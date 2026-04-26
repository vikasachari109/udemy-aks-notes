# AKS Network Plugins: Kubenet vs Azure CNI
### A Beginner-Friendly Deep Dive

> **What this document covers:** How pods and nodes get their IP addresses in AKS, what happens when they talk to each other, and how to choose between Kubenet, Azure CNI, Azure CNI Overlay, and other options. Every term, resource, and trade-off explained from scratch.

---

## Table of Contents
1. [Why Does Networking in Kubernetes Matter?](#1-why-does-networking-in-kubernetes-matter)
2. [Core Networking Concepts](#2-core-networking-concepts)
3. [What is a Network Plugin (CNI)?](#3-what-is-a-network-plugin-cni)
4. [Overview of AKS Network Plugin Options](#4-overview-of-aks-network-plugin-options)
5. [Option 1 — Kubenet (Basic Networking)](#5-option-1--kubenet-basic-networking)
6. [Option 2 — Azure CNI (Advanced Networking)](#6-option-2--azure-cni-advanced-networking)
7. [Option 3 — Azure CNI Overlay](#7-option-3--azure-cni-overlay)
8. [Option 4 — Cilium](#8-option-4--cilium)
9. [Option 5 — Bring Your Own CNI (BYO)](#9-option-5--bring-your-own-cni-byo)
10. [IP Address Planning — How Many IPs Do You Need?](#10-ip-address-planning--how-many-ips-do-you-need)
11. [CIDR Ranges and Avoiding Conflicts](#11-cidr-ranges-and-avoiding-conflicts)
12. [Full Comparison Tables](#12-full-comparison-tables)
13. [How to Choose — Decision Guide](#13-how-to-choose--decision-guide)

---

## 1. Why Does Networking in Kubernetes Matter?

When you run an application in Kubernetes, it runs inside a **Pod** (a small isolated unit). Your cluster may have hundreds or thousands of pods running simultaneously across multiple nodes (VMs). These pods need to:

1. Talk to **each other** (e.g., a web app pod talking to a database pod)
2. Talk to **services outside the cluster** (e.g., an external database or API)
3. Be **reachable from outside** the cluster (e.g., users accessing your web app)

The **network plugin** (also called the **CNI plugin**) is responsible for how all of this happens. It determines:
- What IP address each pod gets
- How pods on different nodes communicate
- Whether pods are visible from your Azure network
- How many pods can run per node
- Whether you'll run out of IP addresses as you scale

This is one of the **most important architectural decisions** you make when creating an AKS cluster, because it **cannot be changed after creation** without rebuilding the cluster.

---

## 2. Core Networking Concepts

Before exploring the options, let's build a solid foundation of networking terms.

### IP Address
An **IP Address** is a unique numerical label (e.g., `192.168.1.5`) assigned to a device on a network so it can send and receive traffic. Think of it like a postal address — packets are delivered to the right destination because each entity has a unique address.

### CIDR Notation
**CIDR (Classless Inter-Domain Routing)** notation expresses a range of IP addresses. For example:
- `10.224.0.0/16` = all IPs from `10.224.0.0` to `10.224.255.255` = **65,536 addresses**
- `10.224.0.0/24` = all IPs from `10.224.0.0` to `10.224.0.255` = **256 addresses**
- `10.224.0.0/28` = all IPs from `10.224.0.0` to `10.224.0.15` = **16 addresses**

The number after `/` is the **prefix length** — the bigger it is, the **smaller** the address range.

### Subnet
A **Subnet** is a subdivision of a larger network (VNet). Each subnet has its own CIDR range. Resources inside a subnet get IPs from that range. Subnets that aren't directly connected cannot communicate without routing rules.

### NAT — Network Address Translation
**NAT** is a technique where a router or firewall replaces the **source IP address** of a packet with a different IP before forwarding it. This hides the original sender's IP. In AKS, NAT is used to let pods (which have internal-only IPs) communicate with resources outside the cluster by substituting the pod's IP with the node's IP.

### UDR — User Defined Routes
A **UDR (User Defined Route)** is a custom routing rule you add to a route table in Azure to override the default routing behavior. For example, you might say: "All traffic destined for `10.244.2.0/24` should go through node `10.224.0.5`."

Kubenet uses UDRs extensively to route pod traffic across nodes.

### Pod
A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more containers and gives them a shared network environment. Each pod gets its own IP address. When a pod is deleted and recreated, it may get a different IP.

### Service
A **Kubernetes Service** is a stable virtual IP (called a **Cluster IP**) that load balances traffic across a group of pods. Services always get IPs from the **Service CIDR** range.

### kube-proxy
**kube-proxy** runs on every node and manages network rules (using iptables or IPVS) to implement Kubernetes Services. When you access a Service's ClusterIP, kube-proxy ensures the traffic gets forwarded to an actual pod.

### Node
A **Node** is a worker virtual machine in the AKS cluster. Each node hosts multiple pods. Nodes get their IPs from the **Azure VNet subnet**.

---

## 3. What is a Network Plugin (CNI)?

**CNI** stands for **Container Network Interface** — it is a standardized specification that defines how networking plugins should configure network connectivity for containers.

When you create an AKS cluster, Kubernetes installs a CNI plugin on every node. This plugin is called every time a pod is created or deleted — it's responsible for:
1. Assigning an IP address to the new pod
2. Setting up the pod's network interface
3. Adding routing rules so other pods can find this pod
4. Removing all of the above when the pod is deleted

Different CNI plugins have different strategies for IP assignment and routing. AKS lets you choose among several options.

---

## 4. Overview of AKS Network Plugin Options

```
┌────────────────────────────────────────────────────────────────────┐
│                    AKS Network Plugin Options                        │
├──────────────────────────────────────────────────────────────────────┤
│  Kubenet (Basic)       — Default. Pods get internal IPs, NAT to nodes│
│  Azure CNI (Flat)      — Pods get real VNet IPs. Rich integration   │
│  Azure CNI Overlay     — Best of both. Pods internal, no IP exhaust │
│  Cilium                — eBPF-powered. High perf, advanced policies  │
│  BYO (Calico, etc.)    — Bring your own 3rd party CNI               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Option 1 — Kubenet (Basic Networking)

### What is Kubenet?
**Kubenet** is the default and simplest networking plugin for AKS. It was designed to conserve IP address space by giving pods **internal-only IP addresses** that are not routable on the Azure VNet.

### How Kubenet Works — Step by Step

```
Azure VNet Subnet (10.224.0.0/16)
├── Node 1: IP = 10.224.0.5  (from VNet subnet)
│     ├── Pod A: IP = 10.244.0.2  (from Pod CIDR, internal only)
│     ├── Pod B: IP = 10.244.0.3
│     └── Pod C: IP = 10.244.0.4
│
├── Node 2: IP = 10.224.0.6
│     ├── Pod D: IP = 10.244.1.2  (from Pod CIDR, internal only)
│     └── Pod E: IP = 10.244.1.3
│
└── Node 3: IP = 10.224.0.4
      └── Pod F: IP = 10.244.2.2
```

- **Nodes** get IPs from the **Azure VNet Subnet** (real, routable Azure IPs)
- **Pods** get IPs from the **Pod CIDR** (e.g., `10.244.0.0/16`) — this is an internal Kubernetes range, NOT part of the Azure VNet

### How Do Pods Talk to Each Other?

Since pods have internal IPs that Azure doesn't know about, Kubernetes needs a way to route traffic between pods on different nodes.

**Kubenet uses two mechanisms:**

#### 1. User Defined Routes (UDR) in the Route Table
AKS automatically creates a **Route Table** and adds entries like:
```
Destination    Next Hop Type    Next Hop IP
10.244.0.0/24  Virtual Appliance  10.224.0.5  ← "Node 1 owns these pod IPs"
10.244.1.0/24  Virtual Appliance  10.224.0.6  ← "Node 2 owns these pod IPs"
10.244.2.0/24  Virtual Appliance  10.224.0.4  ← "Node 3 owns these pod IPs"
```

When Pod D (on Node 2) wants to reach Pod A (on Node 1), Azure routing looks up the table, sees `10.244.0.x → go to 10.224.0.5`, and sends the packet to Node 1. Node 1 then delivers it to Pod A.

#### 2. NAT (Network Address Translation)
When a pod sends traffic **outside** the cluster (e.g., to the internet), its source IP (`10.244.0.2`) is **replaced** with the node's IP (`10.224.0.5`). This is because external resources only know about VNet IPs, not pod IPs.

### Each Node Gets a /24 from the Pod CIDR
Each node is assigned a `/24` block from the Pod CIDR:
- Node 1: `10.244.0.0/24` (hosts pods with IPs .2 through .254)
- Node 2: `10.244.1.0/24`
- Node 3: `10.244.2.0/24`

A `/24` gives 256 addresses (minus reserved ones), so up to **~250 pods per node**.

### What About Services?
Kubernetes Services (ClusterIP) get IPs from the **Service CIDR** (e.g., `10.0.0.0/16`). This is the same in all plugins — Kubenet and CNI alike.

### How to Create a Kubenet Cluster

```bash
# Create resource group
az group create --name rg-aks-kubenet --location westeurope

# Create cluster with Kubenet
az aks create -g rg-aks-kubenet -n aks-kubenet --network-plugin kubenet

# Verify node IPs come from VNet subnet
kubectl get nodes -o wide
# NAME                    INTERNAL-IP    ...
# aks-nodepool1-xxx-0     10.224.0.5
# aks-nodepool1-xxx-1     10.224.0.6

# Verify pod IPs come from Pod CIDR
kubectl get pods -A -o wide
# coredns-xxx   10.244.0.8  ← from Pod CIDR
# nginx-xxx     10.244.1.4  ← from Pod CIDR
```

### Resources Created

| Resource | Type | Purpose |
|----------|------|---------|
| `aks-agentpool-<id>-routetable` | **Route Table** | Custom routes mapping pod CIDRs to node IPs |
| `aks-agentpool-<id>-nsg` | Network Security Group | Firewall rules for nodes |
| `aks-vnet-<id>` | Virtual Network | The network for nodes |

### Kubenet Limitations

| Limitation | Detail |
|-----------|--------|
| **Max cluster size** | 400 nodes (Azure supports max 400 UDR routes) |
| **Windows Nodepools** | ❌ Not supported |
| **Azure Virtual Nodes** | ❌ Not supported |
| **Azure Network Policy** | ❌ Not supported |
| **Calico Network Policy** | ✅ Supported |
| **Additional latency** | Small — packets must make an extra routing hop through UDRs |
| **One cluster per subnet** | ❌ Cannot share a subnet with another AKS cluster |

### Pros and Cons of Kubenet

**✅ Advantages**
- Very IP-efficient: only nodes consume VNet IPs; pods use internal IPs
- Great for IP-constrained environments
- Simpler setup — Azure manages the VNet automatically
- Good for clusters where most pod communication is internal

**❌ Disadvantages**
- Pod IPs are not visible from outside the cluster (need NAT for external access)
- Requires UDRs — if another system manages your route tables, conflicts can occur
- Limited scale (400 nodes)
- Doesn't support Windows, Virtual Nodes, or Azure Network Policy

---

## 6. Option 2 — Azure CNI (Advanced Networking)

### What is Azure CNI?
**Azure CNI** (Container Networking Interface) is the "flat" or "traditional" networking model where every pod gets a **real IP address from the Azure VNet subnet**. Pods are first-class network citizens — they can be directly reached from your VNet, your on-premises network, or other connected VNets without NAT.

### How Azure CNI Works

```
Azure VNet Subnet (10.224.0.0/16)
├── Node 1: IP = 10.224.0.62  (from VNet subnet)
│     ├── Pod A: IP = 10.224.0.10  (ALSO from VNet subnet!)
│     ├── Pod B: IP = 10.224.0.11
│     └── ...up to 30 pods pre-reserved
│
├── Node 2: IP = 10.224.0.33
│     ├── Pod C: IP = 10.224.0.70
│     └── ...up to 30 pods pre-reserved
│
└── Node 3: IP = 10.224.0.4
      └── ...up to 30 pods pre-reserved
```

- **Nodes AND Pods** all get IPs from the **same Azure VNet subnet**
- **No NAT** is required for pod-to-VNet communication
- **No route tables** are needed for pod-to-pod routing
- Every pod is directly addressable from outside the cluster

### IP Reservation — The Critical Detail
When a node joins the cluster, Azure CNI **pre-reserves** IP addresses for the maximum number of pods that node can host (default: 30 per node).

For example, if you have 3 nodes with 30 pods max each:
```
Total IPs reserved = 3 nodes × (1 node IP + 30 pod IPs) = 3 × 31 = 93 IPs
```
These IPs are reserved **even before any pods are actually scheduled**. This ensures fast pod startup but wastes IPs if pods are sparse.

### No Pod CIDR in Azure CNI
Notice the networking panel shows:
```
Pod CIDR:    -   (empty — no separate pod network!)
Service CIDR: 10.0.0.0/16
```
Because pods use real VNet IPs, there's no need for a separate pod CIDR range.

### How to Create an Azure CNI Cluster

```bash
# Create resource group
az group create --name rg-aks-cni --location westeurope

# Create cluster with Azure CNI
az aks create -g rg-aks-cni -n aks-cni --network-plugin azure

# Verify node and pod IPs are from the same subnet
kubectl get nodes -o wide
# NAME                INTERNAL-IP
# aks-nodepool-xxx-0  10.224.0.62
# aks-nodepool-xxx-1  10.224.0.33

kubectl get pods -A -o wide
# coredns-xxx         10.224.0.67  ← SAME VNet subnet range as nodes!
# csi-azuredisk-xxx   10.224.0.33  ← same as node IP (system pods can share)
```

### The IP Exhaustion Problem

This is the **biggest drawback** of traditional Azure CNI.

**Example**: You have a subnet with `/24` (256 IPs). First 3 reserved by Azure = 253 usable.
- With **max 30 pods per node**: `253 ÷ 31 = 8 nodes`, each with 30 pods = 240 pods
- After 8 nodes, you run out of IPs — you can't add more nodes

Compare with Kubenet on the same subnet:
- Only nodes consume subnet IPs: 253 node IPs possible = 253 nodes × 110 pods = 27,830 pods

This disparity is why **Azure CNI Overlay** was created.

### Azure CNI Features vs Kubenet

| Feature | Azure CNI |
|---------|-----------|
| Windows Nodepools | ✅ Yes |
| Azure Virtual Nodes | ✅ Yes |
| Azure Network Policy | ✅ Yes |
| Calico Network Policy | ✅ Yes |
| Pod-to-VNet routing | ✅ Direct (no NAT) |
| Max cluster size | 1000+ nodes |

### Pros and Cons of Azure CNI

**✅ Advantages**
- Pods are directly routable in your VNet — no NAT
- Supports all AKS features (Windows, Virtual Nodes, AGIC, Azure Network Policy)
- Better observability — pod IPs visible in network traces
- Performance: no additional hop, no NAT overhead

**❌ Disadvantages**
- **IP exhaustion**: Heavy IP consumption per node
- Requires careful IP address planning upfront
- May need to rebuild cluster in a larger subnet as you grow

---

## 7. Option 3 — Azure CNI Overlay

### What is Azure CNI Overlay?
**Azure CNI Overlay** is a newer networking model that combines the best of both worlds:
- **Like Kubenet**: Pods get IPs from a separate internal CIDR (not the VNet subnet) → conserves VNet IPs
- **Like Azure CNI**: No complex UDRs or route tables needed → simpler routing, better performance
- **Better than Kubenet**: Scales to 1000 nodes (not just 400), and Windows is supported

Microsoft now recommends Azure CNI Overlay for **all new AKS clusters** (as of 2025).

### How Azure CNI Overlay Works

```
Azure VNet Subnet (10.224.0.0/16) — only nodes here
├── Node 1: IP = 10.224.0.4  (from VNet)
│     Pod CIDR block: 192.168.0.0/24
│     ├── Pod A: IP = 192.168.0.2  (internal, not in VNet)
│     └── Pod B: IP = 192.168.0.3
│
├── Node 2: IP = 10.224.0.5  (from VNet)
│     Pod CIDR block: 192.168.1.0/24
│     └── Pod C: IP = 192.168.1.2
│
└── Node 3: IP = 10.224.0.6  (from VNet)
      Pod CIDR block: 192.168.2.0/24
      └── Pod D: IP = 192.168.2.2
```

### Key Mechanics
- **Nodes** get IPs from the VNet subnet (real Azure IPs)
- **Pods** get IPs from a private CIDR you specify (e.g., `192.168.0.0/16`)
- Each node gets a **/24 block** from the pod CIDR (e.g., Node 1 = `192.168.0.0/24` → up to 250 pods)
- Pod-to-pod traffic within the cluster stays in the overlay network — **no UDRs needed**
- Pod traffic going outside the cluster is **NAT'd** to the node's IP (same as Kubenet)
- **Endpoints outside the cluster cannot directly reach pods** (must go through a Service)
- The **same pod CIDR can be reused** across multiple AKS clusters in the same VNet (since pod IPs are internal and don't leak into the VNet)

### How to Create an Azure CNI Overlay Cluster

```bash
# Create resource group
az group create -n rg-aks-cni-overlay -l westeurope

# Create cluster with Azure CNI Overlay
az aks create -n aks-cni-overlay -g rg-aks-cni-overlay \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16

# Verify: Node IPs from VNet subnet, Pod IPs from Pod CIDR
kubectl get nodes -o wide
# INTERNAL-IP: 10.224.0.4, 10.224.0.5, 10.224.0.6  ← VNet IPs

kubectl get pods -o wide
# nginx-xxx    192.168.1.143  ← from Pod CIDR!
# coredns-xxx  192.168.0.33   ← from Pod CIDR!
```

### Max Pods Per Node
With Azure CNI Overlay, each node gets a `/24` block = **250 pods per node** (compared to 30 with traditional Azure CNI). The max is configurable up to 250.

### Scalability
| Plugin | Max Nodes | Max Pods/Node |
|--------|-----------|---------------|
| Kubenet | 400 | 110 |
| Azure CNI | 1000+ | 30 (default) |
| Azure CNI Overlay | 1000 | 250 |

### Overlay Networking — How Pods Talk Across Nodes
Unlike Kubenet (which uses UDRs), Azure CNI Overlay uses the **Azure network fabric's built-in routing** but wraps pod traffic in an overlay. Specifically:
- Traffic between pods on the **same node** stays local
- Traffic between pods on **different nodes** is routed by Azure's infrastructure using the node IP as the delivery target, and the overlay layer delivers to the specific pod
- **No custom route tables needed in the VNet** — Azure handles it natively

This gives **performance on par with VM-to-VM communication** — much better than Kubenet's extra hop.

### Azure CNI Overlay Limitations
- Cannot use **AGIC (Application Gateway Ingress Controller)** — though AGIC support was added in April 2025 as a preview
- Windows Server 2019 node pools are not supported (Windows Server 2022 is fine)
- External clients cannot directly connect to a pod by its pod IP

### Pros and Cons

**✅ Advantages**
- Best of both worlds: IP-efficient like Kubenet, fast like Azure CNI
- No IP exhaustion — pods use private CIDR, not VNet IPs
- Scales to 1000 nodes with 250 pods per node
- No complex UDRs to manage
- Same pod CIDR can be reused across multiple clusters
- Supports Windows Server 2022, Calico, Cilium, Azure Network Policy

**❌ Disadvantages**
- Pod IPs are not directly routable from outside the cluster (need Services)
- Pod IPs are NAT'd when going outside → harder to trace traffic in logs
- Cannot use AGIC (traditional) — use App Gateway for Containers instead

---

## 8. Option 4 — Cilium

### What is Cilium?
**Cilium** is a high-performance, open-source CNI plugin based on **eBPF** (extended Berkeley Packet Filter) — a Linux kernel technology that allows programs to run inside the kernel without changing kernel code. This makes Cilium extremely fast and capable of enforcing complex security policies.

In AKS, Cilium is available as:
- **Azure CNI Powered by Cilium** — uses Cilium for the dataplane (packet processing) while Azure CNI manages IP allocation
- **Cilium-only (BYO)** — you install Cilium yourself

### What eBPF Means for You
Instead of using iptables (which is slow for large clusters), Cilium processes network packets **directly in the Linux kernel** using eBPF programs. This means:
- Faster network processing
- Better observability (you can see exactly what's happening to every packet)
- More powerful network policies (can enforce Layer 7 policies like HTTP path-based rules)

### When to Use Cilium
- Very large clusters (thousands of nodes)
- You need advanced network security policies (L7 policies, DNS-aware policies)
- You want built-in observability with **Hubble** (Cilium's network observability tool)
- Latency-sensitive workloads that need the fastest possible networking

### Limitations
- Linux-only (no Windows nodes)
- Some Kubernetes features not yet supported (e.g., `internalTrafficPolicy=Local`)

---

## 9. Option 5 — Bring Your Own CNI (BYO)

AKS allows you to bring your own CNI plugin if none of the built-in options meet your needs. Common choices:
- **Calico** (widely used, supports both Kubenet and Azure CNI, good network policies)
- **Canal** (Flannel + Calico policies)
- **Flannel** (simple overlay)
- **Weave** (simple mesh networking)

BYO CNI is for advanced users who have specific requirements not met by the built-in options.

---

## 10. IP Address Planning — How Many IPs Do You Need?

This is one of the most important planning steps. Getting it wrong means rebuilding your cluster.

### IP Consumption by Plugin

| Plugin | Node IPs (from VNet) | Pod IPs (from VNet) | Total VNet IPs per Node |
|--------|---------------------|---------------------|------------------------|
| Kubenet | 1 per node | 0 (from Pod CIDR) | **1** |
| Azure CNI | 1 per node | 30 per node (default) | **31** |
| Azure CNI Overlay | 1 per node | 0 (from Pod CIDR) | **1** |

### Example: /24 Subnet (256 IPs, 253 usable after 3 reserved by Azure)

| Plugin | Nodes Possible | Pods Possible |
|--------|---------------|---------------|
| **Kubenet** | 251 nodes | 251 × 110 = **27,610 pods** |
| **Azure CNI** | 253 ÷ 31 = **8 nodes** | 8 × 30 = **240 pods** |
| **Azure CNI Overlay** | 251 nodes | 251 × 250 = **62,750 pods** |

This is a dramatic difference. With a modest `/24` subnet, Azure CNI can only support 8 nodes, while Kubenet and CNI Overlay can support 251!

### Reserved IPs in a Subnet
Azure reserves the **first 5 IP addresses** in every subnet for internal use:
- `.0` — Network address
- `.1` — Reserved by Azure for the default gateway
- `.2`, `.3` — Reserved by Azure for Azure DNS
- `.255` — Broadcast address

So a `/24` subnet (256 IPs) has **251 usable IPs**.

### Pod CIDR Sizing (Kubenet / CNI Overlay)

Each node gets a `/24` from your Pod CIDR. So:
- `10.244.0.0/16` Pod CIDR = 256 possible `/24` blocks = **256 nodes** maximum

If you want more nodes, use a larger Pod CIDR:
- `10.244.0.0/14` = 1024 nodes

### Service CIDR Sizing
The Service CIDR only needs to be large enough for the number of Kubernetes Services you plan to create. A `/16` (65,536 addresses) is usually more than enough for most clusters.

---

## 11. CIDR Ranges and Avoiding Conflicts

### The Golden Rule
All CIDR ranges used in your cluster **must not overlap** with each other, or with any network they connect to.

### Required Non-Overlapping Ranges
You need separate, non-overlapping CIDRs for:
1. **VNet** (e.g., `10.0.0.0/8`)
2. **Node Subnet** (e.g., `10.224.0.0/16`) — must be within the VNet
3. **Pod CIDR** (Kubenet / CNI Overlay only) — must be separate from VNet and connected networks
4. **Service CIDR** — must be separate from all of the above
5. **Docker Bridge CIDR** — legacy, being phased out

### Forbidden Ranges in AKS
AKS clusters **cannot use** these ranges for Service CIDR, Pod CIDR, or VNet:
- `169.254.0.0/16` — Link-local addresses (used internally by Azure)
- `172.30.0.0/16` — Used by Docker
- `172.31.0.0/16` — Used by Docker
- `192.0.2.0/24` — Reserved for documentation

### Important: These Cannot Be Changed After Creation
Once you create an AKS cluster, you **cannot change** the Pod CIDR, Service CIDR, or VNet configuration. Plan carefully before creating.

### Example Full Configuration

```bash
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --network-plugin kubenet \
  --service-cidr 10.0.0.0/16 \        # For Kubernetes Services
  --dns-service-ip 10.0.0.10 \         # Must be within service-cidr
  --pod-cidr 10.244.0.0/16 \           # For Pods (Kubenet/CNI Overlay)
  --vnet-subnet-id $SUBNET_ID           # Where nodes get their IPs
```

---

## 12. Full Comparison Tables

### At a Glance

| Feature | Kubenet | Azure CNI | Azure CNI Overlay | Cilium |
|---------|---------|-----------|-------------------|--------|
| **Node IPs from** | VNet Subnet | VNet Subnet | VNet Subnet | VNet Subnet |
| **Pod IPs from** | Pod CIDR (internal) | VNet Subnet | Pod CIDR (internal) | VNet Subnet or CIDR |
| **Pod directly routable from VNet?** | ❌ No | ✅ Yes | ❌ No | Depends |
| **Route Table needed?** | ✅ Yes (UDRs) | ❌ No | ❌ No | ❌ No |
| **NAT for external traffic?** | ✅ Yes | ✅ Yes (external) | ✅ Yes | Depends |
| **Max nodes** | 400 | 1000+ | 1000 | 1000+ |
| **Max pods/node** | 110 | 30 (default) | 250 | 250 |
| **Windows support** | ❌ No | ✅ Yes | ✅ (2022 only) | ❌ No |
| **Azure Network Policy** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **Calico Network Policy** | ✅ Yes | ✅ Yes | ✅ Yes | N/A |
| **Azure Virtual Nodes** | ❌ No | ✅ Yes | ❌ No | ❌ No |
| **AGIC support** | ❌ No | ✅ Yes | ✅ Yes (2025+) | ❌ No |
| **IP efficiency** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Performance** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Operational complexity** | Medium | Low-Medium | Low | Medium-High |

### Kubenet vs Azure CNI Overlay (Detailed)

| Area | Azure CNI Overlay | Kubenet |
|------|-------------------|---------|
| Cluster scale | 1000 nodes, 250 pods/node | 400 nodes, 110 pods/node |
| Network configuration | Simple — no extra routes for pods | Complex — requires UDRs on cluster subnet |
| Pod connectivity performance | On par with VMs in VNet | Additional hop adds minor latency |
| Network Policies | Azure Network Policies, Calico, Cilium | Calico only |
| OS Platforms supported | Linux + Windows Server 2022 | Linux only |

---

## 13. How to Choose — Decision Guide

### Step-by-Step Decision Tree

```
Are you creating a NEW cluster?
│
├─► YES
│     │
│     ├─► Do you need Windows node pools?
│     │     ├─► YES → Azure CNI Overlay (or Azure CNI if you need AGIC)
│     │     └─► NO  → Continue below
│     │
│     ├─► Do you have limited VNet IP space?
│     │     ├─► YES → Azure CNI Overlay (pods use internal IPs)
│     │     └─► NO  → Continue below
│     │
│     ├─► Do you need pods to be directly reachable from VNet (no NAT)?
│     │     ├─► YES → Azure CNI (flat, traditional)
│     │     └─► NO  → Azure CNI Overlay (recommended for most cases)
│     │
│     └─► Do you need advanced L7 network policies or huge scale?
│           └─► YES → Cilium
│
└─► NO (migrating existing cluster)
      └─► Generally, you must create a new cluster to change CNI.
          Plan your migration carefully.
```

### Use Kubenet When:
- You have very limited IP address space
- Most pod communication is **within the cluster** (internal)
- You don't need Azure Network Policy, Windows nodes, or Virtual Nodes
- You want the cheapest/simplest setup for dev/test
- ⚠️ Note: Microsoft considers Kubenet **legacy** for new clusters as of 2025

### Use Azure CNI (Traditional) When:
- You have **abundant IP address space** in your VNet
- Pods must be **directly reachable** from outside the cluster (on-premises, peered VNets)
- You need **Windows nodes**, Virtual Nodes, or AGIC
- You need pods to be visible with their real IPs in network traces

### Use Azure CNI Overlay When (Recommended for most new clusters):
- You want **simplicity + scale** without IP exhaustion
- You need Windows Server 2022 support
- Large clusters (hundreds of nodes)
- You want good performance without the complexity of Kubenet's UDRs
- This is **Microsoft's recommended default** for new clusters as of 2025

### Use Cilium When:
- Ultra-high performance networking
- You need **Layer 7 network policies** (HTTP path-based rules)
- Large-scale clusters with advanced observability requirements
- You're comfortable with eBPF-based networking

---

## Summary: The 3-Question Cheat Sheet

| Question | Answer → Plugin |
|----------|----------------|
| IP space is tight, simple cluster | → Kubenet (legacy) or CNI Overlay |
| Pods must be VNet-routable | → Azure CNI (traditional) |
| Best all-around for new clusters | → **Azure CNI Overlay** ✅ |
| Maximum performance + policies | → Cilium |

---

## References & Further Reading

- [AKS Networking Concepts](https://learn.microsoft.com/en-us/azure/aks/concepts-network)
- [Azure CNI Overlay](https://learn.microsoft.com/en-us/azure/aks/concepts-network-azure-cni-overlay)
- [Kubenet documentation](https://learn.microsoft.com/en-us/azure/aks/configure-kubenet)
- [Azure CNI documentation](https://learn.microsoft.com/en-us/azure/aks/configure-azure-cni)
- [Cilium in AKS](https://learn.microsoft.com/en-us/azure/aks/azure-cni-powered-by-cilium)
- [Choose a CNI plugin](https://learn.microsoft.com/en-us/azure/aks/concepts-network#compare-network-models)