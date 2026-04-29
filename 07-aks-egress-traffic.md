# 📘 Doc 07 — AKS Egress Traffic: Complete Beginner-to-Expert Guide

> **Who this is for:** Someone learning AKS networking from scratch. Every concept, resource, command, and decision is explained fully — with zero assumed knowledge.

---

## 🧭 Table of Contents

1. [What Is Egress Traffic?](#1-what-is-egress-traffic)
2. [Why AKS Needs to Manage Egress](#2-why-aks-needs-to-manage-egress)
3. [Kubernetes Networking Foundations](#3-kubernetes-networking-foundations)
4. [What is `outboundType`?](#4-what-is-outboundtype)
5. [OutboundType: LoadBalancer (Default)](#5-outboundtype-loadbalancer-default)
6. [Every Azure Resource Explained](#6-every-azure-resource-explained)
7. [SNAT — The Core Concept](#7-snat--the-core-concept)
8. [SNAT Port Exhaustion — The Real Problem](#8-snat-port-exhaustion--the-real-problem)
9. [Solution 1: More Outbound Public IPs](#9-solution-1-more-outbound-public-ips)
10. [What is Azure NAT Gateway?](#10-what-is-azure-nat-gateway)
11. [OutboundType: ManagedNATGateway](#11-outboundtype-managednatgateway)
12. [OutboundType: UserAssignedNATGateway](#12-outboundtype-userassignednatgateway)
13. [OutboundType: ManagedNATGatewayV2 (Preview)](#13-outboundtype-managednatgatewayv2-preview)
14. [OutboundType: UserDefinedRouting](#14-outboundtype-userdefinedrouting)
15. [Ingress vs Egress — The Interaction](#15-ingress-vs-egress--the-interaction)
16. [Asymmetric Routing Problem](#16-asymmetric-routing-problem)
17. [Decision Guide: Which OutboundType to Use](#17-decision-guide-which-outboundtype-to-use)
18. [Quick Reference Commands](#18-quick-reference-commands)

---

## 1. What Is Egress Traffic?

In computer networking, two words describe the direction of traffic:

- **Ingress** = traffic coming **INTO** your cluster (a user hitting your website)
- **Egress** = traffic going **OUT OF** your cluster (your app calling an external API)

Every time a pod in AKS does any of the following, it creates **egress traffic**:

| Pod Action | Where Traffic Goes |
|-----------|-------------------|
| Pulling a Docker image | Docker Hub / MCR / ACR |
| Calling a payment API | `api.stripe.com`, `api.paypal.com` |
| Sending telemetry / logs | Datadog, Splunk, Azure Monitor |
| Fetching secrets | HashiCorp Vault, Azure Key Vault |
| Downloading OS patches | Ubuntu/CentOS package repos |
| Talking to AKS control plane | Azure API Server |
| Querying an external database | Azure SQL, Cosmos DB, MongoDB Atlas |

All of this is **outbound traffic**. The internet needs a public IP to deliver responses back to your pod — and since pods only have private IPs, Azure must perform an address translation. That translation mechanism is what `outboundType` configures.

---

## 2. Why AKS Needs to Manage Egress

Azure VMs (and AKS nodes are Azure VMs) only have **private IP addresses** from within your VNet (e.g., `172.16.0.4`). Private IPs are not routable on the public internet.

When a pod with IP `10.244.0.5` wants to call `https://api.github.com`:
- GitHub cannot deliver a response to `10.244.0.5` — that IP doesn't exist on the internet
- The packet must be **translated** to have a public, routable source IP before it leaves Azure

Additionally, organizations have security requirements:
- **Audit:** Log every outbound connection for compliance
- **Filter:** Only allow pods to reach approved external services
- **IP Allowlisting:** Guarantee that your pods always appear to come from a known, static IP so partners can whitelist you
- **Scale:** Thousands of pods making thousands of simultaneous connections without dropping

These are precisely the problems that `outboundType` options solve — each in a different way.

---

## 3. Kubernetes Networking Foundations

Before understanding AKS egress, you need three foundational concepts:

### Pods Have Private IPs

Every Kubernetes pod receives its own IP address (e.g., `10.244.1.5`). This is:
- Unique within the cluster
- Reachable by other pods within the cluster
- **NOT reachable from the internet** — it's a private address

### Nodes Are Azure VMs

Each Kubernetes worker node is an Azure VM with:
- A **private IP** within the Azure VNet (e.g., `172.16.0.4`)
- A **NIC (Network Interface Card)** — the hardware-level connection to the VNet
- No public IP by default (public IPs on nodes are a security risk)

### The Packet Journey Outbound

When a pod makes an HTTP request:
```
Pod (10.244.1.5) 
  → Linux network bridge on node 
    → Node NIC (172.16.0.4) 
      → Azure VNet 
        → Egress mechanism (LB / NAT GW / Firewall)
          → Public Internet
```

The "egress mechanism" is what `outboundType` configures.

---

## 4. What is `outboundType`?

`--outbound-type` is an AKS cluster parameter that controls **how outbound internet traffic exits the cluster**.

```bash
az aks create \
  --resource-group my-rg \
  --name my-cluster \
  --outbound-type loadBalancer   # ← This controls egress
```

### All Supported Values (2025/2026)

| Value | Who manages it | VNet requirement | Status |
|-------|---------------|-----------------|--------|
| `loadBalancer` | AKS | AKS-managed or BYO | GA (default) |
| `managedNATGateway` | AKS | AKS-managed VNet ONLY | GA |
| `managedNATGatewayV2` | AKS | AKS-managed VNet ONLY | Preview |
| `userAssignedNATGateway` | You | BYO VNet required | GA |
| `userDefinedRouting` | You | BYO VNet required | GA |
| `none` | N/A | Network Isolated Clusters only | GA |

> **Critical understanding:** `outboundType` affects ONLY **egress**. It has ZERO effect on how users reach your services (ingress). Those are controlled by Kubernetes Services, Ingress Controllers, and Application Gateway.

---

## 5. OutboundType: LoadBalancer (Default)

### Overview

The default AKS egress mechanism. An **Azure Standard Load Balancer** performs SNAT to translate your pods' private IPs to a public IP before they reach the internet.

```
┌────────────────────────────────────────────────────────────┐
│                     aks-egress-vnet                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   aks-subnet                         │  │
│  │  ┌─────────────────────────────────────────────┐    │  │
│  │  │             AKS Cluster                      │    │  │
│  │  │  Node1  Node2  Node3                         │    │  │
│  │  │  [Pod]  [Pod]  [Pod]                         │────┼──┼──► LB (SNAT) ──► 20.x.x.x ──► Internet
│  │  └─────────────────────────────────────────────┘    │  │  (Egress)
│  │                  NSG                                 │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Creating a Cluster (LoadBalancer is the default)

```bash
az aks create \
  --resource-group $AKS_RG \
  --name $AKS_NAME \
  --enable-managed-identity \
  --outbound-type loadBalancer
# Note: --outbound-type loadBalancer is optional; it's the default
```

### Proving Egress Works

```bash
# Run a pod
kubectl run nginx --image=nginx
# pod/nginx created

# Get a shell and check the outbound IP
kubectl exec nginx -it -- /bin/bash
curl ifconfig.me
# Output: 20.126.14.246  ← This is the Load Balancer's public IP
```

### Creating a Public Service Adds Another IP

When you expose a deployment publicly:
```bash
kubectl expose deployment nginx --name nginx --port=80 --type LoadBalancer

kubectl get svc
# NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
# kubernetes   ClusterIP      10.0.0.1       <none>          443/TCP        10h
# nginx        LoadBalancer   10.0.106.59    20.31.208.171   80:31371/TCP   9s
```

AKS adds a **second public IP** (`20.31.208.171`) as another frontend on the **same** `kubernetes` Load Balancer. Now the LB has:
- `20.126.14.246` → used for egress (all pods going out)
- `20.31.208.171` → used for ingress (users reaching the nginx service)

One Load Balancer, two roles, two separate IPs.

---

## 6. Every Azure Resource Explained

When AKS provisions a cluster, it creates these resources in the node resource group (`MC_<rg>_<name>_<region>`). Here is every single one explained:

---

### 🔄 Azure Standard Load Balancer (`kubernetes`)

**What it is at a fundamental level:**  
A Layer 4 (TCP/UDP) network appliance that can distribute incoming connections across multiple backends AND provide outbound NAT for backend VMs.

**Why AKS creates it:**  
It's the primary egress mechanism in the default configuration. Every pod's outbound connection passes through it. It also becomes the entry point for Kubernetes `LoadBalancer` type services.

**How it actually works:**
1. AKS registers all worker nodes in the Load Balancer's **backend pool**
2. AKS configures **outbound rules** that map the backend pool to a public IP
3. When a node initiates an outbound connection, the LB intercepts it, performs SNAT, and forwards it
4. The LB records the mapping: `(node private IP, ephemeral port)` ↔ `(LB public IP, SNAT port)`
5. When a response arrives, the LB reverses the mapping and delivers it to the correct node

**Important constraint:** Standard LB is required. Basic LB does not support zone redundancy or the features AKS needs.

---

### 🌐 Public IP Address (GUID-named)

**What it is:**  
A static IPv4 address that Azure assigns from its global address pool. It's routable anywhere on the internet.

**Why AKS creates it:**  
The Load Balancer needs at least one public IP to perform outbound SNAT. This IP is what external services see when your pods make requests.

**Why Standard SKU matters:**  
Standard Public IPs support:
- Zone-redundancy (survives one AZ failure)
- Attachment to Standard Load Balancers and NAT Gateways
- Inbound flow idle timeout configuration

Basic Public IPs are being deprecated and cannot be used with modern networking features.

**Real-world use:**  
If you integrate with a third-party API, you give them `20.126.14.246` and they add it to their allowlist. All traffic from your AKS cluster appears to come from this IP.

---

### 🔒 Network Security Group (NSG)

**What it is:**  
A stateful, distributed firewall operating at the Azure network level. It contains **security rules** (priority-ordered list of allow/deny) for inbound and outbound traffic.

**Why AKS creates it:**  
To protect the AKS subnet. By default, you don't want arbitrary internet traffic reaching your nodes directly. The NSG acts as the perimeter.

**Default rules AKS sets:**
- Allow inbound from Azure Load Balancer (for health probes)
- Allow inbound from VNet (for node-to-node, pod-to-pod communication)
- Allow all outbound to internet (pods must be able to reach external services)
- Deny all other inbound from internet

**Why you should care:**  
If you add overly restrictive NSG rules, you can accidentally break AKS functionality — for example, blocking the port AKS uses to pull container images will cause `ErrImagePull` on all pods. Always test NSG changes carefully.

---

### 🗺️ Route Table

**What it is:**  
A set of routing rules that override Azure's default system routes for a specific subnet. Each entry says "traffic going to destination X should go to next hop Y."

**Why AKS creates it:**  
In **kubenet** networking mode, AKS must tell Azure how to route pod-to-pod traffic across different nodes (each node has its own pod CIDR). The route table has one entry per node: "pods on `10.244.X.0/24` → go to node VM IP `172.16.0.X`."

**In UDR mode, YOU add to it:**  
You add a route `0.0.0.0/0 → Firewall private IP` to force all internet traffic through the firewall. This route overrides Azure's default internet route.

**Critical gotcha with UDR mode:**  
AKS validates that you don't accidentally create a `0.0.0.0/0 → Internet` route in UDR mode (because that would bypass the firewall). It enforces that 0.0.0.0/0 must point to an NVA/appliance, not directly to Internet.

---

### 🖥️ Virtual Machine Scale Set (VMSS)

**What it is:**  
An Azure resource that manages a collection of identical VMs with shared configuration — your Kubernetes **worker nodes**.

**Why AKS creates it (instead of individual VMs):**  
- **Auto-scaling:** Add or remove nodes by changing the count. Azure provisions/deprovisions VMs automatically.
- **Uniform configuration:** All nodes have the same OS image, disk size, VM size, and network settings.
- **Rolling upgrades:** Update nodes one by one without downtime.
- **Integration with Azure services:** The VMSS is registered as the backend pool of the Load Balancer automatically.

**Why this matters for networking:**  
When AKS applies a networking change (like switching outbound type), it updates the VMSS model, and all nodes receive the new configuration on next scale or upgrade operation. This is why some networking changes require a node image upgrade to take effect.

---

### 🌐 Virtual Network (VNet)

**What it is:**  
A logically isolated, private network in Azure. Think of it as your organization's private LAN, but in the cloud. You define:
- **Address space** (e.g., `10.0.0.0/16`) — the total IP range available
- **Subnets** — subdivisions of that space for different purposes
- **DNS settings** — what DNS servers resources in the VNet use
- **Peering** — connections to other VNets (in the same or different subscriptions/regions)

**Why AKS creates it:**  
All Azure resources (VMs, Load Balancers, NAT Gateways) must live inside a VNet. Without a VNet, there's no networking.

**BYO VNet vs AKS-managed VNet:**  
By default, AKS creates its own VNet (`aks-vnet-XXXXXXXX`). But in enterprise scenarios, you pre-create the VNet yourself with your own IP ranges and settings, then pass `--vnet-subnet-id` to AKS at cluster creation. This is called BYO (Bring Your Own) VNet and is required for hub & spoke and enterprise architectures.

---

### 📦 Subnet

**What it is:**  
A sub-segment of the VNet's IP address space dedicated to a specific resource group. For example, a VNet with address space `10.0.0.0/16` could have:
- Subnet A: `10.0.0.0/24` for AKS nodes
- Subnet B: `10.0.1.0/24` for a jump box VM
- Subnet C: `10.0.2.0/26` for the Azure Firewall

**Why subnets matter for egress:**  
A **NAT Gateway attaches at the subnet level**. ALL resources in a subnet share the NAT Gateway. A **Route Table also attaches at the subnet level**. This means you can have different subnets (for different node pools) with different egress paths.

For example:
- System node pool in Subnet A → NAT Gateway for egress
- GPU node pool in Subnet B → Different NAT Gateway (or firewall)

---

### 🔑 Managed Identity

**What it is:**  
An Azure Active Directory identity that represents the AKS control plane. It's like a service account that Azure manages automatically (no password to rotate, no credentials to store).

**Why AKS needs it:**  
AKS must make Azure API calls to:
- Create/modify Load Balancers and Public IPs
- Read VNet configuration
- Manage disk attachments for persistent volumes
- Read from Azure Container Registry (if configured)

Instead of hardcoding a username/password, AKS uses the Managed Identity. Azure automatically authenticates it.

**Two types:**
- **System-assigned:** Created and deleted with the cluster. Tightly coupled.
- **User-assigned:** Pre-created by you, can be shared across resources.

**Why this matters:**  
If the Managed Identity lacks the right permissions (e.g., "Network Contributor" on your VNet resource group), AKS cannot create Load Balancers or manage networking. This causes cluster provisioning failures or LoadBalancer service failures.

---

## 7. SNAT — The Core Concept

**SNAT = Source Network Address Translation**

It is the process of changing the **source IP address** of an outbound packet before forwarding it.

### Why SNAT Is Necessary

Pod IP: `10.244.1.5` (private)  
The internet doesn't know how to route to `10.244.1.5`.  
The Load Balancer has public IP: `20.126.14.246`

SNAT flow:
```
Pod sends:    SRC=10.244.1.5   DST=api.github.com
              ↓ SNAT at Load Balancer
LB forwards:  SRC=20.126.14.246:PORT_4567   DST=api.github.com
              ↓ GitHub responds to 20.126.14.246:PORT_4567
LB receives:  SRC=api.github.com   DST=20.126.14.246:PORT_4567
              ↓ Reverse mapping lookup
Pod receives: SRC=api.github.com   DST=10.244.1.5
```

The Load Balancer is essentially a stateful translator — it tracks every open connection.

### What Is a SNAT Port?

When many pods share one public IP, the IP alone isn't enough to distinguish connections. Consider:
- Pod A connects to GitHub: appears as `20.x.x.x → GitHub`
- Pod B also connects to GitHub: appears as `20.x.x.x → GitHub`

GitHub sends two responses to `20.x.x.x` — how does the Load Balancer know which one goes to Pod A and which to Pod B?

**Answer: SNAT ports.** The LB assigns a unique **port number** to each connection:
- Pod A → GitHub: LB uses `20.x.x.x:PORT_4567`
- Pod B → GitHub: LB uses `20.x.x.x:PORT_4568`

Now responses to port 4567 go to Pod A, and responses to port 4568 go to Pod B. The port is part of the unique "connection ID."

### SNAT Port Lifecycle

A SNAT port is:
1. **Allocated** when a new outbound connection opens
2. **Held** for the entire duration of the connection
3. **Held longer** during idle periods (connection open but no data flowing)
4. **Released** when the connection closes (TCP FIN/RST) OR when the idle timeout expires

During the time a port is held, **no other connection can use it**.

---

## 8. SNAT Port Exhaustion — The Real Problem

### How Azure Load Balancer Pre-Allocates Ports

With Azure Load Balancer (Standard), each public IP has ~64,000 SNAT ports. These are **pre-allocated** to each node in a fixed block before any connections are made.

**AKS uses a stepped allocation:**

| Backend pool size (nodes) | Ports pre-allocated per node |
|--------------------------|------------------------------|
| 1 – 50 nodes | 1,024 ports/node |
| 51 – 100 nodes | 512 ports/node |
| 101 – 200 nodes | 256 ports/node |
| 201 – 400 nodes | 128 ports/node |
| 401 – 800 nodes | 64 ports/node |
| 801 – 1000 nodes | 32 ports/node |

**Example:** 100-node cluster, 1 public IP → 512 ports per node × 100 nodes = 51,200 (less than the 64k the IP could support — some ports are reserved).

### The Exhaustion Scenario

Imagine Node 5 has 50 pods. Each pod makes 15 simultaneous calls to external payment services. That's:
```
50 pods × 15 connections = 750 concurrent connections
```

Node 5 only has 1,024 pre-allocated SNAT ports. 750 is within limits... until the load spikes to Black Friday levels:
```
50 pods × 25 connections = 1,250 concurrent connections
```

**Port exhaustion!** The 1,251st connection attempt fails. The pod sees a connection timeout.

Meanwhile, Node 3 (which runs batch jobs with rare external calls) has 900 unused ports. But those ports are **pre-allocated to Node 3** and **cannot be shared** with Node 5.

This is the fundamental flaw of pre-allocated SNAT: **wasted ports can't be redistributed.**

### How to Diagnose SNAT Exhaustion

**Azure Monitor query:**
- Navigate to your Load Balancer → Metrics
- Metric: `SNAT Connection Count`
- Split by: `Connection State = Failed`
- If you see spikes here, you have exhaustion

**Application symptoms:**
- Intermittent connection failures to external APIs
- Errors that get worse during high-traffic periods
- Errors that improve when you scale down pods
- Errors correlating with specific nodes (if one node is overloaded)

**kubectl indicators:**
```bash
# Look for pods stuck in crashloopbackoff calling external services
kubectl get pods --all-namespaces

# Check pod logs for connection refused / timeout patterns
kubectl logs <pod-name> --tail=100 | grep -i "timeout\|refused\|SNAT"
```

---

## 9. Solution 1: More Outbound Public IPs

The quickest fix: add more public IPs to the Load Balancer. Each additional IP adds another ~64,000 SNAT ports to the pre-allocated pool.

```bash
az aks update \
  --resource-group rg-aks-lb \
  --name aks-lb \
  --load-balancer-managed-outbound-ip-count 3
```

**Math:** 3 IPs × 64,000 ports = 192,000 total SNAT ports  
**Per-node (100 nodes):** 192,000 ÷ 100 = 1,920 ports per node

**When this is enough:**
- Small to medium clusters
- Moderate outbound traffic
- Budget isn't a major concern (each IP costs ~$3–5/month)

**Why it's still not ideal:**
- Pre-allocation inefficiency remains — ports are still locked to individual nodes
- 16 IP maximum (Azure hard limit) = ~1 million ports maximum
- You're multiplying the waste, not eliminating it
- NAT Gateway achieves the same maximum with NO waste

---

## 10. What is Azure NAT Gateway?

Azure NAT Gateway is a **fully managed, dedicated outbound connectivity service** for Azure VNets. Unlike Load Balancer, which does both inbound and outbound, NAT Gateway is **outbound-only**.

### The Key Innovation: On-Demand SNAT Ports

Instead of pre-allocating ports per-node, NAT Gateway uses a **shared dynamic pool**:

```
LOAD BALANCER (pre-allocated):
  Node A: [1,024 slots] → 700 used, 324 WASTED
  Node B: [1,024 slots] → 50 used, 974 WASTED
  Node C: [1,024 slots] → 1,024 used → EXHAUSTED! ⛔
  
NAT GATEWAY (on-demand):
  Shared pool: [64,512 slots] → 1,774 used, 62,738 available
  Any node can take from the shared pool as needed ✅
```

No waste, no fixed limits per node, no exhaustion unless you genuinely exceed the shared pool.

### NAT Gateway Technical Specifications

| Specification | Value |
|--------------|-------|
| Public IPs supported | 1 to 16 |
| SNAT ports per IP | 64,512 |
| Maximum concurrent connections | 16 × 64,512 = **1,032,192** |
| Protocols supported | TCP, UDP (not ICMP) |
| Idle timeout range | 4 to 120 minutes |
| Maximum throughput | 50 Gbps |
| Availability Zone support | Zonal (Standard) or Zone-redundant (StandardV2) |
| Works at which level | Subnet level |

### What NAT Gateway Does NOT Do

- ❌ Does NOT inspect packet contents
- ❌ Does NOT filter traffic by FQDN or IP
- ❌ Does NOT log individual connections (only aggregate metrics)
- ❌ Does NOT handle inbound (ingress) traffic
- ❌ Does NOT replace a firewall for security purposes

---

## 11. OutboundType: ManagedNATGateway

AKS creates, owns, and manages the NAT Gateway. You just configure the count of IPs and the idle timeout.

### Requirement

Only works with **AKS-managed VNets** (you did NOT specify `--vnet-subnet-id`). For BYO VNet, use `userAssignedNATGateway`.

### Creating the Cluster

```bash
az aks create \
  --resource-group rg-aks-natgateway \
  --name aks-natgateway \
  --outbound-type managedNATGateway \
  --nat-gateway-managed-outbound-ip-count 2 \
  --nat-gateway-idle-timeout 4
```

**Every parameter explained:**

| Parameter | What it means | When to change it |
|-----------|--------------|-------------------|
| `--outbound-type managedNATGateway` | AKS will create and manage a NAT Gateway | Switch from default LB |
| `--nat-gateway-managed-outbound-ip-count 2` | Create 2 public IPs = 2 × 64,512 = 129,024 SNAT ports | Increase if you see SNAT exhaustion |
| `--nat-gateway-idle-timeout 4` | Release idle connections after 4 minutes | Increase for long-lived connections (WebSockets, DB connections); lower for short HTTP calls |

### What Gets Created in Node Resource Group

- ✅ 2 × Public IP Addresses (Standard SKU)
- ✅ 1 × NAT Gateway — attached to the AKS subnet
- ✅ NSG, Route Table, VMSS, VNet, Managed Identity
- ❌ **NO Load Balancer** — not created at cluster creation

### Verifying It Works

```bash
kubectl run nginx --image=nginx
kubectl exec nginx -it -- /bin/bash

# The NAT Gateway has 2 IPs. Traffic round-robins between them:
curl https://ifconfig.me   # → 13.81.209.102
curl https://ifconfig.me   # → 40.115.29.65
```

Both IPs are the NAT Gateway's public IPs. Give both to external services for allowlisting.

### When You Create a Public Service

```bash
kubectl expose deployment nginx --port=80 --type LoadBalancer
```

AKS creates a **brand new, separate Standard Load Balancer** exclusively for this service's ingress. The NAT Gateway is NOT affected. Clear separation:
- NAT Gateway → egress for all pods
- Load Balancer → ingress for this specific service

### Updating an Existing Cluster to Use Managed NAT Gateway

If you have a cluster with a managed VNet (AKS created it):
```bash
az aks update \
  --resource-group <resourceGroup> \
  --name <clusterName> \
  --outbound-type managedNATGateway \
  --nat-gateway-managed-outbound-ip-count 2
```

> ⚠️ **Warning:** This changes the cluster's egress IP. Update any firewall rules or partner allowlists that depended on the old LB public IP.

---

## 12. OutboundType: UserAssignedNATGateway

You pre-create the NAT Gateway and its public IPs. AKS uses the one you've already set up. Required when using a BYO VNet.

### When to Use This

- Your cluster uses a custom VNet (`--vnet-subnet-id` was specified)
- Your organization wants to control networking infrastructure centrally
- You want to share one NAT Gateway across multiple clusters in the same subnet
- You need to pre-provision specific public IPs before the cluster exists

### Full Step-by-Step Setup

```bash
# Set variables
RG="rg-aks-usernatgateway"
LOC="westeurope"

# Step 1: Create a Managed Identity for the cluster
IDENTITY_ID=$(az identity create \
  --resource-group $RG \
  --name natClusterId \
  --location $LOC \
  --query id --output tsv)

# Step 2: Create a public IP for the NAT Gateway
az network public-ip create \
  --resource-group $RG \
  --name myNatGatewayPip \
  --location $LOC \
  --sku standard

# Step 3: Create the NAT Gateway
az network nat gateway create \
  --resource-group $RG \
  --name myNatGateway \
  --location $LOC \
  --public-ip-addresses myNatGatewayPip

# Step 4: Create a VNet
az network vnet create \
  --resource-group $RG \
  --name myVnet \
  --location $LOC \
  --address-prefixes 172.16.0.0/20

# Step 5: Create subnet — attach NAT Gateway at creation time
SUBNET_ID=$(az network vnet subnet create \
  --resource-group $RG \
  --vnet-name myVnet \
  --name natCluster \
  --address-prefixes 172.16.0.0/22 \
  --nat-gateway myNatGateway \
  --query id --output tsv)

# Step 6: Create AKS using that subnet
az aks create \
  --resource-group $RG \
  --name aks-cluster \
  --location $LOC \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_ID \
  --outbound-type userAssignedNATGateway \
  --assign-identity $IDENTITY_ID \
  --generate-ssh-keys
```

**What's different from managedNATGateway:**
- You create the NAT Gateway, Public IP, VNet, and Subnet BEFORE the cluster
- You attach the NAT Gateway to the subnet yourself
- AKS merely declares "I use the NAT Gateway on this subnet" — it doesn't create or modify it

---

## 13. OutboundType: ManagedNATGatewayV2 (Preview)

A newer, enhanced version of managed NAT Gateway using **Azure StandardV2 NAT Gateway**.

### The Core Improvement: Zone Redundancy

Standard NAT Gateway (used by `managedNATGateway`) is **zonal** — it lives in ONE Availability Zone. If that zone fails, NAT Gateway fails, and egress for pods in that zone breaks.

StandardV2 NAT Gateway is **zone-redundant by default** — it automatically distributes across ALL available zones. If Zone 2 goes down, pods in Zone 2 still have egress via Zone 1 and Zone 3.

### Comparison

| Feature | managedNATGateway | managedNATGatewayV2 |
|---------|-------------------|---------------------|
| NAT Gateway SKU | Standard | StandardV2 |
| Zone redundancy | ❌ Zonal | ✅ Zone-redundant by default |
| High Availability | Requires one NAT GW per zone (complex) | Built-in |
| Public IP SKU required | Standard | StandardV2 (NEW requirement) |
| Status | GA | Preview |

### Enabling and Using ManagedNATGatewayV2

```bash
# 1. Register the preview feature
az feature register \
  --namespace "Microsoft.ContainerService" \
  --name "ManagedNATGatewayV2Preview"

# 2. Check registration status (wait until "Registered")
az feature show \
  --namespace "Microsoft.ContainerService" \
  --name "ManagedNATGatewayV2Preview"

# 3. Refresh the provider
az provider register --namespace Microsoft.ContainerService

# 4. Create cluster with managedNATGatewayV2
az aks create \
  --resource-group $RG \
  --name my-cluster \
  --outbound-type managedNATGatewayV2 \
  --nat-gateway-managed-outbound-ip-count 2
```

> ⚠️ **Note:** StandardV2 NAT Gateway requires **StandardV2 Public IPs** — you cannot reuse existing Standard SKU IPs with it.

---

## 14. OutboundType: UserDefinedRouting

### What Is It?

User-Defined Routing (UDR) means **you take full control of how packets are routed** from the AKS subnet. Instead of Azure's default behavior (internet traffic goes directly to the internet via SNAT), you redirect all traffic to a next-hop appliance — usually an **Azure Firewall** or third-party NVA (Network Virtual Appliance).

### Why Enterprises Need This

Organizations with strict compliance requirements (banking, healthcare, government) need to:
- **Log every URL** that every pod accesses (audit trail)
- **Block traffic** to unauthorized domains (`*.torrent.com`, `*.competitor.com`)
- **Allow only specific services** (`*.docker.io`, `*.microsoft.com`, `*.mypartner.com`)
- **Decrypt HTTPS traffic** and inspect contents (SSL/TLS inspection)
- **Enforce data exfiltration prevention** — detect and block sensitive data leaving the cluster

None of these are possible with Load Balancer or NAT Gateway, which work at Layer 4 (TCP/UDP) and don't understand HTTP(S) content.

**Azure Firewall** and NVAs work at Layer 7, can see URLs, decrypt HTTPS, and enforce application-layer policies.

### Architecture: Hub & Spoke with UDR

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure Region                                   │
│                                                                   │
│  ┌─────────────────────────┐   VNet Peering   ┌──────────────┐  │
│  │      SPOKE VNet         │◄────────────────►│   HUB VNet   │  │
│  │      (aks-vnet)         │                  │              │  │
│  │  ┌──────────────────┐   │                  │ ┌──────────┐ │  │
│  │  │   aks-subnet     │   │                  │ │  Azure   │ │  │──► Internet
│  │  │                  │   │                  │ │ Firewall │ │  │  (via FW Public IP)
│  │  │  [AKS Cluster]   │   │                  │ └──────────┘ │  │
│  │  │                  │   │   Route Table:   │              │  │
│  │  │  Pod → Node      │───┼──0.0.0.0/0       │              │  │
│  │  │                  │   │  → FW Private IP │              │  │
│  │  └──────────────────┘   │                  │              │  │
│  └─────────────────────────┘                  └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Full Setup Guide

**Part 1: Create Hub VNet and Azure Firewall**

```bash
# Create hub VNet — firewall lives here
az network vnet create -g $RG -n hub-vnet \
  --address-prefix 10.0.0.0/8

# Azure Firewall REQUIRES a subnet named exactly "AzureFirewallSubnet"
az network vnet subnet create -g $RG --vnet-name hub-vnet \
  -n AzureFirewallSubnet \
  --address-prefix 10.0.2.0/24

# Create firewall's public IP (Standard SKU required)
az network public-ip create -g $RG -n firewall-publicip \
  --sku Standard --allocation-method Static

# Create Azure Firewall
az network firewall create -g $RG -n hub-firewall \
  --enable-dns-proxy true   # DNS proxy allows FQDN-based filtering

# Configure the firewall's IP
az network firewall ip-config create -g $RG \
  -f hub-firewall -n fw-ip-config \
  --public-ip-address firewall-publicip \
  --vnet-name hub-vnet

# Get the firewall's private IP — needed for route table
FW_PRIVATE_IP=$(az network firewall show -g $RG -n hub-firewall \
  --query "ipConfigurations[0].privateIpAddress" -o tsv)

echo "Firewall Private IP: $FW_PRIVATE_IP"
# Example output: 10.0.2.4
```

**Part 2: Create Route Table Forcing Traffic Through Firewall**

```bash
# Create the route table
az network route-table create -g $RG -l $LOC \
  --name firewall-routetable

# Key route: send ALL internet traffic to the Firewall
az network route-table route create -g $RG \
  --route-table-name firewall-routetable \
  -n internet-via-firewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_PRIVATE_IP
```

> ⚠️ **AKS validation:** AKS checks that your `0.0.0.0/0` route does NOT point directly to `Internet`. It must point to a VirtualAppliance (like a Firewall). If it points to Internet, AKS will reject the cluster creation.

**Part 3: Create Spoke VNet for AKS**

```bash
az network vnet create -g $RG -n aks-vnet \
  --address-prefix 172.16.0.0/12

SUBNET_ID=$(az network vnet subnet create -g $RG \
  --vnet-name aks-vnet -n aks-subnet \
  --address-prefix 172.16.0.0/22 \
  --route-table firewall-routetable \
  --query id -o tsv)

# Peer the spoke VNet to the hub VNet (bidirectional)
az network vnet peering create -g $RG \
  -n spoke-to-hub --vnet-name aks-vnet \
  --remote-vnet hub-vnet \
  --allow-vnet-access --allow-forwarded-traffic

az network vnet peering create -g $RG \
  -n hub-to-spoke --vnet-name hub-vnet \
  --remote-vnet aks-vnet \
  --allow-vnet-access --allow-forwarded-traffic
```

**Part 4: Add Firewall Rules for AKS**

AKS needs to access many external services to function. The easiest way: use the `AzureKubernetesService` FQDN tag, which Microsoft maintains automatically:

```bash
# Allow all AKS-required endpoints (MCR, Azure API, OS updates, etc.)
az network firewall application-rule create \
  -g $RG -f hub-firewall \
  --collection-name aksfwar \
  -n fqdn \
  --source-addresses '*' \
  --protocols 'http=80' 'https=443' \
  --fqdn-tags "AzureKubernetesService" \
  --action allow \
  --priority 100
```

The `AzureKubernetesService` FQDN tag is a **managed list** updated by Microsoft whenever AKS adds new dependencies. You don't need to maintain it manually.

For Docker image pulling from Docker Hub:
```bash
az network firewall application-rule create \
  -g $RG -f hub-firewall \
  --collection-name docker-rules \
  -n docker \
  --source-addresses '*' \
  --protocols 'https=443' \
  --target-fqdns \
    hub.docker.com \
    registry-1.docker.io \
    production.cloudflare.docker.com \
    auth.docker.io \
    cdn.auth0.com \
    login.docker.com \
    ifconfig.me \
  --action allow \
  --priority 200
```

**Part 5: Create AKS Cluster in UDR Mode**

```bash
az aks create \
  -g $RG \
  -n $AKSNAME \
  --node-count 3 \
  --network-plugin azure \
  --outbound-type userDefinedRouting \
  --vnet-subnet-id $SUBNET_ID \
  --load-balancer-sku standard
```

**Verifying the Pod Egresses via Firewall:**
```bash
# Pod should initially fail — Docker Hub not allowed yet
kubectl run nginx --image=nginx
kubectl get pods  # STATUS: ErrImagePull

# After adding docker firewall rules:
kubectl get pods  # STATUS: Running

kubectl exec nginx -it -- /bin/bash
curl http://ifconfig.me
# Output: 13.95.91.166  ← This is the FIREWALL's public IP
# NOT the Load Balancer IP ← Firewall is handling egress
```

---

## 15. Ingress vs Egress — The Interaction

`outboundType` only controls egress. Here's how ingress (public Kubernetes services) interacts with each outbound type:

| outboundType | Egress mechanism | What creates ingress (public services)? |
|-------------|-----------------|----------------------------------------|
| `loadBalancer` | Azure LB (also creates the LB) | Additional frontend IP on the SAME LB |
| `managedNATGateway` | NAT Gateway | AKS creates a NEW dedicated LB |
| `userAssignedNATGateway` | NAT Gateway | AKS creates a NEW dedicated LB |
| `userDefinedRouting` | Firewall (via route table) | AKS creates LB ON DEMAND (not at cluster creation) |

---

## 16. Asymmetric Routing Problem

This problem occurs specifically when using `userDefinedRouting` (firewall egress) with Kubernetes services of type `LoadBalancer` (ingress).

### The Problem in Detail

**Inbound path (user → pod):**
```
User → Load Balancer Public IP → LB → Node → Pod
```

**Response path (pod → user):**
```
Pod → Node → Route Table (0.0.0.0/0 → Firewall) → Azure Firewall
```

The Firewall sees a packet from the pod going to the user's IP. But it **never saw the original inbound request** (which arrived via the Load Balancer, bypassing the firewall). The Firewall cannot match this response to any known connection → **drops the packet**.

Result: Users cannot receive responses from your services.

### Solution A: DNAT Rule on Firewall

Add a Destination NAT rule to the Firewall:
- Incoming traffic to `Firewall_Public_IP:80` → DNAT to `LB_IP:80`

This routes ALL inbound traffic through the Firewall first. Now the Firewall sees both directions:
- Inbound: User → Firewall → LB → Pod ✅
- Outbound: Pod → Firewall (response) → User ✅

### Solution B: Use Application Gateway (AGIC) for Ingress

The **Application Gateway Ingress Controller (AGIC)** lives **inside** your AKS VNet. Because it uses the VNet's private IP space, return traffic from pods to the Application Gateway travels through private VNet routing — NOT through the route table that forces traffic to the firewall. No asymmetric routing issue.

This is the **recommended enterprise pattern**:
- **Egress:** All pod outbound → Firewall (via UDR)
- **Ingress:** All user inbound → Application Gateway → Pods (via private VNet)
- No conflict, no asymmetric routing

---

## 17. Decision Guide: Which OutboundType to Use?

```
Are you learning / prototyping?
└─ YES → loadBalancer (default)
└─ NO ↓

Is your cluster getting SNAT exhaustion or you expect high outbound scale?
└─ YES ↓
   Does AKS manage your VNet (you didn't set --vnet-subnet-id)?
   └─ YES → managedNATGateway (or managedNATGatewayV2 for AZ redundancy)
   └─ NO (BYO VNet) → userAssignedNATGateway

└─ NO ↓

Do you need traffic inspection, FQDN filtering, or compliance-level logging?
└─ YES → userDefinedRouting (Azure Firewall or NVA in hub VNet)
└─ NO → loadBalancer or managedNATGateway
```

| Scenario | Recommended | Reason |
|---------|-------------|--------|
| Dev/test, small team | `loadBalancer` | Simple, no extra setup |
| Production, moderate scale | `managedNATGateway` | Solves SNAT, AKS manages it |
| Production, need HA across AZs | `managedNATGatewayV2` | Zone-redundant (preview) |
| BYO VNet, enterprise | `userAssignedNATGateway` | You control the NAT GW |
| Enterprise security, compliance | `userDefinedRouting` | Full L7 inspection via Firewall |
| Hub & Spoke / Landing Zone | `userDefinedRouting` | Standard enterprise pattern |

---

## 18. Quick Reference Commands

```bash
# ── CREATE ─────────────────────────────────────────────────────

# Default Load Balancer
az aks create -g $RG -n $NAME --outbound-type loadBalancer

# Managed NAT Gateway (AKS VNet only)
az aks create -g $RG -n $NAME \
  --outbound-type managedNATGateway \
  --nat-gateway-managed-outbound-ip-count 2 \
  --nat-gateway-idle-timeout 4

# User-Assigned NAT Gateway (BYO VNet)
az aks create -g $RG -n $NAME \
  --outbound-type userAssignedNATGateway \
  --vnet-subnet-id $SUBNET_ID  # subnet must already have NAT GW attached

# UDR / Firewall
az aks create -g $RG -n $NAME \
  --outbound-type userDefinedRouting \
  --vnet-subnet-id $SUBNET_ID  # subnet must have route table with 0.0.0.0/0 → FW

# ── UPDATE ─────────────────────────────────────────────────────

# Scale LB outbound IPs
az aks update -g $RG -n $NAME \
  --load-balancer-managed-outbound-ip-count 3

# Change to managedNATGateway
az aks update -g $RG -n $NAME \
  --outbound-type managedNATGateway \
  --nat-gateway-managed-outbound-ip-count 2

# Change to userAssignedNATGateway
az aks update -g $RG -n $NAME \
  --outbound-type userAssignedNATGateway

# ── VERIFY ─────────────────────────────────────────────────────

# Check egress IP from inside a pod
kubectl exec <pod-name> -- curl -s ifconfig.me

# List NAT Gateway outbound IPs
az network nat gateway show -g $RG -n <natgw-name> \
  --query 'publicIpAddresses[].id' -o tsv | \
  xargs -I{} az network public-ip show --ids {} --query ipAddress -o tsv
```

---

## 📚 Official References

- [AKS Egress OutboundType](https://learn.microsoft.com/en-us/azure/aks/egress-outboundtype)
- [NAT Gateway with AKS](https://learn.microsoft.com/en-us/azure/aks/nat-gateway)
- [Limit egress with Azure Firewall](https://learn.microsoft.com/en-us/azure/aks/limit-egress-traffic)
- [Required AKS outbound FQDNs](https://learn.microsoft.com/en-us/azure/aks/outbound-rules-control-egress)
- [Virtual WAN AKS egress filtering](https://blog.cloudtrooper.net/2023/01/10/filtering-aks-egress-traffic-with-virtual-wan/)