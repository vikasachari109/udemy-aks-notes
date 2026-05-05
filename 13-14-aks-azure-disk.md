# AKS Storage with Azure Disks & Availability Zones
### The Ultimate Beginner-Friendly Deep Dive

> **What this document covers:** Why applications need persistent storage, container ephemeral nature, Azure storage redundancy options (LRS/ZRS/GRS), how pods request volumes, Storage Classes, Azure Disk vs Azure Files vs Azure NetApp, Zone-Redundant Storage for high availability, Shared Disks for multi-node access, and complete practical examples — all explained from absolute scratch.

---

## Table of Contents
1. [The Fundamental Problem: Container Ephemeral Nature](#1-the-fundamental-problem-container-ephemeral-nature)
2. [Why Applications Need Persistent Storage](#2-why-applications-need-persistent-storage)
3. [How a Pod Requests Persistent Storage](#3-how-a-pod-requests-persistent-storage)
4. [Azure Storage Redundancy Options Explained](#4-azure-storage-redundancy-options-explained)
5. [Azure Disk: LRS vs ZRS Deep Dive](#5-azure-disk-lrs-vs-zrs-deep-dive)
6. [Azure Storage Options for AKS Workloads](#6-azure-storage-options-for-aks-workloads)
7. [Storage Classes in AKS — Built-in and Custom](#7-storage-classes-in-aks--built-in-and-custom)
8. [Zone-Redundant Storage (ZRS) for High Availability](#8-zone-redundant-storage-zrs-for-high-availability)
9. [Multi-Zone AKS Cluster with ZRS Disks](#9-multi-zone-aks-cluster-with-zrs-disks)
10. [Azure Shared Disks — Multi-Node Attachment](#10-azure-shared-disks--multi-node-attachment)
11. [Shared Disk with ZRS for Ultimate Availability](#11-shared-disk-with-zrs-for-ultimate-availability)
12. [Complete Examples with YAML](#12-complete-examples-with-yaml)
13. [Comparison Tables and Decision Trees](#13-comparison-tables-and-decision-trees)
14. [Troubleshooting Storage Issues](#14-troubleshooting-storage-issues)
15. [Best Practices and Recommendations](#15-best-practices-and-recommendations)

---

## 1. The Fundamental Problem: Container Ephemeral Nature

### What Does "Ephemeral" Mean?

**Ephemeral** means temporary, short-lived, disposable. In Kubernetes:
- Containers are **ephemeral** — they start, run, and can be killed at any moment
- Pods are **ephemeral** — they can be deleted and recreated with a new identity
- Any data written **inside a container's filesystem is ephemeral** — it disappears when the container stops

### The Problem in Action

Let's say you run a database (PostgreSQL) in a pod:

```bash
# Deploy PostgreSQL pod
kubectl run postgres --image=postgres:15

# Create a database inside the pod
kubectl exec postgres -- psql -U postgres -c "CREATE DATABASE myapp"

# Check the database exists
kubectl exec postgres -- psql -U postgres -c "\l"
# myapp is listed ✅

# Pod crashes or gets deleted
kubectl delete pod postgres

# Kubernetes recreates the pod (assuming a Deployment manages it)
# Check databases again
kubectl exec postgres-new-abc123 -- psql -U postgres -c "\l"
# myapp is GONE! ❌
```

**All data inside the container was lost** because containers are ephemeral.

### The Filesystem Inside a Container

Every container has its own filesystem. By default, this filesystem is:
1. **Created from the container image** (read-only layers)
2. **Writable layer on top** (where the container writes data)
3. **Stored on the node's local disk**
4. **Deleted when the container stops**

```
Container Filesystem:
┌────────────────────────────────────┐
│  Writable Layer (ephemeral)        │  ← Data written here DISAPPEARS
├────────────────────────────────────┤
│  Image Layer 3 (read-only)         │
├────────────────────────────────────┤
│  Image Layer 2 (read-only)         │
├────────────────────────────────────┤
│  Image Layer 1 (read-only)         │
└────────────────────────────────────┘
```

**This is why we need Persistent Volumes** — storage that exists **outside the container** and **survives pod restarts**.

---

## 2. Why Applications Need Persistent Storage

### Stateless vs Stateful Applications

**Stateless Application:**
- Doesn't store data between requests
- Examples: Web frontend, API gateway, load balancer
- Can be restarted without losing anything
- ✅ Works fine with ephemeral storage

**Stateful Application:**
- Stores data that must persist between requests
- Examples: Database, file storage, cache with persistence
- Restarting = data loss = disaster
- ❌ Requires persistent storage

### Common Use Cases for Persistent Storage

| Use Case | Data That Must Persist |
|----------|----------------------|
| **Database** (PostgreSQL, MySQL, MongoDB) | Database files, transaction logs |
| **Content Management** (WordPress) | Uploaded images, media files |
| **Log Aggregation** (Elasticsearch) | Log indices, search data |
| **CI/CD** (Jenkins) | Build artifacts, job configurations |
| **Message Queue** (RabbitMQ, Kafka) | Persistent message queues |
| **Shared Storage** (NFS) | Files shared between multiple pods |
| **Backups** | Database dumps, application backups |
| **ML Models** | Trained models, datasets |

---

## 3. How a Pod Requests Persistent Storage

### The Kubernetes Storage Model

Kubernetes uses a **3-layer abstraction** to decouple pods from actual storage:

```
┌──────────────────────────────────────────────────────────────┐
│  Layer 1: Pod                                                │
│  ↓ References                                                │
│  Layer 2: PersistentVolumeClaim (PVC)                        │
│  ↓ Binds to                                                  │
│  Layer 3: PersistentVolume (PV)                              │
│  ↓ Backed by                                                 │
│  Layer 4: Actual Azure Disk / Azure Files / Azure Blob       │
└──────────────────────────────────────────────────────────────┘
```

### Step-by-Step: Pod to Disk

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Developer writes a Pod YAML                              │
│    pod.spec.volumes points to a PVC named "my-data"         │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Developer creates a PVC YAML                             │
│    PVC requests 50Gi of storage, storageClass: managed-csi  │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Kubernetes sees PVC with storageClass: managed-csi       │
│    Kubernetes asks: "Which provisioner handles this class?" │
│    Answer: disk.csi.azure.com (Azure Disk CSI Driver)       │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Azure Disk CSI Driver calls Azure API                    │
│    "Create a 50GB Premium SSD Managed Disk in RG MC_xxx"    │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. Azure creates the Managed Disk                           │
│    Location: Resource Group MC_myaks_westeurope             │
│    Size: 50 GiB                                             │
│    SKU: Premium_LRS (or Premium_ZRS)                        │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. Kubernetes creates a PersistentVolume (PV)               │
│    PV represents the Azure Disk                             │
│    PV is bound to the PVC                                   │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. Kubernetes schedules the Pod to a node                   │
│    Node attaches the Azure Disk (like plugging in a USB)    │
│    Disk is mounted at /mnt/data (or wherever pod specifies) │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. Application inside pod writes to /mnt/data               │
│    Data is written to the Azure Disk                        │
│    Data persists even if pod restarts!                      │
└─────────────────────────────────────────────────────────────┘
```

### Key Components Explained

**PersistentVolumeClaim (PVC)**
- A **request** for storage by a user/pod
- Specifies size (e.g., 50Gi), access mode (RWO/RWX), storage class
- Think of it as: "I need 50GB of fast storage"

**PersistentVolume (PV)**
- The **actual storage resource** in the cluster
- Backed by Azure Disk, Azure Files, or Azure Blob
- Created automatically (dynamic provisioning) or manually (static)

**StorageClass**
- Defines **how** to create a PV
- Points to a **provisioner** (like `disk.csi.azure.com`)
- Contains parameters: disk SKU (Premium_LRS, Standard_SSD, etc.)

**CSI Driver (Container Storage Interface)**
- A plugin that knows how to talk to Azure to create/attach/detach disks
- AKS has CSI drivers for: Azure Disk, Azure Files, Azure Blob

---

## 4. Azure Storage Redundancy Options Explained

### What is Storage Redundancy?

**Redundancy** = Making copies of your data so you don't lose it if hardware fails.

Azure replicates your data **automatically** in the background. The number of copies and their locations depend on the redundancy option you choose.

### The 4 Redundancy Options

```
┌─────────────────────────────────────────────────────────────┐
│              Single Region                                  │
├────────────────────┬────────────────────────────────────────┤
│                    │                                        │
│       LRS          │           ZRS                          │
│ Locally Redundant  │  Zone-Redundant Storage                │
│     Storage        │                                        │
│                    │                                        │
│   3 copies in      │   3 copies across 3                    │
│   1 datacenter     │   availability zones                   │
│                    │   in the same region                   │
│                    │                                        │
│   Cheapest         │   Higher availability                  │
│   99.999999999%    │   99.9999999999%                       │
│   (11 nines)       │   (12 nines) durability                │
└────────────────────┴────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              Dual Region                                    │
├────────────────────┬────────────────────────────────────────┤
│                    │                                        │
│       GRS          │           GZRS                         │
│  Geo-Redundant     │  Geo-Zone-Redundant                    │
│     Storage        │     Storage                            │
│                    │                                        │
│   3 copies in      │   3 copies across 3 zones              │
│   primary region   │   in primary region                    │
│   +                │   +                                    │
│   3 copies in      │   3 copies in                          │
│   secondary region │   secondary region                     │
│                    │                                        │
│   Disaster recovery│   Best of both worlds                  │
│   99.99999999999999%│  99.99999999999999%                   │
│   (16 nines)       │   (16 nines) durability                │
└────────────────────┴────────────────────────────────────────┘
```

### Redundancy Options Detail

#### LRS (Locally Redundant Storage)

**What:** 3 copies of your data in **one physical location** (one datacenter)

**How it works:**
```
Your Azure Disk
       │
       ▼
┌──────────────────┐
│   Datacenter 1   │
│                  │
│  Copy 1 (Rack A) │
│  Copy 2 (Rack B) │
│  Copy 3 (Rack C) │
└──────────────────┘
```

**Protects against:**
- ✅ Individual disk failures
- ✅ Server rack failures
- ✅ Network switch failures

**Does NOT protect against:**
- ❌ Entire datacenter failure (fire, flood, power outage)
- ❌ Availability zone failure

**When to use:**
- Cost-sensitive workloads
- Non-critical data
- Single-zone AKS clusters

**Cost:** Cheapest option

**Durability:** 99.999999999% (11 nines) = 0.00000001% annual failure rate

#### ZRS (Zone-Redundant Storage)

**What:** 3 copies of your data across **3 availability zones** in the same region

**How it works:**
```
Your Azure Disk
       │
   ┌───┴───┬───────┐
   ▼       ▼       ▼
┌──────┐ ┌──────┐ ┌──────┐
│ Zone │ │ Zone │ │ Zone │
│  1   │ │  2   │ │  3   │
│      │ │      │ │      │
│Copy 1│ │Copy 2│ │Copy 3│
└──────┘ └──────┘ └──────┘
```

**Protects against:**
- ✅ Individual disk/rack/server failures
- ✅ **Entire availability zone failure** (datacenter outage)

**Does NOT protect against:**
- ❌ Entire region failure (rare, but possible)

**When to use:**
- **Production workloads** (recommended)
- **Multi-zone AKS clusters**
- Databases with high availability requirements

**Cost:** ~50% more expensive than LRS

**Durability:** 99.9999999999% (12 nines)

**Write behavior:** Writes are synchronous — data must be written to all 3 zones before the write is acknowledged. This adds ~1-2ms latency compared to LRS.

#### GRS (Geo-Redundant Storage)

**What:** 3 copies in primary region + 3 copies in a **secondary region** hundreds of miles away

**How it works:**
```
Primary Region (West Europe)          Secondary Region (North Europe)
       │                                      │
┌──────────────┐                      ┌──────────────┐
│ Datacenter 1 │                      │ Datacenter X │
│              │  Async Replication   │              │
│  Copy 1      │ ───────────────────► │  Copy 4      │
│  Copy 2      │                      │  Copy 5      │
│  Copy 3      │                      │  Copy 6      │
└──────────────┘                      └──────────────┘
```

**Protects against:**
- ✅ All LRS failures
- ✅ **Entire region failure** (earthquake, hurricane, war)

**Does NOT protect against:**
- ❌ Account-level corruption (unlikely)

**When to use:**
- Critical data that must survive region-wide disasters
- Compliance requirements (data must be replicated to another region)

**Cost:** ~2x more expensive than LRS

**Durability:** 99.99999999999999% (16 nines)

**Note:** Replication to secondary region is **asynchronous** (a few seconds delay). You cannot read from secondary region unless you trigger a failover.

#### GZRS (Geo-Zone-Redundant Storage)

**What:** Combines ZRS (in primary region) + GRS (replication to secondary region)

This is the **most durable** option but also the most expensive.

---

## 5. Azure Disk: LRS vs ZRS Deep Dive

### What is an Azure Managed Disk?

An **Azure Managed Disk** is a virtual hard drive (VHD) managed by Azure. It's like attaching an external hard drive to your computer, except:
- It's virtual (no physical disk)
- Azure handles all the low-level management (snapshots, replication, encryption)
- It can be attached to VMs or AKS pods

### LRS Disk Limitations in Multi-Zone AKS

**Scenario:** You have a multi-zone AKS cluster (nodes in Zone 1, 2, and 3). You create a pod with an LRS disk.

```
┌──────────────────────────────────────────────────────────────┐
│                    AKS Cluster (3 Zones)                     │
│                                                              │
│  Zone 1           Zone 2           Zone 3                    │
│  ┌──────┐        ┌──────┐        ┌──────┐                  │
│  │ Node │        │ Node │        │ Node │                  │
│  │  A   │        │  B   │        │  C   │                  │
│  └──┬───┘        └──────┘        └──────┘                  │
│     │                                                        │
│     │ Pod with LRS Disk is scheduled here                   │
│     ▼                                                        │
│  ┌──────┐                                                   │
│  │ Pod  │                                                   │
│  │ + PV │                                                   │
│  └──┬───┘                                                   │
│     │                                                        │
│     ▼                                                        │
│  ┌────────────────┐                                         │
│  │  LRS Disk      │                                         │
│  │  (in Zone 1)   │                                         │
│  └────────────────┘                                         │
│                                                              │
│  ⚠️ Problem: Zone 1 fails (power outage)                    │
│     ↓                                                        │
│  ❌ Node A is down                                           │
│  ❌ LRS Disk is in Zone 1 (unavailable)                     │
│  ❌ Kubernetes tries to reschedule pod to Zone 2 or 3       │
│  ❌ CANNOT attach the disk (it's stuck in Zone 1)           │
│  ❌ Pod is stuck in "Pending" state                         │
│  ❌ APPLICATION IS DOWN until Zone 1 recovers               │
└──────────────────────────────────────────────────────────────┘
```

**Why?** Azure Disks are **zonal resources** when using LRS. An LRS disk created in Zone 1 can only be attached to VMs in Zone 1.

### ZRS Disk: The Solution

With a **ZRS disk**, the data is replicated across all 3 zones:

```
┌──────────────────────────────────────────────────────────────┐
│                    AKS Cluster (3 Zones)                     │
│                                                              │
│  Zone 1           Zone 2           Zone 3                    │
│  ┌──────┐        ┌──────┐        ┌──────┐                  │
│  │ Node │        │ Node │        │ Node │                  │
│  │  A   │        │  B   │        │  C   │                  │
│  └──┬───┘        └──────┘        └──┬───┘                  │
│     │                              │                         │
│     │ Pod initially here           │ Pod moved here          │
│     ▼                              ▼ after Zone 1 fails      │
│  ┌──────┐                       ┌──────┐                    │
│  │ Pod  │ ─────X──────────────► │ Pod  │                    │
│  └──┬───┘  Zone 1 fails         └──┬───┘                    │
│     │                              │                         │
│     ▼                              ▼                         │
│  ┌───────────────────────────────────────────┐              │
│  │          ZRS Disk                         │              │
│  │  Replicated across Zone 1, 2, 3           │              │
│  │                                            │              │
│  │  Zone 1: Copy (unavailable)                │              │
│  │  Zone 2: Copy ✅ (accessible)              │              │
│  │  Zone 3: Copy ✅ (accessible)              │              │
│  └───────────────────────────────────────────┘              │
│                                                              │
│  ✅ Zone 1 fails                                             │
│  ✅ Kubernetes reschedules pod to Zone 3                    │
│  ✅ ZRS disk is attached to Node C in Zone 3                │
│  ✅ Pod starts successfully                                 │
│  ✅ APPLICATION IS BACK ONLINE in ~1-2 minutes              │
└──────────────────────────────────────────────────────────────┘
```

**Key insight:** A ZRS disk can be attached to any node in any zone within the same region.

### LRS vs ZRS Comparison Table

| Aspect | LRS | ZRS |
|--------|-----|-----|
| **Copies** | 3 (all in one datacenter) | 3 (one per zone) |
| **Durability** | 99.999999999% (11 nines) | 99.9999999999% (12 nines) |
| **Availability** | ~99.9% | ~99.99% |
| **Can survive zone failure?** | ❌ No | ✅ Yes |
| **Can attach to node in any zone?** | ❌ No (zonal) | ✅ Yes (regional) |
| **Write latency** | Lower (~1ms) | Slightly higher (~2-3ms) |
| **Cost** | Baseline | +50% |
| **Use case** | Single-zone clusters, dev/test | **Multi-zone clusters, production** |
| **Supported SKUs** | Standard_LRS, Premium_LRS, Ultra | StandardSSD_ZRS, Premium_ZRS |

### When Zone Failover Happens

**Automatic failover** (managed by Azure and Kubernetes):
```
1. Zone 1 loses power
      ↓
2. Node A (Zone 1) becomes NotReady (30-60 seconds)
      ↓
3. Kubernetes marks pod for rescheduling (5 minutes default)
      ↓
4. Kubernetes detaches disk from Zone 1
      ↓
5. Kubernetes schedules pod to Node C (Zone 3)
      ↓
6. Azure attaches ZRS disk to Node C
      ↓
7. Pod starts, mounts disk, application resumes
```

**Total downtime with ZRS:** ~5-8 minutes (mostly waiting for Kubernetes to detect failure)

**Total downtime with LRS:** Until Zone 1 recovers (hours to days)

---

## 6. Azure Storage Options for AKS Workloads

### The 4 Main Options

| Storage Type | Protocol | Access Mode | Use Case | CSI Driver |
|-------------|----------|-------------|----------|------------|
| **Azure Disk** | SCSI (block) | RWO (single node) | Databases, block storage | `disk.csi.azure.com` |
| **Azure Files** | SMB / NFS | RWX (multi-node) | Shared files, WordPress uploads | `file.csi.azure.com` |
| **Azure Blob** | NFS v3 / Blobfuse | RWX (multi-node) | Object storage, data lakes | `blob.csi.azure.com` |
| **Azure NetApp Files** | NFS v3/v4.1 | RWX (multi-node) | High-perf shared storage | Trident CSI driver |

### Azure Disk (Block Storage)

**What:** Virtual hard drives (like a USB drive attached to a computer)

**Access Mode:** `ReadWriteOnce` (RWO) — can only be mounted by **one node** at a time

**When to use:**
- Databases (MySQL, PostgreSQL, SQL Server, MongoDB)
- Caching layers (Redis)
- CI/CD build agents
- Any workload needing block-level access

**SKUs:**
- **Standard HDD**: Up to 500 IOPS, 60 MB/s — lowest cost, slow
- **Standard SSD**: Up to 6,000 IOPS, 750 MB/s — balanced
- **Premium SSD v1**: Up to 20,000 IOPS, 900 MB/s — high performance
- **Premium SSD v2**: Up to 80,000 IOPS, 1,200 MB/s — ultra-high performance
- **Ultra Disk**: Up to 160,000 IOPS, 4,000 MB/s — extreme performance

**Redundancy:** LRS or ZRS

**Limitations:**
- ❌ Cannot be shared between multiple nodes simultaneously (unless using Shared Disks feature)

### Azure Files (File Storage)

**What:** Fully managed file shares (like a network drive)

**Access Mode:** `ReadWriteMany` (RWX) — can be mounted by **multiple nodes** simultaneously

**When to use:**
- WordPress media uploads (multiple pods need access)
- Shared configuration files
- Logging (multiple pods writing to same log directory)
- Legacy applications expecting file shares

**Protocols:**
- **SMB (Server Message Block)**: Windows-compatible
- **NFS v4.1**: Linux-native, better performance than SMB

**SKUs:**
- **Standard**: Up to 10,000 IOPS (per share)
- **Premium**: Up to 100,000 IOPS

**Redundancy:** LRS, ZRS, GRS, GZRS

**Limitations:**
- ❌ Higher latency than Azure Disk (~5-10ms vs ~1-3ms)
- ❌ Not suitable for databases (POSIX compliance issues)

### Azure Blob (Object Storage)

**What:** Object storage (like AWS S3)

**Access Mode:** `ReadWriteMany` (RWX)

**When to use:**
- Backup storage
- Data lake / analytics workloads
- Media files, images, videos
- HPC (High-Performance Computing)

**Protocols:**
- **Blobfuse**: FUSE adapter (filesystem in userspace)
- **NFS v3**: Native mounting

**SKUs:**
- **Standard**: General purpose
- **Premium**: High performance (optimized for small files)

**Redundancy:** LRS, ZRS, GRS, GZRS

**Limitations:**
- ❌ Not POSIX-compliant (file locking doesn't work reliably)
- ❌ Not suitable for databases

### Azure NetApp Files

**What:** Enterprise-grade file storage powered by NetApp's ONTAP technology

**Access Mode:** `ReadWriteMany` (RWX)

**When to use:**
- High-performance NFS workloads
- SAP HANA
- HPC applications
- When you need sub-millisecond latency

**SKUs:**
- **Standard**: 16 MB/s per TiB
- **Premium**: 64 MB/s per TiB
- **Ultra**: 128 MB/s per TiB

**Cost:** Most expensive option (~3-10x more than Azure Files)

### Decision Tree

```
Do you need multi-node shared access (RWX)?
│
├─ NO → Use Azure Disk
│       │
│       └─ Is it a database or requires high IOPS?
│          ├─ YES → Premium SSD (ZRS for prod)
│          └─ NO  → Standard SSD
│
└─ YES → Need shared access
         │
         ├─ Is it file shares / legacy app?
         │  └─ Use Azure Files (Premium for high IOPS)
         │
         ├─ Is it object storage / analytics?
         │  └─ Use Azure Blob (NFS)
         │
         └─ Need enterprise performance / SAP / HPC?
            └─ Use Azure NetApp Files
```

---

## 7. Storage Classes in AKS — Built-in and Custom

### What is a StorageClass?

A **StorageClass** is a Kubernetes object that defines:
1. **Which provisioner** to use (`disk.csi.azure.com`, `file.csi.azure.com`)
2. **What parameters** to pass to the provisioner (disk SKU, redundancy, region)
3. **Reclaim policy** (what happens when PVC is deleted)
4. **Volume binding mode** (when to provision the disk)

Think of it as a **template** or **recipe** for creating storage.

### Built-in StorageClasses in AKS

```bash
kubectl get storageclass
```

**For Kubernetes 1.29+ (Multi-Zone Clusters):**
```
NAME                    PROVISIONER              RECLAIMPOLICY   VOLUMEBINDINGMODE
azurefile               file.csi.azure.com       Delete          Immediate
azurefile-csi           file.csi.azure.com       Delete          Immediate
azurefile-csi-premium   file.csi.azure.com       Delete          Immediate
azurefile-premium       file.csi.azure.com       Delete          Immediate
azureblob-nfs-premium   blob.csi.azure.com       Delete          Immediate
azureblob-fuse-premium  blob.csi.azure.com       Delete          Immediate
default                 disk.csi.azure.com       Delete          WaitForFirstConsumer
managed                 disk.csi.azure.com       Delete          WaitForFirstConsumer
managed-csi (default)   disk.csi.azure.com       Delete          WaitForFirstConsumer  ← ZRS (K8s 1.29+)
managed-csi-premium     disk.csi.azure.com       Delete          WaitForFirstConsumer  ← ZRS (K8s 1.29+)
```

**Important Change in AKS 1.29+:**
- **Before Kubernetes 1.29:** `managed-csi` and `managed-csi-premium` used **LRS** by default
- **After Kubernetes 1.29 (for multi-zone clusters):** They use **ZRS** by default

This means:
```yaml
# Before K8s 1.29
storageClassName: managed-csi-premium
# Created: Premium_LRS disk

# After K8s 1.29 (multi-zone cluster)
storageClassName: managed-csi-premium
# Created: Premium_ZRS disk ✅
```

### Built-in StorageClass Details

**`managed-csi` (Default)**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi
provisioner: disk.csi.azure.com
parameters:
  skuname: StandardSSD_ZRS   # ZRS in K8s 1.29+ multi-zone
  # skuname: StandardSSD_LRS # LRS in K8s 1.28 or single-zone
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**`managed-csi-premium`**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi-premium
provisioner: disk.csi.azure.com
parameters:
  skuname: Premium_ZRS       # ZRS in K8s 1.29+ multi-zone
  # skuname: Premium_LRS     # LRS in K8s 1.28 or single-zone
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### Creating a Custom StorageClass (Explicit LRS for Cost Savings)

If you're on Kubernetes 1.29+ and want to use **LRS instead of ZRS** (to save costs for non-critical workloads):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi-lrs-custom
provisioner: disk.csi.azure.com
parameters:
  skuname: Premium_LRS          # Explicitly LRS
  cachingmode: ReadOnly         # Optional: caching mode
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f custom-storageclass.yaml
```

Now use it in PVCs:
```yaml
storageClassName: managed-csi-lrs-custom
```

### StorageClass Parameters for Azure Disk

| Parameter | Values | Description |
|-----------|--------|-------------|
| `skuname` | `Standard_LRS`, `StandardSSD_LRS`, `StandardSSD_ZRS`, `Premium_LRS`, `Premium_ZRS`, `UltraSSD_LRS` | Disk SKU |
| `cachingmode` | `None`, `ReadOnly`, `ReadWrite` | Host caching (improves performance) |
| `diskEncryptionSetID` | Azure Resource ID | Customer-managed encryption keys |
| `networkAccessPolicy` | `AllowAll`, `DenyAll` | Network access to disk |
| `maxShares` | `2`, `3`, etc. | Shared disk (multi-node attach) |

### Volume Binding Mode

**`WaitForFirstConsumer` (Recommended)**
- Disk is **NOT created** until a pod uses the PVC
- Ensures disk is created in the **same zone as the pod**
- Prevents cross-zone attach failures

**`Immediate`**
- Disk is created **immediately** when PVC is created
- May cause issues in multi-zone clusters (disk in Zone 1, pod in Zone 2)

**Always use `WaitForFirstConsumer` for Azure Disk.**

---

## 8. Zone-Redundant Storage (ZRS) for High Availability

### What Problem Does ZRS Solve?

**Problem:** Your application has 99.9% availability SLA. But:
- LRS disk availability: ~99.9%
- VM availability (single zone): ~99.9%
- **Combined availability: ~99.8%** (multiplication of probabilities)

You miss your SLA!

**Solution with ZRS:**
- ZRS disk availability: ~99.99%
- Multi-zone VM deployment: ~99.99%
- **Combined availability: ~99.98%**

You exceed your SLA ✅

### How ZRS Works Under the Hood

```
┌─────────────────────────────────────────────────────────────┐
│  When you write data to a ZRS disk:                         │
│                                                              │
│  Application writes 1 MB of data                            │
│           │                                                  │
│           ▼                                                  │
│  Azure Disk service receives write                          │
│           │                                                  │
│      ┌────┴────┬────────┐                                   │
│      │         │        │                                   │
│      ▼         ▼        ▼                                   │
│   Zone 1    Zone 2   Zone 3                                 │
│   Copy 1    Copy 2   Copy 3                                 │
│      │         │        │                                   │
│      └─────────┴────────┘                                   │
│           │                                                  │
│           ▼                                                  │
│  All 3 writes acknowledged                                  │
│           │                                                  │
│           ▼                                                  │
│  Write operation returns SUCCESS to application             │
│                                                              │
│  Total time: ~2-3 milliseconds                              │
└─────────────────────────────────────────────────────────────┘
```

**Synchronous replication** means:
- The write is not considered successful until **all 3 copies** are confirmed
- This adds 1-2ms latency compared to LRS
- But guarantees **zero data loss** even if a zone fails immediately after write

### ZRS Availability Zones Deep Dive

**What is an Availability Zone?**

An **Availability Zone** is a physically separate datacenter within an Azure region, with:
- Independent power supply
- Independent cooling
- Independent networking
- Physically separated by kilometers

```
┌──────────────────────────────────────────────────────────────┐
│                  Azure Region: West Europe                   │
│                                                              │
│   ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│   │  Zone 1        │  │  Zone 2        │  │  Zone 3      │ │
│   │                │  │                │  │              │ │
│   │  Datacenter A  │  │  Datacenter B  │  │  Datacenter C│ │
│   │  (Amsterdam)   │  │  (Haarlem)     │  │  (Rotterdam) │ │
│   │                │  │                │  │              │ │
│   │  Own power     │  │  Own power     │  │  Own power   │ │
│   │  Own cooling   │  │  Own cooling   │  │  Own cooling │ │
│   │  Own network   │  │  Own network   │  │  Own network │ │
│   └────────────────┘  └────────────────┘  └──────────────┘ │
│          │                     │                   │         │
│          └─────────────────────┴───────────────────┘         │
│                    < 2ms latency between zones               │
└──────────────────────────────────────────────────────────────┘
```

**Regions with Availability Zones:**
Not all regions have zones. As of 2025, 60+ regions support availability zones, including:
- US: East US, West US 2, Central US
- Europe: West Europe, North Europe, UK South
- Asia: Southeast Asia, Japan East, Australia East

Check the latest: [Azure Availability Zones](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-service-support)

### ZRS Use Cases

✅ **Production databases** — Cannot afford multi-hour outages

✅ **Financial applications** — Regulatory requirements for HA

✅ **E-commerce** — Revenue loss during downtime

✅ **SaaS platforms** — Customer-facing applications

❌ **Dev/test environments** — LRS is fine (save costs)

❌ **Non-critical batch jobs** — Can restart if zone fails

---

## 9. Multi-Zone AKS Cluster with ZRS Disks

### Creating a Multi-Zone AKS Cluster

```bash
az aks create \
  --resource-group my-rg \
  --name my-aks-cluster \
  --location westeurope \
  --node-count 3 \
  --zones 1 2 3 \              # ← Enable availability zones
  --kubernetes-version 1.29 \
  --generate-ssh-keys
```

Verify nodes are spread across zones:
```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone

# NAME                                ZONE
# aks-nodepool1-12345678-vmss000000   westeurope-1
# aks-nodepool1-12345678-vmss000001   westeurope-2
# aks-nodepool1-12345678-vmss000002   westeurope-3
```

### Complete YAML: ZRS Disk Deployment

**Step 1: Create Custom ZRS StorageClass (Optional, as built-in now uses ZRS)**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi-zrs
provisioner: disk.csi.azure.com
parameters:
  skuname: Premium_ZRS          # Zone-Redundant Storage
  cachingmode: ReadOnly
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**Step 2: Create PersistentVolumeClaim**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-managed-disk-zrs
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-csi-zrs
  resources:
    requests:
      storage: 50Gi
```

**Step 3: Create Deployment Using the PVC**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-zrs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-zrs
  template:
    metadata:
      labels:
        app: nginx-zrs
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: azuredisk-zrs
          mountPath: "/mnt/azuredisk"
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
      volumes:
      - name: azuredisk-zrs
        persistentVolumeClaim:
          claimName: azure-managed-disk-zrs
```

**Deploy:**
```bash
kubectl apply -f storageclass-zrs.yaml
kubectl apply -f pvc-zrs.yaml
kubectl apply -f deployment-zrs.yaml
```

**Verify:**
```bash
# Check PVC is bound
kubectl get pvc azure-managed-disk-zrs
# NAME                      STATUS   VOLUME                                     CAPACITY   STORAGECLASS
# azure-managed-disk-zrs    Bound    pvc-abc123-def4-5678-90ab-cdef12345678    50Gi       managed-csi-zrs

# Check which node the pod is on
kubectl get pod -l app=nginx-zrs -o wide
# NAME                        READY   STATUS    NODE
# nginx-zrs-7d4f8c9b6-x7k2m   1/1     Running   aks-nodepool1-...-vmss000001 (Zone 2)

# Exec into pod and write data
kubectl exec -it nginx-zrs-7d4f8c9b6-x7k2m -- bash
echo "Hello ZRS!" > /mnt/azuredisk/test.txt
exit

# Simulate zone failure by deleting the pod
kubectl delete pod nginx-zrs-7d4f8c9b6-x7k2m

# Deployment recreates pod (possibly in a different zone)
kubectl get pod -l app=nginx-zrs -o wide
# NAME                        READY   STATUS    NODE
# nginx-zrs-7d4f8c9b6-p9q4r   1/1     Running   aks-nodepool1-...-vmss000002 (Zone 3)

# Data persists!
kubectl exec -it nginx-zrs-7d4f8c9b6-p9q4r -- cat /mnt/azuredisk/test.txt
# Hello ZRS!
```

### ZRS Disk Cost Calculation

**Example: 1TB Premium SSD**

| Redundancy | Monthly Cost (West Europe) | Cost Difference |
|-----------|---------------------------|----------------|
| Premium_LRS | ~$135 | Baseline |
| Premium_ZRS | ~$203 | +50% |

**Formula:**
```
ZRS Cost = LRS Cost × 1.5
```

Is it worth it? **Yes, for production workloads.** Consider:
- Downtime cost: If 1 hour of downtime = $10,000 in lost revenue
- ZRS prevents multi-hour outages during zone failures
- Extra $68/month is a tiny insurance premium

---

## 10. Azure Shared Disks — Multi-Node Attachment

### What is a Shared Disk?

A **Shared Disk** is an Azure Managed Disk that can be attached to **multiple VMs (or AKS nodes) simultaneously**.

**Normal Azure Disk:**
```
Disk ──► VM 1  (single attachment)
```

**Shared Disk:**
```
       ┌─► VM 1
       │
Disk ──┼─► VM 2
       │
       └─► VM 3
```

### Why Shared Disks?

**Use cases:**
- **Clustered databases** (SQL Server Failover Cluster, Oracle RAC)
- **Windows Server Failover Cluster (WSFC)**
- **SAP ASCS/SCS clustering**
- **Parallel file systems** (GFS2, OCFS2)
- **High-availability applications** that need shared block storage

**How it differs from Azure Files:**
- Azure Files: File-level sharing (RWX with SMB/NFS protocol)
- Shared Disk: Block-level sharing (multiple nodes see raw disk, coordinate access with cluster software)

### Shared Disk Limitations

⚠️ **Important constraints:**

1. **No filesystem by default** — It's raw block storage. You must use cluster-aware filesystems (GFS2, OCFS2) or coordinate access at the application level

2. **Max shares limit:**
   - Premium SSD: Up to 2-5 shares (depends on disk size)
   - Ultra Disk: Up to 5 shares

3. **Requires special handling in Kubernetes:**
   - `volumeMode: Block` (not `Filesystem`)
   - `devicePath` (not `mountPath`)
   - Application must manage raw block device

4. **Caching must be disabled:** `cachingmode: None`

5. **Not all regions support it** — Check [Azure Shared Disk regions](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-shared)

### Shared Disk in Kubernetes

**Access Mode:** `ReadWriteMany` (but in **Block mode**, not Filesystem mode)

**StorageClass:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shared-disk-sc
provisioner: disk.csi.azure.com
parameters:
  skuname: Premium_LRS
  maxShares: "3"              # ← Enable shared disk for up to 3 nodes
  cachingMode: None           # ← Required when maxShares > 1
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc-azuredisk
spec:
  accessModes:
  - ReadWriteMany
  volumeMode: Block           # ← Block mode, not Filesystem
  storageClassName: shared-disk-sc
  resources:
    requests:
      storage: 256Gi
```

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shared-disk-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shared-disk-app
  template:
    metadata:
      labels:
        app: shared-disk-app
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeDevices:         # ← volumeDevices, not volumeMounts
        - name: azuredisk
          devicePath: /dev/sdx # ← Raw block device path
      volumes:
      - name: azuredisk
        persistentVolumeClaim:
          claimName: shared-pvc-azuredisk
```

**Key differences:**
- `volumeDevices` instead of `volumeMounts`
- `devicePath: /dev/sdx` instead of `mountPath: /mnt/data`
- Application sees a raw block device, not a filesystem

### Shared Disk Behavior

```
┌──────────────────────────────────────────────────────────────┐
│                  AKS Cluster (3 Nodes)                       │
│                                                              │
│  Node 1              Node 2              Node 3              │
│  ┌──────┐          ┌──────┐          ┌──────┐              │
│  │ Pod  │          │ Pod  │          │ Pod  │              │
│  │  A   │          │  B   │          │  C   │              │
│  └──┬───┘          └──┬───┘          └──┬───┘              │
│     │                 │                 │                   │
│     │   All 3 pods see the same raw block device            │
│     └─────────────────┼─────────────────┘                   │
│                       │                                     │
│                       ▼                                     │
│               ┌──────────────────┐                          │
│               │   Shared Disk    │                          │
│               │   /dev/sdx       │                          │
│               └──────────────────┘                          │
│                                                              │
│  ⚠️ Application must coordinate writes!                     │
│  Without coordination → data corruption                      │
└──────────────────────────────────────────────────────────────┘
```

**Why coordination is needed:**
- All 3 pods can write to `/dev/sdx` simultaneously
- Without locks or coordination, writes can overwrite each other
- This is why Shared Disks are used with cluster-aware software (SQL FCI, Oracle RAC)

---

## 11. Shared Disk with ZRS for Ultimate Availability

### Combining Shared Disk + ZRS

You can combine **both features** for maximum availability:
- **Shared Disk:** Multiple nodes can attach
- **ZRS:** Data replicated across 3 availability zones

**StorageClass:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: zrs-shared-managed-csi
provisioner: disk.csi.azure.com
parameters:
  skuname: Premium_ZRS        # ← Zone-Redundant Storage
  maxShares: "3"              # ← Shared disk
  cachingMode: None           # ← Required for shared disks
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**Use case:** SQL Server Failover Cluster Instance across multiple zones

```
┌──────────────────────────────────────────────────────────────┐
│               AKS Cluster (3 Zones)                          │
│                                                              │
│  Zone 1              Zone 2              Zone 3              │
│  ┌──────┐          ┌──────┐          ┌──────┐              │
│  │ SQL  │          │ SQL  │          │ SQL  │              │
│  │Primary          │Standby│          │Standby│              │
│  └──┬───┘          └──┬───┘          └──┬───┘              │
│     │                 │                 │                   │
│     └─────────────────┼─────────────────┘                   │
│                       │                                     │
│                       ▼                                     │
│               ┌──────────────────┐                          │
│               │  Shared ZRS Disk │                          │
│               │                  │                          │
│               │  Replicated in:  │                          │
│               │  Zone 1, 2, 3    │                          │
│               └──────────────────┘                          │
│                                                              │
│  ✅ Zone 1 fails → SQL fails over to Zone 2 node           │
│  ✅ Disk remains accessible (ZRS)                           │
│  ✅ Zone 2 node takes over as primary                       │
│  ✅ Downtime: ~30 seconds (SQL FCI failover time)           │
└──────────────────────────────────────────────────────────────┘
```

**Benefits:**
- ✅ Survives zone failures (ZRS)
- ✅ Fast failover between nodes (Shared Disk)
- ✅ No data loss (synchronous replication)

**Cost:** ~50% more than LRS Shared Disk

---

## 12. Complete Examples with YAML

### Example 1: PostgreSQL with ZRS Disk

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_PASSWORD: "MySecurePassword123!"
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi-zrs-premium
provisioner: disk.csi.azure.com
parameters:
  skuname: Premium_ZRS
  cachingmode: ReadOnly
reclaimPolicy: Retain              # Keep data if PVC is deleted
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-csi-zrs-premium
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
  clusterIP: None  # Headless service
```

### Example 2: Monitoring Deployment with Azure Files (Shared Storage)

```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-zrs
provisioner: file.csi.azure.com
parameters:
  skuName: Premium_ZRS
  protocol: nfs
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-logs-pvc
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: azurefile-zrs
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-aggregator
spec:
  replicas: 3
  selector:
    matchLabels:
      app: log-aggregator
  template:
    metadata:
      labels:
        app: log-aggregator
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:latest
        volumeMounts:
        - name: logs
          mountPath: /var/log/shared
      volumes:
      - name: logs
        persistentVolumeClaim:
          claimName: shared-logs-pvc
```

All 3 replicas share the same Azure Files volume.

---

## 13. Comparison Tables and Decision Trees

### Storage Type Comparison

| Feature | Azure Disk (LRS) | Azure Disk (ZRS) | Azure Files | Azure Blob | Shared Disk |
|---------|-----------------|-----------------|-------------|------------|-------------|
| **Access Mode** | RWO | RWO | RWX | RWX | RWX (Block) |
| **Multi-node** | ❌ No | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **Zone failure** | ❌ Fails | ✅ Survives | ✅ Survives (ZRS) | ✅ Survives (ZRS) | ✅ Survives (ZRS) |
| **Performance** | Excellent | Excellent | Good | Fair | Excellent |
| **Latency** | ~1-2ms | ~2-3ms | ~5-10ms | ~10-20ms | ~1-2ms |
| **Max IOPS** | 20,000 | 20,000 | 100,000 | Variable | 20,000 |
| **Use case** | Databases | HA Databases | Shared files | Object storage | Clustered DBs |
| **Cost** | $ | $$ (+50%) | $$ | $ | $$$ |

### Redundancy Comparison

| Option | Copies | Locations | Durability | Cost | Use Case |
|--------|--------|-----------|------------|------|----------|
| **LRS** | 3 | 1 datacenter | 11 nines | $ | Dev/test |
| **ZRS** | 3 | 3 zones | 12 nines | $$ | **Production** |
| **GRS** | 6 | 2 regions | 16 nines | $$$ | Disaster recovery |
| **GZRS** | 6 | 3 zones + 2nd region | 16 nines | $$$$ | Critical data |

### Decision Tree: Which Storage to Use?

```
START: What does your application need?
│
├─ Database (MySQL, PostgreSQL, SQL Server, MongoDB)?
│  └─ Production?
│     ├─ YES → Azure Disk (Premium_ZRS)
│     └─ NO  → Azure Disk (Premium_LRS or Standard_LRS)
│
├─ Shared files between pods (WordPress uploads, logs)?
│  └─ Azure Files (ZRS for production)
│
├─ Clustered database (SQL FCI, Oracle RAC)?
│  └─ Shared Disk with ZRS
│
├─ Object storage (backups, analytics, media)?
│  └─ Azure Blob (NFS or Blobfuse)
│
└─ High-performance enterprise NFS?
   └─ Azure NetApp Files
```

---

## 14. Troubleshooting Storage Issues

### Problem 1: PVC Stuck in Pending

**Symptoms:**
```bash
kubectl get pvc
# NAME        STATUS    VOLUME   CAPACITY   STORAGECLASS
# my-pvc      Pending            0                      managed-csi-zrs
```

**Common Causes:**
1. No nodes available in the required zone
2. StorageClass doesn't exist
3. Quota exceeded
4. Invalid parameters in StorageClass

**Troubleshooting:**
```bash
# Describe PVC to see error
kubectl describe pvc my-pvc
# Look for Events section

# Common error: "failed to provision volume"
# Solution: Check StorageClass exists
kubectl get storageclass

# Check node zones
kubectl get nodes -o custom-columns=NAME:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone
```

### Problem 2: Pod Cannot Attach Disk (Cross-Zone Error)

**Symptoms:**
```bash
kubectl describe pod my-pod
# Events:
#   Warning  FailedAttachVolume  1m  attachdetach-controller  AttachVolume.Attach failed: disk is in zone westeurope-1 but pod is scheduled on node in zone westeurope-2
```

**Cause:** Using an LRS disk with `volumeBindingMode: Immediate` in a multi-zone cluster

**Solution:**
```yaml
# Change StorageClass to use WaitForFirstConsumer
volumeBindingMode: WaitForFirstConsumer

# Or switch to ZRS disk
parameters:
  skuname: Premium_ZRS
```

### Problem 3: Slow Disk Performance

**Symptoms:** Database is slow, high latency

**Troubleshooting:**
```bash
# Check disk SKU
kubectl get pv
kubectl describe pv <pv-name>
# Look for: Parameters.skuname

# Is it Standard_LRS? Upgrade to Premium_ZRS

# Check IOPS throttling in Azure Portal:
# Navigate to the Managed Disk resource
# Metrics → Disk IOPS → Check if hitting limits
```

### Problem 4: Out of Disk Space

**Symptoms:**
```bash
# Application logs show "No space left on device"
```

**Solution: Expand the PVC (if `allowVolumeExpansion: true`)**
```bash
# Edit PVC
kubectl edit pvc my-pvc

# Change:
spec:
  resources:
    requests:
      storage: 100Gi  # From 50Gi to 100Gi

# Restart pod to apply
kubectl rollout restart deployment my-app
```

---

## 15. Best Practices and Recommendations

### Storage Selection

✅ **Use ZRS for all production databases** — 50% cost increase is worth the availability

✅ **Use Premium SSD for databases** — Much better IOPS than Standard

✅ **Size disks appropriately** — Performance scales with size in Azure

✅ **Use `volumeBindingMode: WaitForFirstConsumer`** — Prevents cross-zone issues

✅ **Use StatefulSets for databases** — Guarantees pod-to-disk binding

### Cost Optimization

✅ **Use LRS for dev/test** — Save 50% compared to ZRS

✅ **Right-size disks** — Don't over-provision (you pay for allocated space, not used)

✅ **Use Standard SSD for non-critical workloads** — Much cheaper than Premium

✅ **Enable snapshots and backups** — Cheaper than replication for long-term retention

### High Availability

✅ **Multi-zone AKS cluster** — Distribute nodes across 3 zones

✅ **ZRS disks** — Survive zone failures

✅ **Pod anti-affinity** — Spread pods across zones

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: myapp
      topologyKey: topology.kubernetes.io/zone
```

✅ **Health probes** — Detect failures quickly

✅ **Resource limits** — Prevent noisy neighbors

### Monitoring

✅ **Monitor disk IOPS** — Azure Monitor metrics

✅ **Set alerts for disk space** — Before reaching 80%

✅ **Track PVC usage** — Kubernetes metrics

```bash
kubectl top pv
```

✅ **Monitor latency** — Application-level metrics

### Security

✅ **Enable encryption at rest** — All Azure Disks are encrypted by default

✅ **Use customer-managed keys (CMK)** — For compliance requirements

✅ **Network policies** — Restrict which pods can access storage

✅ **RBAC** — Limit who can create/delete PVCs

---

## References & Further Reading

- [Azure Storage Redundancy (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Zone-Redundant Storage for Azure Disks (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-redundancy)
- [AKS Storage Concepts (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/concepts-storage)
- [Azure Disk CSI Driver in AKS (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/azure-disk-csi)
- [Azure Shared Disks (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-shared)
- [Storage Best Practices for AKS (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-storage)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Container Storage Interface (CSI)](https://kubernetes-csi.github.io/docs/)