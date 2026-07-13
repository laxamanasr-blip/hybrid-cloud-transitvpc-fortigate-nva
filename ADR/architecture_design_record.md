# Architecture Decision Record (ADR)

> This document captures the key architectural decisions made throughout this project, including the alternatives considered, the rationale behind each decision, and the associated trade-offs. The goal was not simply to build a functional hybrid cloud environment, but to design one that reflects common enterprise networking patterns while providing opportunities to explore AWS networking, hybrid connectivity, and cloud security concepts.

---

# Project Goal

The initial requirement was straightforward:

> Host an AdGuard Home instance in AWS to serve as a secondary DNS server for my on-premises homelab.

The simplest solution would have been:

```
Home Lab
    │
AWS Site-to-Site VPN (VGW)
    │
Application VPC
    │
AdGuard
```

While this architecture would satisfy the functional requirement, it offers limited scalability and does not reflect how many enterprise AWS environments are designed.

Instead, this project prioritized **architecture over simplicity**.

---

# Decision 1 — Use a Transit VPC Instead of Connecting Directly to the Application VPC

## Decision

Deploy a dedicated Transit VPC to host network infrastructure instead of terminating hybrid connectivity directly within the Application VPC.

## Alternatives Considered

### Option A — VPN directly into the Application VPC

**Pros**

- Simple
- Lower cost
- Fewer AWS resources
- Faster deployment

**Cons**

- VPN tightly coupled with workloads
- Difficult to expand
- Poor separation of concerns
- Networking and applications share the same VPC

---

### Option B — Dedicated Transit VPC (**Chosen**)

**Pros**

- Dedicated networking layer
- Better separation of infrastructure and workloads
- Easier onboarding of future VPCs
- More closely resembles enterprise architectures

**Cons**

- Additional AWS resources
- Higher cost
- Increased routing complexity

## Why I Chose This

Although the current workload is only an AdGuard DNS server, the objective was to build a reusable network foundation rather than optimize for today's requirements.

---

# Decision 2 — Use AWS Transit Gateway Instead of Virtual Private Gateway

## Decision

Use AWS Transit Gateway as the central routing hub.

## Alternatives Considered

### Virtual Private Gateway (VGW)

Suitable for:

- Single VPC
- Small deployments
- Simple hybrid connectivity

Limitations:

- VPN terminates directly into the workload VPC
- Limited scalability

---

### AWS Transit Gateway (**Chosen**)

Advantages:

- Central routing hub
- Hub-and-spoke topology
- Multiple VPC attachments
- Simplified expansion
- Cleaner routing architecture

## Why I Chose This

Transit Gateway allows future application VPCs to be connected without redesigning the hybrid connectivity.

---

# Decision 3 — Deploy a FortiGate Network Virtual Appliance

## Decision

Terminate the VPN on a FortiGate VM inside the Transit VPC instead of using native AWS VPN termination.

## Alternatives Considered

### Native AWS Site-to-Site VPN

Advantages

- Managed service
- Simpler deployment
- Lower operational overhead

Limitations

- Limited inspection capabilities
- Less control over routing and security
- Reduced exposure to enterprise firewall technologies

---

### FortiGate NVA (**Chosen**)

Advantages

- Enterprise firewall
- Stateful inspection
- Advanced routing
- Familiar enterprise platform
- Future IPS/IDS capabilities

Trade-offs

- Additional management
- Compute cost
- More complex deployment

## Why I Chose This

Many enterprise environments deploy third-party firewalls inside AWS to centralize routing and security services.

This project was designed to gain practical experience with that model.

---

# Decision 4 — Use Route-Based IPsec Instead of Policy-Based IPsec

## Decision

Deploy a Route-Based IPsec VPN using Virtual Tunnel Interfaces (VTIs).

## Why

Advantages:

- Supports dynamic routing
- Easier to maintain
- Simplifies expansion
- No crypto ACL maintenance

This aligns well with Transit Gateway architectures and enterprise firewall deployments.

---

# Decision 5 — Use eBGP Instead of Static Routes

## Decision

Exchange routes dynamically between OPNsense and FortiGate using eBGP.

## Alternatives

### Static Routes

Advantages

- Simple
- Easy for small environments

Disadvantages

- Manual updates
- Poor scalability
- Operational overhead

---

### eBGP (**Chosen**)

Advantages

- Automatic route advertisement
- Dynamic route learning
- Better scalability
- Reflects enterprise networking practices

## Why I Chose This

The objective was to understand how enterprise hybrid cloud environments exchange routes rather than simply establish connectivity.

---

# Decision 6 — Separate Networking from Applications

One of the primary design goals was to avoid coupling networking infrastructure with application workloads.

Final architecture:

```
Home Lab
      │
IPsec VPN
      │
Transit VPC
      │
FortiGate NVA
      │
Transit Gateway
      │
Application VPC
      │
AdGuard Home
```

This separation provides:

- Cleaner architecture
- Better security boundaries
- Easier maintenance
- Improved scalability

---

# Design Principles

Throughout the project, several guiding principles influenced every architectural decision.

### Build for Tomorrow, Not Just Today

The environment currently hosts a single workload.

However, every decision was made assuming additional workloads would eventually be introduced.

---

### Separate Responsibilities

Networking infrastructure and application infrastructure should remain independent whenever possible.

---

### Prefer Dynamic Over Manual

Where practical, routing should adapt automatically rather than rely on manually maintained configuration.

---

### Learn Enterprise Patterns

The goal of this homelab is not simply to deploy AWS resources.

It is to understand the architectural patterns commonly used in enterprise cloud environments.

---

# Lessons Learned

This project provided practical experience with:

- AWS Transit Gateway
- Transit VPC design
- Network Virtual Appliances (NVAs)
- Route-Based IPsec VPN
- eBGP dynamic routing
- AWS route propagation
- Hybrid cloud networking
- Enterprise network segmentation
- Designing for scalability instead of immediate requirements

---

# Final Thoughts

Many of the architectural decisions documented here introduced additional complexity compared to simpler alternatives.

That complexity was intentional.

Rather than optimizing for the smallest or least expensive deployment, this project was designed to explore enterprise networking concepts that are difficult to appreciate through documentation alone.

The resulting architecture may be larger than required for a single DNS server, but it provides a scalable foundation capable of supporting additional workloads, application VPCs, and future security services without significant redesign.

Ultimately, the primary outcome of this project was not the deployment itself—it was the experience gained through evaluating trade-offs, implementing the architecture, troubleshooting issues, and documenting the decisions behind every major component.
