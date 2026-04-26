# AKS Control Plane: Public, Private & VNET Integration
### A Beginner-Friendly Deep Dive

> **What this document covers:** How the AKS Control Plane (the "brain" of your cluster) can be exposed — publicly over the internet, privately inside your network, or a hybrid using VNET Integration. Every resource, term, and concept is explained from scratch.

---

## Table of Contents
1. [What is Kubernetes and AKS?](#1-what-is-kubernetes-and-aks)
2. [Core Concepts: Control Plane, Worker Nodes, API Server](#2-core-concepts-control-plane-worker-nodes-api-server)
3. [Who Needs Access to the Control Plane?](#3-who-needs-access-to-the-control-plane)
4. [Azure Resources You Will Encounter](#4-azure-resources-you-will-encounter)
5. [The 4 AKS Control Plane Access Options](#5-the-4-aks-control-plane-access-options)
6. [Option 1 — Public AKS Cluster](#6-option-1--public-aks-cluster)
7. [Option 2 — Private AKS Cluster (Private Endpoint)](#7-option-2--private-aks-cluster-private-endpoint)
8. [Option 3 — Public Cluster with VNET Integration](#8-option-3--public-cluster-with-vnet-integration)
9. [Option 4 — Private Cluster with VNET Integration](#9-option-4--private-cluster-with-vnet-integration)
10. [How to Access a Private Cluster](#10-how-to-access-a-private-cluster)
11. [Full Comparison Table](#11-full-comparison-table)
12. [Quick Decision Guide](#12-quick-decision-guide)

---

## 1. What is Kubernetes and AKS?

### Kubernetes
**Kubernetes** (often shortened to **K8s**) is an open-source platform for automating the deployment, scaling, and management of containerized applications. Think of it as an operating system for your cloud infrastructure — it tells your servers what to run, keeps things healthy, and restarts services that crash.

### AKS — Azure Kubernetes Service
**AKS** is Microsoft Azure's fully managed Kubernetes offering. "Managed" means Azure takes care of the most complex parts — particularly the **Control Plane** (the brain of Kubernetes). You don't have to install it, patch it, or worry about it going down. Azure handles that.

You, as a developer or DevOps engineer, only manage the **Worker Nodes** — the actual servers where your applications run.

---

## 2. Core Concepts: Control Plane, Worker Nodes, API Server

### The Control Plane
The **Control Plane** is the management layer of a Kubernetes cluster. It makes global decisions about the cluster — for example, scheduling applications, detecting failures, and restarting pods. In AKS, this runs in a **Microsoft-managed subscription** (you don't see it in your Azure portal directly).

Think of it like the **cockpit of an airplane**: the pilots (Control Plane) manage everything, but the passengers (your apps) ride in the body of the plane (Worker Nodes).

### Worker Nodes (Nodepool)
These are the actual virtual machines where your containerized applications run. In AKS, they live **inside your Azure subscription** in a resource group prefixed with `MC_` (Managed Cluster). You can see and manage them through your Azure portal.

Each group of similar nodes (same VM size, OS, config) is called a **Nodepool**.

### The API Server
The **API Server** is the front door of the Control Plane. Every interaction with Kubernetes goes through it:
- When you run `kubectl get pods`, you're talking to the API Server.
- When your application scales up, the scheduler talks to the API Server.
- When CI/CD pipelines deploy code, they talk to the API Server.

The API Server is exposed via a **URL (FQDN)** and listens on **HTTPS port 443**.

### FQDN — Fully Qualified Domain Name
An **FQDN** is a complete, human-readable domain name for a server. In AKS, the API Server gets a name like:
```
aks-cluster-rg-abc123.hcp.westeurope.azmk8s.io
```
This name can resolve to either a **public IP** (reachable from the internet) or a **private IP** (only reachable from inside a private network), depending on how you set up your cluster.

---

## 3. Who Needs Access to the Control Plane?

There are three types of clients that need to communicate with the Control Plane:

| Client | Tool | Why? |
|--------|------|-------|
| Developer / DevOps Engineer | `kubectl` (Kubernetes CLI) | Deploy applications, inspect pods, debug |
| CI/CD Pipeline | Azure DevOps, GitHub Actions, etc. | Automate deployments |
| Worker Nodes | Internal Kubernetes agents (kubelet) | Nodes must constantly report their status and receive instructions |

**The big security question**: Should the API Server be reachable from the public internet, or should it be locked inside a private network? This is exactly what the 4 access options solve.

---

## 4. Azure Resources You Will Encounter

Before diving into the access options, here is a glossary of every Azure resource mentioned — explained simply.

### Resource Group
A **Resource Group** is a logical container that holds related Azure resources. Everything you create (VMs, networks, IPs) belongs to a resource group. Think of it as a folder in a file system.

### Virtual Network (VNet)
A **Virtual Network (VNet)** is a private, isolated network inside Azure. Resources inside the same VNet can talk to each other using private IP addresses, just like computers in an office LAN. Nothing outside can access them unless you explicitly allow it.

### Subnet
A **Subnet** is a subdivided portion of a VNet. You break a VNet into subnets to logically group resources (e.g., one subnet for the AKS nodes, another for the API server). Each subnet has its own IP address range (e.g., `10.224.0.0/16`).

### Public IP Address
A **Public IP Address** is an IP that is reachable over the internet. When the AKS Control Plane has a public IP, anyone on the internet can attempt to reach it (access is still controlled by authentication, but the endpoint is exposed).

### Private IP Address
A **Private IP Address** is only reachable from within your VNet (or connected networks). It is not exposed to the internet at all.

### Load Balancer
A **Load Balancer** distributes incoming network traffic across multiple backend resources (like VM instances). AKS uses two kinds:
- **External/Public Load Balancer**: Exposed to the internet. Used for end-user traffic to your apps.
- **Internal Load Balancer (ILB)**: Only reachable from inside your VNet. Used for private communication between nodes and the Control Plane.

### Private Endpoint
A **Private Endpoint** is a special network interface that maps an Azure service (like the AKS Control Plane) to a **private IP address inside your VNet**. Without a Private Endpoint, you'd need to go over the public internet to reach that Azure service.

Think of it like running a private phone line directly into your office instead of using the public telephone network.

### Private DNS Zone
A **Private DNS Zone** is a DNS server that only responds to queries from within a specific VNet. When a Private Endpoint is created for AKS, a Private DNS Zone is also created so that the API Server's domain name resolves to the private IP, not a public one.

**Why is this needed?** DNS translates human-readable names (like `my-cluster.hcp.westeurope.azmk8s.io`) to IP addresses. If the private endpoint has IP `10.224.0.4`, then the Private DNS Zone must map the cluster's FQDN to that IP — so that `kubectl` and the nodes know where to connect.

### Network Security Group (NSG)
An **NSG** is a firewall for your VNet. It contains rules that **allow or deny** network traffic to/from resources. AKS automatically configures NSGs to protect node-to-node and node-to-control-plane traffic.

### Route Table
A **Route Table** defines how network packets are routed within a VNet. AKS uses route tables (especially with the Kubenet network plugin) to tell nodes how to reach pods on other nodes.

### Virtual Machine Scale Set (VMSS)
A **VMSS** is a group of identical VMs that can automatically scale up or down. AKS Worker Nodes are deployed as VMSS — this is how AKS can add or remove nodes as your workload changes.

### Managed Identity
A **Managed Identity** is an Azure AD identity assigned to an Azure resource (like AKS) that allows it to authenticate to other Azure services without needing a username and password stored anywhere. For example, AKS uses a Managed Identity to pull images from Azure Container Registry.

### Subnet Delegation
**Subnet Delegation** gives a specific Azure service (like AKS) exclusive rights to deploy resources inside a subnet. When you use VNET Integration, AKS needs a dedicated subnet delegated to `Microsoft.ContainerService/managedClusters` so it can inject the API Server's Internal Load Balancer into it.

---

## 5. The 4 AKS Control Plane Access Options

Azure supports four ways to configure access to the AKS Control Plane:

```
┌─────────────────────────────────────────────────────────────────────┐
│                   AKS Control Plane Access Options                    │
├──────────────────────────────────────────────────────────────────────┤
│  1. Public Cluster          - Public IP, exposed to internet          │
│  2. Private Cluster         - Private IP via Private Endpoint         │
│  3. Public + VNET Integr.   - Public IP externally, private internally│
│  4. Private + VNET Integr.  - Private IP only, no Private Endpoint   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 6. Option 1 — Public AKS Cluster

### What is it?
In a **Public Cluster**, the API Server (Control Plane) is given a **public FQDN and a public IP address**. The endpoint is reachable from anywhere on the internet over HTTPS (port 443).

### Architecture Diagram
```
    Developer / kubectl / CI-CD Pipeline
              │
              │  HTTPS (public internet)
              ▼
    ┌─────────────────────┐          ┌─────────────────────────┐
    │  Control Plane      │          │ Microsoft Managed Sub.   │
    │  Public IP          │──────────│ AKS Control Plane (API)  │
    └─────────────────────┘          └─────────────────────────┘
                                               ▲
                                               │ Worker nodes access
                                               │ Control Plane
                                               │ (via public IP)
                                    ┌─────────────────────────┐
                                    │ AKS VNET                 │
                                    │  [Nodepool (VMs)]        │
                                    └─────────────────────────┘
```

### Key Characteristics
- The API Server FQDN resolves to a **public IP address** (e.g., `20.103.218.175`)
- The Control Plane endpoint **is exposed to the internet**
- You can restrict which IPs can connect using **Authorized IP Ranges**
- Worker nodes connect to the Control Plane **over the public IP** (though the traffic stays within the Azure backbone — it does NOT go over the open internet)
- There is **no Private FQDN** (empty when queried)

### What is "Authorized IP Ranges"?
This is a security feature that lets you whitelist only specific IP addresses or CIDR ranges that are allowed to reach the API Server. For example, you could allow only your company's VPN IP: `203.0.113.5/32`. Everyone else gets blocked.

### How to Create a Public AKS Cluster

```bash
# Step 1: Create a Resource Group
az group create -n rg-aks-public -l westeurope

# Step 2: Create the AKS cluster (public by default)
az aks create -n aks-cluster -g rg-aks-public --generate-ssh-keys

# Verify the public FQDN
az aks show -n aks-cluster -g rg-aks-public --query fqdn
# Output: "aks-cluster-rg-aks-public-17b128.hcp.westeurope.azmk8s.io"

# Verify it resolves to a public IP
nslookup aks-cluster-rg-aks-public-17b128.hcp.westeurope.azmk8s.io
# Output: Address: 20.103.218.175  ← PUBLIC IP

# Check if private FQDN exists
az aks show -n aks-cluster -g rg-aks-public --query privateFqdn
# Output: null  ← No private FQDN
```

### Resources Created in the MC_ Resource Group

When AKS creates a cluster, it automatically creates a second resource group (called `MC_<rg>_<cluster>_<region>`) that holds all the infrastructure:

| Resource | Type | Purpose |
|----------|------|---------|
| `<uuid>` | Public IP address | The IP the Control Plane is exposed on |
| `aks-agentpool-<id>-nsg` | Network Security Group | Firewall rules for the nodes |
| `aks-agentpool-<id>-routetable` | Route table | Routes for pod-to-pod traffic |
| `aks-cluster-agentpool` | Managed Identity | Identity for AKS to access Azure services |
| `aks-nodepool1-<id>-vmss` | Virtual Machine Scale Set | The actual Worker Node VMs |
| `aks-vnet-<id>` | Virtual Network | The private network for nodes |
| `kubernetes` | Load Balancer | Exposes services to external users |

### How Do Worker Nodes Connect to the Control Plane?

In a public cluster, worker nodes connect to the Control Plane **through the public IP address**. This means if you run:
```bash
kubectl describe svc kubernetes
```
You'll see the endpoint listed as the public IP, e.g., `20.103.218.175:443`.

> ⚠️ This traffic stays within the **Azure backbone network** — it doesn't literally travel over the open internet — but the endpoint itself is publicly addressable.

### Pros and Cons

**✅ Advantages**
- Simple to set up — no extra networking required
- `kubectl` works out of the box from anywhere
- CI/CD pipelines can connect from cloud agents easily

**❌ Disadvantages**
- The API Server endpoint is visible on the public internet
- Not acceptable for strict security or compliance requirements
- Worker nodes talk to the Control Plane via a public endpoint

---

## 7. Option 2 — Private AKS Cluster (Private Endpoint)

### What is it?
In a **Private Cluster**, the API Server is given a **private IP address** via a **Private Endpoint** inside your VNet. The Control Plane endpoint is **NOT exposed to the internet at all**.

A public FQDN still exists (for convenience), but it resolves to the **private IP** — so it's only usable from inside your VNet. You can optionally disable the public FQDN entirely.

### Architecture Diagram
```
    Developer / kubectl (BLOCKED from internet)
              │  ✗  No public access
              ▼
    ┌─────────────────────────┐
    │  AKS Control Plane       │  ← In Microsoft managed subscription
    │  (no public exposure)    │
    └─────────────────────────┘
                    ▲
                    │
         ┌──────────────────────────────────────────┐
         │ AKS VNET (Your subscription)              │
         │                                           │
         │  [Private Endpoint]──────────────────────►│
         │        │              (private IP)        │
         │  [Private DNS Zone]                       │
         │        │                                  │
         │  [Nodepool (Worker Nodes)]                │
         │        │                                  │
         │  [JumpBox VM] ◄── Developer access via   │
         │                    this VM only           │
         └──────────────────────────────────────────┘
```

### Key Characteristics
- Control Plane has a **public FQDN** but it resolves to a **private IP** (e.g., `10.224.0.4`)
- The private IP is only reachable from **inside your VNet** or peered networks
- A **Private Endpoint** is created inside your VNet — this "injects" the Control Plane into your network
- A **Private DNS Zone** is created to map the private FQDN to the private IP
- The **public FQDN can be disabled** for maximum isolation
- Worker nodes connect to the Control Plane **through the Private Endpoint**
- `kubectl` from outside the VNet **cannot reach the cluster** directly

### What Happens When You Run nslookup?
- From inside the VNet: the FQDN resolves to `10.224.0.4` ✅
- From outside the VNet: `Non-existent domain` ❌ (because the Private DNS Zone is only linked to the cluster VNet)

### How to Create a Private AKS Cluster

```bash
# Step 1: Create Resource Group
az group create -n rg-aks-private -l westeurope

# Step 2: Create private cluster
az aks create -n aks-cluster -g rg-aks-private --enable-private-cluster

# Check the public FQDN (still exists but resolves to private IP)
az aks show -n aks-cluster -g rg-aks-private --query fqdn
# Output: "aks-cluster-rg-aks-private-17b128.hcp.westeurope.azmk8s.io"

# nslookup from internet resolves to PRIVATE IP
# Address: 10.224.0.4  ← private!

# Check private FQDN
az aks show -n aks-cluster -g rg-aks-private --query privateFqdn
# Output: "aks-cluster-rg-aks-private-17b128-6d8d.628fd8ef.privatelink.westeurope.azmk8s.io"

# Optionally disable the public FQDN
az aks update -n aks-cluster -g rg-aks-private --disable-public-fqdn
```

### Additional Resources in the MC_ Resource Group

Compared to a public cluster, a private cluster adds:

| New Resource | Type | Purpose |
|--------------|------|---------|
| `<uuid>.privatelink.westeurope.azmk8s.io` | **Private DNS Zone** | Maps cluster's private FQDN → private IP |
| `kube-apiserver` | **Private Endpoint** | The "door" from your VNet to the Control Plane |
| `kube-apiserver.nic.<id>` | Network Interface | The network card attached to the private endpoint |

### Private DNS Zone — Deep Dive

The Private DNS Zone (e.g., `628fd8ef.privatelink.westeurope.azmk8s.io`) contains an **A record** like:
```
aks-cluster-rg-aks-private  →  10.224.0.4  (private IP of the API Server)
```

This DNS zone is **linked to your VNet**, so only machines inside the VNet can resolve that name. Machines outside see "Non-existent domain."

> **Analogy**: Imagine the company directory at your office maps "CEO's office" to room 204. But this directory only exists inside the office. If you look up from outside, there's no listing.

### How Worker Nodes Connect
In a private cluster, the API Server is "projected" into the AKS subnet through the Private Endpoint. The Kubernetes service inside the cluster shows:
```
Endpoints: 10.224.0.4:443  ← private IP of the Control Plane
```
So worker nodes communicate with the Control Plane entirely within the private network. No public IP involved.

### Pros and Cons

**✅ Advantages**
- Complete network isolation — API Server never touches the public internet
- Meets strict compliance requirements (financial, government, healthcare)
- Minimal attack surface

**❌ Disadvantages**
- Complex setup — requires networking expertise
- `kubectl` must be run from inside the VNet (e.g., a JumpBox VM, VPN, or ExpressRoute)
- CI/CD pipelines need to run on self-hosted agents inside the VNet
- Restarting a private cluster creates a new Private Endpoint with a **different private IP** (important for DNS records)

---

## 8. Option 3 — Public Cluster with VNET Integration

### What is it?
**VNET Integration** is a hybrid approach. The Control Plane still has a **public FQDN and public IP** (so `kubectl` and CI/CD can connect from the internet), but **Worker Nodes connect to the Control Plane via a private IP** inside the VNet — using an **Internal Load Balancer**.

This solves the biggest weakness of the pure public cluster: worker nodes no longer need to communicate with the Control Plane over the public IP.

### Architecture Diagram
```
    Developer / kubectl / CI-CD Pipeline
              │
              │  HTTPS (public internet, via public IP)
              ▼
    ┌─────────────────────┐          ┌─────────────────────────┐
    │  Control Plane       │          │ Microsoft Managed Sub.   │
    │  Public IP           │──────────│ AKS Control Plane (API)  │
    └─────────────────────┘          └──────────────┬──────────┘
                                                     │ Subnet Delegation
                                    ┌────────────────▼────────────────┐
                                    │ AKS VNET (Your subscription)     │
                                    │                                   │
                                    │  [AKS Subnet]   [API Server Sub] │
                                    │  [Nodepool]─────►[Internal LB]   │
                                    │  (Worker Nodes)  (private access) │
                                    └──────────────────────────────────┘
```

### Key Characteristics
- API Server FQDN resolves to a **public IP** (externally accessible)
- **Public FQDN cannot be disabled** (only possible on private clusters)
- Worker Nodes connect to Control Plane through **Internal Load Balancer (private IP)** — not the public IP
- DevOps pipelines can connect through **either** the public or private IP
- VNET Integration creates a **new dedicated subnet** in your VNet (called something like `aks-apiserver-subnet`)
- That subnet is **delegated** to `Microsoft.ContainerService/managedClusters` — AKS owns it
- No Private Endpoint is created

### What is the Internal Load Balancer (ILB)?
The **Internal Load Balancer** (called `kube-apiserver` in the `MC_` resource group) sits inside the dedicated API server subnet. It has a private IP address (e.g., `10.226.0.4`). Worker nodes send their API requests to this private IP instead of going out to the public IP.

This keeps **all node-to-control-plane traffic private** while still allowing external operators to use the public endpoint.

### Subnet Delegation — Why is it Needed?
When AKS sets up VNET Integration, it needs to inject the Internal Load Balancer into a subnet. Azure requires you to **delegate** that subnet to `Microsoft.ContainerService/managedClusters`, which means:
- Only AKS can place resources in that subnet
- AKS can manage the subnet's configuration freely

You'll see this in the portal:
```
Subnet: aks-apiserver-subnet  10.226.0.0/28
Delegated to: Microsoft.ContainerService/managedClusters
```

### How to Create a Public Cluster with VNET Integration

```bash
# Step 1: Create Resource Group
az group create -n rg-aks-public-vnet-integration -l eastus2

# Step 2: Create cluster with VNET Integration flag
az aks create -n aks-cluster -g rg-aks-public-vnet-integration \
  --enable-apiserver-vnet-integration

# Check the FQDN — resolves to a PUBLIC IP
az aks show -n aks-cluster -g rg-aks-public-vnet-integration --query fqdn
# Output: "aks-cluster.hcp.eastus2.azmk8s.io"
# nslookup resolves to: 20.94.16.207  ← PUBLIC IP

# No private FQDN (cluster is still "public")
az aks show -n aks-cluster -g rg-aks-public-vnet-integration --query privateFqdn
# Output: null
```

### How Worker Nodes Connect — Verified with kubectl
```bash
kubectl describe svc kubernetes
# Endpoints: 10.226.0.4:443  ← PRIVATE IP of the Internal Load Balancer
```
Compare this to the public cluster where you'd see `20.103.218.175:443` (public IP). With VNET Integration, even though the cluster is "public," the nodes use a private path.

### Additional Resources Created

| New Resource | Type | Purpose |
|--------------|------|---------|
| `kube-apiserver` | **Load Balancer (Internal)** | Private access to API Server from within VNet |
| New subnet in VNet | **Subnet** | Hosts the Internal Load Balancer |

### Pros and Cons

**✅ Advantages**
- Easy to manage externally (public endpoint still works)
- Worker node traffic stays private (no public IP needed for nodes)
- No complex Private Endpoint setup

**❌ Disadvantages**
- API Server is still reachable from the internet (public IP exists)
- Public FQDN cannot be disabled
- Not suitable for strict "no internet exposure" requirements

---

## 9. Option 4 — Private Cluster with VNET Integration

### What is it?
The most secure option that uses VNET Integration. The Control Plane is **only accessible via private IP** through the Internal Load Balancer. There is **no Private Endpoint** created (unlike Option 2). The public FQDN still exists but resolves to the **private IP** of the ILB, and it can be disabled.

### Architecture Diagram
```
    Developer / kubectl (BLOCKED from internet)
              │  ✗  No public access
              ▼
    ┌────────────────────────┐          ┌──────────────────────────┐
    │  AKS Control Plane      │          │ Microsoft Managed Sub.    │
    │  (no public exposure)   │──────────│ API Server pods projected │
    └────────────────────────┘          │ via Subnet Delegation     │
                                        └──────────────────────────┘
                                                    │
                                    ┌───────────────▼──────────────┐
                                    │ AKS VNET                      │
                                    │  [AKS Subnet]  [API Svr Sub]  │
                                    │  [Nodepool]────►[Internal LB] │
                                    │  (Worker Nodes) (private only)│
                                    └──────────────────────────────┘
```

### Key Characteristics
- Control Plane is accessible **only via private IP** of the Internal Load Balancer
- Public FQDN exists but resolves to the **private IP** (e.g., `10.226.0.4`) — can be disabled
- DevOps CD pipelines can **only** access through private IP (internal LB)
- Nodes access Control Plane through Internal Load Balancer (private IP)
- **No Private Endpoint** is created (unlike Option 2)
- A **Private DNS Zone** is created (maps private FQDN → private IP of ILB)

### How to Create

```bash
# Step 1: Create Resource Group
az group create -n rg-aks-private-vnet-integration -l eastus2

# Step 2: Create private cluster with VNET Integration
az aks create -n aks-cluster -g rg-aks-private-vnet-integration \
  --enable-apiserver-vnet-integration \
  --enable-private-cluster

# Check FQDN — resolves to PRIVATE IP (even though it's a "public" FQDN)
az aks show -n aks-cluster -g rg-aks-private-vnet-integration --query fqdn
# nslookup resolves to: 10.226.0.4  ← PRIVATE IP

# Private FQDN exists too
az aks show -n aks-cluster -g rg-aks-private-vnet-integration --query privateFqdn
# Output: "aks-cluster.2788811a.private.eastus2.azmk8s.io"
# nslookup from outside: Non-existent domain  ← only resolvable inside VNet
```

### Difference from Option 2 (Private Endpoint)
| Aspect | Option 2 (Private Endpoint) | Option 4 (Private + VNET Integration) |
|--------|----------------------------|---------------------------------------|
| Access mechanism | Private Endpoint | Internal Load Balancer |
| Private Endpoint created? | ✅ Yes | ❌ No |
| Private DNS Zone created? | ✅ Yes | ✅ Yes |
| Subnet Delegation needed? | ❌ No | ✅ Yes |
| Worker node to CP traffic | Via Private Endpoint | Via Internal Load Balancer |

---

## 10. How to Access a Private Cluster

Since `kubectl` from your laptop cannot reach a private cluster directly, here are your options:

### Option A: JumpBox VM inside the AKS VNet
Deploy a Virtual Machine inside the same VNet as AKS. SSH into that VM, then run `kubectl` from inside:
```bash
# Inside the JumpBox VM
az aks get-credentials --resource-group rg-aks-private --name aks-cluster
kubectl get pods
```
**Best for**: Small teams, dev environments.

### Option B: Azure Bastion
**Azure Bastion** is a managed service that lets you securely connect to VMs over SSL without exposing them to the internet. Connect to your JumpBox VM through Bastion from the Azure Portal.

### Option C: VPN or ExpressRoute
Connect your on-premises office network to the Azure VNet using a **Site-to-Site VPN** or **Azure ExpressRoute**. Once connected, your laptop is effectively "inside" the VNet.

**Best for**: Enterprise environments with existing hybrid connectivity.

### Option D: kubectl command invoke
Azure CLI provides a way to run kubectl commands without direct network access:
```bash
az aks command invoke \
  --resource-group rg-aks-private \
  --name aks-cluster \
  --command "kubectl get pods"
```
Azure executes the command server-side through a temporary secure tunnel. **Best for**: Quick debugging, one-off commands.

### Option E: Private Endpoint Connection
Create another Private Endpoint in a different VNet that connects to the cluster's Private Endpoint. Useful for hub-and-spoke architectures.

---

## 11. Full Comparison Table

| Feature | Public Cluster | Private Cluster | Public + VNET Integration | Private + VNET Integration |
|---------|---------------|-----------------|--------------------------|---------------------------|
| **Public FQDN** | ✅ Yes (public IP) | ✅ Yes (private IP) | ✅ Yes (public IP) | ✅ Yes (private IP) |
| **Private FQDN** | ❌ No | ✅ Yes | ❌ No | ✅ Yes |
| **Public FQDN can be disabled?** | ❌ No | ✅ Yes | ❌ No | ✅ Yes |
| **How to access Control Plane** | Public IP/FQDN | Private Endpoint + Private DNS | Public IP or private ILB IP | Internal LB only |
| **Private Endpoint created?** | ❌ No | ✅ Yes | ❌ No | ❌ No |
| **Internal Load Balancer?** | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| **Private DNS Zone?** | ❌ No | ✅ Yes | ❌ No | ✅ Yes |
| **Worker node → CP traffic** | Over public IP | Via Private Endpoint | Via Internal LB (private) | Via Internal LB (private) |
| **Internet exposure** | Full | None | API Server exposed | None |
| **Complexity** | Low | High | Medium | High |
| **`kubectl` from laptop** | ✅ Direct | ❌ Need VPN/JumpBox | ✅ Direct | ❌ Need VPN/JumpBox |

---

## 12. Quick Decision Guide

```
Is exposing the API Server to the internet acceptable?
│
├─► YES
│     └─► Do you want node-to-CP traffic to be private?
│           ├─► YES → Option 3: Public + VNET Integration
│           └─► NO  → Option 1: Public Cluster
│
└─► NO (fully private required)
      └─► Are you okay with the complexity of Private Endpoints?
            ├─► YES (want traditional private cluster) → Option 2: Private Cluster
            └─► Prefer simpler ILB-based approach     → Option 4: Private + VNET Integration
```

### Microsoft's Current Recommendation (2025)
- For new production clusters: **Private Cluster with VNET Integration** or **Private Cluster with Private Endpoint**
- For development/staging: **Public Cluster with VNET Integration** is a good balance
- Avoid pure public clusters in production environments

---

## References & Further Reading

- [Create a private AKS cluster](https://learn.microsoft.com/en-us/azure/aks/private-clusters)
- [API Server VNet Integration](https://learn.microsoft.com/en-us/azure/aks/api-server-vnet-integration)
- [Access a private AKS cluster](https://learn.microsoft.com/en-us/azure/aks/private-cluster-connect)
- [AKS Authorized IP Ranges](https://learn.microsoft.com/en-us/azure/aks/api-server-authorized-ip-ranges)