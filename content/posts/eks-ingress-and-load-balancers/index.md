---
title: "EKS Networking Refresher, Part 1: Ingress and Load Balancers"
date: 2026-10-23T11:30:00+05:30
description: "How ingress has evolved, how the AWS Load Balancer Controller picks subnets, how IngressGroups share ALBs, and how a load balancer actually reaches your pods."
tags: ["AWS", "EKS", "Kubernetes", "Networking"]
series: ["EKS Networking Refresher"]
draft: false
---

# EKS Networking Refresher, Part 1: Ingress and Load Balancers

If you last worked on EKS a few years ago, the picture in your head is probably "a `LoadBalancer` Service in front of an nginx ingress controller". That still works, but most of it has moved on. Ingress is frozen, the most popular ingress controller has been retired, and load balancers can talk to pods directly.

This is the first of two posts. This one covers how traffic gets into a cluster, and how the AWS Load Balancer Controller decides where to put a load balancer. [Part 2](/posts/eks-pod-networking-vpc-cni/) covers how pods get their IPs, and how to stop them draining your VPC.

---

## Ingress: How the Model Evolved

### The classic model

```
Internet -> L4 cloud LB (Service type=LoadBalancer)
         -> ingress controller pods (nginx / Kong / Traefik / Envoy)
         -> backend pods
```

One cloud load balancer sits in front of a proxy that runs inside the cluster. `Ingress` objects are just configuration for that proxy, and all the routing logic (host, path, rewrites, auth, rate limits) lives in the proxy pods.

### What changed

**Ingress is frozen, Gateway API is the successor.** The `Ingress` API is feature-frozen. It only understands HTTP host and path matching, so everything else (rewrites, timeouts, auth, canaries) ended up in vendor-specific annotations. Gateway API, which went GA in October 2023, replaces that with role-oriented resources:

| Resource | Purpose | Typical owner |
|---|---|---|
| `GatewayClass` | Which implementation to use | Infrastructure provider |
| `Gateway` | Listeners, ports, TLS | Platform team |
| `HTTPRoute`, `GRPCRoute` | Routing rules | App teams |

It has typed fields for header matching, weighted splits, redirects and cross-namespace attachment, which replaces most of the annotation soup. One caveat: GA covered `Gateway`, `GatewayClass` and `HTTPRoute`. Other route types, such as `TCPRoute`, are still in the experimental channel, so check the status of what you need before you commit to it.

**ingress-nginx has been retired.** The community ingress-nginx project was announced for retirement in November 2025, with best-effort maintenance continuing only until March 2026. After that there are no further releases, no bug fixes and no security patches. If you still run it, treat it as unmaintained software and plan the move.

**There are two ways to route.** The real decision is whether routing happens in a proxy you run, or in the cloud load balancer:

| | In-cluster proxy | Cloud-native LB controller |
|---|---|---|
| Data path | L4 LB -> proxy pods -> backend pods | L7 cloud LB -> backend pods directly |
| Where routing lives | In the proxy | In the cloud LB (listener rules, target groups) |
| Examples | Kong, APISIX, Traefik, Envoy Gateway, Istio | AWS Load Balancer Controller, GKE Ingress |
| Pros | Portable, rich features, fast config changes | No proxy to run, native WAF/ACM/auth, managed scaling |
| Cons | You operate and scale the proxy | Cloud-specific annotations, rule and target limits, slower reconcile |

A common production pattern uses both. A cloud ALB or NLB handles TLS, WAF and public entry, and sits in front of an in-cluster API gateway (Kong, APISIX, Envoy) that handles auth and rate limiting. Each layer does what it is good at.

Some supporting pieces have become standard along the way:

- **cert-manager** for in-cluster certificates (or ACM when TLS ends at an AWS load balancer).
- **external-dns** to create DNS records from ingress and route hostnames.
- **Shared gateways** instead of one load balancer per app, to cut cost.
- **Sidecar-less service mesh** (Istio ambient, Cilium) for east-west traffic.

---

## The AWS Load Balancer Controller

The AWS Load Balancer Controller (LBC) turns Kubernetes objects into real AWS resources:

| Kubernetes object | AWS resource |
|---|---|
| `Ingress` (class `alb`) | Application Load Balancer, listeners, rules, target groups |
| `Service` type `LoadBalancer` (with the right annotations) | Network Load Balancer |

A typical install is the `aws-load-balancer-controller` Helm chart in `kube-system`, with an IAM role bound to its service account (IRSA or EKS Pod Identity). The flags that matter are `--cluster-name`, `--ingress-class` and `--aws-vpc-id`, and the chart creates the `alb` `IngressClass` for you.

The important difference from an in-cluster proxy is that **there is no proxy in the data path**. Routing happens inside the ALB itself.

