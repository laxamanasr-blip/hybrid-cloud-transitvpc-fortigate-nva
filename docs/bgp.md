# Dynamic Routing with eBGP

## Overview

This hybrid cloud environment uses **External Border Gateway Protocol (eBGP)** to exchange routes between the on-premises OPNsense firewall and the FortiGate Network Virtual Appliance (NVA) deployed in AWS.

Instead of maintaining static routes on both VPN endpoints, routes are dynamically advertised across the Route-Based IPsec VPN (VTI). This simplifies administration and allows the network to scale without manual route updates.

---

# Why eBGP?

Static routing would have worked for this environment since only a few prefixes are exchanged.

However, the goal of this project was to model an enterprise hybrid cloud architecture rather than simply establish connectivity.

Using eBGP provides several advantages:

- Automatic route advertisement
- Easier expansion to additional VPCs
- Reduced administrative overhead
- Faster adaptation to routing changes
- Design consistency with enterprise hybrid networks

As additional VPCs are connected through AWS Transit Gateway, the routing architecture can evolve without requiring extensive changes to the VPN configuration.

---

# BGP Topology

```
          Home Lab
        OPNsense Firewall
          ASN 65002
               │
      Route-Based IPsec VPN
               │
        FortiGate NVA
          ASN 65001
               │
      AWS Transit Gateway
               │
         Application VPCs
```

---

# BGP Peering

| Device | ASN |
|----------|------|
| FortiGate | 65001 |
| OPNsense | 65002 |

---

# Advertised Routes

## OPNsense → AWS

| Prefix | Description |
|---------|-------------|
| 10.72.15.0/24 | Infrastructure Network |
| 192.168.69.0/24 | Client Network |

---

## AWS → OPNsense

| Prefix | Description |
|---------|-------------|
| 10.123.0.0/16 | Transit VPC |
| 172.31.0.0/16 | Main Application VPC |

---

# Route Advertisement Flow

```
Home Networks

↓

OPNsense

↓

eBGP

↓

FortiGate

↓

Transit Gateway

↓

Application VPCs
```

---

# FortiGate Configuration

The FortiGate establishes the eBGP session over the VPN tunnel interface and advertises AWS network prefixes while learning on-premises routes.

## Screenshot

![FortiGate BGP](/screenshots/FGT_BGP_Routing_Table.png)

---

# OPNsense FRRouting

FRRouting (FRR) is used on OPNsense to establish the BGP neighbor relationship with the FortiGate.

The firewall advertises local networks and installs AWS routes learned through BGP.

## Screenshot

![OPNsense FRR](/screenshots/OPNSENSE_BGP_Routing_table.png)

---

# Learned Routes

After the BGP session is established, both firewalls automatically populate their routing tables with learned prefixes.

---

# Verification

The BGP session was validated by confirming:

- Neighbor state is **Established**
- Expected prefixes are exchanged
- Learned routes appear in both routing tables
- Bidirectional connectivity between AWS and on-premises networks

---

# Lessons Learned

Implementing eBGP provided practical experience with:

- Dynamic routing over Route-Based IPsec
- Hybrid cloud route advertisement
- Enterprise network scalability
- Transit Gateway integration
- Route propagation and troubleshooting

Although static routes would have been sufficient for this lab, eBGP better reflects how many enterprise environments manage hybrid connectivity between on-premises infrastructure and AWS.
