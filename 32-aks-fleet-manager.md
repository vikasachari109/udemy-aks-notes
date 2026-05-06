# Azure Kubernetes Fleet Manager
### The Ultimate Beginner-Friendly Deep Dive

> **What this document covers:** What Fleet Manager is, why you need multi-cluster management, fleet types (with/without hub), member clusters, update runs and strategies, resource propagation, multi-cluster load balancing, auto-upgrade profiles, managed namespaces, and complete setup workflows — all explained from absolute scratch.

---

## Table of Contents
1. [The Problem: Managing Multiple AKS Clusters](#1-the-problem-managing-multiple-aks-clusters)
2. [What is Azure Kubernetes Fleet Manager?](#2-what-is-azure-kubernetes-fleet-manager)
3. [Fleet Types: With and Without Hub Cluster](#3-fleet-types-with-and-without-hub-cluster)
4. [What Azure Manages vs What You Manage](#4-what-azure-manages-vs-what-you-manage)
5. [Core Concepts: Fleets, Members, Hub Cluster](#5-core-concepts-fleets-members-hub-cluster)
6. [Creating a Fleet and Joining Members](#6-creating-a-fleet-and-joining-members)
7. [Update Orchestration — Safe Multi-Cluster Upgrades](#7-update-orchestration--safe-multi-cluster-upgrades)
8. [Update Strategies: Stages, Groups, and Ordering](#8-update-strategies-stages-groups-and-ordering)
9. [Auto-Upgrade Profiles](#9-auto-upgrade-profiles)
10. [Resource Propagation — Deploying Across Clusters](#10-resource-propagation--deploying-across-clusters)
11. [DNS-Based Multi-Cluster Load Balancing](#11-dns-based-multi-cluster-load-balancing)
12. [Managed Fleet Namespaces](#12-managed-fleet-namespaces)
13. [Networking and Security](#13-networking-and-security)
14. [Monitoring and Troubleshooting](#14-monitoring-and-troubleshooting)
15. [Best Practices and Recommendations](#15-best-practices-and-recommendations)

---

## 1. The Problem: Managing Multiple AKS Clusters

### Why Multiple Clusters?

Enterprises typically run multiple AKS clusters for:
- **Multi-region** — Low latency for global users (East US + West Europe + Japan East)
- **Environment isolation** — Dev, staging, production clusters
- **Compliance** — Data residency requirements per region
- **Blast radius** — Limit impact of failures to one cluster
- **Team isolation** — Different teams own different clusters

### The Management Challenge

With 10+ clusters, you face:
- ❌ Upgrading Kubernetes versions cluster by cluster (tedious, error-prone)
- ❌ Deploying apps to each cluster individually
- ❌ No centralized view of cluster health
- ❌ Inconsistent configurations across clusters
- ❌ No coordinated rollout strategy (upgrade dev→staging→prod)

**Fleet Manager solves all of these.**

---

## 2. What is Azure Kubernetes Fleet Manager?

**Azure Kubernetes Fleet Manager** is a service that provides **at-scale management** of multiple AKS clusters. It enables:

```
┌──────────────────────────────────────────────────────────────────┐
│              Fleet Manager Capabilities                          │
│                                                                  │
│  1. UPDATE ORCHESTRATION                                        │
│     Safely upgrade K8s versions and node images across          │
│     multiple clusters with controlled rollout                   │
│                                                                  │
│  2. RESOURCE PROPAGATION (requires hub cluster)                 │
│     Deploy Kubernetes resources from a central hub to           │
│     selected member clusters                                    │
│                                                                  │
│  3. MULTI-CLUSTER LOAD BALANCING (requires hub cluster)         │
│     DNS-based traffic distribution across clusters              │
│                                                                  │
│  4. MANAGED NAMESPACES                                          │
│     Enforce quotas, policies, RBAC across fleet namespaces      │
│                                                                  │
│  5. AUTO-UPGRADE PROFILES                                       │
│     Automatically trigger upgrades when new versions release    │
└──────────────────────────────────────────────────────────────────┘
```

**Pricing:** Fleet Manager itself is **free**. You only pay for the hub cluster AKS resources (if you enable a hub).

---

## 3. Fleet Types: With and Without Hub Cluster

| Feature | Without Hub | With Hub |
|---------|------------|----------|
| **Update orchestration** | ✅ Yes | ✅ Yes |
| **Auto-upgrade profiles** | ✅ Yes | ✅ Yes |
| **Resource propagation** | ❌ No | ✅ Yes |
| **Multi-cluster load balancing** | ❌ No | ✅ Yes |
| **Managed namespaces** | ❌ No | ✅ Yes |
| **Cost** | Free | Hub cluster cost (~$70-150/mo) |
| **Use case** | Upgrade orchestration only | Full multi-cluster management |

**Without hub** = ARM-level grouping only (upgrades)
**With hub** = Full Kubernetes control plane for fleet operations

---

## 4. What Azure Manages vs What You Manage

```
┌──────────────────────────────────────────────────────────────────┐
│              AZURE MANAGES                                       │
│                                                                  │
│  ✅ Hub cluster lifecycle (auto-upgraded to latest K8s)         │
│  ✅ KubeFleet controllers on hub cluster                        │
│  ✅ Fleet networking components                                 │
│  ✅ Member cluster agent deployment                             │
│  ✅ Hub cluster security (mutations blocked)                    │
├──────────────────────────────────────────────────────────────────┤
│              YOU MANAGE                                           │
│                                                                  │
│  ⚠️ Creating fleet and joining member clusters                  │
│  ⚠️ Defining update strategies (stages, groups, order)          │
│  ⚠️ Creating and running update runs                            │
│  ⚠️ Defining resource placement policies                       │
│  ⚠️ Monitoring update progress and health                      │
│  ⚠️ Member cluster lifecycle (create, configure, delete)        │
│  ⚠️ Application deployments to hub for propagation             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. Core Concepts: Fleets, Members, Hub Cluster

### Fleet

The top-level resource. A logical grouping of AKS clusters. Supports up to **100 member clusters**.

### Member Cluster

Any AKS cluster (or Arc-enabled K8s cluster in preview) that is **joined** to the fleet. Members can be in different regions, resource groups, or subscriptions (same Entra ID tenant).

### Hub Cluster

A Microsoft-managed AKS cluster that acts as the **control plane** for the fleet. You deploy resources to the hub, and Fleet Manager propagates them to members.

```
┌──────────────────────────────────────────────────────────────────┐
│                    Fleet Manager                                 │
│                                                                  │
│  ┌──────────────────┐                                           │
│  │   Hub Cluster    │  ← Microsoft-managed AKS cluster          │
│  │   (Control Plane)│  ← You deploy resources here              │
│  │                  │  ← Auto-upgraded by Azure                 │
│  └────────┬─────────┘                                           │
│           │                                                      │
│     ┌─────┼─────┬──────────┐                                    │
│     │     │     │          │                                    │
│     ▼     ▼     ▼          ▼                                    │
│  ┌──────┐┌──────┐┌──────┐┌──────┐                              │
│  │AKS   ││AKS   ││AKS   ││AKS   │                              │
│  │East  ││West  ││Japan ││Dev   │                              │
│  │US    ││EU    ││East  ││      │                              │
│  │(Prod)││(Prod)││(Prod)││(Dev) │                              │
│  └──────┘└──────┘└──────┘└──────┘                              │
│   Member  Member  Member  Member                                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. Creating a Fleet and Joining Members

### Create a Fleet (Without Hub)

```bash
az fleet create \
  --resource-group my-fleet-rg \
  --name my-fleet \
  --location westeurope \
  --enable-managed-identity
```

### Create a Fleet (With Hub)

```bash
az fleet create \
  --resource-group my-fleet-rg \
  --name my-fleet \
  --location westeurope \
  --enable-hub \
  --enable-managed-identity
```

### Join Member Clusters

```bash
# Join AKS cluster 1
az fleet member create \
  --resource-group my-fleet-rg \
  --fleet-name my-fleet \
  --name aks-eastus \
  --member-cluster-id "/subscriptions/<sub>/resourceGroups/rg-east/providers/Microsoft.ContainerService/managedClusters/aks-eastus"

# Join AKS cluster 2
az fleet member create \
  --resource-group my-fleet-rg \
  --fleet-name my-fleet \
  --name aks-westeu \
  --member-cluster-id "/subscriptions/<sub>/resourceGroups/rg-west/providers/Microsoft.ContainerService/managedClusters/aks-westeu"

# List members
az fleet member list --resource-group my-fleet-rg --fleet-name my-fleet -o table
```

---

## 7. Update Orchestration — Safe Multi-Cluster Upgrades

### What is an Update Run?

An **update run** is a fleet-wide operation that upgrades Kubernetes versions and/or node images across multiple member clusters in a controlled manner.

```bash
# Create and start an update run
az fleet updaterun create \
  --resource-group my-fleet-rg \
  --fleet-name my-fleet \
  --name upgrade-to-130 \
  --upgrade-type Full \
  --kubernetes-version 1.30.4 \
  --node-image-selection Latest

# Start the run
az fleet updaterun start \
  --resource-group my-fleet-rg \
  --fleet-name my-fleet \
  --name upgrade-to-130
```

### Upgrade Types

| Type | What It Upgrades |
|------|-----------------|
| `Full` | Control plane + node pools + node images |
| `ControlPlaneOnly` | Only the control plane (API server) |
| `NodeImageOnly` | Only node images (no K8s version change) |

### Node Image Selection

| Option | Behavior |
|--------|---------|
| `Latest` | Uses the latest image available in each cluster's region |
| `Consistent` | Uses the latest image common across ALL regions (ensures consistency) |

---

## 8. Update Strategies: Stages, Groups, and Ordering

### Why Strategies?

Without a strategy, all clusters upgrade simultaneously. With a strategy, you control the **order**: dev → staging → production.

```
┌──────────────────────────────────────────────────────────────────┐
│              Update Strategy: "safe-rollout"                     │
│                                                                  │
│  Stage 1: "dev-stage" ──────────────────────────────────────    │
│    Group: dev-clusters                                          │
│    Members: aks-dev                                             │
│    → Upgrade dev first                                          │
│    → Wait for completion                                        │
│                                                                  │
│  Stage 2: "staging-stage" ──────────────────────────────────    │
│    Group: staging-clusters                                      │
│    Members: aks-staging                                         │
│    → Upgrade staging second                                     │
│    → Wait + optional manual approval                            │
│                                                                  │
│  Stage 3: "prod-stage" ─────────────────────────────────────    │
│    Group: prod-eastus, prod-westeu                              │
│    Members: aks-prod-eastus, aks-prod-westeu                    │
│    → Upgrade production last                                    │
│    → Both prod clusters upgrade in parallel within this stage   │
└──────────────────────────────────────────────────────────────────┘
```

### Creating an Update Strategy

```bash
# Create strategy
az fleet updatestrategy create \
  --resource-group my-fleet-rg \
  --fleet-name my-fleet \
  --name safe-rollout \
  --stages '[
    {"name":"dev","groups":[{"name":"dev-group"}]},
    {"name":"staging","groups":[{"name":"staging-group"}],"afterStageWaitInSeconds":3600},
    {"name":"prod","groups":[{"name":"prod-group"}]}
  ]'

# Use strategy in an update run
az fleet updaterun create \
  --resource-group my-fleet-rg \
  --fleet-name my-fleet \
  --name run-2 \
  --upgrade-type Full \
  --kubernetes-version 1.30.4 \
  --update-strategy-name safe-rollout
```

---

## 9. Auto-Upgrade Profiles

Auto-upgrade profiles **automatically trigger** update runs when new versions are available:

```bash
az fleet autoupgradeprofile create \
  --resource-group my-fleet-rg \
  --fleet-name my-fleet \
  --name auto-stable \
  --channel Stable \
  --update-strategy-name safe-rollout
```

### Channels

| Channel | Behavior |
|---------|---------|
| `Stable` | N-1 minor version (recommended for production) |
| `Rapid` | Latest GA version |
| `NodeImage` | Node image updates only |

---

## 10. Resource Propagation — Deploying Across Clusters

**(Requires hub cluster)**

Deploy resources to the hub cluster, and Fleet Manager propagates them to selected member clusters.

```yaml
# Deploy to hub cluster
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app-ns
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: myacr.azurecr.io/myapp:v1
---
# ClusterResourcePlacement: tells Fleet WHERE to deploy
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterResourcePlacement
metadata:
  name: my-app-placement
spec:
  resourceSelectors:
  - group: ""
    kind: Namespace
    name: my-app-ns
    version: v1
  policy:
    placementType: PickAll       # Deploy to ALL member clusters
    # OR
    # placementType: PickN       # Deploy to N clusters
    # numberOfClusters: 2
    # OR
    # placementType: PickFixed   # Deploy to specific clusters
```

---

## 11. DNS-Based Multi-Cluster Load Balancing

**(Preview, requires hub cluster)**

Fleet Manager integrates with Azure Traffic Manager to load-balance traffic across services in multiple clusters.

---

## 12. Managed Fleet Namespaces

Fleet Manager can enforce namespace-level policies across all member clusters:
- Resource quotas
- Network policies
- RBAC role bindings

---

## 13. Networking and Security

- Hub cluster can be **public or private** (cannot change after creation)
- Member clusters communicate with hub via Fleet agent
- All clusters must be in the same **Entra ID tenant**
- Fleet Manager supports **system-assigned and user-assigned managed identities**
- Update runs honor member cluster **maintenance windows**

---

## 14. Monitoring and Troubleshooting

```bash
# Check update run status
az fleet updaterun show \
  --resource-group my-fleet-rg \
  --fleet-name my-fleet \
  --name upgrade-to-130

# List all update runs
az fleet updaterun list \
  --resource-group my-fleet-rg \
  --fleet-name my-fleet -o table

# Access hub cluster
az fleet get-credentials \
  --resource-group my-fleet-rg \
  --name my-fleet

kubectl get memberclusters
kubectl get clusterresourceplacements
```

---

## 15. Best Practices and Recommendations

✅ **Use update strategies** — Always upgrade dev → staging → prod

✅ **Enable auto-upgrade profiles** — Automate recurring upgrades

✅ **Use `Consistent` node image selection** — Ensures same image across regions

✅ **Set maintenance windows on member clusters** — Fleet Manager respects them

✅ **Use hub cluster** for full capabilities (resource propagation, load balancing)

✅ **Start with update orchestration only** — Simplest entry point (no hub needed)

✅ **Monitor update runs** — Check status after each stage completes

✅ **Use `afterStageWaitInSeconds`** — Add validation time between stages

✅ **Test with fewer clusters first** — Validate strategy before fleet-wide rollout

---

## References & Further Reading

- [Fleet Manager Overview (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/overview)
- [Choosing Fleet Type (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/concepts-choosing-fleet)
- [Update Orchestration (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/update-orchestration)
- [Resource Propagation (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/concepts-multi-cluster-workload-management)
- [Fleet Manager FAQ (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/faq)
- [KubeFleet Open Source (CNCF)](https://github.com/Azure/fleet)