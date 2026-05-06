# Karpenter & Node Auto-Provisioning (NAP) in AKS
### The Ultimate Beginner-Friendly Deep Dive

> **What this document covers:** What Karpenter is, how Node Auto-Provisioning (NAP) works in AKS, how it differs from Cluster Autoscaler, the NodePool and AKSNodeClass CRDs, SKU selection, Spot vs On-Demand, consolidation, disruption policies, networking requirements, upgrades, what Azure manages vs what you manage, and complete setup workflows — all explained from absolute scratch.

---

## Table of Contents
1. [The Problem: Traditional Node Pool Scaling](#1-the-problem-traditional-node-pool-scaling)
2. [What is Karpenter?](#2-what-is-karpenter)
3. [What is Node Auto-Provisioning (NAP)?](#3-what-is-node-auto-provisioning-nap)
4. [Karpenter vs Cluster Autoscaler](#4-karpenter-vs-cluster-autoscaler)
5. [How Karpenter Works — The Provisioning Lifecycle](#5-how-karpenter-works--the-provisioning-lifecycle)
6. [What Azure Manages vs What You Manage](#6-what-azure-manages-vs-what-you-manage)
7. [Prerequisites and Networking Requirements](#7-prerequisites-and-networking-requirements)
8. [Enabling NAP on AKS](#8-enabling-nap-on-aks)
9. [The NodePool CRD — Defining What Gets Provisioned](#9-the-nodepool-crd--defining-what-gets-provisioned)
10. [The AKSNodeClass CRD — Azure-Specific Configuration](#10-the-aksnodeclass-crd--azure-specific-configuration)
11. [SKU Selection — Families, Architectures, Spot Instances](#11-sku-selection--families-architectures-spot-instances)
12. [Consolidation — Automatic Cost Optimization](#12-consolidation--automatic-cost-optimization)
13. [Disruption Policies and Node Disruption Budgets](#13-disruption-policies-and-node-disruption-budgets)
14. [Node Image Upgrades for NAP](#14-node-image-upgrades-for-nap)
15. [Troubleshooting NAP](#15-troubleshooting-nap)
16. [Best Practices and Recommendations](#16-best-practices-and-recommendations)

---

## 1. The Problem: Traditional Node Pool Scaling

### The Old Way — Cluster Autoscaler

With the traditional **Cluster Autoscaler**, you must pre-define node pools with a fixed VM size:

```
┌──────────────────────────────────────────────────────────────┐
│  Traditional Approach (Cluster Autoscaler)                   │
│                                                              │
│  Node Pool "general"    → Standard_D4s_v5 (4 vCPU, 16GB)   │
│  Node Pool "memory"     → Standard_E8s_v5 (8 vCPU, 64GB)   │
│  Node Pool "gpu"        → Standard_NC6s_v3 (GPU)            │
│  Node Pool "spot"       → Standard_D4s_v5 (Spot)            │
│  Node Pool "arm"        → Standard_D4pds_v6 (ARM)           │
│                                                              │
│  ⚠️ You must guess the right VM sizes IN ADVANCE            │
│  ⚠️ Each pool has ONE fixed VM size                         │
│  ⚠️ Over-provisioning wastes money                          │
│  ⚠️ Under-provisioning = pending pods                       │
│  ⚠️ Managing 5-10 pools is complex                          │
└──────────────────────────────────────────────────────────────┘
```

### The New Way — Karpenter / NAP

```
┌──────────────────────────────────────────────────────────────┐
│  NAP / Karpenter Approach                                    │
│                                                              │
│  NodePool "default" → Constraints:                           │
│    - D-series family                                         │
│    - AMD64 or ARM64                                          │
│    - On-Demand or Spot                                       │
│    - Max 100 vCPUs total                                     │
│                                                              │
│  Pod needs 2 vCPU, 8GB → Karpenter picks D2as_v5            │
│  Pod needs 16 vCPU, 128GB → Karpenter picks E16s_v5         │
│  Pod needs GPU → Karpenter picks NC6s_v3                     │
│                                                              │
│  ✅ Karpenter picks the OPTIMAL VM for each workload        │
│  ✅ One NodePool replaces many traditional pools             │
│  ✅ Automatic right-sizing and consolidation                │
│  ✅ Mixed VM sizes, architectures, pricing                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. What is Karpenter?

**Karpenter** is an open-source Kubernetes node provisioner originally created by AWS and donated to the **CNCF (Cloud Native Computing Foundation)**. It watches for unschedulable pods and provisions the optimal compute resources in response.

**Key principles:**
- **Pod-driven**: Provisions nodes based on actual pod requirements, not pre-defined pools
- **Right-sized**: Selects the cheapest VM that satisfies pod resource requests
- **Consolidation**: Continuously bin-packs workloads and removes underutilized nodes
- **Cloud-agnostic**: Supports AWS (native), Azure (via AKS provider), and more

GitHub: `https://github.com/kubernetes-sigs/karpenter`
AKS Provider: `https://github.com/Azure/karpenter-provider-azure`

---

## 3. What is Node Auto-Provisioning (NAP)?

**Node Auto-Provisioning (NAP)** is AKS's managed implementation of Karpenter. Azure deploys, configures, and manages Karpenter as a control plane add-on — you don't install or maintain it yourself.

**NAP became GA in mid-July 2025.**

```
┌──────────────────────────────────────────────────────────────┐
│                    NAP Architecture                           │
│                                                              │
│  AKS Control Plane (Azure Managed)                          │
│  ┌────────────────────────────────────────────────┐         │
│  │  API Server   │  etcd   │  Karpenter Controller│         │
│  │               │         │  (managed by Azure)  │         │
│  └────────────────────────────────────────────────┘         │
│           │                         │                        │
│           │ Watches pods            │ Provisions VMs         │
│           ▼                         ▼                        │
│  ┌──────────────┐         ┌──────────────────┐              │
│  │ Pending Pod  │         │ New VM (right-   │              │
│  │ needs 4 CPU, │  ──────►│ sized for the    │              │
│  │ 16GB RAM     │         │ pod's needs)     │              │
│  └──────────────┘         └──────────────────┘              │
│                                                              │
│  Custom Resources (you configure):                          │
│  • NodePool     → Constraints (families, capacity, limits)  │
│  • AKSNodeClass → Azure settings (OS, disk, subnet)         │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Karpenter vs Cluster Autoscaler

| Feature | Cluster Autoscaler | Karpenter / NAP |
|---------|-------------------|-----------------|
| **VM selection** | Fixed per node pool | Dynamic per pod |
| **Mixed VM sizes** | ❌ One size per pool | ✅ Any size within constraints |
| **Provisioning speed** | ~2-5 minutes | ~1-2 minutes |
| **Consolidation** | ❌ Only removes empty nodes | ✅ Bin-packs and right-sizes |
| **Spot + On-Demand mix** | Requires separate pools | ✅ In single NodePool |
| **AMD + ARM mix** | Requires separate pools | ✅ In single NodePool |
| **Scaling unit** | Node pool (VMSS) | Individual VM |
| **Complexity** | Multiple pools to manage | 1-2 NodePool CRDs |
| **Cost optimization** | Manual right-sizing | Automatic |
| **GA on AKS** | Yes (since 2019) | Yes (July 2025) |

### When to Use NAP vs Cluster Autoscaler

**Use NAP when:**
- You want automatic VM right-sizing
- You have diverse workloads (different CPU/memory ratios)
- You want cost optimization via consolidation
- You want Spot + On-Demand mixed in one pool

**Use Cluster Autoscaler when:**
- You need a specific, predictable VM size
- You have compliance requirements for fixed node configurations
- You're running AKS < 1.26 (NAP not available)

---

## 5. How Karpenter Works — The Provisioning Lifecycle

```
Step 1: Pod created, cannot be scheduled (Pending)
  Scheduler says: "No node has enough resources"
         │
         ▼
Step 2: Karpenter detects unschedulable pod
  Evaluates: CPU requests, memory requests, node affinity,
             tolerations, topology spread constraints
         │
         ▼
Step 3: Karpenter selects optimal VM SKU
  Filters: NodePool requirements (family, arch, capacity type)
  Ranks: Cheapest VM that fits all pending pods
  Example: 3 pods need 2 CPU each → picks D8as_v5 (8 CPU)
           instead of 3x D2as_v5 (more efficient packing)
         │
         ▼
Step 4: Karpenter creates the VM
  Calls Azure API → Creates individual VM (not VMSS)
  Bootstraps Kubernetes → Joins cluster
  Time: ~60-90 seconds
         │
         ▼
Step 5: Pod is scheduled on the new node
  Scheduler places pod on the newly provisioned node
         │
         ▼
Step 6: Continuous consolidation
  Karpenter monitors node utilization
  If a node becomes underutilized → migrates pods elsewhere
  Then deletes the underutilized node
  Saves money automatically
```

---

## 6. What Azure Manages vs What You Manage

```
┌──────────────────────────────────────────────────────────────┐
│              AZURE MANAGES                                    │
│                                                              │
│  ✅ Karpenter controller deployment and upgrades            │
│  ✅ Karpenter CRD installation                              │
│  ✅ Default NodePool and AKSNodeClass creation               │
│  ✅ System-surge NodePool for system pool scaling            │
│  ✅ Node image updates (automatic by default)               │
│  ✅ Karpenter health monitoring                              │
│  ✅ Integration with AKS control plane                      │
├──────────────────────────────────────────────────────────────┤
│              YOU MANAGE                                       │
│                                                              │
│  ⚠️ NodePool configuration (SKU families, limits, labels)   │
│  ⚠️ AKSNodeClass configuration (OS, disk size, subnet)     │
│  ⚠️ Disruption policies and Node Disruption Budgets        │
│  ⚠️ Pod resource requests (Karpenter needs accurate values)│
│  ⚠️ Maintenance windows for node image upgrades            │
│  ⚠️ RBAC permissions for custom subnets                    │
│  ⚠️ Monitoring and logging                                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. Prerequisites and Networking Requirements

### Required Networking Configuration

NAP requires **Azure CNI Overlay with Cilium**:

```bash
# NAP requires these specific network settings:
--network-plugin azure
--network-plugin-mode overlay
--network-dataplane cilium
```

**Not supported:**
- ❌ Kubenet
- ❌ Azure CNI (flat/non-overlay)
- ❌ Azure CNI without Cilium
- ❌ Existing clusters with Cluster Autoscaler enabled

### RBAC Permissions for Custom Subnets

If using custom subnets, grant the cluster identity **Network Contributor**:

```bash
CLUSTER_IDENTITY=$(az aks show -g my-rg -n my-aks \
  --query "identity.principalId" -o tsv)

az role assignment create \
  --assignee $CLUSTER_IDENTITY \
  --role "Network Contributor" \
  --scope "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Network/virtualNetworks/<vnet>"
```

---

## 8. Enabling NAP on AKS

### New Cluster

```bash
az aks create \
  --name my-aks \
  --resource-group my-rg \
  --node-provisioning-mode Auto \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium \
  --generate-ssh-keys
```

### Existing Cluster

```bash
# First, disable cluster autoscaler on all node pools
az aks nodepool update -g my-rg --cluster-name my-aks \
  -n nodepool1 --disable-cluster-autoscaler

# Enable NAP
az aks update -g my-rg -n my-aks \
  --node-provisioning-mode Auto
```

### Verify

```bash
# Check Karpenter CRDs exist
kubectl api-resources | grep karpenter
# aksnodeclasses.karpenter.azure.com
# nodeclaims.karpenter.sh
# nodepools.karpenter.sh

# Check default NodePool
kubectl get nodepools
# NAME      NODECLASS   DESIRED   READY   AGE
# default   default     0         0       5m
```

---

## 9. The NodePool CRD — Defining What Gets Provisioned

### Default NodePool Created by NAP

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    spec:
      nodeClassRef:
        name: default
      expireAfter: Never
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: kubernetes.io/os
        operator: In
        values: ["linux"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand"]
      - key: karpenter.azure.com/sku-family
        operator: In
        values: ["D"]      # Default: D-series only
```

### Custom NodePool Example (Production)

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-workloads
spec:
  # Resource limits for this pool
  limits:
    cpu: 200              # Max 200 vCPUs total
    memory: 800Gi         # Max 800 GB RAM total

  # Disruption settings
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    budgets:
    - nodes: "20%"        # Max 20% of nodes disrupted at once

  # Weight for priority (higher = preferred)
  weight: 50

  template:
    metadata:
      labels:
        workload-type: general
    spec:
      nodeClassRef:
        apiVersion: karpenter.azure.com/v1beta1
        kind: AKSNodeClass
        name: default
      expireAfter: 720h    # Nodes expire after 30 days (force refresh)

      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64", "arm64"]    # Both architectures
      - key: kubernetes.io/os
        operator: In
        values: ["linux"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand", "spot"]  # Mix Spot and On-Demand
      - key: karpenter.azure.com/sku-family
        operator: In
        values: ["D", "E"]            # General + Memory optimized
      - key: karpenter.azure.com/sku-version
        operator: In
        values: ["5", "6"]            # v5 and v6 generations only
```

---

## 10. The AKSNodeClass CRD — Azure-Specific Configuration

```yaml
apiVersion: karpenter.azure.com/v1beta1
kind: AKSNodeClass
metadata:
  name: default
spec:
  # OS image family: Ubuntu or AzureLinux
  imageFamily: Ubuntu

  # OS disk size (default: 128 GB, minimum: 30 GB)
  osDiskSizeGB: 128

  # Custom VNet subnet (optional)
  vnetSubnetID: "/subscriptions/.../subnets/aks-subnet"

  # FIPS compliance mode
  fipsMode: Disabled        # or FIPS

  # Max pods per node (10-250, default: 110)
  maxPods: 110

  # Kubelet configuration
  kubelet:
    maxPods: 110
```

---

## 11. SKU Selection — Families, Architectures, Spot Instances

### Available Selectors

| Selector | Example Values | Description |
|----------|---------------|-------------|
| `karpenter.azure.com/sku-family` | `D`, `E`, `F`, `L`, `N` | VM family |
| `karpenter.azure.com/sku-name` | `Standard_D4s_v5` | Exact VM SKU |
| `karpenter.azure.com/sku-version` | `3`, `5`, `6` | Generation |
| `kubernetes.io/arch` | `amd64`, `arm64` | CPU architecture |
| `karpenter.sh/capacity-type` | `on-demand`, `spot` | Pricing model |

### Spot Priority

When both Spot and On-Demand are specified, Karpenter **prioritizes Spot** instances first (cheapest). If Spot is unavailable, it falls back to On-Demand.

---

## 12. Consolidation — Automatic Cost Optimization

Karpenter continuously evaluates whether nodes can be consolidated:

```
BEFORE consolidation:
  Node A (D8s_v5, 8 CPU): Using 2 CPU  ← 25% utilized
  Node B (D8s_v5, 8 CPU): Using 3 CPU  ← 37% utilized
  Total: 16 CPU allocated, 5 CPU used

AFTER consolidation:
  Node C (D8s_v5, 8 CPU): Using 5 CPU  ← 62% utilized
  Node A: DELETED (pods moved to C)
  Node B: DELETED (pods moved to C)
  Total: 8 CPU allocated, 5 CPU used
  Savings: 50% fewer nodes!
```

**Consolidation policies:**
- `WhenEmpty` — Only remove nodes with zero pods
- `WhenEmptyOrUnderutilized` — Remove empty OR underutilized nodes (recommended)

---

## 13. Disruption Policies and Node Disruption Budgets

Control how Karpenter disrupts nodes:

```yaml
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    budgets:
    - nodes: "20%"           # Max 20% disrupted simultaneously
    - nodes: "0"             # During maintenance window: no disruption
      schedule: "0 9 * * 1-5"  # Mon-Fri 9 AM
      duration: 8h              # For 8 hours
```

---

## 14. Node Image Upgrades for NAP

NAP nodes are **automatically updated** when new images are available. You can control timing with maintenance windows:

```bash
az aks maintenanceconfiguration add \
  -g my-rg --cluster-name my-aks \
  --name aksManagedNodeOSUpgradeSchedule \
  --schedule-type Weekly \
  --day-of-week Sunday \
  --start-time 01:00 \
  --duration 4
```

NAP forces image updates if the node image is older than **90 days**.

---

## 15. Troubleshooting NAP

### View Karpenter Logs

```bash
# Enable control plane logs, then query:
AKSControlPlane
| where Category == "karpenter-events"
| order by TimeGenerated desc
```

### Common Issues

**Pods stuck Pending:** Check NodePool limits (CPU/memory caps reached)

**No SKU available:** Broaden SKU families in requirements

**Subnet errors:** Verify Network Contributor role on VNet

---

## 16. Best Practices and Recommendations

✅ **Set accurate pod resource requests** — Karpenter uses these to right-size nodes

✅ **Use multiple SKU families** — Broader selection = better availability and pricing

✅ **Enable Spot + On-Demand** — Karpenter prioritizes Spot for cost savings

✅ **Set resource limits** on NodePools — Prevent runaway scaling

✅ **Use `WhenEmptyOrUnderutilized` consolidation** — Maximum cost savings

✅ **Set `expireAfter`** — Force node refresh (e.g., 720h = 30 days)

✅ **Configure maintenance windows** — Control when node image updates happen

✅ **Monitor with Azure Monitor** — Track Karpenter metrics and events

---

## References & Further Reading

- [NAP Overview (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/node-auto-provisioning)
- [Enable NAP (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/use-node-auto-provisioning)
- [Configure NodePools (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/node-auto-provisioning-node-pools)
- [Configure AKSNodeClass (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/node-auto-provisioning-aksnodeclass)
- [NAP Networking (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/node-auto-provisioning-networking)
- [NAP Node Image Upgrades (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/node-auto-provisioning-upgrade-image)
- [Karpenter Documentation](https://karpenter.sh/)
- [AKS Karpenter Provider (GitHub)](https://github.com/Azure/karpenter-provider-azure)