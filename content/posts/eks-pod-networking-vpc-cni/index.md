---
title: "EKS Networking Refresher, Part 2: Pod Networking with the VPC CNI"
date: 2026-10-23T12:00:00+05:30
description: "How the VPC CNI hands out pod IPs, how a secondary CIDR with custom networking and ENIConfig keeps them from draining your VPC, and what SNAT does to traffic leaving it."
tags: ["AWS", "EKS", "Kubernetes", "Networking"]
series: ["EKS Networking Refresher"]
draft: false
---

# EKS Networking Refresher, Part 2: Pod Networking with the VPC CNI

[Part 1](/posts/eks-ingress-and-load-balancers/) covered how traffic gets into an EKS cluster. This part is about the other half: how pods get their IP addresses, and what happens when those addresses start running out. It covers the secondary IP range, custom networking, `ENIConfig`, security groups and SNAT.

---

## Why Pod IPs Are a Problem on EKS

In Part 1 we saw that an ALB in `target-type: ip` mode registers **pod IPs as targets** and talks to pods directly. That only works because the AWS VPC CNI gives every pod a real VPC IP, taken as a secondary address on an ENI.

That is convenient, but it has a cost. The VPC's primary CIDR is usually part of a corporate-routable range that is shared with peered VPCs, Transit Gateway and on-prem networks. A busy node can hold dozens of pods, so that range drains quickly, and you cannot get it back without renumbering.

---

## VPC CNI and the Secondary IP Range

### The fix: a secondary CIDR just for pods

1. Attach a secondary CIDR to the VPC. The `100.64.0.0/10` block (the CG-NAT range) is popular because it is large and rarely clashes with corporate ranges.
2. Create one subnet per AZ in that CIDR. These are the "pod subnets".
3. Turn on custom networking in the CNI and tell it, per AZ, which pod subnet to use (see the ENIConfig section below).

### Example layout

```
VPC
 |- Primary CIDR   10.0.0.0/16        (routable beyond the VPC)
 |    |- public   subnets   10.0.0.0/24   x 3 AZs   -> internet-facing LBs
 |    |- private  subnets   10.0.16.0/20  x 3 AZs   -> nodes, internal LBs
 |
 |- Secondary CIDR 100.64.0.0/16      (VPC-only, not advertised outside)
      |- pod subnets        100.64.0.0/18 x 3 AZs   -> pod IPs only, no ELB tags
```

