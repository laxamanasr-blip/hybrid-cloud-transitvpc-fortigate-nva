# AWS Transit Gateway

## Overview

AWS Transit Gateway (TGW) serves as the central routing hub of this hybrid cloud environment.

Instead of connecting the on-premises network directly to the application VPC, all connectivity is centralized through the Transit Gateway. This enables a scalable hub-and-spoke architecture where multiple VPCs can communicate with on-premises resources through a single hybrid connectivity point.

In this project, the Transit Gateway connects:

- Transit VPC
- Main Application VPC

The Transit VPC hosts the FortiGate Network Virtual Appliance (NVA), which terminates the Route-Based IPsec VPN from the on-premises OPNsense firewall.

---

# Why AWS Transit Gateway?

A Virtual Private Gateway (VGW) would have been sufficient for a single VPC.

However, the objective of this project was to build an architecture that resembles how enterprise environments are commonly designed.

Using Transit Gateway provides several advantages:

- Centralized routing
- Simplified multi-VPC connectivity
- Easy future expansion
- Reduced VPN complexity
- Better separation between networking and application workloads

This allows new application VPCs to be attached without creating additional VPN tunnels.

---

# Architecture

```
                   Home Lab
                       │
             Route-Based IPsec VPN
                       │
                FortiGate NVA
                 (Transit VPC)
                       │
          Transit Gateway Attachment
                       │
         AWS Transit Gateway (Hub)
                       │
      ┌────────────────┴────────────────┐
      │                                 │
 Transit VPC                     Main VPC
```

---

# Transit Gateway Attachments

This project uses two VPC attachments.

| Attachment | Purpose |
|------------|---------|
| Transit VPC | Connects the FortiGate NVA to the Transit Gateway |
| Main VPC | Provides connectivity to application workloads |

Future application VPCs can be attached without modifying the VPN architecture.

---

# Screenshot

![Transit Gateway Attachments](/screenshots/TransitGW_Attachments.png)

**Figure 1:** AWS Transit Gateway with VPC attachments connecting the Transit VPC and the Main Application VPC.

---

# Transit Gateway Route Table

The Transit Gateway route table determines how traffic is forwarded between connected attachments.

Routes include:

| Destination | Attachment |
|------------|------------|
| 10.123.0.0/16 | Transit VPC |
| 172.31.0.0/16 | Main VPC |

Traffic destined for the application VPC is forwarded to the appropriate VPC attachment, while traffic destined for the Transit VPC is sent toward the FortiGate appliance.

---

# Screenshot

![Transit Gateway Route Table](/screenshots/TransitGW_Routes.png)

**Figure 2:** Transit Gateway route table showing routes to connected VPC attachments.

---

# Traffic Flow

Example: Home client accessing AdGuard Home

```
Client
192.168.69.100
      │
      ▼
OPNsense
      │
      ▼
Route-Based IPsec VPN
      │
      ▼
FortiGate
      │
      ▼
Transit Gateway
      │
      ▼
Main VPC
      │
      ▼
AdGuard Home
```

Transit Gateway forwards packets solely based on destination prefixes and attachment associations.

---

# Why a Transit Gateway Instead of Direct VPC Routing?

Without Transit Gateway:

```
Home

↓

VPN

↓

Main VPC
```

Problems:

- VPN terminates inside workload VPC
- Difficult to add more VPCs
- Security services mixed with applications
- Additional VPNs may be required

---

With Transit Gateway:

```
Home

↓

VPN

↓

Transit VPC

↓

Transit Gateway

↓

Application VPCs
```

Benefits:

- Dedicated networking layer
- Central routing hub
- Simplified expansion
- Cleaner network segmentation
- Enterprise-inspired architecture

---

# Future Expansion

One of the main reasons for selecting Transit Gateway is scalability.

Additional workloads can be connected simply by attaching new VPCs.

Example:

```
                    Transit Gateway

          ┌─────────┼─────────┐
          │         │         │
     Shared VPC   Dev VPC   Production VPC
```

No additional VPN tunnels are required.

The FortiGate continues to provide hybrid connectivity for all attached VPCs.

---

# Validation

The Transit Gateway deployment was validated by confirming:

- Transit VPC attachment is available
- Main VPC attachment is available
- Route table contains expected routes
- On-premises clients can access AWS workloads
- Return traffic follows the expected path

---

# Lessons Learned

Implementing AWS Transit Gateway provided practical experience with:

- Hub-and-spoke networking
- Hybrid cloud routing
- VPC attachments
- Transit Gateway route tables
- Network segmentation
- Enterprise hybrid cloud design

While a Virtual Private Gateway would have satisfied the functional requirements of this lab, Transit Gateway provided a scalable foundation that better reflects enterprise AWS networking architectures.
