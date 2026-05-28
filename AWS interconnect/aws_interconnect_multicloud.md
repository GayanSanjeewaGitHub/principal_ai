# AWS Interconnect - Multicloud

> Announced: April 14, 2026 — General Availability  
> First launch partner: Google Cloud  
> Coming later in 2026: Microsoft Azure, Oracle Cloud Infrastructure (OCI)

---

## What Is It?

AWS Interconnect - multicloud is the **first purpose-built product** for private, high-speed connections between AWS and other cloud service providers (CSPs). Previously, customers had to "DIY" their multicloud networking — managing complex global multi-layered networks at scale.

**Key capabilities:**
- Private, secure, dedicated bandwidth connections between Amazon VPCs and other cloud environments
- Built-in resiliency (redundant physical paths)
- Scale to multiple VPCs or Regions by associating with **AWS Transit Gateway** or **AWS Cloud WAN**
- New single-fee pricing based on selected bandwidth and geographical scope
- **One free local 500Mbps interconnect per Region** starting May 2026

---

## Scenario: AWS (2 Regions, 2 AZs each) ↔ GCP (3 Singapore-area Regions)

### Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════════════════╗
║                                        AWS SIDE                                              ║
║                                                                                              ║
║  ┌─────────────────────────────────────┐    ┌─────────────────────────────────────┐         ║
║  │  AWS Region 1 (ap-southeast-1 SG)   │    │  AWS Region 2 (ap-southeast-3 BKK)  │         ║
║  │                                     │    │                                     │         ║
║  │  ┌──────────┐   ┌──────────┐        │    │  ┌──────────┐   ┌──────────┐        │         ║
║  │  │  AZ 1a   │   │  AZ 1b   │        │    │  │  AZ 2a   │   │  AZ 2b   │        │         ║
║  │  │ [EC2/svc]│   │ [EC2/svc]│        │    │  │ [EC2/svc]│   │ [EC2/svc]│        │         ║
║  │  └────┬─────┘   └────┬─────┘        │    │  └────┬─────┘   └────┬─────┘        │         ║
║  │       └──────┬────────┘             │    │       └──────┬────────┘             │         ║
║  │         VPC route table             │    │         VPC route table             │         ║
║  │    (GCP CIDR → TGW attachment)      │    │    (GCP CIDR → TGW attachment)      │         ║
║  │              │                      │    │              │                      │         ║
║  │      ┌───────▼────────┐             │    │      ┌───────▼────────┐             │         ║
║  │      │ Transit Gateway│             │    │      │ Transit Gateway│             │         ║
║  │      │  (Region 1)    │             │    │      │  (Region 2)    │             │         ║
║  │      └───────┬────────┘             │    │      └───────┬────────┘             │         ║
║  └──────────────┼──────────────────────┘    └──────────────┼──────────────────────┘         ║
║                 │                                          │                                 ║
║                 │    ┌──────────────────────────┐          │                                 ║
║                 └───►│   AWS Cloud WAN / TGW    │◄─────────┘                                 ║
║                      │   Global Peering         │                                            ║
║                      └──────────┬───────────────┘                                            ║
║                                 │                                                            ║
║                      ┌──────────▼───────────────┐                                            ║
║                      │  AWS Interconnect         │                                            ║
║                      │  - multicloud             │                                            ║
║                      │  (provisioned per region) │                                            ║
║                      │  dedicated bandwidth,     │                                            ║
║                      │  single-fee pricing       │                                            ║
║                      └──────────┬───────────────┘                                            ║
╚═════════════════════════════════╪════════════════════════════════════════════════════════════╝
                                  │
              ╔═══════════════════╪══════════════════════════╗
              ║  PRIVATE FIBER    │  (not public internet)   ║
              ║  dedicated link   │  high-speed, encrypted   ║
              ╚═══════════════════╪══════════════════════════╝
                                  │
