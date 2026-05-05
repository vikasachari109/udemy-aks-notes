# AKS Upgrade Options — End-to-End Guide
### The Ultimate Beginner-Friendly Deep Dive

> **What this document covers:** What needs upgrading in AKS (control plane, node pools, node images, OS), the Kubernetes version support policy (N-2 rule), manual vs automatic upgrades, all 5 auto-upgrade channels, all 4 node OS upgrade channels, maintenance windows, the upgrade process step-by-step (surge, drain, cordon), what Azure manages vs what you manage, Pod Disruption Budgets, blue-green strategies, Long Term Support, troubleshooting, and complete best practices — all explained from absolute scratch.

---

## Table of Contents
1. [What Needs Upgrading in AKS?](#1-what-needs-upgrading-in-aks)
2. [Kubernetes Version Support Policy — The N-2 Rule](#2-kubernetes-version-support-policy--the-n-2-rule)
3. [What Azure Manages vs What You Manage](#3-what-azure-manages-vs-what-you-manage)
4. [The 3 Types of AKS Upgrades](#4-the-3-types-of-aks-upgrades)
5. [How the Upgrade Process Works Step by Step](#5-how-the-upgrade-process-works-step-by-step)
6. [Manual Upgrades — Control Plane and Node Pools](#6-manual-upgrades--control-plane-and-node-pools)
7. [Cluster Auto-Upgrade Channels](#7-cluster-auto-upgrade-channels)
8. [Node OS Auto-Upgrade Channels](#8-node-os-auto-upgrade-channels)
9. [Maintenance Windows — Scheduling Upgrades](#9-maintenance-windows--scheduling-upgrades)
10. [Max Surge — Controlling Upgrade Speed and Disruption](#10-max-surge--controlling-upgrade-speed-and-disruption)
11. [Pod Disruption Budgets (PDBs)](#11-pod-disruption-budgets-pdbs)
12. [Blue-Green Upgrade Strategy](#12-blue-green-upgrade-strategy)
13. [Long Term Support (LTS)](#13-long-term-support-lts)
14. [OS Version Upgrades (Ubuntu, Azure Linux)](#14-os-version-upgrades-ubuntu-azure-linux)
15. [Troubleshooting Upgrade Failures](#15-troubleshooting-upgrade-failures)
16. [Best Practices and Recommendations](#16-best-practices-and-recommendations)

---

## 1. What Needs Upgrading in AKS?

### The 4 Layers That Need Upgrading

An AKS cluster has multiple layers, each upgraded independently:

```
┌──────────────────────────────────────────────────────────────────┐
│  Layer 1: KUBERNETES VERSION (Control Plane)                     │
│  What: API server, etcd, scheduler, controller manager           │
│  Example: 1.29.4 → 1.30.2                                       │
│  Frequency: ~Every 3-4 months (new minor versions)               │
│  Managed by: Azure (but YOU trigger the upgrade)                │
│                                                                  │
│  Layer 2: KUBERNETES VERSION (Node Pools)                        │
│  What: kubelet, kube-proxy on each worker node                   │
│  Example: 1.29.4 → 1.30.2                                       │
│  Can be different from control plane (up to N-2)                │
│  Managed by: Azure (but YOU trigger the upgrade)                │
│                                                                  │
│  Layer 3: NODE IMAGE                                             │
│  What: The OS image (Ubuntu/Azure Linux) with all patches       │
│  Example: AKSUbuntu-2204-202410.27.0 → AKSUbuntu-2204-202411.05│
│  Frequency: Weekly (Linux), Monthly (Windows)                   │
│  Managed by: Azure creates images; YOU apply them               │
│                                                                  │
│  Layer 4: NODE OS SECURITY PATCHES                               │
│  What: Kernel patches, CVE fixes, security hotfixes              │
│  Frequency: As needed (can be daily for critical CVEs)          │
│  Managed by: Depends on channel (Unmanaged, SecurityPatch, etc.)│
└──────────────────────────────────────────────────────────────────┘
```

### Why All 4 Layers Matter

```
Layer 1: K8s version (Control Plane)
  → New Kubernetes API features, deprecations, security fixes
  → MUST stay within N-2 support window

Layer 2: K8s version (Node Pools)
  → kubelet/kube-proxy on nodes must be compatible with control plane
  → Can lag up to 2 minor versions behind control plane

Layer 3: Node Image
  → Contains OS kernel, container runtime (containerd), CSI drivers
  → New images weekly with bug fixes and security patches
  → Old images may have known vulnerabilities

Layer 4: OS Security Patches
  → Critical CVE fixes (e.g., kernel exploits)
  → Applied between node image releases
  → May or may not require node reboot
```

---

## 2. Kubernetes Version Support Policy — The N-2 Rule

### What is N-2?

AKS supports the **3 most recent GA minor versions** of Kubernetes at any time. If the latest version is 1.31, then:

```
1.31 = N   (latest, fully supported)
1.30 = N-1 (supported)
1.29 = N-2 (supported)
1.28 = N-3 (PLATFORM SUPPORT ONLY — reduced support, no new patches)
1.27 = UNSUPPORTED (will be auto-upgraded to N-2)
```

### Semantic Versioning

```
1.30.4
│  │  │
│  │  └── Patch version (bug fixes, CVEs)
│  └───── Minor version (new features, deprecations)
└──────── Major version (breaking changes — Kubernetes has been v1 since 2015)
```

### Version Upgrade Rules

**Rule 1:** You can only upgrade **one minor version at a time**
```
✅ 1.29 → 1.30     (one minor version jump)
✅ 1.30 → 1.31     (one minor version jump)
❌ 1.29 → 1.31     (skipping 1.30 — NOT allowed)
```

**Rule 2:** Control plane version must always be **≥ node pool version**
```
✅ Control Plane: 1.31, Node Pools: 1.29  (2 versions behind — OK)
✅ Control Plane: 1.31, Node Pools: 1.31  (same version — OK)
❌ Control Plane: 1.29, Node Pools: 1.31  (nodes ahead — NOT possible)
```

**Rule 3:** Upgrade control plane first, then node pools
```
Step 1: Control Plane 1.29 → 1.30
Step 2: Node Pool A    1.29 → 1.30
Step 3: Node Pool B    1.29 → 1.30
```

### Checking Available Upgrades

```bash
# Check current version
az aks show --resource-group my-rg --name my-aks \
  --query "kubernetesVersion" -o tsv
# 1.29.4

# Check available upgrade versions
az aks get-upgrades --resource-group my-rg --name my-aks -o table
# Name    ResourceGroup    MasterVersion    Upgrades
# -----   -------------    -------------    --------
# default my-rg            1.29.4           1.29.7, 1.30.0, 1.30.2, 1.30.4
```

### Version Release Cadence

| Event | Frequency |
|-------|-----------|
| New Kubernetes minor version (upstream) | ~Every 4 months |
| AKS GA release of new minor version | ~2-4 weeks after upstream |
| Patch version releases | Monthly or as-needed for CVEs |
| Oldest supported version deprecated | ~30 days after new minor is GA |
| Node image updates (Linux) | Weekly |
| Node image updates (Windows) | Monthly |

---

## 3. What Azure Manages vs What You Manage

### The Shared Responsibility Model

```
┌──────────────────────────────────────────────────────────────────┐
│              AZURE MANAGES (Fully Automated)                     │
│                                                                  │
│  ✅ Control plane infrastructure (API server, etcd, scheduler)   │
│  ✅ Control plane high availability (multi-zone)                │
│  ✅ Control plane security patches (automatic, transparent)      │
│  ✅ etcd backups (automatic, every 5 minutes)                   │
│  ✅ Certificate rotation (automatic)                            │
│  ✅ Creating new node images (weekly)                           │
│  ✅ CSI driver updates (bundled with node images)               │
│  ✅ Azure Load Balancer management                              │
│  ✅ Kubernetes API server availability (99.95% SLA with SLA tier)│
├──────────────────────────────────────────────────────────────────┤
│              YOU MANAGE (Your Responsibility)                     │
│                                                                  │
│  ⚠️ Triggering Kubernetes version upgrades (or enabling auto)   │
│  ⚠️ Triggering node image upgrades (or enabling auto)           │
│  ⚠️ Choosing auto-upgrade channels                              │
│  ⚠️ Setting maintenance windows                                 │
│  ⚠️ Testing upgrades in staging before production               │
│  ⚠️ Pod Disruption Budgets (PDBs) for your workloads           │
│  ⚠️ Application compatibility testing with new K8s versions     │
│  ⚠️ Monitoring upgrade progress and health                     │
│  ⚠️ OS version migration (Ubuntu 22.04 → 24.04)                │
│  ⚠️ Handling deprecated Kubernetes APIs in your manifests       │
│  ⚠️ Node pool scaling during upgrades                           │
│  ⚠️ Backup and recovery strategy                                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. The 3 Types of AKS Upgrades

### Type 1: Full Cluster Upgrade (Control Plane + All Node Pools)

```bash
az aks upgrade \
  --resource-group my-rg \
  --name my-aks \
  --kubernetes-version 1.30.4
```

**What happens:**
1. Control plane upgrades first (5-15 minutes)
2. Each node pool upgrades sequentially
3. Within each pool, nodes upgrade one by one (rolling)

### Type 2: Control Plane Only Upgrade

```bash
az aks upgrade \
  --resource-group my-rg \
  --name my-aks \
  --kubernetes-version 1.30.4 \
  --control-plane-only
```

**What happens:**
1. Only the API server, etcd, scheduler upgrade
2. Node pools stay at their current version
3. You upgrade node pools separately later

**Use case:** Validate API compatibility before upgrading nodes.

### Type 3: Node Image Only Upgrade (No K8s Version Change)

```bash
az aks nodepool upgrade \
  --resource-group my-rg \
  --cluster-name my-aks \
  --name nodepool1 \
  --node-image-only
```

**What happens:**
1. Kubernetes version stays the same
2. Node OS image is updated (latest patches, kernel, containerd)
3. Nodes are reimaged one by one (rolling)

**Use case:** Apply weekly security patches without changing K8s version.

---

## 5. How the Upgrade Process Works Step by Step

### The Rolling Upgrade Process

When upgrading a node pool, AKS performs a **rolling upgrade** — replacing nodes one at a time:

```
Node Pool: 3 nodes (v1.29) → Target: v1.30
Max Surge: 1 (default)

STEP 1: Create 1 surge node (v1.30)
┌─────────────────────────────────────────────────────┐
│  node-0 (v1.29) ✅ Running pods                     │
│  node-1 (v1.29) ✅ Running pods                     │
│  node-2 (v1.29) ✅ Running pods                     │
│  node-3 (v1.30) 🆕 NEW surge node (empty)           │
└─────────────────────────────────────────────────────┘

STEP 2: Cordon node-0 (mark unschedulable)
  node-0 is marked as "no new pods can be scheduled here"

STEP 3: Drain node-0 (evict all pods)
  All pods on node-0 are gracefully terminated
  Kubernetes scheduler moves them to other nodes (including node-3)

STEP 4: Delete node-0

STEP 5: Create new node-0 (v1.30)
┌─────────────────────────────────────────────────────┐
│  node-0 (v1.30) 🆕 New node                         │
│  node-1 (v1.29) ✅ Running pods                     │
│  node-2 (v1.29) ✅ Running pods                     │
│  node-3 (v1.30) ✅ Running migrated pods            │
└─────────────────────────────────────────────────────┘

STEP 6: Repeat for node-1 (cordon → drain → delete → create new)

STEP 7: Repeat for node-2

STEP 8: Delete surge node-3
┌─────────────────────────────────────────────────────┐
│  node-0 (v1.30) ✅ Running pods                     │
│  node-1 (v1.30) ✅ Running pods                     │
│  node-2 (v1.30) ✅ Running pods                     │
│  ✅ UPGRADE COMPLETE                                 │
└─────────────────────────────────────────────────────┘
```

### Pre-Upgrade Validations

Before starting, AKS automatically checks:
- ✅ Valid upgrade path (no skipping minor versions)
- ✅ Sufficient IP addresses in subnet (for surge nodes)
- ✅ Sufficient vCPU quota (for surge nodes)
- ✅ PDB configurations (warns if misconfigured)
- ✅ Certificates not expired
- ✅ Deprecated API usage detection

---

## 6. Manual Upgrades — Control Plane and Node Pools

### Full Manual Upgrade Workflow

```bash
# Step 1: Check current versions
az aks show -g my-rg -n my-aks \
  --query "{ControlPlane:kubernetesVersion, NodePools:agentPoolProfiles[].{Name:name,Version:currentOrchestratorVersion}}" -o table

# Step 2: Check available upgrades
az aks get-upgrades -g my-rg -n my-aks -o table

# Step 3: Upgrade control plane first
az aks upgrade -g my-rg -n my-aks \
  --kubernetes-version 1.30.4 \
  --control-plane-only

# Step 4: Upgrade node pools one at a time
az aks nodepool upgrade \
  -g my-rg --cluster-name my-aks \
  --name systempool \
  --kubernetes-version 1.30.4

az aks nodepool upgrade \
  -g my-rg --cluster-name my-aks \
  --name userpool \
  --kubernetes-version 1.30.4

# Step 5: Verify
az aks show -g my-rg -n my-aks \
  --query "{ControlPlane:kubernetesVersion, NodePools:agentPoolProfiles[].{Name:name,Version:currentOrchestratorVersion}}"
```

### Node Image Only Upgrade

```bash
# Check current node image version
az aks nodepool show -g my-rg --cluster-name my-aks -n nodepool1 \
  --query nodeImageVersion

# Check available node images
az aks nodepool get-upgrades \
  -g my-rg --cluster-name my-aks -n nodepool1

# Upgrade node image only
az aks nodepool upgrade \
  -g my-rg --cluster-name my-aks \
  -n nodepool1 --node-image-only
```

---

## 7. Cluster Auto-Upgrade Channels

### The 5 Channels

AKS offers 5 auto-upgrade channels for Kubernetes version upgrades:

| Channel | What It Does | Risk Level | Best For |
|---------|-------------|------------|----------|
| **none** | No automatic upgrades | Manual control | Teams that test every upgrade manually |
| **patch** | Auto-upgrades to latest patch (e.g., 1.30.2 → 1.30.4) | Low | Production (conservative) |
| **stable** | Auto-upgrades to N-1 minor version | Medium | **Production (recommended)** |
| **rapid** | Auto-upgrades to latest GA minor version | Higher | Dev/test, staging |
| **node-image** | Only auto-upgrades node images (not K8s version) | Low | When you manage K8s version manually |

### Channel Behavior Explained

```
Current latest GA: 1.31

Channel: none
  → No changes. You upgrade manually.

Channel: patch
  → If you're on 1.30.2, upgrades to 1.30.4 (latest patch)
  → Does NOT upgrade to 1.31 (that's a minor version)

Channel: stable
  → Upgrades to N-1 = 1.30.x (one behind latest)
  → Waits for 1.30 to be "proven" before applying

Channel: rapid
  → Upgrades to N = 1.31.x (latest)
  → Applied as soon as it's GA

Channel: node-image
  → K8s version unchanged
  → Node images updated weekly
```

### Setting the Auto-Upgrade Channel

```bash
# Set to stable (recommended for production)
az aks update -g my-rg -n my-aks \
  --auto-upgrade-channel stable

# Set to patch (most conservative)
az aks update -g my-rg -n my-aks \
  --auto-upgrade-channel patch

# Disable auto-upgrade
az aks update -g my-rg -n my-aks \
  --auto-upgrade-channel none

# Check current channel
az aks show -g my-rg -n my-aks \
  --query "autoUpgradeProfile"
```

### Recommended Channels by Environment

| Environment | Cluster Channel | Node OS Channel | Why |
|------------|----------------|-----------------|-----|
| **Production** | `stable` | `SecurityPatch` | Proven versions, minimal disruption |
| **Staging** | `stable` | `NodeImage` | Match production, test first |
| **Development** | `rapid` | `NodeImage` | Latest features, early testing |
| **Regulated/Finance** | `patch` or `none` | `SecurityPatch` | Maximum control |

---

## 8. Node OS Auto-Upgrade Channels

### Separate from Cluster Upgrades

Node OS upgrades are **independent** from Kubernetes version upgrades. You configure them separately:

| Channel | What It Does | Disruption | Reboot Required? |
|---------|-------------|-----------|-----------------|
| **None** | No OS updates at all | Zero | No |
| **Unmanaged** | Ubuntu/Azure Linux nightly apt updates | Low | You manage reboots (use kured) |
| **SecurityPatch** | AKS-tested security patches only | Low-Medium | Only when kernel patches need it |
| **NodeImage** | Full node image replacement weekly | Medium | Yes (node reimaged) |

### SecurityPatch vs NodeImage

```
SecurityPatch:
  ┌──────────────────────────────────────────┐
  │ • Patches applied IN-PLACE when possible │
  │ • Only reimages node when NECESSARY      │
  │ • Faster (less disruption)               │
  │ • Typically 5 days after CVE published   │
  │ • Extra cost: VHD storage in your RG     │
  │ • Best for: Production                   │
  └──────────────────────────────────────────┘

NodeImage:
  ┌──────────────────────────────────────────┐
  │ • Full VHD replacement every week        │
  │ • Always reimages all nodes              │
  │ • More disruptive (drain + reimage)      │
  │ • Contains ALL updates (security + bugs) │
  │ • No extra storage cost                  │
  │ • Best for: Staging, dev                 │
  └──────────────────────────────────────────┘
```

### Setting Node OS Channel

```bash
# Set to SecurityPatch (recommended for production)
az aks update -g my-rg -n my-aks \
  --node-os-upgrade-channel SecurityPatch

# Set to NodeImage (full weekly updates)
az aks update -g my-rg -n my-aks \
  --node-os-upgrade-channel NodeImage

# Check current setting
az aks show -g my-rg -n my-aks \
  --query "autoUpgradeProfile.nodeOSUpgradeChannel"
```

---

## 9. Maintenance Windows — Scheduling Upgrades

### Why Maintenance Windows?

Without a maintenance window, auto-upgrades can happen at any time — potentially during peak traffic. Maintenance windows let you control **when** upgrades occur.

### Two Separate Windows

```
┌─────────────────────────────────────────────────────────────────┐
│  Window 1: aksManagedAutoUpgradeSchedule                        │
│  Controls: Kubernetes version auto-upgrades                     │
│  Example: "Upgrade K8s version on Saturdays at 2 AM"           │
│                                                                  │
│  Window 2: aksManagedNodeOSUpgradeSchedule                      │
│  Controls: Node OS security patch auto-upgrades                 │
│  Example: "Apply OS patches on Sundays at 3 AM"                │
└─────────────────────────────────────────────────────────────────┘
```

### Creating Maintenance Windows

```bash
# Cluster auto-upgrade window (Saturdays 2-6 AM UTC)
az aks maintenanceconfiguration add \
  -g my-rg --cluster-name my-aks \
  --name aksManagedAutoUpgradeSchedule \
  --schedule-type Weekly \
  --day-of-week Saturday \
  --start-time 02:00 \
  --utc-offset +00:00 \
  --duration 4

# Node OS upgrade window (Sundays 3-7 AM UTC)
az aks maintenanceconfiguration add \
  -g my-rg --cluster-name my-aks \
  --name aksManagedNodeOSUpgradeSchedule \
  --schedule-type Weekly \
  --day-of-week Sunday \
  --start-time 03:00 \
  --utc-offset +00:00 \
  --duration 4
```

**Minimum window duration:** 4 hours (recommended by Microsoft).

---

## 10. Max Surge — Controlling Upgrade Speed and Disruption

### What is Max Surge?

**Max surge** controls how many extra nodes are created during an upgrade. More surge = faster upgrade but more resources needed.

| Max Surge | Behavior | Speed | Resource Cost |
|-----------|----------|-------|-------------|
| `1` (default) | 1 extra node at a time | Slow, safe | Low |
| `33%` | 33% extra nodes (e.g., 1 extra for 3-node pool) | Medium | Medium |
| `50%` | 50% extra nodes | Fast | High |
| `100%` | Double all nodes at once | Fastest (blue-green) | Highest |

### Setting Max Surge

```bash
# Set max surge to 33% (recommended for production)
az aks nodepool update \
  -g my-rg --cluster-name my-aks \
  -n nodepool1 \
  --max-surge 33%

# Set max surge to 1 node (most conservative)
az aks nodepool update \
  -g my-rg --cluster-name my-aks \
  -n nodepool1 \
  --max-surge 1
```

### Zone Balancing Tip

For multi-zone clusters, set max surge to a **multiple of 3** to maintain zone balance:
```
3-node pool:  max-surge = 3  (one per zone)
6-node pool:  max-surge = 3  (one per zone)
9-node pool:  max-surge = 3  (one per zone)
```

---

## 11. Pod Disruption Budgets (PDBs)

### What is a PDB?

A **Pod Disruption Budget** tells Kubernetes how many pods of a given application can be unavailable during voluntary disruptions (like node upgrades).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2            # At least 2 pods must always be running
  # OR
  # maxUnavailable: 1        # At most 1 pod can be down at a time
  selector:
    matchLabels:
      app: my-app
```

### How PDBs Affect Upgrades

During a node upgrade, Kubernetes drains pods from the old node. If evicting a pod would violate a PDB, the drain **waits** (up to the drain timeout).

```
Deployment: my-app (3 replicas)
PDB: minAvailable: 2

Node upgrade starts → Drain node-0:
  Pod my-app-abc on node-0 needs to be evicted
  Currently running: 3 pods (node-0, node-1, node-2)
  After eviction: 2 pods → Meets PDB (minAvailable: 2) → ✅ Allowed

  Next: Pod my-app-def on node-0 also needs eviction
  Currently running: 2 pods (node-1, node-2)
  After eviction: 1 pod → Violates PDB (minAvailable: 2) → ❌ BLOCKED
  Wait until rescheduled pod is Running on another node...
  Now 3 pods again → Eviction allowed → ✅ Continue
```

### Common PDB Mistake That Blocks Upgrades

```yaml
# ❌ BAD: This blocks ALL evictions!
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: bad-pdb
spec:
  maxUnavailable: 0          # No pods can ever be unavailable
  selector:
    matchLabels:
      app: my-app
# Result: Upgrade hangs forever, then fails with timeout
```

**Fix:** Always allow at least 1 pod to be unavailable:
```yaml
maxUnavailable: 1    # Allow 1 pod down during upgrades
```

---

## 12. Blue-Green Upgrade Strategy

### What is Blue-Green?

Instead of in-place rolling upgrades, you create a **second node pool** with the new version, migrate workloads, then delete the old pool.

```
BEFORE:
┌──────────────────────────────────────────────┐
│  Node Pool "blue" (v1.29) — 3 nodes          │
│  All pods running here                       │
└──────────────────────────────────────────────┘

STEP 1: Create new pool
┌──────────────────────────────────────────────┐
│  Node Pool "blue"  (v1.29) — 3 nodes ✅      │
│  Node Pool "green" (v1.30) — 3 nodes 🆕     │
└──────────────────────────────────────────────┘

STEP 2: Cordon old pool, pods migrate to new
┌──────────────────────────────────────────────┐
│  Node Pool "blue"  (v1.29) — CORDONED 🚫     │
│  Node Pool "green" (v1.30) — All pods here ✅│
└──────────────────────────────────────────────┘

STEP 3: Validate, then delete old pool
┌──────────────────────────────────────────────┐
│  Node Pool "green" (v1.30) — 3 nodes ✅      │
│  ✅ UPGRADE COMPLETE                         │
└──────────────────────────────────────────────┘
```

### Blue-Green CLI Commands

```bash
# Step 1: Add new node pool with target version
az aks nodepool add \
  -g my-rg --cluster-name my-aks \
  --name green \
  --kubernetes-version 1.30.4 \
  --node-vm-size Standard_D4s_v5 \
  --node-count 3 \
  --zones 1 2 3

# Step 2: Cordon old node pool
kubectl cordon -l agentpool=blue

# Step 3: Drain old node pool
kubectl drain -l agentpool=blue \
  --ignore-daemonsets \
  --delete-emptydir-data

# Step 4: Validate everything is running on green
kubectl get pods -o wide --all-namespaces

# Step 5: Delete old node pool
az aks nodepool delete \
  -g my-rg --cluster-name my-aks \
  --name blue
```

**Advantages:** Zero-downtime, instant rollback (just uncordon blue)
**Disadvantages:** Double the cost during migration, more complex

---

## 13. Long Term Support (LTS)

### What is LTS?

Standard AKS support for a Kubernetes minor version is **~12 months**. With **Long Term Support (LTS)**, you get **2 years** of support for select versions.

| Feature | Standard | LTS |
|---------|---------|-----|
| **Support duration** | ~12 months | 2 years |
| **Patch updates** | Yes | Yes (extended) |
| **Security fixes** | Yes | Yes (extended) |
| **Cost** | Included in Standard tier | Requires **Premium tier** ($0.10/cluster/hr) |
| **Versions supported** | N, N-1, N-2 | Specific LTS versions (e.g., 1.27, 1.30) |

### Enabling LTS

```bash
az aks update -g my-rg -n my-aks \
  --tier premium \
  --k8s-support-plan AKSLongTermSupport
```

---

## 14. OS Version Upgrades (Ubuntu, Azure Linux)

### Current OS Versions

| OS SKU | Status | End of Life |
|--------|--------|------------|
| Ubuntu 20.04 | ❌ Retired March 2027 | Migrate to 24.04 |
| Ubuntu 22.04 | ⚠️ Supported until June 2027 | Migrate to 24.04 |
| **Ubuntu 24.04** | ✅ Current default (K8s 1.35+) | Active |
| Azure Linux 2.0 | ❌ Retired Nov 2025 | Migrate to 3.0 |
| **Azure Linux 3.0** | ✅ Current default | Active |

### How to Migrate OS Version

```bash
# Check current OS SKU
az aks nodepool show -g my-rg --cluster-name my-aks -n nodepool1 \
  --query osSku

# Upgrade by creating new node pool with target OS
az aks nodepool add \
  -g my-rg --cluster-name my-aks \
  --name newpool \
  --os-sku Ubuntu2404 \
  --kubernetes-version 1.32.0 \
  --node-count 3

# Migrate workloads, then delete old pool
```

---

## 15. Troubleshooting Upgrade Failures

### Common Failure: PDB Blocking Drain

```
Error: Cannot evict pod as it would violate the pod's disruption budget
```
**Fix:** Review and fix PDB (ensure `maxUnavailable >= 1`)

### Common Failure: Insufficient IP Addresses

```
Error: InsufficientSubnetSize
```
**Fix:** Ensure subnet has enough IPs for surge nodes

### Common Failure: Quota Exceeded

```
Error: QuotaExceeded
```
**Fix:** Request quota increase in Azure Portal → Quotas

### Force Upgrade (Last Resort)

```bash
az aks upgrade -g my-rg -n my-aks \
  --kubernetes-version 1.30.4 \
  --enable-force-upgrade \
  --upgrade-override-until 2025-12-01T00:00:00Z
```
⚠️ **Warning:** Force upgrade bypasses PDBs and may cause downtime.

---

## 16. Best Practices and Recommendations

### Upgrade Strategy

✅ **Enable auto-upgrade** with `stable` channel for production

✅ **Enable node OS auto-upgrade** with `SecurityPatch` channel

✅ **Set maintenance windows** during low-traffic hours (minimum 4 hours)

✅ **Test upgrades in staging first** before production

✅ **Use blue-green strategy** for critical production clusters

### PDB Configuration

✅ **Always set PDBs** for production workloads

✅ **Never set `maxUnavailable: 0`** — it blocks upgrades

✅ **Ensure enough replicas** to satisfy PDB during drain

### Monitoring

✅ **Monitor upgrade progress** via Azure Activity Log

✅ **Set up alerts** for upgrade failures

✅ **Check AKS release notes** before every upgrade

✅ **Use `az aks get-upgrades`** to plan ahead

### Version Management

✅ **Stay within N-2** — don't let versions fall behind

✅ **Upgrade at least every 3-4 months** (when new minor releases)

✅ **Handle deprecated APIs** before upgrading (check with `kubectl convert`)

✅ **Consider LTS** for regulated environments needing stability

---

## References & Further Reading

- [AKS Upgrade Overview (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/upgrade-cluster-components)
- [Upgrade AKS Control Plane (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/upgrade-aks-control-plane)
- [Upgrade Node Images (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/upgrade-node-image)
- [Auto-Upgrade Cluster (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/auto-upgrade-cluster)
- [Auto-Upgrade Node OS Images (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/auto-upgrade-node-os-image)
- [Upgrade Options and Recommendations (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/upgrade-options)
- [AKS Supported Kubernetes Versions (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions)
- [Planned Maintenance Windows (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/planned-maintenance)
- [AKS Day-2 Patch & Upgrade Guide (Azure Architecture Center)](https://learn.microsoft.com/en-us/azure/architecture/operator-guides/aks/aks-upgrade-practices)
- [Blue-Green Deployment of AKS Clusters (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/upgrade-blue-green)
- [AKS Release Notes & Tracker](https://github.com/Azure/AKS/releases)