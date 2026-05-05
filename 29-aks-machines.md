# AKS Node VM Sizes, Machine Types & Selection Guide
### The Ultimate Beginner-Friendly Deep Dive (Linux Focus)

> **What this document covers:** What Azure VM sizes are, how the naming convention works, every VM family explained (D/E/F/B/L/N/M), system vs user node pools, what enterprises use in production, how to select the right machine type, Generation 1 vs Gen 2, cost optimization strategies, Reserved Instances vs Spot VMs, node pool design patterns, and complete best practices — all explained from absolute scratch with a focus on Linux nodes.

---

## Table of Contents
1. [What is a Node in AKS?](#1-what-is-a-node-in-aks)
2. [Azure VM Naming Convention — Decoding the Names](#2-azure-vm-naming-convention--decoding-the-names)
3. [VM Family Overview — All Families Explained](#3-vm-family-overview--all-families-explained)
4. [General Purpose VMs — The D-Series Workhorses](#4-general-purpose-vms--the-d-series-workhorses)
5. [Burstable VMs — The B-Series for Dev/Test](#5-burstable-vms--the-b-series-for-devtest)
6. [Memory-Optimized VMs — The E-Series and M-Series](#6-memory-optimized-vms--the-e-series-and-m-series)
7. [Compute-Optimized VMs — The F-Series](#7-compute-optimized-vms--the-f-series)
8. [Storage-Optimized VMs — The L-Series](#8-storage-optimized-vms--the-l-series)
9. [GPU VMs — The N-Series for ML/AI](#9-gpu-vms--the-n-series-for-mlai)
10. [Generation 1 vs Generation 2 VMs](#10-generation-1-vs-generation-2-vms)
11. [System Node Pool vs User Node Pool](#11-system-node-pool-vs-user-node-pool)
12. [What Enterprises Actually Use in Production](#12-what-enterprises-actually-use-in-production)
13. [How to Select the Right VM Size — Decision Framework](#13-how-to-select-the-right-vm-size--decision-framework)
14. [Cost Optimization — Pricing Models and Strategies](#14-cost-optimization--pricing-models-and-strategies)
15. [Node Pool Design Patterns](#15-node-pool-design-patterns)
16. [Best Practices and Recommendations](#16-best-practices-and-recommendations)

---

## 1. What is a Node in AKS?

### Nodes = Virtual Machines

In AKS, every **node** is an Azure **Virtual Machine (VM)**. When you create a node pool with 3 nodes, Azure provisions 3 VMs behind the scenes. These VMs run Linux (Ubuntu by default, or Azure Linux), have the Kubernetes **kubelet** agent installed, and join the AKS cluster automatically.

```
┌──────────────────────────────────────────────────────────────┐
│                     AKS Cluster                              │
│                                                              │
│  System Node Pool (runs kube-system pods)                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │ Node 0   │  │ Node 1   │  │ Node 2   │                  │
│  │ D4s_v5   │  │ D4s_v5   │  │ D4s_v5   │                  │
│  │ 4 vCPU   │  │ 4 vCPU   │  │ 4 vCPU   │                  │
│  │ 16GB RAM │  │ 16GB RAM │  │ 16GB RAM │                  │
│  │ Ubuntu   │  │ Ubuntu   │  │ Ubuntu   │                  │
│  └──────────┘  └──────────┘  └──────────┘                  │
│                                                              │
│  User Node Pool (runs your application pods)                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Node 0   │  │ Node 1   │  │ Node 2   │  │ Node 3   │   │
│  │ D8s_v5   │  │ D8s_v5   │  │ D8s_v5   │  │ D8s_v5   │   │
│  │ 8 vCPU   │  │ 8 vCPU   │  │ 8 vCPU   │  │ 8 vCPU   │   │
│  │ 32GB RAM │  │ 32GB RAM │  │ 32GB RAM │  │ 32GB RAM │   │
│  │ Ubuntu   │  │ Ubuntu   │  │ Ubuntu   │  │ Ubuntu   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### What Each Node Contains

```
┌────────────────────────────────────────┐
│           AKS Node (VM)                │
│                                        │
│  ┌──────────────────────────────┐     │
│  │  OS: Ubuntu 22.04 or Azure   │     │
│  │       Linux (Mariner)        │     │
│  ├──────────────────────────────┤     │
│  │  Kubernetes Components:      │     │
│  │  • kubelet (agent)           │     │
│  │  • kube-proxy (networking)   │     │
│  │  • containerd (runtime)      │     │
│  ├──────────────────────────────┤     │
│  │  CSI Drivers:                │     │
│  │  • Azure Disk CSI            │     │
│  │  • Azure File CSI            │     │
│  │  • Azure Blob CSI (optional) │     │
│  ├──────────────────────────────┤     │
│  │  Kubernetes Reserved:        │     │
│  │  • ~750 MB RAM (kubelet)     │     │
│  │  • ~100m CPU (system)        │     │
│  │  • Eviction threshold: 100MB │     │
│  ├──────────────────────────────┤     │
│  │  Your Pods:                  │     │
│  │  • Pod 1 (app container)     │     │
│  │  • Pod 2 (sidecar)           │     │
│  │  • Pod 3 (monitoring agent)  │     │
│  └──────────────────────────────┘     │
└────────────────────────────────────────┘
```

### Allocatable vs Total Resources

Not all of a node's CPU and memory is available for your pods. Kubernetes reserves some for itself:

**Memory reservation formula (per node):**
- First 4 GB: 25% reserved (1 GB)
- Next 4 GB: 20% reserved (0.8 GB)
- Next 8 GB: 10% reserved (0.8 GB)
- Next 112 GB: 6% reserved (varies)
- Anything above 128 GB: 2% reserved

**Example for a D4s_v5 (16 GB RAM):**
```
Total RAM:        16 GB
Kubernetes reserved: ~2.6 GB (25% of 4 + 20% of 4 + 10% of 8)
Eviction threshold:  0.1 GB
Allocatable:       ~13.3 GB available for pods
```

**CPU reservation:**
- First core: 60 millicores
- Second core: 10 millicores
- Third and fourth cores: 5 millicores each
- Every core above 4: 2.5 millicores each

---

## 2. Azure VM Naming Convention — Decoding the Names

### The Naming Pattern

Every Azure VM size follows a structured naming convention:

```
Standard_D8s_v5
    │     ││ │
    │     ││ └── Version (generation): v5 = 5th generation hardware
    │     │└──── Suffix: s = Premium Storage capable
    │     └───── vCPU count: 8 vCPUs
    └─────────── Family: D = General Purpose
```

### Full Naming Breakdown

```
Standard _ [Family] [Sub-family] [# of vCPUs] [Constrained vCPUs] [Additive Features] [Accelerator Type] _ [Version]
```

### Family Letters

| Letter | Family | Optimized For |
|--------|--------|---------------|
| **B** | Burstable | Variable workloads, dev/test |
| **D** | General Purpose | Balanced CPU/memory, production workhorse |
| **E** | Memory Optimized | High memory-to-CPU ratio (8 GB per vCPU) |
| **F** | Compute Optimized | High CPU-to-memory ratio (2 GB per vCPU) |
| **L** | Storage Optimized | High local disk IOPS, NVMe storage |
| **M** | Ultra Memory | Massive memory (up to 4 TB RAM) |
| **N** | GPU | Machine learning, graphics rendering |

### Suffix Letters

| Suffix | Meaning |
|--------|---------|
| **a** | AMD processor (often 5-10% cheaper than Intel) |
| **d** | Local temp disk included |
| **i** | Isolated (dedicated physical server) |
| **l** | Low memory (2 GB per vCPU instead of 4) |
| **m** | High memory variant |
| **p** | ARM-based (Ampere Altra) processor |
| **s** | Premium Storage capable (SSD support) |
| **t** | Tiny memory |

### Reading Real Examples

```
Standard_D4s_v5
  D = General Purpose
  4 = 4 vCPUs
  s = Premium Storage
  v5 = 5th generation
  → 4 vCPUs, 16 GB RAM, Premium SSD support

Standard_E8as_v5
  E = Memory Optimized
  8 = 8 vCPUs
  a = AMD processor
  s = Premium Storage
  v5 = 5th generation
  → 8 vCPUs, 64 GB RAM, AMD, Premium SSD

Standard_F16s_v2
  F = Compute Optimized
  16 = 16 vCPUs
  s = Premium Storage
  v2 = 2nd generation
  → 16 vCPUs, 32 GB RAM, high clock speed

Standard_D4pds_v6
  D = General Purpose
  4 = 4 vCPUs
  p = ARM processor (Ampere)
  d = Local disk
  s = Premium Storage
  v6 = 6th generation (latest)
  → 4 vCPUs, 16 GB RAM, ARM, local disk, latest gen
```

---

## 3. VM Family Overview — All Families Explained

### Visual Family Map

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure VM Families for AKS                    │
│                                                                 │
│  CPU ◄─────────────────────────────────────────────► Memory    │
│                                                                 │
│  ┌─────────────┐                                               │
│  │  F-Series   │ Compute Optimized (2 GB/vCPU)                 │
│  │  "CPU Heavy" │ Batch processing, analytics, gaming          │
│  └─────────────┘                                               │
│                                                                 │
│       ┌─────────────┐                                          │
│       │  D-Series   │ General Purpose (4 GB/vCPU)              │
│       │ "Balanced"  │ Most production workloads                │
│       └─────────────┘                                          │
│                                                                 │
│            ┌─────────────┐                                     │
│            │  E-Series   │ Memory Optimized (8 GB/vCPU)        │
│            │"Memory Heavy"│ Databases, in-memory analytics     │
│            └─────────────┘                                     │
│                                                                 │
│                 ┌─────────────┐                                │
│                 │  M-Series   │ Ultra Memory (up to 4 TB RAM)  │
│                 │ "Mega RAM"  │ SAP HANA, in-memory DBs        │
│                 └─────────────┘                                │
│                                                                 │
│  ┌─────────────┐                                               │
│  │  B-Series   │ Burstable (variable CPU credits)              │
│  │"Budget/Dev" │ Dev/test, low traffic apps                    │
│  └─────────────┘                                               │
│                                                                 │
│  ┌─────────────┐          ┌─────────────┐                     │
│  │  L-Series   │          │  N-Series   │                     │
│  │"High IOPS"  │          │ "GPU/ML/AI" │                     │
│  │ NVMe storage│          │ Training,    │                     │
│  └─────────────┘          │ inference    │                     │
│                           └─────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

### Summary Comparison Table

| Family | vCPU:RAM Ratio | vCPU Range | RAM Range | Best For | Cost Level |
|--------|---------------|------------|-----------|----------|------------|
| **B** | Variable | 1-20 | 0.5-80 GB | Dev/test, burstable workloads | $ |
| **D** | 1:4 | 2-128 | 8-512 GB | **Most production workloads** | $$ |
| **E** | 1:8 | 2-128 | 16-1024 GB | Databases, analytics, caching | $$$ |
| **F** | 1:2 | 2-96 | 4-192 GB | Batch processing, CI/CD, web servers | $$ |
| **L** | 1:8 | 8-96 | 64-768 GB | Cassandra, Elasticsearch, data warehousing | $$$$ |
| **M** | 1:16+ | 8-416 | 218-11,400 GB | SAP HANA, in-memory databases | $$$$$ |
| **N** | Varies | 4-96 | 28-1344 GB | ML training/inference, rendering | $$$$$ |

---

## 4. General Purpose VMs — The D-Series Workhorses

### Why D-Series is the Enterprise Default

The D-series is the **most commonly used VM family in AKS**. With a balanced 1:4 vCPU-to-memory ratio (4 GB RAM per vCPU), it handles most production workloads efficiently.

### Current D-Series Generations

| Size | vCPUs | RAM (GB) | Temp Disk | Max Data Disks | Network (Gbps) | ~Monthly Cost (Linux, East US) |
|------|-------|----------|-----------|----------------|----------------|-------------------------------|
| **Standard_D2s_v5** | 2 | 8 | No temp | 4 | 12.5 | ~$70 |
| **Standard_D4s_v5** | 4 | 16 | No temp | 8 | 12.5 | ~$140 |
| **Standard_D8s_v5** | 8 | 32 | No temp | 16 | 12.5 | ~$281 |
| **Standard_D16s_v5** | 16 | 64 | No temp | 32 | 12.5 | ~$562 |
| **Standard_D32s_v5** | 32 | 128 | No temp | 32 | 16 | ~$1,124 |
| **Standard_D48s_v5** | 48 | 192 | No temp | 32 | 24 | ~$1,686 |
| **Standard_D64s_v5** | 64 | 256 | No temp | 32 | 30 | ~$2,249 |
| **Standard_D96s_v5** | 96 | 384 | No temp | 32 | 35 | ~$3,373 |

### AMD Variants (5-10% Cheaper)

| Size | vCPUs | RAM (GB) | ~Monthly Cost |
|------|-------|----------|---------------|
| **Standard_D2as_v5** | 2 | 8 | ~$63 |
| **Standard_D4as_v5** | 4 | 16 | ~$126 |
| **Standard_D8as_v5** | 8 | 32 | ~$253 |
| **Standard_D16as_v5** | 16 | 64 | ~$506 |

### ARM Variants (Cheapest, Ampere Altra)

| Size | vCPUs | RAM (GB) | ~Monthly Cost |
|------|-------|----------|---------------|
| **Standard_D2pds_v6** | 2 | 8 | ~$55 |
| **Standard_D4pds_v6** | 4 | 16 | ~$110 |
| **Standard_D8pds_v6** | 8 | 32 | ~$220 |
| **Standard_D16pds_v6** | 16 | 64 | ~$440 |

**ARM VMs offer ~20-30% savings** over Intel equivalents. Most Linux container workloads run perfectly on ARM with multi-arch images.

### When to Use D-Series

✅ API backends and microservices
✅ Web applications
✅ Small to medium databases
✅ CI/CD build agents
✅ General Kubernetes workloads
✅ System node pools (Microsoft recommends D16ds_v5 for large clusters)

---

## 5. Burstable VMs — The B-Series for Dev/Test

### How Burstable VMs Work

B-series VMs use a **CPU credit system**. They have a baseline CPU performance (e.g., 20% of a vCPU) and accumulate credits when idle. When your workload needs more CPU, it "bursts" by spending accumulated credits.

```
CPU Credits:
                    ┌── Burst! (spending credits)
                    │
100% CPU ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                   /│\
                  / │ \
Baseline 20% ───/──│──\──────────────────────
               /   │   \
Idle (earning credits)   Idle (earning credits)
```

### Common B-Series Sizes for AKS Dev/Test

| Size | vCPUs | RAM (GB) | Baseline CPU | ~Monthly Cost |
|------|-------|----------|-------------|---------------|
| **Standard_B2ms** | 2 | 8 | 60% | ~$30 |
| **Standard_B4ms** | 4 | 16 | 90% | ~$60 |
| **Standard_B8ms** | 8 | 32 | 135% | ~$121 |
| **Standard_B2s_v2** | 2 | 8 | 40% | ~$28 |
| **Standard_B4s_v2** | 4 | 16 | 50% | ~$55 |

### When to Use B-Series

✅ Development and testing environments
✅ CI/CD runners with variable load
✅ Internal tools with low traffic
✅ Proof-of-concept clusters

### When NOT to Use B-Series

❌ **Production workloads** — Performance is unpredictable when credits are exhausted
❌ **System node pools** — AKS explicitly blocks B-series for system pools
❌ **Databases** — Inconsistent CPU = inconsistent query performance
❌ **Latency-sensitive applications**

---

## 6. Memory-Optimized VMs — The E-Series and M-Series

### E-Series (Memory Optimized — 8 GB per vCPU)

The E-series offers **double the memory** of D-series at the same vCPU count. Ideal for workloads that need lots of RAM but not proportionally more CPU.

| Size | vCPUs | RAM (GB) | ~Monthly Cost |
|------|-------|----------|---------------|
| **Standard_E2s_v5** | 2 | 16 | ~$92 |
| **Standard_E4s_v5** | 4 | 32 | ~$184 |
| **Standard_E8s_v5** | 8 | 64 | ~$368 |
| **Standard_E16s_v5** | 16 | 128 | ~$737 |
| **Standard_E32s_v5** | 32 | 256 | ~$1,474 |
| **Standard_E64s_v5** | 64 | 512 | ~$2,948 |

### When to Use E-Series

✅ In-memory databases (Redis, Memcached with large datasets)
✅ Java applications with large heaps (JVM-heavy workloads)
✅ Elasticsearch / OpenSearch clusters
✅ Data analytics and processing
✅ Apache Spark and Kafka
✅ SAP applications

### M-Series (Ultra Memory — Up to 4 TB RAM)

For extreme memory requirements:

| Size | vCPUs | RAM (GB) | ~Monthly Cost |
|------|-------|----------|---------------|
| **Standard_M32ms** | 32 | 875 | ~$4,300 |
| **Standard_M64ms** | 64 | 1,792 | ~$8,600 |
| **Standard_M128ms** | 128 | 3,892 | ~$17,200 |

**Use case:** SAP HANA, large in-memory databases. Rarely used in AKS.

---

## 7. Compute-Optimized VMs — The F-Series

### F-Series (2 GB per vCPU)

F-series provides more CPU power per dollar than D-series, at the cost of less memory. Higher clock speeds and newer processors make them ideal for CPU-bound workloads.

| Size | vCPUs | RAM (GB) | ~Monthly Cost |
|------|-------|----------|---------------|
| **Standard_F2s_v2** | 2 | 4 | ~$62 |
| **Standard_F4s_v2** | 4 | 8 | ~$124 |
| **Standard_F8s_v2** | 8 | 16 | ~$248 |
| **Standard_F16s_v2** | 16 | 32 | ~$496 |
| **Standard_F32s_v2** | 32 | 64 | ~$993 |
| **Standard_F48s_v2** | 48 | 96 | ~$1,489 |
| **Standard_F72s_v2** | 72 | 144 | ~$2,234 |

### When to Use F-Series

✅ Batch processing and data transformations
✅ Web servers handling many concurrent requests
✅ CI/CD build pipelines (compilation is CPU-heavy)
✅ Video encoding / transcoding
✅ Scientific simulations
✅ Gaming servers

### When NOT to Use F-Series

❌ Applications needing lots of RAM (only 2 GB per vCPU)
❌ Databases (too little memory for buffer pools)
❌ Java applications (JVM needs RAM)

---

## 8. Storage-Optimized VMs — The L-Series

### L-Series (High Local NVMe Storage)

L-series VMs include extremely fast **local NVMe SSDs** attached directly to the VM. These local disks offer millions of IOPS with sub-millisecond latency.

| Size | vCPUs | RAM (GB) | Local NVMe Storage | ~Monthly Cost |
|------|-------|----------|-------------------|---------------|
| **Standard_L8s_v3** | 8 | 64 | 1x 1.92 TB NVMe | ~$500 |
| **Standard_L16s_v3** | 16 | 128 | 2x 1.92 TB NVMe | ~$1,000 |
| **Standard_L32s_v3** | 32 | 256 | 4x 1.92 TB NVMe | ~$2,000 |
| **Standard_L48s_v3** | 48 | 384 | 6x 1.92 TB NVMe | ~$3,000 |
| **Standard_L64s_v3** | 64 | 512 | 8x 1.92 TB NVMe | ~$4,000 |

**⚠️ Warning:** Local NVMe storage is **ephemeral** — data is lost if the VM is deallocated or fails. Use only for temporary/cacheable data.

### When to Use L-Series

✅ Cassandra, ScyllaDB (distributed NoSQL needing fast local disk)
✅ Elasticsearch with hot-tier indices
✅ Kafka with high-throughput message brokers
✅ Data warehousing (temporary staging)

---

## 9. GPU VMs — The N-Series for ML/AI

### GPU VM Families

| Family | GPU | VRAM | Use Case |
|--------|-----|------|----------|
| **NC-series** (v3) | NVIDIA V100 | 16-32 GB | ML training, inference |
| **NC-series** (A100) | NVIDIA A100 | 80 GB | Large model training |
| **ND-series** (H100) | NVIDIA H100 | 80 GB | LLM training, deep learning |
| **NV-series** | NVIDIA T4 | 16 GB | Remote visualization, VDI |

### Common GPU Sizes for AKS

| Size | vCPUs | RAM (GB) | GPUs | GPU VRAM | ~Monthly Cost |
|------|-------|----------|------|----------|---------------|
| **Standard_NC6s_v3** | 6 | 112 | 1x V100 | 16 GB | ~$2,200 |
| **Standard_NC24s_v3** | 24 | 448 | 4x V100 | 64 GB | ~$8,800 |
| **Standard_NC24ads_A100_v4** | 24 | 220 | 1x A100 | 80 GB | ~$2,700 |
| **Standard_ND96asr_v4** | 96 | 900 | 8x A100 | 640 GB | ~$22,000 |

**AKS GPU setup:**
```bash
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-aks \
  --name gpupool \
  --node-count 1 \
  --node-vm-size Standard_NC6s_v3 \
  --node-taints sku=gpu:NoSchedule  # Only GPU pods land here
```

---

## 10. Generation 1 vs Generation 2 VMs

### What Changed in Gen 2?

| Feature | Gen 1 | Gen 2 |
|---------|-------|-------|
| **Boot** | BIOS | UEFI (faster boot) |
| **OS Disk** | Up to 2 TB | Up to 4 TB |
| **Disk Type** | VHD | VHDX |
| **NVMe Support** | ❌ | ✅ |
| **Trusted Launch** | ❌ | ✅ (Secure Boot, vTPM) |
| **Memory** | Limited | Increased capacity |
| **Performance** | Baseline | Improved |

### AKS Default Behavior (Kubernetes 1.29+)

- **Linux node pools:** Default to Gen 2 (unless VM size only supports Gen 1)
- **v6 series VMs:** Exclusively Gen 2

**Recommendation:** Always use Gen 2 for new clusters.

---

## 11. System Node Pool vs User Node Pool

### System Node Pool

**Purpose:** Runs Kubernetes system pods (`kube-system` namespace) — CoreDNS, metrics-server, tunnelfront, etc.

**Requirements:**
- Minimum 2 vCPUs and 4 GB RAM (AKS enforced)
- B-series NOT allowed
- At least 1 system pool per cluster

**Recommended sizes:**
```
Small clusters (<50 nodes):    Standard_D4s_v5  (4 vCPUs, 16 GB)
Medium clusters (50-200):      Standard_D8s_v5  (8 vCPUs, 32 GB)
Large clusters (200-1000):     Standard_D16s_v5 (16 vCPUs, 64 GB)
Very large (1000+ nodes):      Standard_D16ds_v5 with ephemeral OS
```

### User Node Pool

**Purpose:** Runs your application workloads.

**Flexibility:** Any supported VM size, including GPU, burstable, storage-optimized.

**Multiple pools for different workloads:**
```bash
# General workload pool
az aks nodepool add --name general --node-vm-size Standard_D4s_v5 --node-count 5

# Memory-heavy pool (databases)
az aks nodepool add --name memory --node-vm-size Standard_E8s_v5 --node-count 3

# GPU pool (ML inference)
az aks nodepool add --name gpu --node-vm-size Standard_NC6s_v3 --node-count 1
```

---

## 12. What Enterprises Actually Use in Production

### The Most Common Enterprise Configurations

Based on Azure best practices and real-world deployments:

**System Node Pool (Tier 1 — Standard):**
```
VM Size: Standard_D4s_v5 or Standard_D8s_v5
Nodes: 3 (spread across 3 availability zones)
OS: Azure Linux (Mariner) or Ubuntu 22.04
Disk: Ephemeral OS disk (faster, no extra cost)
```

**General Workload Pool (Tier 2 — Most Common):**
```
VM Size: Standard_D4s_v5 (4 vCPU, 16 GB)  ← Most popular
         Standard_D8s_v5 (8 vCPU, 32 GB)  ← For larger pods
Nodes: 5-20 (autoscaled)
OS: Azure Linux or Ubuntu
```

**Database/Cache Pool (Tier 3 — Memory Heavy):**
```
VM Size: Standard_E4s_v5 (4 vCPU, 32 GB)
         Standard_E8s_v5 (8 vCPU, 64 GB)
Nodes: 3-5
Taints: workload=database:NoSchedule
```

### Enterprise Cost Optimization Pattern

Many enterprises use a **mixed approach**:
```
70% of workloads → D-series (general purpose, reliable)
15% of workloads → E-series (memory-heavy, databases)
10% of workloads → B-series (dev/test, non-critical)
5%  of workloads → F-series or GPU (specialized)
```

### Linux vs Windows Cost Difference

| OS | D4s_v5 Monthly | Why? |
|----|---------------|------|
| **Linux** | ~$140 | No OS license cost |
| **Windows** | ~$280 | Windows Server license included |

**Linux is ~50% cheaper** for the same VM size because there's no OS licensing fee. This is why most enterprises run Linux nodes exclusively and only add Windows node pools when absolutely necessary (legacy .NET Framework apps).

---

## 13. How to Select the Right VM Size — Decision Framework

### The Decision Tree

```
START: What is your workload's primary bottleneck?
│
├─ CPU-bound (high CPU, low memory)?
│  └─ F-series (Fsv2)
│     Example: Batch processing, video encoding
│
├─ Memory-bound (high memory, moderate CPU)?
│  └─ E-series (Es_v5)
│     Example: Redis, Elasticsearch, Java apps
│
├─ Balanced (equal CPU and memory)?
│  └─ D-series (Ds_v5) ← DEFAULT CHOICE
│     Example: Web APIs, microservices, general apps
│
├─ Storage I/O bound (high disk throughput)?
│  └─ L-series (Ls_v3)
│     Example: Cassandra, Kafka, data pipelines
│
├─ GPU required (ML training/inference)?
│  └─ N-series (NC/ND/NV)
│     Example: TensorFlow, PyTorch, image processing
│
├─ Variable / bursty load (idle most of the time)?
│  └─ B-series (Bs_v2) — DEV/TEST ONLY
│     Example: Internal tools, staging environments
│
└─ Extreme memory (500 GB+ RAM)?
   └─ M-series
      Example: SAP HANA, in-memory analytics
```

### Step-by-Step Selection Process

**Step 1: Profile your workload**
```bash
# Monitor current resource usage
kubectl top nodes
kubectl top pods --all-namespaces

# Check resource requests vs actual usage
kubectl describe nodes | grep -A 5 "Allocated resources"
```

**Step 2: Calculate required resources**
```
Total pod CPU requests:     12 vCPUs
Total pod memory requests:  40 GB
Safety buffer (30%):        +3.6 vCPU, +12 GB
Kubernetes overhead:        +1 vCPU, +2.6 GB per node
───────────────────────────────────────────
Need:                       ~16.6 vCPUs, ~54.6 GB RAM
```

**Step 3: Choose between fewer large nodes vs many small nodes**

| Strategy | Fewer Large Nodes | Many Small Nodes |
|----------|------------------|-----------------|
| **Example** | 2x D16s_v5 (16 vCPU each) | 8x D4s_v5 (4 vCPU each) |
| **Total resources** | 32 vCPU, 128 GB | 32 vCPU, 128 GB |
| **Pros** | Less overhead, larger pods possible | Better fault tolerance, finer scaling |
| **Cons** | One node failure = 50% capacity loss | More Kubernetes overhead |
| **Recommendation** | ✅ When pods need >4 vCPU | ✅ When pods are small (<1 vCPU) |

**General rule:** Aim for nodes that can fit **10-30 pods each**. Too few pods per node = wasted resources. Too many = noisy neighbor issues.

---

## 14. Cost Optimization — Pricing Models and Strategies

### The 4 Pricing Models

| Model | Savings | Commitment | Best For |
|-------|---------|-----------|----------|
| **Pay-As-You-Go (PAYG)** | 0% (baseline) | None | Short-term, unpredictable workloads |
| **Azure Savings Plan** | Up to 25% | 1 or 3 year | Consistent compute, flexible VM changes |
| **Reserved Instances (RI)** | Up to 72% | 1 or 3 year, specific VM size + region | Predictable, stable workloads |
| **Spot VMs** | Up to 90% | None (can be evicted) | Batch jobs, fault-tolerant workloads |

### Reserved Instances Deep Dive

```
Example: Standard_D8s_v5 in East US (Linux)

PAYG:              $281/month
1-Year Reserved:   $176/month  (37% savings)
3-Year Reserved:   $112/month  (60% savings)

Annual savings (3-year, 10 nodes):
  PAYG:     $281 × 10 × 12 = $33,720/year
  3-Year RI: $112 × 10 × 12 = $13,440/year
  ─────────────────────────────────────────
  Savings:                     $20,280/year
```

### Spot VMs for AKS

**Spot VMs** use unused Azure capacity at massive discounts (up to 90% off). Azure can **evict** them with 30 seconds notice when capacity is needed.

```bash
# Create a Spot node pool
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-aks \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \         # Pay up to PAYG price
  --node-vm-size Standard_D4s_v5 \
  --node-count 5 \
  --node-taints "kubernetes.azure.com/scalesetpriority=spot:NoSchedule"
```

**Good for:** Batch processing, CI/CD builds, data processing, dev/test
**Bad for:** Production APIs, databases, stateful workloads

### Cost Comparison Summary (D8s_v5, 10 nodes, Linux, East US, per year)

| Model | Monthly per Node | Annual (10 nodes) | Savings |
|-------|-----------------|-------------------|---------|
| **PAYG** | $281 | $33,720 | Baseline |
| **Savings Plan (3yr)** | $197 | $23,640 | 30% |
| **Reserved (3yr)** | $112 | $13,440 | 60% |
| **Spot** | ~$56 | ~$6,720 | 80% |

---

## 15. Node Pool Design Patterns

### Pattern 1: Simple (Small Teams)

```
┌─────────────────────────────┐
│  System Pool: D4s_v5 × 3    │
│  User Pool:   D4s_v5 × 3-10 │
│  (autoscaler: min 3, max 10)│
└─────────────────────────────┘
```

### Pattern 2: Workload Separation (Medium Teams)

```
┌─────────────────────────────────────────────────┐
│  System Pool:   D4s_v5 × 3                      │
│  General Pool:  D8s_v5 × 5-20 (autoscaled)     │
│  Database Pool: E8s_v5 × 3 (tainted)            │
│  Batch Pool:    D4s_v5 Spot × 0-10 (autoscaled) │
└─────────────────────────────────────────────────┘
```

### Pattern 3: Enterprise Multi-Zone (Large Teams)

```
┌─────────────────────────────────────────────────────────┐
│  System Pool:   D8s_v5 × 3  (zones 1,2,3)              │
│  API Pool:      D4s_v5 × 6-30 (zones 1,2,3, autoscaled)│
│  Worker Pool:   D8s_v5 × 3-15 (zones 1,2,3, autoscaled)│
│  Database Pool: E8s_v5 × 3 (zones 1,2,3, tainted)      │
│  GPU Pool:      NC6s_v3 × 1-3 (tainted, autoscaled)    │
│  Spot Pool:     D4s_v5 × 0-20 (Spot, batch jobs)       │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Best Practices and Recommendations

### VM Selection

✅ **Start with D-series** for most workloads — it's the safest default

✅ **Use D4s_v5 or D8s_v5** — the two most common production sizes

✅ **Consider AMD variants** (`a` suffix) — 5-10% cheaper, same performance

✅ **Consider ARM variants** (`p` suffix) — 20-30% cheaper if your images support multi-arch

✅ **Use latest generation** — v5 or v6 offers better price/performance than v3/v4

✅ **Use Gen 2 VMs** — Better security, faster boot, NVMe support

### Node Pool Design

✅ **Separate system and user pools** — Different scaling and lifecycle

✅ **Use multiple user pools** for different workload types

✅ **Enable cluster autoscaler** — Scale nodes based on demand

✅ **Spread across 3 availability zones** — High availability

✅ **Use ephemeral OS disks** — Faster, cheaper, no disk to manage

✅ **Set resource requests and limits** on all pods — Required for autoscaler

### Cost Optimization

✅ **Use Reserved Instances** for stable production workloads (60% savings)

✅ **Use Spot VMs** for batch/dev/test workloads (up to 90% savings)

✅ **Right-size regularly** — Monitor actual CPU/memory usage and adjust

✅ **Use Linux nodes** — 50% cheaper than Windows (no license fee)

✅ **Scale to zero** where possible — Spot pools and batch pools during off-hours

✅ **Use Azure Advisor** — Automatic right-sizing recommendations

### Linux-Specific Recommendations

✅ **Use Azure Linux (Mariner)** — Smaller image, faster boot, Microsoft-maintained

✅ **Or Ubuntu 22.04** — Widest ecosystem support, most community documentation

✅ **Enable ephemeral OS** — Uses node's temp disk for OS, faster and free

```bash
az aks nodepool add \
  --name mypool \
  --node-vm-size Standard_D4s_v5 \
  --os-type Linux \
  --os-sku AzureLinux \
  --node-osdisk-type Ephemeral \
  --node-count 3 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 20
```

### Monitoring and Right-Sizing

```bash
# Check node utilization
kubectl top nodes

# Check pod resource usage
kubectl top pods --all-namespaces --sort-by=memory

# Azure Advisor recommendations
az advisor recommendation list \
  --filter "Category eq 'Cost'" \
  --output table
```

---

## References & Further Reading

- [Azure VM Sizes Overview (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview)
- [VM Sizes for AKS (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/aks-virtual-machine-sizes)
- [AKS Quotas, SKUs, Regions (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/quotas-skus-regions)
- [AKS Cost Optimization Best Practices (GitHub)](https://github.com/Azure/k8s-best-practices/blob/master/Cost_Optimization.md)
- [Performance & Scale Best Practices for Large AKS (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/best-practices-performance-scale-large)
- [Azure VM Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/)
- [VM Generations and AKS (AKS Blog)](https://blog.aks.azure.com/2025/04/23/vm-generations-and-aks)
- [Azure VM Series Pricing](https://azure.microsoft.com/en-us/pricing/details/virtual-machines/series/)