### Example: a public ALB ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: public-gateway
  namespace: gateway
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/subnets: subnet-pub-a, subnet-pub-b, subnet-pub-c
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:<region>:<account-id>:certificate/<cert-id>
    alb.ingress.kubernetes.io/backend-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-path: /status
    alb.ingress.kubernetes.io/healthcheck-port: "9080"
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 80
```

Reading it top to bottom: internet to a public ALB across three AZs, HTTPS ends at the ALB with an ACM certificate, plain HTTP goes on to the gateway pod IPs, and health checks hit `/status`. A single catch-all rule sends everything to the gateway, which does the real routing.

### A failure mode worth knowing

An `Ingress` can point at a Service that does not exist, for example because the gateway Deployment was scaled to zero. The ALB is still created, but its target group has no healthy targets and every request returns a 503. When debugging, check target health in the ALB target group, not just whether the Ingress has an address.

---

## Subnet Discovery for Load Balancers

An ALB or NLB needs network interfaces in VPC subnets. Kubernetes knows nothing about subnets, so the controller has to decide which ones to use. You can tell it explicitly, or you can let it discover them from tags. Discovery is worth the effort because it keeps subnet IDs out of every app team's manifests: the platform team tags subnets once, and internet-facing load balancers land in public subnets while internal ones land in private subnets, spread across AZs.

### The tags

| Tag | Meaning |
|---|---|
| `kubernetes.io/role/elb = 1` | Candidate subnet for **internet-facing** load balancers |
| `kubernetes.io/role/internal-elb = 1` | Candidate subnet for **internal** load balancers |
| `kubernetes.io/cluster/<cluster-name> = shared` or `owned` | Optional. Scopes a subnet to one cluster, and is needed if subnets are shared between clusters |

The role tags work with a value of `1` or an empty value.

### How the controller chooses

1. If the Ingress has `alb.ingress.kubernetes.io/subnets` (or a Service has `service.beta.kubernetes.io/aws-load-balancer-subnets`), those subnets are used and nothing is discovered.
2. Otherwise the controller lists subnets in the cluster's VPC and keeps the ones with the role tag that matches the load balancer's scheme. It then drops subnets that carry a cluster tag for a different cluster, and subnets with fewer than 8 free IP addresses.
3. If several subnets in the same AZ are still eligible, the controller picks one per AZ. A subnet tagged for the current cluster wins, and any remaining tie is broken by the lowest subnet ID, compared as a string.

That last rule is deterministic, but it is not something you want to depend on. It is easy to end up with a load balancer in a subnet you did not intend, purely because of how an ID sorts.

### Common gotchas

- **Public subnets missing `elb=1`.** Internet-facing load balancers fail to discover anything, and teams work around it with hardcoded subnet annotations that nobody revisits.
- **Two tagged subnets in one AZ**, such as a large one and a small one. Which one the load balancer lands on depends on the tie-break above, and a small subnet can run out of IPs. An ALB needs free addresses in each of its subnets and uses more of them as it scales up.
- **Hardcoded subnet IDs** go stale when infrastructure is rebuilt.

### What I would do

1. Tag public subnets `elb=1` and private subnets `internal-elb=1` through infrastructure as code, and rely on discovery.
2. Tag exactly one subnet per AZ per tier, so the tie-break never matters.
3. Give load balancer subnets room. A /24 or larger is a reasonable floor.
4. Use the cluster tag when subnets are shared across clusters.
5. Keep explicit subnet annotations for real exceptions only.
6. Consider `IngressClassParams`, which can set `scheme` and `subnets` once at the IngressClass level instead of on every Ingress.

---

## Ingress Groups: One ALB or Many

By default, **every `Ingress` creates its own ALB**. That gets expensive and sprawls quickly, because each ALB has its own hourly cost, its own IPs in your subnets and its own security group.

### IngressGroup

Annotate Ingresses with the same group name and they merge into **one ALB**. Each Ingress contributes its own host and path rules.

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/group.name: shared-internal
    alb.ingress.kubernetes.io/group.order: "100"   # rule priority within the group
    alb.ingress.kubernetes.io/scheme: internal
```

This is the closest equivalent to the old "one load balancer for everything" setup, except the routing now lives in the ALB instead of a proxy.

### Trade-offs

- All Ingresses in a group share one set of listeners and one certificate set.
- The scheme (internal or internet-facing) is per ALB, so you cannot mix schemes in a group.
- A broken Ingress can block reconciliation for the whole group.
- ALBs have a quota on rules per listener (100 by default, and adjustable). A group with dozens of Ingresses and many rules can get close, so check the current quota for your account.
- Fewer ALBs means lower cost and fewer IPs consumed in load balancer subnets.

### A sensible layout

| ALB | Scheme | Contents |
|---|---|---|
| Shared internal group | internal | The bulk of service Ingresses |
| One or two small groups | internal | Anything needing isolation (different certificate, different security group, rule limit pressure) |
| Public ALB | internet-facing | Only what truly must be public, ideally just the API gateway |

