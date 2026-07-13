# Routing Design

## Overview

This document explains how traffic flows between the on-premises homelab and AWS.

The hybrid cloud environment uses a dedicated Transit VPC, a FortiGate Network Virtual Appliance (NVA), AWS Transit Gateway, and dynamic routing with eBGP to provide secure connectivity between on-premises resources and AWS workloads.

Rather than relying on static routes, routing information is exchanged dynamically between OPNsense and FortiGate. AWS Transit Gateway then distributes traffic to the appropriate VPC attachments.

---

# Network Overview

| Network | CIDR | Location |
|----------|------|----------|
| Infrastructure | 10.72.15.0/24 | On-Premises |
| Client Network | 192.168.69.0/24 | On-Premises |
| Transit VPC | 10.123.0.0/16 | AWS |
| Main Application VPC | 172.31.0.0/16 | AWS |

---

# High-Level Traffic Flow

```
Client
    │
OPNsense
    │
Route-Based IPsec VPN
    │
FortiGate NVA
    │
Transit Gateway
    │
Application VPC
    │
AdGuard Home
```

Traffic entering AWS always traverses the FortiGate before reaching the Transit Gateway.

Likewise, return traffic follows the reverse path.

---

# Route Advertisement

Routing information is exchanged dynamically through eBGP.

## Advertised from OPNsense

| Prefix | Description |
|---------|-------------|
| 10.72.15.0/24 | Infrastructure Network |
| 192.168.69.0/24 | Client Network |

---

## Advertised from FortiGate

| Prefix | Description |
|---------|-------------|
| 10.123.0.0/16 | Transit VPC |
| 172.31.0.0/16 | Main Application VPC |

This eliminates the need to manually maintain static routes whenever additional prefixes are introduced.

---

# Packet Walkthrough

The following example demonstrates a DNS request from a home client to the AdGuard server hosted in AWS.

---

## Step 1 – Client

The client generates a DNS request.

| Source | Destination |
|----------|-------------|
| 192.168.69.100 | 172.31.10.53 |

The destination belongs to an AWS network.

---

## Step 2 – OPNsense

OPNsense performs a route lookup.

Because the AWS prefixes are learned via eBGP, the packet is forwarded through the Route-Based IPsec VPN interface.

### Screenshot

![OPNSENSERT](/screenshots/OPNSENSE_ROUTE_TABLE.png)

---

## Step 3 – Route-Based IPsec Tunnel

The packet is encrypted and transmitted across the VPN tunnel.

Unlike a policy-based VPN, routing determines which traffic enters the tunnel.

No crypto ACLs are required.

---

## Step 4 – FortiGate

The FortiGate receives the encrypted packet, decrypts it, and performs another routing lookup.

Traffic destined for the Main Application VPC is forwarded to the Transit Gateway attachment.

### Screenshot

![FGTBGPROUTE](/screenshots/FGT_BGP_Routing_Table.png)

---

## Step 5 – AWS Transit Gateway

Transit Gateway receives the packet from the Transit VPC attachment.

Using its route table, TGW determines the correct attachment for the destination prefix.

### Screenshot

![TGWRT](/screenshots/TransitGW_Routes.png)

---

## Step 6 – Main Application VPC

The Main VPC route table contains a local route for the destination subnet.

The packet is delivered to the EC2 instance hosting AdGuard Home.

### Screenshot

![MAINVPCRT](/screenshots/Main_VPC_Route_Table.png)

---

## Step 7 – Return Traffic

The response follows the reverse path.

```
AdGuard

↓

Main VPC

↓

Transit Gateway

↓

FortiGate

↓

IPsec VPN

↓

OPNsense

↓

Client
```

Bidirectional routing is maintained through eBGP advertisements.

---

# Route Tables

## OPNsense

| Destination | Next Hop |
|-------------|----------|
| 172.31.0.0/16 | VPN Tunnel |
| 10.123.0.0/16 | VPN Tunnel |

---

## FortiGate

| Destination | Next Hop |
|-------------|----------|
| 10.72.15.0/24 | VPN Tunnel |
| 192.168.69.0/24 | VPN Tunnel |
| 172.31.0.0/16 | Transit Gateway |

### Screenshot


---

## Transit Gateway

| Destination | Attachment |
|-------------|------------|
| Transit VPC | Transit Attachment |
| Main VPC | Main VPC Attachment |


---

## Main VPC

| Destination | Next Hop |
|-------------|----------|
| 10.72.15.0/24 | Transit Gateway |
| 192.168.69.0/24 | Transit Gateway |
| 172.31.0.0/16 | Local |

### Screenshot

---

# Validation

Hybrid connectivity was verified using the following tests:

- Successful ICMP communication
- DNS resolution through AdGuard Home
- Established BGP neighbor relationship
- Expected routes present on both firewalls
- Correct Transit Gateway route propagation

### Ping Test

![TraffictoAWS](screenshots/Traffic_to_aws.png)
---

### Tunnel Status

![FGTIPSEC](/screenshots/Foritate_IPSEC.png)

---

# Design Considerations

This project intentionally uses dynamic routing and a Transit Gateway architecture rather than a direct VPN into the application VPC.

The routing design provides:

- Centralized routing
- Separation of networking and workloads
- Simplified expansion
- Reduced administrative overhead
- Enterprise-inspired architecture

As additional application VPCs are introduced, no changes are required to the VPN topology. New VPCs can simply be attached to the Transit Gateway and advertised through the existing routing infrastructure.

---

# Lessons Learned

This implementation provided hands-on experience with:

- Route-Based IPsec VPNs
- Dynamic routing using eBGP
- AWS Transit Gateway route propagation
- Hybrid cloud packet forwarding
- AWS VPC route tables
- Enterprise hub-and-spoke networking
- Network segmentation
