---
title: "Integrating Spot Instances at Scale"
date: 2025-07-17T21:19:54+05:30
description: "How we safely integrated Amazon EC2 Spot Instances into our ECS clusters to optimize cloud compute costs."
tags: ["AWS", "ECS", "Spot Instances", "Cloud Cost Optimization"]
draft: false
---

# Optimizing ECS Costs with EC2 Spot Instances

As part of my current role I was tasked with integrating **Amazon EC2 Spot Instances** within our **Amazon ECS clusters**. While we had already benefited from **Reserved Instances** and **Savings Plans**, Spot Instances presented an opportunity for even greater savings, often up to **90% compared to On-Demand pricing**.

However, Spot Instances come with trade-offs. Their availability can fluctuate, and they can be terminated with just a two-minute warning. Safely integrating them into a production ECS environment requires thoughtful architecture, workload-level filtering, and resilient automation.

---

## The Challenge: Balancing Cost and Reliability

Our ECS environment initially ran entirely on **On-Demand EC2 instances**, managed through **Auto Scaling Groups (ASGs)** connected to **ECS Capacity Providers**. This setup offered high reliability, but also left significant savings on the table.

Our goal was to introduce **Spot capacity** into our clusters and strategically place eligible workloads on it. The primary challenge? **Mitigating service degradation** caused by the unpredictable nature of Spot infrastructure.

---

## Understanding Spot Limitations

Spot Instances offer impressive discounts, but with a few important caveats:

- **Ephemeral nature**: They can be terminated at any time with a 2-minute warning.
- **Variable capacity**: Availability fluctuates based on AWS’s excess capacity in each Availability Zone.
- **Scaling constraints**: Auto Scaling might stall if your preferred instance types are in short supply.

To safely adopt Spot, workloads must be **resilient, stateless, and capable of shutting down quickly**. This led us to define a clear, technical set of criteria for Spot eligibility.

---

## Workload Eligibility Criteria

To ensure safe and effective use of Spot Instances, we established the following conditions for workload eligibility:

### Fast Shutdown Capability

- **Requirement**: Containers must define a `stopTimeout` less than 120 seconds.
- **Why it matters**: Ensures graceful shutdown within AWS's two-minute termination window.

### Resilience via Redundancy

- **Requirement**: The ECS service should have a `desiredCount` of 2 or more.
- **Why it matters**: Guarantees availability by avoiding single-point failures.

### Statelessness

- **Requirement**: No usage of persistent volumes or local storage.
- **Why it matters**: Spot terminations risk data loss if workloads aren’t stateless.

### Load Balancer Deregistration < 120s

- **Requirement**: Target deregistration delay should be less than 120 seconds.
- **Why it matters**: Ensures that tasks are removed from load balancers before forced shutdown, preventing traffic loss.

These criteria allowed us to systematically identify workloads that can tolerate interruptions without user impact.

---

## Spot Placement Score (SPS) & Predictive Confidence

To make smarter placement decisions, we introduced a **Spot Placement Score (SPS)** metric. This metric reflects the likelihood of acquiring Spot Instances for a specific configuration and is based on:

- The **instance families and sizes** defined in our Spot ASGs
- The **maximum capacity** targeted by those ASGs
- **Regional and zone-level availability** of excess Spot capacity

We derive this score using [**AWS’s Spot Placement Score API**](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-placement-score.html), which provides insight into AWS’s internal view of instance availability. We calculate a custom SPS for each cluster configuration and **emit it as a metric** to track Spot capacity health over time.

- **High SPS**: Spot capacity is readily available; interruptions are unlikely.
- **Low SPS**: Increased risk of interruption; guides us to scale back Spot usage.

---

## Dynamic Adjustments with Automation

We built automation around this score to dynamically adjust Spot usage across services and clusters. Our system allows us to:

- Proactively reduce Spot usage when SPS trends downward.
- Disable Spot entirely during critical periods (e.g., product launches).
- Quickly re-enable Spot when the environment stabilizes.

This is achieved via a global **“Spot percentage” setting in ECS Capacity Providers**, which controls the ratio of tasks placed on Spot vs. On-Demand. Adjusting this percentage allows us to dial in or out of Spot with minimal effort.

---

## Architecture: Automating the Rollout

To scale this across environments, we built a modular system composed of five key components:

### Orchestrator (Step Function)

Coordinates the entire workflow:

- Identifies eligible services
- Updates capacity provider strategies
- Validates post-deployment health
- Retries updates for unstable services

### Determiner (Lambda)

Applies the eligibility criteria and detects drift between current and desired configurations.

### Updater (Lambda)

Applies the updated strategy only if needed, ensuring idempotent and safe deployments.

### Stability Checker (Lambda)

Validates that each service has stabilized (all desired tasks are running, none pending).

### Scheduler (EventBridge)

Triggers the orchestrator on a configurable cadence to maintain up-to-date configurations.

We leverage **ECS capacity provider strategies** to control Spot usage per service. These strategies define how tasks are distributed across capacity providers by assigning weights (e.g., 70% On-Demand, 30% Spot).

---

## Per-Service Granularity

Rather than enforce a blanket rule across all services, we opted for **per-service strategy definitions**. This enables us to:

- Tailor strategies to each service's reliability and behavior
- Perform safe, gradual rollouts
- Fine-tune Spot usage at the workload level

This granularity gives teams confidence in adoption **without compromising uptime**.

---

## Real-World Results

Even with a conservative rollout, capping usage at 10-15% we achieved **meaningful compute cost savings** without impacting availability or performance.

As our confidence and tooling mature, we plan to increase this percentage, expanding Spot adoption.

---

## Key Takeaways

- **Spot is powerful, but risky**: The savings are real, but only if workloads are interruption-tolerant and stateless.
- **Objective rules matter**: Hard constraints like shutdown time and statelessness help scale safely and avoid human judgment errors.
- **Automation is essential**: Managing Spot at scale across many services is only feasible with automated tooling.
- **Stability checks are non-negotiable**: Post-deployment validation ensures confidence in rollout safety.
- **Placement scoring is a game-changer**: SPS provides predictive insight that helps us make informed Spot allocation decisions.

---

If you're managing a large ECS footprint, **Spot Instances can be a major lever** for compute cost optimization, but only with the right guardrails in place. With objective eligibility criteria, an automation framework, and real-time placement scoring, Spot can become a **safe, reliable, and cost-effective** part of your ECS strategy.