╔═════════════════════════════════╪════════════════════════════════════════════════════════════╗
║                                 │         GCP SIDE                                           ║
║                      ┌──────────▼───────────────┐                                            ║
║                      │  GCP Cloud Interconnect   │                                            ║
║                      │  Attachment (VLAN)        │                                            ║
║                      │  + Cloud Router (BGP)     │  ← BGP exchanges routes                   ║
║                      └──────────┬───────────────┘     (AWS VPC CIDRs ↔ GCP VPC CIDRs)       ║
║                                 │                                                            ║
║         ┌───────────────────────┼───────────────────────┐                                    ║
║         │                       │                       │                                    ║
║  ┌──────▼──────┐        ┌───────▼─────┐        ┌───────▼─────┐                              ║
║  │ GCP Region 1│        │ GCP Region 2│        │ GCP Region 3│                              ║
║  │asia-se1 (SG)│        │asia-se2(JKT)│        │ asia-e1(TW) │                              ║
║  │             │        │             │        │             │                              ║
║  │ ┌─────────┐ │        │ ┌─────────┐ │        │ ┌─────────┐ │                              ║
║  │ │Zone-a   │ │        │ │Zone-a   │ │        │ │Zone-a   │ │                              ║
║  │ │[VM/svc] │ │        │ │[VM/svc] │ │        │ │[VM/svc] │ │                              ║
║  │ ├─────────┤ │        │ ├─────────┤ │        │ ├─────────┤ │                              ║
║  │ │Zone-b   │ │        │ │Zone-b   │ │        │ │Zone-b   │ │                              ║
║  │ │[VM/svc] │ │        │ │[VM/svc] │ │        │ │[VM/svc] │ │                              ║
║  │ ├─────────┤ │        │ └─────────┘ │        │ └─────────┘ │                              ║
║  │ │Zone-c   │ │        │             │        │             │                              ║
║  │ │[VM/svc] │ │        │  GCP VPC    │        │  GCP VPC    │                              ║
║  │ └─────────┘ │        │  (internal  │        │  (internal  │                              ║
║  │             │        │   routing)  │        │   routing)  │                              ║
║  │  GCP VPC    │        └─────────────┘        └─────────────┘                              ║
║  └─────────────┘                                                                             ║
╚══════════════════════════════════════════════════════════════════════════════════════════════╝
```

---

## How the Packet Actually Travels — Step by Step

```
EC2 in AWS AZ-1a  →  wants to reach GCP VM in asia-southeast1-a

Step 1   EC2 sends packet to GCP IP (e.g. 10.128.0.5)
         Subnet route table says:
           10.128.0.0/16  →  tgw-xxxxxxxx  (Transit Gateway)

Step 2   Transit Gateway receives it
         TGW route table says:
           10.128.0.0/16  →  interconnect-attachment

Step 3   Packet hits AWS Interconnect - multicloud endpoint
         Leaves AWS network on DEDICATED PRIVATE LINK
         (never touches public internet)

Step 4   GCP Cloud Router receives it via BGP session
         Already knows AWS routes because BGP peering
         was established at setup time

Step 5   GCP VPC internal routing takes over
         Routes to the correct Region → correct Zone → VM
```

---

## Key Concepts

### Why Transit Gateway?
Your 2 regions × 2 AZs = 4 possible sources. Without TGW you'd need an Interconnect attachment per VPC. TGW is the **single hub** — any AZ in any region funnels through it.

### How does GCP know AWS CIDRs (and vice versa)?
**BGP session** between AWS Interconnect and GCP Cloud Router. At setup time they exchange route tables. After that, routing is automatic — any new subnet added to either side propagates via BGP.

### Zone-level routing on GCP side?
GCP Zones are **NOT** separate routing domains — they all share the same VPC. Once the packet enters the GCP VPC, GCP's internal SDN routes it to the correct zone transparently. You target a VM IP, GCP handles the rest.

### Resiliency
AWS provisions **redundant physical paths** at the Interconnect level, so a single fiber cut doesn't drop your connection.

---

## Pricing Summary

| Model | Detail |
|---|---|
| Single-fee | Based on selected bandwidth + geographical scope |
| Free tier | 1 free local 500Mbps interconnect per Region (from May 2026) |
| Scaling | Associate with TGW or Cloud WAN to reach multiple VPCs/Regions without re-provisioning |
