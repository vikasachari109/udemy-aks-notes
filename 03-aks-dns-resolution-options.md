# AKS DNS Resolution at Scale: Private Clusters, Centralized vs Decentralized DNS
### A Beginner-Friendly Deep Dive

> **What this document covers:** How DNS works in Azure, why private AKS clusters have unique DNS challenges, what "Hub-and-Spoke" networking means, and the two major patterns — **Centralized** and **Decentralized** DNS resolution — with their trade-offs, limits, and CLI flags.

---

## Table of Contents
1. [What is DNS and Why Does It Matter?](#1-what-is-dns-and-why-does-it-matter)
2. [DNS in the Context of Private AKS](#2-dns-in-the-context-of-private-aks)
3. [Azure DNS: Public vs Private DNS Zones](#3-azure-dns-public-vs-private-dns-zones)
4. [Hub-and-Spoke Network Architecture](#4-hub-and-spoke-network-architecture)
5. [The DNS Challenge for Private AKS](#5-the-dns-challenge-for-private-aks)
6. [Option A: Public FQDN Resolving to Private IP (No Private DNS Zone)](#6-option-a-public-fqdn-resolving-to-private-ip-no-private-dns-zone)
7. [Option B: Centralized DNS Resolution](#7-option-b-centralized-dns-resolution)
8. [Option C: Decentralized DNS Resolution](#8-option-c-decentralized-dns-resolution)
9. [Azure Private DNS Zone Limits](#9-azure-private-dns-zone-limits)
10. [Comparison Table](#10-comparison-table)
11. [Decision Guide](#11-decision-guide)

---

## 1. What is DNS and Why Does It Matter?

### DNS in Plain English

**DNS** stands for **Domain Name System**. It is the phonebook of the internet. Instead of memorizing IP addresses like `20.103.218.175`, you use human-readable names like `my-cluster.westeurope.azmk8s.io`. DNS translates the name into the IP address your computer actually needs to make the connection.

**How a DNS lookup works (simplified):**

```
You type: my-cluster.westeurope.azmk8s.io
     │
     ▼
Your computer asks your DNS server:
"What is the IP for my-cluster.westeurope.azmk8s.io?"
     │
     ▼
DNS Server responds:
"That name maps to IP 20.103.218.175"
     │
     ▼
Your computer connects to 20.103.218.175
```

### DNS Record Types You'll Encounter

| Record Type | What It Does | Example |
|-------------|-------------|---------|
| **A Record** | Maps a name to an IPv4 address | `my-cluster → 10.1.0.4` |
| **AAAA Record** | Maps a name to an IPv6 address | `my-cluster → 2001:db8::1` |
| **CNAME Record** | Maps a name to another name (alias) | `www → myapp.azure.com` |
| **NS Record** | Identifies the authoritative name server for a domain | `mycompany.com → ns1.azure-dns.com` |
| **SOA Record** | Start of Authority — contains admin info about a zone | metadata record |
| **TXT Record** | Arbitrary text — used for verification, External-DNS ownership | `"heritage=external-dns..."` |

### TTL — Time to Live
Every DNS record has a **TTL** (Time to Live) measured in seconds. This tells DNS caches how long they should store the answer before asking again. A TTL of `300` means: "cache this answer for 5 minutes, then re-query."

Low TTL = fast to update but more DNS queries (higher traffic).
High TTL = fewer queries but slow to propagate changes.

---

## 2. DNS in the Context of Private AKS

### Recap: What is the AKS Control Plane FQDN?
When you create an AKS cluster, Azure gives the API Server (the brain of the cluster) a **Fully Qualified Domain Name (FQDN)** — a unique, human-readable name like:
```
aks-cluster-rg-aks-17b128.hcp.westeurope.azmk8s.io
```

This FQDN can resolve to:
- A **public IP** → publicly accessible API Server
- A **private IP** → only accessible from inside a VNet

### Why DNS Is Critical for Private Clusters
In a **private AKS cluster**, the API Server has no public IP. Its FQDN points to a **Private Endpoint** inside your VNet, which has a private IP (e.g., `10.1.0.4`).

For anyone — a developer, a CI/CD pipeline, or a worker node — to reach the API Server, their DNS resolver must be able to translate the FQDN to that private IP. If the DNS resolver doesn't know about the private IP, the connection fails entirely.

This is the core DNS challenge for private AKS: **making sure the right machines can resolve the private FQDN to the correct private IP.**

---

## 3. Azure DNS: Public vs Private DNS Zones

Azure offers two types of DNS zones. Understanding both is essential.

### Azure Public DNS Zone
A **Public DNS Zone** is a standard internet-facing DNS zone hosted in Azure. It is authoritative for a domain like `mycompany.com`. Anyone on the internet can query it.

Example: If you own `mycompany.com` and host it in Azure DNS, you can create records like:
```
app1.mycompany.com  →  A  →  20.30.40.50
```
Anyone in the world can resolve `app1.mycompany.com`.

**Cost**: ~$0.50/month per zone + per-query charges.

### Azure Private DNS Zone
A **Private DNS Zone** is a DNS zone that is **only visible from within specific VNets** that you link to it. It is not exposed to the internet.

Example: You create a Private DNS Zone for `privatelink.westeurope.azmk8s.io` and create a record:
```
aks-cluster-rg-17b128  →  A  →  10.1.0.4
```
Only VMs/pods inside VNets **linked** to this Private DNS Zone can resolve that name to `10.1.0.4`. Machines outside get "Non-existent domain."

### Virtual Network Links
A **Virtual Network Link** is the connection between a Private DNS Zone and a VNet. Without a link, the VNet cannot resolve names in that zone. You can link one Private DNS Zone to many VNets, and one VNet can be linked to many Private DNS Zones.

**Why does AKS need a VNet link?** When AKS creates a Private Endpoint for the Control Plane, it also creates a Private DNS Zone and links it to the cluster's VNet. This is what allows the worker nodes (which are inside that VNet) to resolve the API Server's FQDN to the private IP.

---

## 4. Hub-and-Spoke Network Architecture

Before diving into DNS patterns, you need to understand **Hub-and-Spoke** — because this is the context in which "centralized vs decentralized DNS" becomes meaningful.

### What is Hub-and-Spoke?
**Hub-and-Spoke** is a network topology commonly used in enterprise Azure deployments. Think of it like an airport:

- The **Hub** is the central airport — it has shared services like a DNS server, a firewall (Azure Firewall), a VPN gateway, and connectivity to on-premises.
- The **Spokes** are the terminal buildings — each one is a separate VNet for a specific team, application, or environment (dev, staging, prod).
- The Spokes connect back to the Hub via **VNet Peering** — a direct, low-latency connection between two VNets.

```
             ┌─────────────────────────────────────┐
             │            HUB VNET                  │
             │  ┌─────────────┐  ┌───────────────┐  │
             │  │ Azure        │  │ DNS Resolver /│  │
             │  │ Firewall     │  │ DNS Forwarder  │  │
             │  └─────────────┘  └───────────────┘  │
             │  ┌─────────────┐  ┌───────────────┐  │
             │  │ VPN Gateway │  │  On-premises  │  │
             │  └─────────────┘  │  connectivity │  │
             └────────┬──────────┴───────────────────┘
                      │ VNet Peering
          ┌───────────┼────────────┐
          │           │            │
   ┌──────▼───┐  ┌────▼─────┐  ┌──▼───────┐
   │ Spoke 1  │  │ Spoke 2  │  │ Spoke 3  │
   │ AKS Prod │  │ AKS Dev  │  │ AKS Stage│
   └──────────┘  └──────────┘  └──────────┘
```

### Why Hub-and-Spoke?
- **Centralized security**: All traffic passes through the Hub's firewall
- **Shared services**: One VPN gateway, one DNS server, one Azure Bastion for all spokes
- **Cost efficiency**: Shared infrastructure costs are split across many teams
- **Isolation**: Each spoke is isolated from other spokes (they don't peer with each other directly)

### VNet Peering
**VNet Peering** is a direct, private network connection between two Azure VNets. Traffic between peered VNets uses Azure's backbone (not the internet) and is fast and low-latency.

**Important DNS point**: By default, VNet Peering does NOT allow a VNet to use the Private DNS Zones linked to the other VNet. If Spoke 1's Private DNS Zone is only linked to Spoke 1's VNet, then the Hub VNet cannot resolve names in that zone — even if they're peered. This is what creates the DNS challenge.

---

## 5. The DNS Challenge for Private AKS

### The Problem
Imagine you have 5 private AKS clusters, each in its own Spoke VNet. Each cluster has its own Private Endpoint with a different private IP, and each cluster's FQDN needs to resolve to its specific private IP.

Now you want your:
- JumpBox VM in the Hub to be able to `kubectl` into any of the 5 clusters
- On-premises developers (connected via VPN to the Hub) to access all clusters
- CI/CD pipelines running in the Hub to deploy to any cluster

For this to work, **the Hub VNet must be able to resolve the private FQDNs of all 5 clusters**. But by default, each cluster's Private DNS Zone is only linked to its own Spoke VNet.

The question becomes: **how do you make the Hub and other VNets able to resolve these private names?**

There are **three approaches**:
1. No Private DNS Zone at all (public FQDN resolves to private IP)
2. **Centralized DNS**: One shared Private DNS Zone for all clusters
3. **Decentralized DNS**: Each cluster has its own Private DNS Zone, linked to the Hub

---

## 6. Option A: Public FQDN Resolving to Private IP (No Private DNS Zone)

### How It Works
This is the simplest approach. When you create a private cluster with `--private-dns-zone="none"`, AKS **does not create a Private DNS Zone**. Instead, the **public FQDN** (which is resolvable by anyone on the internet) resolves directly to the **private IP** of the Private Endpoint.

```bash
nslookup aks-prbob7iw.hcp.swedencentral.azmk8s.io
# Output:
# Address: 10.1.0.4    ← PRIVATE IP! Visible to anyone who resolves it.
```

The FQDN is publicly exposed (it appears in public DNS), but it points to a private IP (`10.1.0.4`). Only machines inside the VNet (or connected networks) can actually **reach** that IP, but anyone can **see** the IP by querying public DNS.

### CLI Flag
```bash
az aks create \
  --name my-cluster \
  --resource-group my-rg \
  --enable-private-cluster \
  --private-dns-zone none   # ← No Private DNS Zone created
```

### Advantages
- **Simplest setup** — no DNS infrastructure to maintain
- No Private DNS Zone, no VNet links, no linked zones
- Works immediately with any machine that has internet DNS resolution

### Disadvantages
- ⚠️ **Security concern**: The private IP address of your internal infrastructure is visible to anyone who queries public DNS. A malicious actor now knows that IP `10.1.0.4` exists in your network — useful for network reconnaissance
- Not suitable for high-security environments

### When to Use
- Development or test clusters where security exposure of internal IPs is acceptable
- Environments where you cannot manage Private DNS Zones

---

## 7. Option B: Centralized DNS Resolution

### The Big Picture
In **Centralized DNS**, you create **one single Private DNS Zone** for the AKS `privatelink` domain (e.g., `privatelink.swedencentral.azmk8s.io`) and use it for **all** your AKS clusters. All cluster FQDNs across all spoke VNets are registered as A records in this single, shared zone.

```
One Centralized Private DNS Zone:
privatelink.swedencentral.azmk8s.io

Records:
aks-cluster-spoke1  →  10.1.0.4    (Cluster 1 in Spoke 1)
aks-cluster-spoke2  →  10.2.0.4    (Cluster 2 in Spoke 2)
aks-cluster-spoke3  →  10.3.0.4    (Cluster 3 in Spoke 3)

Zone is linked to:
  - Hub VNet          ← operators and CI/CD can resolve
  - Spoke 1 VNet      ← (technical requirement for AKS)
  - Spoke 2 VNet      ← (technical requirement for AKS)
  - Spoke 3 VNet      ← (technical requirement for AKS)
```

### Why Must the Zone Be Linked to Both Hub AND Spoke?
The Private DNS Zone must be linked to the Spoke VNet (as an internal technical requirement by AKS — the worker nodes need to resolve their own API Server's FQDN). AND it must be linked to the Hub VNet so that Hub-based machines (JumpBox, CI/CD agents, VPN-connected developers) can also resolve it.

### Architecture Diagram
```
                    ┌──────────────────────────┐
                    │    CENTRALIZED           │
                    │ Private DNS Zone          │
                    │ privatelink.*.azmk8s.io  │
                    │                          │
                    │  cluster1 → 10.1.0.4     │
                    │  cluster2 → 10.2.0.4     │
                    └──────────┬───────────────┘
           VNet Links          │
      ┌────────────────────────┼──────────┐
      │                        │          │
┌─────▼────┐           ┌───────▼───┐  ┌──▼──────┐
│ Hub VNet │           │ Spoke 1   │  │ Spoke 2 │
│ JumpBox  │           │ AKS       │  │ AKS     │
│ CI/CD    │           │ Cluster 1 │  │Cluster 2│
└──────────┘           └───────────┘  └─────────┘
```

### CLI Flag
```bash
az aks create \
  --name my-cluster \
  --resource-group my-rg \
  --enable-private-cluster \
  --private-dns-zone "system"        # AKS manages the zone
  # OR
  --private-dns-zone "/subscriptions/.../privatelink.azmk8s.io"  # BYO zone
```

Using `--private-dns-zone="system"` tells AKS to create/use the centralized zone. Using a specific Zone ID (`"ZoneID"`) lets you pass in an existing zone you already created and manage.

### nslookup Verification
```bash
nslookup aks-89eprte0.202.privatelink.swedencentral.azmk8s.io
# Output:
# Address: 10.1.0.4   ← Private IP, only resolvable from linked VNets
```

### Advantages
- **Simple management**: one zone to maintain, one place to check records
- **Hub-native**: Hub-linked zone means everyone connected to the Hub (VPN users, CI/CD agents in Hub) can automatically reach all clusters
- Works well for small to medium numbers of clusters

### Disadvantages
- ⚠️ **Security risk**: All clusters can read and modify the shared Private DNS Zone (unless you lock down RBAC carefully). A misconfigured cluster could overwrite another cluster's DNS record
- As clusters grow (hundreds of clusters), the single zone becomes large and harder to manage
- Requires careful RBAC to prevent one team's cluster from affecting another team's DNS records

---

## 8. Option C: Decentralized DNS Resolution

### The Big Picture
In **Decentralized DNS**, each AKS cluster gets **its own Private DNS Zone**. The zone name is unique per cluster (it contains the cluster's UUID). Each cluster only manages its own zone.

```
Cluster 1 Private DNS Zone:
628fd8ef.privatelink.swedencentral.azmk8s.io
  └── aks-cluster-spoke1 → 10.1.0.4

Cluster 2 Private DNS Zone:
7a9b1234.privatelink.swedencentral.azmk8s.io
  └── aks-cluster-spoke2 → 10.2.0.4
```

### The Challenge: Linking to the Hub
Each cluster's Private DNS Zone is only linked to its own Spoke VNet by default. For the Hub VNet to resolve each cluster's FQDN, you must:

1. **Create a VNet Link** from each cluster's Private DNS Zone to the Hub VNet
2. This must happen automatically as new clusters are created — typically via **Azure Policy** that detects new Private DNS Zones and auto-creates the Hub link

### Architecture Diagram
```
┌─────────────────────────────────────────────┐
│              Hub VNet                        │
│  DNS links:                                  │
│    ← linked to Zone A (Cluster 1's zone)    │
│    ← linked to Zone B (Cluster 2's zone)    │
└──────────────────────────────────────────────┘
         ↑ VNet Links (added by Azure Policy)
    ┌────┴─────┐           ┌──────────┐
    │  Zone A  │           │  Zone B  │
    │Cluster 1 │           │Cluster 2 │
    │privatelink│          │privatelink│
    └────┬─────┘           └────┬─────┘
         │                      │
    ┌────▼─────┐           ┌────▼─────┐
    │ Spoke 1  │           │ Spoke 2  │
    │ AKS      │           │ AKS      │
    │ Cluster 1│           │ Cluster 2│
    └──────────┘           └──────────┘
```

### CLI Flag
```bash
# Same flags as centralized, but each cluster creates its own zone:
az aks create \
  --name my-cluster \
  --resource-group my-rg \
  --enable-private-cluster \
  --private-dns-zone "system"   # Each cluster gets its own zone
```

### Automating Hub Links with Azure Policy
When a new cluster is created, an **Azure Policy** can automatically detect the new Private DNS Zone and add a VNet Link to the Hub. This prevents manual work for every new cluster.

**Why is this hard without automation?** If you have 50 clusters, you'd need to manually create 50 VNet Links to the Hub. One mistake means one cluster's FQDN is unresolvable from the Hub. Azure Policy eliminates this toil.

### nslookup Verification
```bash
# Same result as centralized — private IP resolution within linked VNets:
nslookup aks-89eprte0.202.privatelink.swedencentral.azmk8s.io
# Address: 10.1.0.4

# From outside linked VNets:
# *** Can't find xxx: Non-existent domain
```

### Advantages
- **Full isolation**: Each cluster's DNS zone is completely independent. No risk of one cluster overwriting another's records
- **Better security posture**: Each team only has access to their own zone
- Better at scale — each zone stays small (just one record per cluster)
- Easier audit and RBAC per zone

### Disadvantages
- ⚠️ Requires automation (Azure Policy) to keep Hub VNet links up to date as new clusters are created
- More zones to manage overall (N clusters = N zones)
- Initial setup complexity is higher

---

## 9. Azure Private DNS Zone Limits

These are hard limits you must plan around:

| Limit | Value | Implication |
|-------|-------|-------------|
| VNet links per Private DNS Zone | **1,000** | A centralized zone can be linked to at most 1,000 VNets |
| Private DNS Zones per VNet | **1,000** | A VNet can be linked to at most 1,000 different Private DNS Zones |
| VNet Peerings per VNet | **500** | A Hub VNet can peer with at most 500 spoke VNets |

### What Do These Limits Mean in Practice?

**Centralized model**: If you link one zone to the Hub + all spoke VNets, and you have 999 spokes, you hit the 1,000 link limit. Most enterprises have far fewer than this, so centralized is usually fine.

**Decentralized model**: The Hub VNet can be linked to at most 1,000 Private DNS Zones. If you have more than 1,000 AKS clusters (each with its own zone), you'll hit the VNet link limit on the Hub. For most organizations, 1,000 clusters is enormous.

**VNet Peering limit**: The Hub can only peer with 500 spoke VNets. If you have more than 500 spoke VNets, you need a hierarchical (Hub of Hubs) architecture.

---

## 10. Comparison Table

| Aspect | Public FQDN / No DNS Zone | Centralized DNS | Decentralized DNS |
|--------|--------------------------|-----------------|-------------------|
| **Private DNS Zone?** | ❌ No | ✅ One shared zone | ✅ One zone per cluster |
| **FQDN visible publicly?** | ✅ Yes (resolves to private IP) | ❌ No | ❌ No |
| **Private IP exposed?** | ⚠️ Yes (public DNS shows it) | ❌ No | ❌ No |
| **Hub can resolve all clusters?** | ✅ Yes (public DNS) | ✅ Yes (Hub linked to zone) | ✅ Yes (with Azure Policy) |
| **Cluster isolation** | N/A | ⚠️ Shared zone (all clusters can modify) | ✅ Full isolation per cluster |
| **Azure Policy needed?** | ❌ No | ❌ No | ✅ Yes (for Hub auto-linking) |
| **Setup complexity** | Low | Medium | High |
| **Scale** | Best (no DNS infra) | Good (up to 1,000 VNet links) | Best (each zone is small) |
| **Security** | ⚠️ IP exposed publicly | ⚠️ All clusters share zone | ✅ Full isolation |
| **CLI flag** | `--private-dns-zone none` | `--private-dns-zone system/ZoneID` | `--private-dns-zone system/ZoneID` |

---

## 11. Decision Guide

```
Do you need maximum security (private IP never exposed)?
│
├─► NO  → Option A: Public FQDN (--private-dns-zone none)
│         Simple, quick, fine for dev/test
│
└─► YES
      │
      ├─► Do you have < 100 clusters or a small org?
      │     └─► YES → Centralized DNS (one shared zone)
      │               Simple to manage, good for most enterprises
      │
      └─► Do you need full DNS isolation per team/cluster?
            └─► YES → Decentralized DNS (one zone per cluster)
                       Add Azure Policy for automatic Hub linking
```

### Microsoft's Recommendation for Enterprise
For most enterprises using Hub-and-Spoke:
- Use **Centralized DNS** for simplicity if teams can be trusted with shared zone RBAC
- Use **Decentralized DNS** if teams need strict isolation or you have a large number of independent teams

---

## References & Further Reading
- [Azure Private DNS Zone documentation](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview)
- [Private AKS cluster DNS options](https://learn.microsoft.com/en-us/azure/aks/private-clusters#options-for-connecting-to-the-private-cluster)
- [Hub-and-Spoke topology in Azure](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke)
- [Azure DNS limits](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-dns-limits)