Note that the pod subnets carry no `elb` tags, so the [load balancer subnet discovery from Part 1](/posts/eks-ingress-and-load-balancers/#subnet-discovery-for-load-balancers) never picks them up.

### How it works at runtime

- The node's **primary ENI** stays in the normal private subnet and provides the node IP.
- With custom networking on, the CNI does **not** use the primary ENI for pod IPs. It attaches **secondary ENIs** in the pod subnet named by the node's `ENIConfig`.
- With prefix delegation enabled, each ENI slot gets a `/28` prefix (16 IPs) from that subnet and pods draw from those prefixes. This greatly increases pods per node.
- Pod traffic **inside the VPC** (pod to pod, ALB to pod, pod to a database in the VPC) keeps the pod IP as its source.
- Pod traffic **leaving the VPC** (internet, peering, Transit Gateway, on-prem) is source-NATed to the node's primary IP. More on that below.

### Key CNI settings

| Setting | Meaning |
|---|---|
| `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true` | Enable custom networking: pods use the `ENIConfig` subnet, not the node's |
| `ENI_CONFIG_LABEL_DEF=topology.kubernetes.io/zone` | Which node label selects the `ENIConfig` |
| `ENABLE_PREFIX_DELEGATION=true` | Assign `/28` prefixes instead of individual IPs |
| `WARM_PREFIX_TARGET`, `WARM_ENI_TARGET`, `WARM_IP_TARGET` | How many addresses to keep warm |
| `AWS_VPC_K8S_CNI_EXTERNALSNAT` | `false` (the default) means the CNI does the SNAT. `true` means you rely on an external NAT |

### What you need to create

1. The secondary CIDR and one pod subnet per AZ, usually in Terraform. Pod subnets only need the local route, because off-VPC traffic is SNAT'd to the node IP.
2. One `ENIConfig` per AZ.
3. The CNI settings above, applied through the EKS managed add-on configuration or by editing the `aws-node` DaemonSet.
4. **Replacement or new nodes.** Existing nodes keep their old ENIs, so roll your nodes after enabling this.

### Trade-offs and alternatives

- Anything outside the VPC cannot address pods directly. This is usually fine because traffic goes through a load balancer or a service anyway.
- Pods on a node no longer use the primary ENI, so adjust `max-pods` if you are not using prefix delegation.
- A missing or mismatched `ENIConfig` leaves pods stuck in `ContainerCreating`.
- There are other ways to solve the same problem: prefix delegation on its own (fewer wasted IPs per node), or an IPv6 cluster, where address exhaustion goes away.

---

## ENIConfig

`ENIConfig` tells the VPC CNI which **subnet and security groups** to use for the extra ENIs that hold pod IPs. Without custom networking, pod ENIs go in the same subnet and use the same security groups as the node's primary ENI. With it on, the CNI has to be told where else to put them, and `ENIConfig` is how. The environment variable only says "do not use the node's subnet for pods". The `ENIConfig` supplies the answer.

```yaml
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-west-2a                 # must equal the node's zone label value
spec:
  subnet: subnet-pod-a             # pod subnet in this AZ
  securityGroups:
    - sg-cluster-primary
    - sg-cluster
    - sg-node
```

It has just two fields, `subnet` and `securityGroups`.

### How a node finds its ENIConfig

When a node joins, the CNI reads the label named by `ENI_CONFIG_LABEL_DEF` (usually the zone label) and looks up the `ENIConfig` whose **name equals that label's value**. That is why the objects are named after zones. You can also annotate a node with `k8s.amazonaws.com/eniConfig: <name>` to pick one explicitly. You need one `ENIConfig` per AZ because a subnet lives in exactly one AZ.

### Behaviours to remember

- A node with no matching `ENIConfig` cannot get pod IPs.
- Changes apply only to ENIs created afterwards, so replace nodes to pick them up.
- If `securityGroups` is omitted, pod ENIs fall back to the security groups on the node's primary ENI.
- The `last-applied-configuration` annotation can drift from the live object if someone edits it outside your normal apply path. Find the real source of truth before you change anything.

### Which security groups to use

None of the groups above are pod-specific. In a typical Terraform-built cluster there are three, and the ENIConfigs reuse them on pod ENIs:

| Security group | Created by |
|---|---|
| EKS primary cluster SG (`eks-cluster-sg-<cluster>-<number>`) | **EKS itself**, automatically at cluster creation |
| Cluster SG (`<cluster>-cluster-<suffix>`) | Terraform, if you use the community `terraform-aws-modules/eks` module |
| Node shared SG (`<cluster>-node-<suffix>`) | Terraform, from the same module |

Reusing them lets pods behave like nodes for control-plane callbacks (webhooks, kubelet) and node-to-node traffic. Two more things to keep in mind:

- The load balancer's security group must be allowed to reach pods. The LBC normally manages its own rules for this, so look at what it creates before adding manual ones.
- Database and cache security groups must allow the **pod subnet CIDR or the node/pod security group**, because within the VPC the source is the pod IP, not the node IP.

Pods using `ENABLE_POD_ENI` (security groups for pods) are a separate feature that gives individual pods their own groups.

---

## SNAT

**SNAT (Source Network Address Translation)** is when a device rewrites the **source IP** of outgoing packets to a different IP and remembers the mapping so replies can be translated back. The destination is untouched. Your home router does this: your laptop is `192.168.1.5`, but websites see the router's public IP. When it also translates ports, it is often called masquerade or PAT.

### In an EKS cluster with custom networking

A pod IP like `100.64.x.x` only exists inside the VPC.

- **Without SNAT**, a pod calling something over a peering link, Transit Gateway or the internet would send packets with an unroutable source, and replies would never arrive.
- **With SNAT** (`EXTERNALSNAT=false`), iptables rules on the node rewrite the source to the node's primary IP, which is routable. The reply comes back to the node, which maps it to the pod.

The AWS documentation is explicit about where the line is. The CNI translates the pod IP only when the destination is not within a CIDR block associated with the VPC. That includes secondary CIDRs, so a pod talking to a database in the same VPC is seen with its real pod IP. One exception to know about is that pods with `hostNetwork: true` already use the node's IP, so nothing is translated for them.

### Consequences

- Systems outside the VPC see the **node IP**, not the pod IP. Their allowlists and firewalls must permit node IPs, or the NAT gateway IP for internet traffic.
- You lose per-pod identity at the remote end.
- `EXTERNALSNAT=true` skips node-level SNAT and relies on an external NAT such as a NAT gateway. It only works if pod IPs are routable there, and nodes must be in private subnets with a route to that NAT gateway.

For completeness, DNAT is the opposite: it rewrites the **destination**. kube-proxy does this when it turns a Service ClusterIP into a pod IP.

---

## Best Practices Checklist

- Use a secondary CIDR (`100.64.0.0/10`) for pods if the primary range is scarce.
- Enable prefix delegation for pod density.
- Keep `ENIConfig` objects and CNI settings in code, and know where the source of truth lives.
- Replace nodes after changing custom networking.
- Allow pod subnet CIDRs in downstream security groups.
- Remember that anything outside the VPC sees the node IP, and update allowlists to match.

---

## Read-Only Commands for Inspecting a Cluster

These commands only read state, so they are safe to run against a live cluster.

```bash
# VPC CIDRs, including secondary ones
aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[0].CidrBlockAssociationSet[].CidrBlock'

# VPC CNI custom networking
kubectl get eniconfig -o yaml
kubectl get ds aws-node -n kube-system -o jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}'
aws eks describe-addon --cluster-name <cluster> --addon-name vpc-cni --query 'addon.configurationValues'

# Security groups referenced by an ENIConfig
aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[].{id:GroupId,name:GroupName,desc:Description}'
```

---

## Key Takeaways

- **Pods on EKS use real VPC IPs**, which is what makes direct-to-pod load balancing possible, and also what drains your address space.
- **Give pods their own address range** with a secondary CIDR, custom networking and one `ENIConfig` per AZ, and roll your nodes afterwards.
- **Use prefix delegation** to raise pod density per node.
- **Traffic leaving the VPC shows up as the node IP**, so allowlists need node IPs, not pod IPs. Traffic inside the VPC keeps the pod IP.