Keeping public and internal ALBs separate is normal and desirable.

---

## How a Load Balancer Reaches Pods

### The Service is for discovery, not the data path

An `Ingress` names a Service, but that Service is mostly how the controller discovers backends: the selector, the ports and the EndpointSlices. In almost every implementation, **the ClusterIP is not in the data path**.

| Implementation | What receives the traffic |
|---|---|
| In-cluster proxies (Kong, Traefik, Envoy) | They read EndpointSlices and send **directly to pod IPs**, bypassing kube-proxy |
| AWS LBC, `target-type: ip` | The ALB registers **pod IPs as targets** and talks to pods directly |
| AWS LBC, `target-type: instance` | The ALB targets **node IPs on a NodePort**, and kube-proxy forwards to a pod |

### Whether it works depends on the CNI

| CNI | Pod IP is | Direct to pod possible |
|---|---|---|
| AWS VPC CNI | A real VPC IP (a secondary IP on an ENI) | Yes |
| GKE VPC-native, Azure CNI | A real VPC or VNet IP | Yes |
| Calico or similar with an IPIP/VXLAN overlay | A virtual address, encapsulated between nodes | No, the cloud network has no route to it |
| Calico with BGP and no encapsulation | Routable if BGP peers with the fabric | Yes, in on-prem or specially built setups |

With an overlay CNI the load balancer can only target node IPs, so you are stuck with `instance` mode: LB, then NodePort, then kube-proxy, then pod. On EKS with the VPC CNI you can use `ip` mode and skip those extra hops. You can also run Calico on EKS in policy-only mode while the VPC CNI still handles addressing. If Calico is the overlay CNI instead, `ip` mode will not work.

### Operational consequences of direct-to-pod

- **Rollouts.** A pod can be deregistered from the target group while the load balancer is still sending it traffic, which causes brief 5xx errors. Mitigate with a `preStop` sleep, an adequate `terminationGracePeriodSeconds`, and pod readiness gates, where the LBC holds a pod "ready" until its target is healthy.
- **Security groups.** Pods must accept traffic from the load balancer's security group. The LBC normally manages its own rules for this, so look at what it creates before adding manual ones.
- **IP consumption.** With the VPC CNI, pods use real VPC IPs, so every pod behind an `ip` mode target spends a VPC address. [Part 2](/posts/eks-pod-networking-vpc-cni/) is about how to keep that from becoming a problem.

---

## Best Practices Checklist

**Load balancing**

- Use Gateway API for new work where the controller supports it, and treat `Ingress` as legacy.
- Put a load balancer in front for TLS and WAF, and an in-cluster gateway behind it if you need API features.
- Use IngressGroups to share ALBs, with a separate public ALB.
- Use `target-type: ip` where pod IPs are VPC-routable.
- Add `preStop` sleeps, a sensible termination grace period and pod readiness gates to avoid 5xx errors during rollouts.
- Keep external-dns and cert-manager (or ACM) in the picture so DNS and certificates are not manual.

**Subnets**

- `elb=1` on public subnets and `internal-elb=1` on private subnets, exactly one per AZ per tier.
- Size load balancer subnets at /24 or larger.
- Avoid hardcoded subnet IDs except for deliberate exceptions.

---

## Read-Only Commands for Inspecting a Cluster

These commands only read state, so they are safe to run against a live cluster.

```bash
# Which load balancer controller and ingress classes exist
helm list -A | grep -i -E 'load-balancer|ingress|gateway'
kubectl get ingressclass

# Ingresses, their scheme and groups
kubectl get ingress -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{.metadata.annotations.alb\.ingress\.kubernetes\.io/scheme}{"\t"}{.metadata.annotations.alb\.ingress\.kubernetes\.io/group\.name}{"\n"}{end}'

# Does the Service behind an Ingress actually exist
kubectl get svc <name> -n <namespace>

# Subnets and their discovery tags
aws ec2 describe-subnets --filters Name=vpc-id,Values=<vpc-id> \
  --query 'Subnets[].{id:SubnetId,az:AvailabilityZone,cidr:CidrBlock,elb:Tags[?Key==`kubernetes.io/role/elb`]|[0].Value,internal:Tags[?Key==`kubernetes.io/role/internal-elb`]|[0].Value}'

# Target health for an ALB target group
aws elbv2 describe-target-health --target-group-arn <target-group-arn>
```

---

## Key Takeaways

- **Pick where routing lives.** Either in a proxy you run, or in the cloud load balancer. Many production setups use both.
- **Tag subnets once, and tag them cleanly.** One subnet per AZ per tier makes discovery boring, which is what you want.
- **Share ALBs with IngressGroups**, and keep the public one separate.
- **`target-type: ip` removes hops**, but it makes rollouts and security groups your problem.

Next up: [Part 2, Pod Networking with the VPC CNI](/posts/eks-pod-networking-vpc-cni/).
