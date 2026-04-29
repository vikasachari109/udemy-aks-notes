# 📘 Doc 09 — AKS NAT Gateway Outbound: Deep Dive

> **Goal:** Understand in-depth how NAT Gateway functions as the AKS egress mechanism — traffic flows to internet vs VNet vs peered VNets, what happens when VMs share the AKS subnet, Availability Zone constraints, and the zonal architecture needed for production HA clusters.

---

## 🧭 Table of Contents

1. [Quick Recap: Why NAT Gateway?](#1-quick-recap-why-nat-gateway)
2. [How NAT Gateway Works at the Subnet Level](#2-how-nat-gateway-works-at-the-subnet-level)
3. [Traffic Flow: Egress to Internet](#3-traffic-flow-egress-to-internet)
4. [Traffic Flow: AKS to Same VNet (No NAT)](#4-traffic-flow-aks-to-same-vnet-no-nat)
5. [Traffic Flow: AKS to Peered VNet (No NAT)](#5-traffic-flow-aks-to-peered-vnet-no-nat)
6. [What Happens When a VM NIC Is in the AKS Subnet](#6-what-happens-when-a-vm-nic-is-in-the-aks-subnet)
7. [Effective Routes Explained](#7-effective-routes-explained)
8. [NAT Gateway Is a Zonal Resource (Standard SKU)](#8-nat-gateway-is-a-zonal-resource-standard-sku)
9. [Availability Zones in Azure — Explained from Scratch](#9-availability-zones-in-azure--explained-from-scratch)
10. [The Zonal NAT Gateway Problem for AKS](#10-the-zonal-nat-gateway-problem-for-aks)
11. [Solution: One NAT Gateway Per Availability Zone](#11-solution-one-nat-gateway-per-availability-zone)
12. [Implementing Multi-Zone NAT Gateway for AKS](#12-implementing-multi-zone-nat-gateway-for-aks)
13. [StandardV2 NAT Gateway: Zone-Redundant Alternative](#13-standardv2-nat-gateway-zone-redundant-alternative)
14. [SNAT Port Dynamics with NAT Gateway](#14-snat-port-dynamics-with-nat-gateway)
15. [Using Public IP Prefix Instead of Individual IPs](#15-using-public-ip-prefix-instead-of-individual-ips)
16. [NAT Gateway vs Other Egress Options — When Each Wins](#16-nat-gateway-vs-other-egress-options--when-each-wins)
17. [Monitoring NAT Gateway Health](#17-monitoring-nat-gateway-health)
18. [Cost Considerations](#18-cost-considerations)

---

## 1. Quick Recap: Why NAT Gateway?

Azure Load Balancer (the default AKS egress) pre-allocates a fixed number of SNAT ports per node. As your cluster scales, each node gets fewer ports, leading to **SNAT exhaustion** — connection failures from pods.

NAT Gateway solves this with **on-demand, shared SNAT port allocation**. Instead of fixed per-node allocations, all nodes in a subnet share a pool. Ports are allocated dynamically when connections open and released when they close.

| Dimension | Load Balancer | NAT Gateway |
|-----------|--------------|-------------|
| Port allocation | Pre-allocated per node | On-demand, shared |
| Wasted ports | Yes — locked to idle nodes | No waste |
| Max ports (1 IP) | 64,000 but distributed fixedly | 64,512 fully on-demand |
| Scales with cluster size | Ports per node decrease | Capacity unchanged |
| Outbound only | No (also handles ingress LB) | Yes (egress only) |

---

## 2. How NAT Gateway Works at the Subnet Level

NAT Gateway is a **subnet-level resource** — not a VM-level or cluster-level resource. This is a fundamental and important characteristic.

### What "Subnet Level" Means

When you attach a NAT Gateway to a subnet:
- **ALL** resources with a NIC in that subnet automatically use the NAT Gateway for outbound internet traffic
- This includes AKS nodes, other VMs you manually deploy into the subnet, Azure Container Instances, etc.
- There is no per-VM configuration needed — it's automatic

### NAT Gateway Intercepts Egress

The Azure network fabric processes traffic at the subnet boundary. When any VM in the subnet sends a packet with a destination outside the VNet (internet), the fabric intercepts it and routes it through the NAT Gateway before it leaves Azure.

```
┌────────────────────────────────────────────────────────────────┐
│                         aks-subnet                              │
│                                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                   │
│  │  Node 1  │   │  Node 2  │   │  Node 3  │                   │
│  │  (VM)    │   │  (VM)    │   │  (VM)    │                   │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘                   │
│       │              │              │                           │
│       └──────────────┴──────────────┘                          │
│                      │                                          │
│                      ▼                                          │
│              [Subnet Boundary]                                  │
│          Outbound to Internet?                                   │
│          YES → Route through NAT Gateway                        │
│          NO (VNet/peered) → Send directly                       │
│                      │                                          │
└──────────────────────┼─────────────────────────────────────────┘
                       │
               ┌───────▼────────┐
               │  NAT Gateway   │  ← SNAT: Private IP → Public IP
               └───────┬────────┘
                       │
                  Public IP(s)
                       │
                   Internet
```

---

## 3. Traffic Flow: Egress to Internet

When an AKS pod makes an outbound internet request, here is the exact path:

### Step-by-Step

**1. Pod initiates request**
```
Pod (10.244.0.5) → wants to reach https://api.stripe.com
```

**2. Kubernetes routing within the node**
```
Pod NIC → Linux bridge (cbr0 or similar) → Node NIC
Pod IP (10.244.0.5) → Node IP (172.16.0.4)
```

**3. Azure VNet routing**
```
Node NIC (172.16.0.4) → Azure VNet fabric
Destination: api.stripe.com → not in VNet or peered VNets → must go to internet
```

**4. NAT Gateway intercepts**
```
Azure network fabric checks: this subnet has a NAT Gateway attached
→ Route through NAT Gateway before sending to internet
```

**5. SNAT — address translation**
```
NAT Gateway:
  Source IP 172.16.0.4 → replaced with NAT Gateway Public IP (e.g., 40.115.29.65)
  Source port (ephemeral) → replaced with a SNAT port from the pool (e.g., :52341)
  
  Connection table entry created:
  172.16.0.4:54312 ↔ 40.115.29.65:52341 → api.stripe.com:443
```

**6. Packet reaches internet**
```
Packet: SRC=40.115.29.65:52341, DST=api.stripe.com:443
Stripe sees the request coming from 40.115.29.65
```

**7. Response returns**
```
Stripe responds: SRC=api.stripe.com:443, DST=40.115.29.65:52341
NAT Gateway receives it → looks up connection table → 
   maps 40.115.29.65:52341 → 172.16.0.4:54312
→ delivers to the node → pod receives response
```

### Live Demo

```bash
# Deploy a pod
kubectl run nginx --image=nginx

# Check what IP the internet sees
kubectl exec nginx -it -- /bin/bash
curl https://ifconfig.me   # → 40.115.29.65   (NAT GW IP 1)
curl https://ifconfig.me   # → 13.81.209.102  (NAT GW IP 2, round-robin)

# The NAT Gateway's outbound IPs from Azure portal match these values
```

---

## 4. Traffic Flow: AKS to Same VNet (No NAT)

**NAT Gateway ONLY applies to internet-bound traffic.** Traffic destined for another resource within the same VNet bypasses NAT Gateway entirely.

### Example

AKS cluster VNet: `172.16.0.0/20`  
AKS nodes subnet: `172.16.0.0/22` (with NAT Gateway attached)  
A database VM also in the VNet: `172.16.4.10`

```
AKS Pod (10.244.0.5) → Node (172.16.0.4) → Database VM (172.16.4.10)

Traffic path:
  172.16.0.4 → 172.16.4.10
  Destination 172.16.4.10 is within VNet (172.16.0.0/20)
  → Azure routes directly via VNet — NAT Gateway is NOT invoked
  → No SNAT occurs — the source IP arrives as 172.16.0.4
```

**Why this matters:**
- Internal traffic is free (no NAT Gateway data processing charges)
- Internal traffic is lower latency (no extra hop through NAT Gateway)
- The database VM sees the actual node private IP as the source (useful for IP-based access control)
- NAT Gateway capacity (SNAT ports) is NOT consumed by internal traffic

---

## 5. Traffic Flow: AKS to Peered VNet (No NAT)

**VNET Peering** is a mechanism to connect two Azure VNets so their resources can communicate as if they're in the same network (using private IPs). Peered VNet traffic also bypasses the NAT Gateway.

```
┌──────────────────────────────┐          ┌──────────────────────────────┐
│         VNET-AKS             │          │          VNET-VM              │
│         172.16.0.0/20        │          │          10.0.0.0/16          │
│                              │          │                              │
│  ┌────────────────────┐      │          │   ┌────────────────────┐    │
│  │    AKS Cluster     │      │◄────────►│   │    VM              │    │
│  │    Subnet          │      │ Peering  │   │    10.0.1.5        │    │
│  │    172.16.0.0/22   │      │          │   └────────────────────┘    │
│  │    (NAT GW here)   │      │          │                              │
│  └────────────────────┘      │          └──────────────────────────────┘
│                              │
│  ┌─────────────┐             │
│  │ NAT Gateway │──────────────────────────────────────────────────► Internet
│  └─────────────┘             │
└──────────────────────────────┘
```

**Traffic from AKS pod → VM in VNET-VM:**
```
Pod (10.244.0.5) → Node (172.16.0.4) → [VNet peering] → VM (10.0.1.5)
```

The peering handles routing. Azure knows that `10.0.0.0/16` is reachable via the peering. The NAT Gateway only handles `0.0.0.0/0` (default route) traffic that isn't matched by any more specific route.

**Why this matters:**
- AKS can freely communicate with databases, VMs, and other services in peered VNets WITHOUT consuming NAT Gateway ports
- This supports common architectures where the "hub" VNet has shared services (DNS, Active Directory, Bastion) accessed via peering

---

## 6. What Happens When a VM NIC Is in the AKS Subnet

Sometimes you might deploy a jump-box VM or a monitoring agent VM directly into the AKS subnet — or you might be curious about what happens in such scenarios. Any VM with its NIC in the AKS subnet automatically inherits the NAT Gateway.

### Scenario Setup

```
AKS Subnet: 172.16.0.0/22
NAT Gateway: nat-gateway (public IP: 40.115.29.65)
AKS nodes: 172.16.0.4, 172.16.0.5, 172.16.0.6
External VM: NIC configured in this same subnet → 172.16.0.100
```

The external VM (at 172.16.0.100) will also route internet-bound traffic through the NAT Gateway. Its effective routes are the same as the AKS nodes.

### The Effective Routes

When you check "Effective routes" on the VM's NIC in the Azure portal (NIC → Effective routes), you see:

| Source | State | Address Prefix | Next Hop Type | Meaning |
|--------|-------|---------------|---------------|---------|
| Default | Active | 172.16.0.0/20 | Virtual network | Traffic to AKS VNet goes directly within VNet |
| Default | Active | 10.0.0.0/16 | VNet peering | Traffic to peered VNET goes via peering |
| Default | Active | 0.0.0.0/0 | Internet | All other traffic → internet (via NAT Gateway) |
| Default | Active | 10.0.0.0/8 | None | RFC 1918 private range → no route (dropped) |
| Default | Active | 127.0.0.0/8 | None | Loopback → no route |
| Default | Active | 172.16.0.0/12 | None | RFC 1918 → no route |
| Default | Active | 100.64.0.0/10 | None | Shared address space → no route |

**The critical route:** `0.0.0.0/0 → Internet` combined with the NAT Gateway on the subnet means all internet traffic goes through the NAT Gateway (not directly to the internet).

**Important note:** "Next Hop Type = Internet" here does NOT mean traffic bypasses the NAT Gateway. The NAT Gateway is attached at the subnet level, below the routing layer. Even traffic routed to "Internet" will be intercepted by the NAT Gateway if the subnet has one.

---

## 7. Effective Routes Explained

"Effective routes" are the actual routing rules in effect for a VM's NIC — the combination of:
- Azure system default routes (always present)
- Routes from any associated Route Table
- Routes from VNet peering
- Routes from special services like NAT Gateway

### Reading the Route Table

```
Destination   Next Hop Type    Next Hop Address
─────────────────────────────────────────────────
172.16.0.0/20  Virtual network  (local VNet)
10.0.0.0/16    VNet Peering     (to peered VNet)
0.0.0.0/0      Internet         (default route)
10.0.0.0/8     None             (dropped - RFC 1918)
```

**Route matching is longest-prefix first:**
- A packet to `172.16.5.0` matches `172.16.0.0/20` (/20 is more specific) → goes within VNet
- A packet to `8.8.8.8` doesn't match any specific route → falls through to `0.0.0.0/0` → Internet

### How NAT Gateway Fits In

The NAT Gateway doesn't appear in the route table as a next hop. Instead, it's transparently attached to the subnet. When Azure sees a packet being routed to "Internet" from a subnet with a NAT Gateway, it automatically intercepts it and routes it through the NAT Gateway. This is an Azure networking layer behavior, below the routing table abstraction.

---

## 8. NAT Gateway Is a Zonal Resource (Standard SKU)

This is one of the most important and most misunderstood aspects of NAT Gateway.

**Azure NAT Gateway (Standard SKU) is ZONAL** — meaning it is physically located in ONE Availability Zone and ONLY serves resources in that zone.

When you create a Standard NAT Gateway:
```bash
az network nat gateway create \
  --resource-group $RG \
  --name nat-gateway \
  --zone 1 \                    # ← THIS specifies the zone
  --public-ip-addresses pip-1
```

This NAT Gateway lives in Zone 1. It will serve VMs in Zone 1 of the subnet. VMs in Zone 2 and Zone 3 will NOT be served by this NAT Gateway.

**What happens if you create a NAT Gateway WITHOUT specifying a zone?**  
Azure creates a "no-zone" NAT Gateway that is not associated with any particular zone. This is NOT zone-redundant — it is simply unzoned (zone-agnostic). If there's a zone failure, an unzoned NAT Gateway may or may not be affected depending on where Azure placed it.

For true HA, you need a deliberately zonal setup (one NAT GW per zone) or StandardV2 (zone-redundant).

---

## 9. Availability Zones in Azure — Explained from Scratch

Before we can understand the NAT Gateway zone problem, you need to understand Azure Availability Zones.

### What Is a Data Center?

A data center is a building full of servers, power supplies, cooling systems, and network connections. If that building loses power, loses cooling, or has a network cut, all servers in it go offline.

### What Is an Availability Zone?

An **Availability Zone (AZ)** is a physically separate data center (or cluster of data centers) within a single Azure region, with:
- Its own independent power supply (separate utility feed, UPS, generators)
- Its own independent cooling
- Its own independent network connections to the internet and other zones
- Physical separation (different buildings, often different cities within the metro area)

If Zone 2 has a power failure, Zone 1 and Zone 3 are completely unaffected.

### Azure Regions with Availability Zones

Most Azure regions have 3 Availability Zones: Zone 1, Zone 2, Zone 3.

```
Azure East US 2 Region
┌─────────────────────────────────────────────────────┐
│                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   Zone 1    │  │   Zone 2    │  │   Zone 3    │ │
│  │             │  │             │  │             │ │
│  │ Data Center │  │ Data Center │  │ Data Center │ │
│  │ (Building A)│  │ (Building B)│  │ (Building C)│ │
│  │             │  │             │  │             │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│   ~km apart        ~km apart         ~km apart      │
│   <2ms latency between zones                        │
└─────────────────────────────────────────────────────┘
```

### Why AKS Uses Multiple Zones

Deploying your AKS cluster across 3 zones provides:
- **99.99% uptime SLA** (vs 99.95% for single-zone)
- Survives one entire zone going offline (planned maintenance or failure)
- Zero downtime for workloads (pods in Zone 1 and Zone 3 keep running while Zone 2 is offline)

```bash
az aks create -g $RG -n my-cluster \
  --zones 1 2 3 \   # ← Deploy nodes across all 3 zones
  --node-count 6    # ← 2 nodes per zone
```

---

## 10. The Zonal NAT Gateway Problem for AKS

Here's the critical conflict:

**AKS is deployed across 3 zones** (for HA):
- 2 nodes in Zone 1
- 2 nodes in Zone 2  
- 2 nodes in Zone 3

**Standard NAT Gateway only serves ONE zone** (by design, for isolation).

If you create one NAT Gateway in Zone 1 and attach it to the AKS subnet:

```
Zone 1 nodes → NAT Gateway → Internet   ✅ Works
Zone 2 nodes → NAT Gateway → ???         ❌ Can't reach Zone 1 NAT GW
Zone 3 nodes → NAT Gateway → ???         ❌ Can't reach Zone 1 NAT GW
```

Wait — actually the Azure networking fabric CAN route across zones within a region. The zonal nature of NAT Gateway means it's optimally co-located with Zone 1 resources, but resources in Zone 2 and Zone 3 CAN still use it (with slightly higher latency and less isolation).

**The real problem is failure isolation:**
- If Zone 1 experiences an outage, the NAT Gateway (in Zone 1) fails
- Nodes in Zone 2 and Zone 3 lose egress — even though they're in healthy zones
- Your app is deployed across 3 zones for HA, but your egress has a single zone failure point

This defeats the purpose of multi-zone deployment.

---

## 11. Solution: One NAT Gateway Per Availability Zone

The correct production architecture for a multi-zone AKS cluster:

```
                              Internet
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
        Public IP AZ1      Public IP AZ2      Public IP AZ3
              │                  │                  │
       NAT Gateway AZ1    NAT Gateway AZ2    NAT Gateway AZ3
              │                  │                  │
          Subnet A            Subnet B            Subnet C
          Zone 1 only         Zone 2 only         Zone 3 only
              │                  │                  │
        AKS Nodes AZ1      AKS Nodes AZ2      AKS Nodes AZ3
              │                  │                  │
              └──────────────────┼──────────────────┘
                                 │
                          AKS Cluster
                       (Virtual Network)
```

**Benefits:**
- If Zone 2 fails: NAT Gateway AZ2 fails, but AZ1 and AZ3 continue working. Pods in AZ1 and AZ3 have uninterrupted egress.
- Full zone isolation — no cross-zone dependency for egress
- Egress IPs are zone-specific (if needed for compliance)

**Trade-offs:**
- 3× the NAT Gateway cost
- 3× the management (IP addresses, configurations)
- Requires 3 separate subnets (one per zone)
- Requires 3 separate node pools (one per zone/subnet)

---

## 12. Implementing Multi-Zone NAT Gateway for AKS

Full implementation of a production-grade multi-zone AKS cluster with per-zone NAT Gateways:

### Part 1: Create Public IPs (one per zone)

```bash
for ZONE in 1 2 3; do
  az network public-ip create \
    --resource-group $RG \
    --name pip-nat-az${ZONE} \
    --sku standard \
    --zone $ZONE \
    --allocation-method Static \
    --location $LOCATION
done
```

Note: `--zone $ZONE` on the Public IP creates a **zone-pinned** IP. This IP is only available within that zone, providing full failure isolation.

### Part 2: Create NAT Gateways (one per zone)

```bash
for ZONE in 1 2 3; do
  az network nat gateway create \
    --resource-group $RG \
    --name nat-gateway-az${ZONE} \
    --zone $ZONE \
    --public-ip-addresses pip-nat-az${ZONE} \
    --idle-timeout 4 \
    --location $LOCATION
done
```

### Part 3: Create VNet with Three Subnets

```bash
az network vnet create \
  --resource-group $RG \
  --name vnet-aks-multizone \
  --address-prefixes 10.0.0.0/8 \
  --location $LOCATION

for ZONE in 1 2 3; do
  az network vnet subnet create \
    --resource-group $RG \
    --vnet-name vnet-aks-multizone \
    --name subnet-az${ZONE} \
    --address-prefixes 10.${ZONE}.0.0/16 \
    --nat-gateway nat-gateway-az${ZONE}  # Attach zone-specific NAT GW
done
```

### Part 4: Create AKS Cluster and Node Pools

```bash
# Get subnet IDs
SUBNET_AZ1=$(az network vnet subnet show -g $RG --vnet-name vnet-aks-multizone -n subnet-az1 --query id -o tsv)
SUBNET_AZ2=$(az network vnet subnet show -g $RG --vnet-name vnet-aks-multizone -n subnet-az2 --query id -o tsv)
SUBNET_AZ3=$(az network vnet subnet show -g $RG --vnet-name vnet-aks-multizone -n subnet-az3 --query id -o tsv)

# Create cluster with system node pool in Zone 1
az aks create \
  --resource-group $RG \
  --name aks-multizone \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_AZ1 \
  --outbound-type userAssignedNATGateway \
  --zones 1 \
  --node-count 2 \
  --generate-ssh-keys

# Add node pool for Zone 2
az aks nodepool add \
  --resource-group $RG \
  --cluster-name aks-multizone \
  --name nodepool2 \
  --vnet-subnet-id $SUBNET_AZ2 \
  --zones 2 \
  --node-count 2

# Add node pool for Zone 3
az aks nodepool add \
  --resource-group $RG \
  --cluster-name aks-multizone \
  --name nodepool3 \
  --vnet-subnet-id $SUBNET_AZ3 \
  --zones 3 \
  --node-count 2
```

### Verifying Zone Isolation

```bash
# Schedule a pod on a Zone 1 node and check egress IP
kubectl run test-az1 \
  --image=curlimages/curl \
  --restart=Never \
  --overrides='{"spec":{"nodeSelector":{"topology.kubernetes.io/zone":"1"}}}' \
  -- curl -s ifconfig.me

kubectl logs test-az1
# Should return pip-nat-az1's IP

# Repeat for Zone 2 and Zone 3
```

---

## 13. StandardV2 NAT Gateway: Zone-Redundant Alternative

Azure's newer **StandardV2 NAT Gateway** eliminates the multi-zone complexity by being **zone-redundant by default**.

### How Zone-Redundant Works

StandardV2 NAT Gateway automatically distributes across all available Availability Zones. Instead of being pinned to one zone, it uses Azure's zone-redundant infrastructure:
- Even if Zone 2 fails, the NAT Gateway continues serving zones 1 and 3
- No manual setup of multiple NAT Gateways required
- One NAT Gateway resource, one subnet, full zone redundancy

### Use With AKS

Use `managedNATGatewayV2` outbound type (currently Preview):

```bash
# Register the preview feature
az feature register \
  --namespace "Microsoft.ContainerService" \
  --name "ManagedNATGatewayV2Preview"

# Create cluster with zone-redundant NAT Gateway
az aks create \
  --resource-group $RG \
  --name aks-natv2 \
  --outbound-type managedNATGatewayV2 \
  --nat-gateway-managed-outbound-ip-count 2 \
  --zones 1 2 3  # Cluster spans 3 zones, NAT GW handles all
```

### Comparing Standard vs StandardV2 for AKS

| Aspect | Standard NAT Gateway | StandardV2 NAT Gateway |
|--------|---------------------|----------------------|
| Zone HA | Requires 1 NAT GW per zone | Built-in zone redundancy |
| Complexity | High (3 NAT GWs, 3 subnets, 3 nodepools) | Low (1 NAT GW, 1 subnet) |
| Cost | 3× NAT Gateway charges | 1× (but StandardV2 premium) |
| Public IP SKU | Standard | StandardV2 (new) |
| AKS integration | userAssignedNATGateway | managedNATGatewayV2 |
| Status | GA | Preview |

---

## 14. SNAT Port Dynamics with NAT Gateway

### On-Demand Allocation Explained

Unlike Load Balancer's pre-allocated ports, NAT Gateway uses a dynamic model:

**At t=0 (cluster start, no traffic):**
```
NAT Gateway SNAT port pool: [64,512 available, 0 in use]
```

**At t=1 (pod makes first connection to Stripe):**
```
NAT Gateway allocates port 1024 for this connection
Pool: [64,511 available, 1 in use]
Connection table: 172.16.0.4:32000 ↔ 40.115.29.65:1024 ↔ api.stripe.com:443
```

**At t=2 (100 pods each make 10 connections):**
```
Pool: [63,512 available, 1,000 in use]
```

**At t=3 (connections close after HTTP response):**
```
Ports released after idle timeout (4 min default)
Pool: [64,512 available, 0 in use]  ← Returns to full capacity
```

### Connection Limits in Practice

With 1 public IP (64,512 ports):
```
Max concurrent connections = 64,512
Typical microservice (50ms avg connection duration): 
  64,512 / 0.05s = 1.29 million connections per second possible
```

With 16 public IPs:
```
Max concurrent connections = 1,032,192 (~1 million)
```

Very few production clusters approach these limits.

### When to Add More IPs

Monitor the metric `SNATConnectionCount` with state `Failed` in Azure Monitor:

```
If Failed SNAT connections > 0 → consider adding IPs
```

```bash
# Add a second public IP to NAT Gateway
az network public-ip create -g $RG -n pip-nat-2 --sku standard
az network nat gateway update \
  -g $RG -n nat-gateway \
  --public-ip-addresses pip-nat pip-nat-2
# Now: 128,024 max concurrent connections
```

### Idle Timeout and Port Recycling

The idle timeout determines how long NAT Gateway holds a SNAT port mapping for an idle connection (no data flowing):

| Timeout | Use Case |
|---------|---------|
| 4 minutes (min) | Standard HTTP/HTTPS APIs. Connections close after each request/response. |
| 10–30 minutes | Database connections that stay idle between queries |
| 30–60 minutes | Long polling connections that infrequently exchange data |
| 120 minutes (max) | WebSocket connections that stay connected for very long periods |

**Important:** Setting a very high idle timeout holds SNAT ports for longer, reducing effective pool capacity. Set it to match your actual connection behavior.

---

## 15. Using Public IP Prefix Instead of Individual IPs

Instead of attaching individual Public IPs to a NAT Gateway, you can attach a **Public IP Prefix** — a contiguous block of consecutive IP addresses.

### What Is a Public IP Prefix?

A /28 prefix gives you 16 consecutive IPs:
```
40.115.29.64/28 gives:
  40.115.29.64
  40.115.29.65
  40.115.29.66
  ...
  40.115.29.79
```

### Benefits of Using a Prefix

1. **Simplified allowlisting:** External partners can allowlist `40.115.29.64/28` instead of 16 individual IPs
2. **Routing simplicity:** One CIDR block in firewall rules covers all your egress IPs
3. **Guaranteed contiguous range:** All IPs are consecutive — no gaps

### Creating and Using a Prefix

```bash
# Create a /28 prefix (16 IPs)
az network public-ip prefix create \
  --resource-group $RG \
  --name prefix-natgateway \
  --length 28 \
  --location $LOCATION

# Create NAT Gateway using the prefix (instead of individual IPs)
az network nat gateway create \
  --resource-group $RG \
  --name nat-gateway \
  --public-ip-prefixes prefix-natgateway \
  --idle-timeout 4

# OR: Add prefix to an existing NAT Gateway
az network nat gateway update \
  --resource-group $RG \
  --name nat-gateway \
  --public-ip-prefixes prefix-natgateway
```

### Available Prefix Sizes

| Prefix length | Number of IPs | SNAT ports |
|--------------|--------------|-----------|
| /31 | 2 | 129,024 |
| /30 | 4 | 258,048 |
| /29 | 8 | 516,096 |
| /28 | 16 | 1,032,192 |

A /28 prefix gives you the maximum SNAT port capacity (same as 16 individual IPs) with the simplicity of one CIDR block.

---

## 16. NAT Gateway vs Other Egress Options — When Each Wins

| Criterion | Load Balancer | NAT Gateway | Azure Firewall (UDR) |
|-----------|--------------|-------------|---------------------|
| **Simplicity** | ✅ Default, zero config | ✅ Simple | ❌ Complex setup |
| **SNAT at scale** | ❌ Pre-allocated, exhaustion risk | ✅ On-demand | ✅ (FW handles SNAT) |
| **Traffic inspection** | ❌ Layer 4 only | ❌ Layer 4 only | ✅ Layer 7 |
| **FQDN filtering** | ❌ | ❌ | ✅ |
| **Audit logging** | Aggregate only | Aggregate only | ✅ Per-connection |
| **Cost** | Included | ~$0.045/hr + data | ~$1.25/hr + data |
| **Zone redundancy** | ✅ (Standard LB) | ✅ (StandardV2) | ✅ (w/ zone config) |
| **IP predictability** | ✅ Static | ✅ Static | ✅ Static |
| **Max connections** | ~1M (16 IPs) | ~1M (16 IPs) | Depends on FW SKU |

**NAT Gateway wins when:**
- You need scalable egress without SNAT exhaustion
- You don't need traffic filtering/inspection
- You want simple setup without firewall management

**Firewall wins when:**
- You need L7 inspection, URL filtering, and compliance logging
- Your organization requires centralized egress control
- Enterprise/hub-spoke architecture

---

## 17. Monitoring NAT Gateway Health

### Key Metrics to Monitor

| Metric | What It Tells You | Alert Condition |
|--------|------------------|----------------|
| `SNATConnectionCount (State=Allocated)` | Current active connections | Baseline: know your normal |
| `SNATConnectionCount (State=Failed)` | Failed SNAT attempts (port exhaustion) | Alert if > 0 |
| `DroppedPackets` | Packets dropped by NAT GW | Alert if increasing |
| `ByteCount` | Data throughput | Monitor for anomalies |
| `PacketCount` | Packet throughput | Monitor for anomalies |
| `DatapathAvailability` | NAT Gateway health | Alert if < 100% |

### Setting Up Monitoring

```bash
# Get NAT Gateway resource ID
NAT_GW_ID=$(az network nat gateway show \
  -g $RG -n nat-gateway --query id -o tsv)

# Create an alert for SNAT failures
az monitor metrics alert create \
  --resource-group $RG \
  --name "NAT-Gateway-SNAT-Failure" \
  --scopes $NAT_GW_ID \
  --condition "avg SNATConnectionCount > 0 where ConnectionState='Failed'" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action <action-group-id> \
  --description "NAT Gateway SNAT port exhaustion detected"
```

### Using kubectl to Detect Symptoms

```bash
# Pods with SNAT exhaustion show connection failures in logs
kubectl logs <pod-name> | grep -i "connection refused\|timeout\|ECONNREFUSED\|i/o timeout"

# Check if pod can reach external services
kubectl exec -it <pod-name> -- curl -v --max-time 5 https://api.github.com/zen
```

---

## 18. Cost Considerations

### NAT Gateway Pricing Components

1. **NAT Gateway resource:** ~$0.045/hour (~$32.40/month per NAT Gateway)
2. **Processed data:** ~$0.045 per GB of data processed
3. **Public IP addresses:** ~$3–4/month per Standard IP

### Example Monthly Cost

Single-region cluster, 1 NAT Gateway, 1 public IP, moderate traffic (100 GB/month):
```
NAT Gateway resource:   $32.40
Data processing:        100 GB × $0.045 = $4.50
Public IP:              $3.65
─────────────────────────────
Total:                  ~$40.55/month
```

Multi-zone cluster (3 NAT Gateways, 3 IPs):
```
3 × NAT Gateway resources:   $97.20
Data processing (shared):    $4.50
3 × Public IPs:              $10.95
─────────────────────────────────
Total:                       ~$112.65/month
```

### Cost Optimization Tips

1. **Private endpoints for Azure services:** Traffic to Azure Storage, SQL, Cosmos DB via private endpoints bypasses NAT Gateway → zero data processing cost for internal Azure traffic.

2. **Right-size IP count:** Start with 1 IP (64,512 ports). Only add more if monitoring shows SNAT failures. Avoid over-provisioning.

3. **Keep idle timeout low:** Lower idle timeout = ports released faster = better port utilization per IP = possibly avoid needing more IPs.

4. **Use StandardV2 in dev/test:** One zone-redundant NAT GW instead of 3 Standard NAT GWs in production = 66% cost reduction while still maintaining availability.

---

## 📚 Official References

- [NAT Gateway Resource](https://learn.microsoft.com/en-us/azure/virtual-network/nat-gateway/natgateway-resource)
- [NAT Gateway Availability Zones](https://learn.microsoft.com/en-us/azure/nat-gateway/nat-availability-zones)
- [AKS with NAT Gateway](https://learn.microsoft.com/en-us/azure/aks/nat-gateway)
- [Monitoring NAT Gateway](https://learn.microsoft.com/en-us/azure/nat-gateway/nat-metrics-alerts)
- [NAT Gateway Pricing](https://azure.microsoft.com/en-us/pricing/details/azure-nat-gateway/)