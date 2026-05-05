# AKS Storage with Azure Blob & BlobFuse
### The Ultimate Beginner-Friendly Deep Dive

> **What this document covers:** What Azure Blob Storage is, how it differs from Azure Disk and Azure Files, how the Blob CSI driver works in AKS, BlobFuse vs NFS v3 mounting protocols, dynamic vs static provisioning, Managed Identity authentication, bring-your-own storage accounts, Key Vault integration, StorageClass parameters, limitations, and complete practical examples — all explained from absolute scratch.

---

## Table of Contents
1. [What is Azure Blob Storage?](#1-what-is-azure-blob-storage)
2. [Why Use Blob Storage with AKS?](#2-why-use-blob-storage-with-aks)
3. [How Blob Storage Differs from Disk and Files](#3-how-blob-storage-differs-from-disk-and-files)
4. [The Azure Blob CSI Driver](#4-the-azure-blob-csi-driver)
5. [BlobFuse vs NFS v3 — Two Ways to Mount Blob](#5-blobfuse-vs-nfs-v3--two-ways-to-mount-blob)
6. [Enabling the Blob CSI Driver in AKS](#6-enabling-the-blob-csi-driver-in-aks)
7. [Built-in Storage Classes for Blob](#7-built-in-storage-classes-for-blob)
8. [Dynamic Provisioning — Automatic Blob Containers](#8-dynamic-provisioning--automatic-blob-containers)
9. [Static Provisioning — Bring Your Own Storage Account](#9-static-provisioning--bring-your-own-storage-account)
10. [Authentication Methods for Blob Storage](#10-authentication-methods-for-blob-storage)
11. [BlobFuse with Managed Identity](#11-blobfuse-with-managed-identity)
12. [Storing Secrets in Azure Key Vault](#12-storing-secrets-in-azure-key-vault)
13. [Custom StorageClass Parameters](#13-custom-storageclass-parameters)
14. [StatefulSets with Azure Blob Storage](#14-statefulsets-with-azure-blob-storage)
15. [Limitations of Azure Blob Storage in AKS](#15-limitations-of-azure-blob-storage-in-aks)
16. [Troubleshooting and Best Practices](#16-troubleshooting-and-best-practices)

---

## 1. What is Azure Blob Storage?

### The Basics

**Azure Blob Storage** is Microsoft's object storage solution for the cloud. Think of it as a massive, virtually unlimited storage locker where you can store:
- Files of any type (images, videos, logs, backups, datasets)
- Massive amounts of unstructured data (terabytes to petabytes)
- Data that is read frequently but written less often

**"Blob"** stands for **Binary Large Object** — essentially any file.

### Blob Storage Hierarchy

```
Storage Account                    ← Top-level resource (unique name globally)
   │
   ├── Container 1                 ← Like a folder/bucket
   │     ├── blob1.jpg             ← Individual files (blobs)
   │     ├── blob2.pdf
   │     └── subfolder/blob3.csv
   │
   ├── Container 2
   │     ├── backup-2024-01-01.tar.gz
   │     └── backup-2024-02-01.tar.gz
   │
   └── Container 3
         └── dataset/train/image001.png
```

**Storage Account:** The top-level Azure resource. Has a globally unique name (e.g., `mystorageaccount123`). Contains containers.

**Container:** A grouping mechanism (like a folder or S3 bucket). You can have unlimited containers per storage account.

**Blob:** An individual file. Can be up to 190.7 TiB in size per blob.

### Types of Blobs

| Type | Max Size | Use Case |
|------|----------|----------|
| **Block Blob** | 190.7 TiB | General purpose (images, videos, logs, backups) |
| **Append Blob** | 195 GiB | Logs, audit trails (optimized for append operations) |
| **Page Blob** | 8 TiB | Virtual hard disks (VHDs), random read/write |

In the AKS context, you'll almost always work with **Block Blobs**.

### Access Tiers

| Tier | Access Frequency | Cost | Use Case |
|------|-----------------|------|----------|
| **Hot** | Frequent | Higher storage, lower access | Active data, APIs |
| **Cool** | Infrequent (30+ days) | Lower storage, higher access | Backups, staging |
| **Cold** | Rarely (90+ days) | Even lower storage | Compliance archives |
| **Archive** | Almost never (180+ days) | Cheapest storage | Long-term retention |

---

## 2. Why Use Blob Storage with AKS?

### The Use Cases

Blob storage in AKS is ideal when your pods need to:

**1. Access Massive Datasets**
```
AI/ML Training Pod → Reads 500GB dataset from Blob Container
                     (Multiple pods read simultaneously)
```

**2. Write Logs to Centralized Storage**
```
App Pod 1 → Writes logs to /mnt/blob/pod1.log
App Pod 2 → Writes logs to /mnt/blob/pod2.log
App Pod 3 → Writes logs to /mnt/blob/pod3.log
             All writing to the same Blob Container
```

**3. Store and Serve User Uploads**
```
Upload API Pod → Writes file to /mnt/blob/uploads/image.jpg
CDN/Web Pod    → Reads file from /mnt/blob/uploads/image.jpg
                 Both pods mount the same Blob Container
```

**4. HPC / Big Data Processing**
```
Spark Pod 1 → Reads partition-1 from Blob
Spark Pod 2 → Reads partition-2 from Blob
Spark Pod 3 → Reads partition-3 from Blob
               Parallel access to massive data
```

**5. Backup Storage**
```
Database Pod → Writes nightly backup to /mnt/blob/backups/
               Blob storage is cheap, durable, geo-replicated
```

### Why Not Just Use Azure Disk?

| Scenario | Azure Disk | Azure Blob |
|----------|-----------|------------|
| Multiple pods reading same data | ❌ RWO (single node) | ✅ RWX (multi-node) |
| Storing 10TB+ of data | 💰 Expensive | ✅ Cheap |
| Frequent small writes (database) | ✅ Best performance | ❌ Higher latency |
| Large sequential reads (ML data) | ✅ Good | ✅ Good |
| Cost per GB | $$$ | $ |

**Rule of thumb:** Use Azure Disk for databases, use Azure Blob for everything else that's large and shared.

---

## 3. How Blob Storage Differs from Disk and Files

### Comparison Table

| Feature | Azure Disk | Azure Files | Azure Blob |
|---------|-----------|-------------|------------|
| **Type** | Block storage | File storage (SMB/NFS) | Object storage |
| **Protocol** | SCSI | SMB, NFS v4.1 | BlobFuse, NFS v3 |
| **Access Mode** | RWO | RWX | RWX |
| **Multi-pod** | ❌ Single node | ✅ Multi-node | ✅ Multi-node |
| **Max Size** | 64 TiB per disk | 100 TiB per share | Virtually unlimited |
| **Latency** | ~1-3ms | ~5-10ms | ~10-50ms |
| **IOPS** | Up to 160K (Ultra) | Up to 100K | Variable |
| **Cost per GB** | $$$ | $$ | $ |
| **POSIX compliant** | ✅ Yes | ✅ Yes | ⚠️ Limited |
| **Ideal for** | Databases | Shared files | Large datasets, backups |
| **CSI Driver** | `disk.csi.azure.com` | `file.csi.azure.com` | `blob.csi.azure.com` |

### Visual Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure Disk                                   │
│                                                                 │
│  ┌──────┐                                                      │
│  │ Pod  │───► /dev/sda (block device) ───► Azure Managed Disk  │
│  └──────┘    One pod, one node only                            │
│              Fastest, most expensive                            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Azure Files                                  │
│                                                                 │
│  ┌──────┐                                                      │
│  │ Pod1 │───┐                                                  │
│  └──────┘   │                                                  │
│  ┌──────┐   ├──► /mnt/share (SMB/NFS) ───► Azure File Share   │
│  │ Pod2 │───┘    Multiple pods, multiple nodes                 │
│  └──────┘        Medium speed, medium cost                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Azure Blob                                   │
│                                                                 │
│  ┌──────┐                                                      │
│  │ Pod1 │───┐                                                  │
│  └──────┘   │                                                  │
│  ┌──────┐   ├──► /mnt/blob (BlobFuse/NFS) ► Blob Container    │
│  │ Pod2 │───┘    Multiple pods, multiple nodes                 │
│  └──────┘        Cheapest, highest latency                     │
│  ┌──────┐   │    Virtually unlimited storage                   │
│  │ Pod3 │───┘                                                  │
│  └──────┘                                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. The Azure Blob CSI Driver

### What is a CSI Driver?

**CSI (Container Storage Interface)** is a standard that defines how Kubernetes talks to storage providers. Each storage type has its own CSI driver:

| Storage | CSI Driver | What It Does |
|---------|-----------|-------------|
| Azure Disk | `disk.csi.azure.com` | Creates/attaches/detaches Azure Managed Disks |
| Azure Files | `file.csi.azure.com` | Creates/mounts Azure File Shares |
| **Azure Blob** | **`blob.csi.azure.com`** | **Creates/mounts Azure Blob Containers** |

### What Runs in the Cluster?

When you enable the Blob CSI driver, AKS deploys pods in `kube-system`:

```bash
kubectl get pods -n kube-system | grep csi-blob
# csi-blob-node-8spm4    3/3    Running    0    100m
# csi-blob-node-ctv9c    3/3    Running    0    100m
# csi-blob-node-jbx9r    3/3    Running    0    100m
```

**`csi-blob-node-*`** — One pod per node (DaemonSet). These pods:
- Run the BlobFuse/NFS mount process on each node
- Handle volume mount/unmount operations
- Manage authentication to Azure Blob Storage

Each `csi-blob-node` pod contains **3 containers**:
1. **`liveness-probe`** — Health check
2. **`node-driver-registrar`** — Registers the CSI driver with kubelet
3. **`blob`** — The actual BlobFuse/NFS mount logic

---

## 5. BlobFuse vs NFS v3 — Two Ways to Mount Blob

### Method 1: BlobFuse (Filesystem in Userspace)

**BlobFuse** is a virtual filesystem driver that makes Azure Blob Storage appear as a regular Linux filesystem. It translates filesystem operations (read, write, mkdir, ls) into Azure Blob REST API calls.

```
Application writes to /mnt/blob/file.txt
         │
         ▼
BlobFuse intercepts the write (FUSE layer in userspace)
         │
         ▼
BlobFuse translates to Azure Blob REST API:
  PUT https://myaccount.blob.core.windows.net/mycontainer/file.txt
         │
         ▼
Azure Blob Storage stores the data
```

**BlobFuse2** is the current version (BlobFuse v1 is deprecated — migrate by September 2026).

**Protocols available:**
- `fuse` — BlobFuse v1 (deprecated)
- `fuse2` — BlobFuse v2 (recommended)

**Authentication required:** Yes — needs account key, SAS token, Managed Identity, or Service Principal

**Pros:**
- ✅ Works with any storage account (no special configuration needed)
- ✅ Local file caching for frequently accessed files
- ✅ Supports Managed Identity authentication
- ✅ Works across VNets (uses REST API over HTTPS)
- ✅ Supports Azure Data Lake Storage Gen2 (ADLS)

**Cons:**
- ❌ Higher latency (REST API overhead + FUSE layer)
- ❌ Not fully POSIX compliant (no file locking, limited symlinks)
- ❌ File caching uses local disk space on nodes

### Method 2: NFS v3 (Network File System)

**NFS v3** mounts the blob container as a native NFS mount. The storage account must have NFS v3 enabled at creation time.

```
Application writes to /mnt/blob/file.txt
         │
         ▼
Linux kernel NFS client sends NFS write request
         │
         ▼
Azure Blob Storage NFS endpoint receives write
         │
         ▼
Azure Blob Storage stores the data
```

**Authentication:** None needed (NFS v3 doesn't use account keys). Security is provided by VNet restrictions.

**Pros:**
- ✅ Better performance than BlobFuse (kernel-level NFS)
- ✅ No authentication secrets needed
- ✅ More POSIX-like behavior
- ✅ No local caching needed

**Cons:**
- ❌ Storage account must have NFS v3 enabled (cannot be changed after creation)
- ❌ AKS cluster must be in the **same or peered VNet** as the storage account
- ❌ Cannot use GRS/GZRS redundancy (only LRS/ZRS)
- ❌ Cannot use all blob features (soft delete, snapshots limited)

### BlobFuse vs NFS Comparison

| Feature | BlobFuse (fuse2) | NFS v3 |
|---------|-----------------|--------|
| **Protocol** | FUSE (userspace) | NFS (kernel) |
| **Performance** | Good (with caching) | Better (native) |
| **Latency** | ~20-50ms | ~10-20ms |
| **Authentication** | Key, SAS, MSI, SPN | None (VNet-based) |
| **VNet requirement** | ❌ Not required | ✅ Required (same/peered VNet) |
| **POSIX compliance** | ⚠️ Limited | ⚠️ Limited (better than fuse) |
| **File caching** | ✅ Yes (local cache) | ❌ No |
| **Cross-subscription** | ✅ Yes | ❌ No |
| **ADLS Gen2** | ✅ Yes | ❌ No |
| **GRS/GZRS** | ✅ Yes | ❌ No (LRS/ZRS only) |
| **Built-in StorageClass** | `azureblob-fuse-premium` | `azureblob-nfs-premium` |

### When to Use Which?

```
Need cross-VNet or cross-subscription access?
  └─ YES → BlobFuse

Need ADLS Gen2 (Data Lake)?
  └─ YES → BlobFuse

Need best performance?
  └─ YES → NFS v3

Need GRS/GZRS replication?
  └─ YES → BlobFuse

Need simplest auth (no secrets)?
  └─ YES → NFS v3

Default recommendation for most workloads?
  └─ NFS v3 (if VNet requirement is met)
```

---

## 6. Enabling the Blob CSI Driver in AKS

### On a New Cluster

```bash
az aks create \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --enable-blob-driver \
  --generate-ssh-keys
```

### On an Existing Cluster

```bash
az aks update \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --enable-blob-driver
```

### Verify the Driver is Running

```bash
kubectl get pods -n kube-system | grep csi-blob
# csi-blob-node-8spm4    3/3    Running    0    10m
# csi-blob-node-ctv9c    3/3    Running    0    10m
# csi-blob-node-jbx9r    3/3    Running    0    10m
```

One `csi-blob-node` pod per node. All should be `3/3 Running`.

### Verify Storage Classes

```bash
kubectl get storageclass | grep blob
# azureblob-fuse-premium    blob.csi.azure.com    Delete    Immediate    true    10m
# azureblob-nfs-premium     blob.csi.azure.com    Delete    Immediate    true    10m
```

### Disable the Driver (if needed)

```bash
az aks update \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --disable-blob-driver
```

---

## 7. Built-in Storage Classes for Blob

AKS creates two built-in StorageClasses when the Blob driver is enabled:

### `azureblob-fuse-premium`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azureblob-fuse-premium
provisioner: blob.csi.azure.com
parameters:
  skuName: Premium_LRS
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

- **Protocol:** BlobFuse (default `fuse2`)
- **SKU:** Premium_LRS (Premium performance with LRS)
- **Use case:** High-performance BlobFuse with local caching

### `azureblob-nfs-premium`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azureblob-nfs-premium
provisioner: blob.csi.azure.com
parameters:
  skuName: Premium_LRS
  protocol: nfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

- **Protocol:** NFS v3
- **SKU:** Premium_LRS
- **Use case:** Native NFS mounting with better performance
- **Requirement:** AKS cluster must be in same/peered VNet as storage account

---

## 8. Dynamic Provisioning — Automatic Blob Containers

### What is Dynamic Provisioning?

**Dynamic provisioning** = Kubernetes automatically creates the Storage Account and Blob Container when a PVC is created. You don't pre-create anything in Azure.

### NFS Example (Dynamic)

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: blob-nfs-pvc
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: azureblob-nfs-premium
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blob-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: blob-app
  template:
    metadata:
      labels:
        app: blob-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        volumeMounts:
        - name: blob-storage
          mountPath: /mnt/blob
      volumes:
      - name: blob-storage
        persistentVolumeClaim:
          claimName: blob-nfs-pvc
```

**What happens automatically:**
1. PVC is created
2. Blob CSI driver creates a new Storage Account in the MC_ resource group
3. Blob CSI driver creates a Blob Container inside that Storage Account
4. A PV is created and bound to the PVC
5. All 3 pods mount the container at `/mnt/blob`

### BlobFuse Example (Dynamic)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: blob-fuse-pvc
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: azureblob-fuse-premium
  resources:
    requests:
      storage: 100Gi
```

**Automatic resource creation:**
- Storage Account: `fuse8d87a1d1a8324d7484f` (auto-generated name)
- Blob Container: auto-created inside the storage account
- Kubernetes Secret: `azure-storage-account-fuse8d87a1d1a8324d7484f-secret` (contains account key)

### Verifying Created Resources

```bash
# Check PVC
kubectl get pvc blob-nfs-pvc
# NAME            STATUS   VOLUME                                     CAPACITY   STORAGECLASS
# blob-nfs-pvc    Bound    pvc-e458e45c-47ca-443f-807a-6b101a9b9614   100Gi      azureblob-nfs-premium

# Check PV
kubectl get pv
# NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY
# pvc-e458e45c-47ca-443f-807a-6b101a9b9614   100Gi      RWX            Delete

# For BlobFuse: Check auto-created secret (contains storage account key)
kubectl get secrets | grep azure-storage
# azure-storage-account-fuse8d87a1d1a8324d7484f-secret   Opaque   2   46m
```

---

## 9. Static Provisioning — Bring Your Own Storage Account

### When to Use Static Provisioning?

- You already have a Storage Account with data in it
- You want to use a specific storage account name
- You need cross-subscription access
- You want to control the storage account configuration (networking, encryption)

### NFS Static Provisioning

**Prerequisites:**
- Storage Account with NFS v3 enabled
- Storage Account in same/peered VNet as AKS
- Blob Container already created

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-blob-nfs
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: azureblob-nfs-premium
  csi:
    driver: blob.csi.azure.com
    readOnly: false
    volumeHandle: myresourcegroup_mystorageaccount_mycontainer
    volumeAttributes:
      resourceGroup: myresourcegroup
      storageAccount: mystorageaccount
      containerName: mycontainer
      protocol: nfs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-blob-nfs
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  volumeName: pv-blob-nfs
  storageClassName: azureblob-nfs-premium
```

**Important:** The `volumeHandle` must be **globally unique** within the cluster. Common format: `resourceGroup_storageAccount_containerName`.

### BlobFuse Static Provisioning (with Account Key)

**Step 1: Create the Kubernetes Secret**

```bash
kubectl create secret generic azure-secret \
  --from-literal=azurestorageaccountname=mystorageaccount \
  --from-literal=azurestorageaccountkey="Ey4HupQDodC2v9gMrwLsybiFblobJlZjxyz..."
```

**Step 2: Create PV and PVC**

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-blob-fuse
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: azureblob-fuse-premium
  mountOptions:
  - -o allow_other
  - --file-cache-timeout-in-seconds=120
  csi:
    driver: blob.csi.azure.com
    readOnly: false
    volumeHandle: pv-blob-fuse-unique-id
    volumeAttributes:
      resourceGroup: myresourcegroup
      storageAccount: mystorageaccount
      containerName: mycontainer
    nodeStageSecretRef:
      name: azure-secret
      namespace: default
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-blob-fuse
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  volumeName: pv-blob-fuse
  storageClassName: azureblob-fuse-premium
```

### Bring Your Own StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: blob-fuse-custom
provisioner: blob.csi.azure.com
parameters:
  resourceGroup: MY_EXISTING_RESOURCE_GROUP
  storageAccount: MY_EXISTING_STORAGE_ACCOUNT
  containerName: MY_EXISTING_CONTAINER
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

---

## 10. Authentication Methods for Blob Storage

### Overview

BlobFuse needs credentials to access Azure Blob Storage. The CSI driver supports 4 authentication methods:

| Method | `AzureStorageAuthType` | Secrets Needed | Recommended |
|--------|----------------------|----------------|-------------|
| **Account Key** | `key` | Yes (key in Secret) | ❌ Least secure |
| **SAS Token** | `sas` | Yes (SAS in Secret) | ⚠️ Time-limited |
| **Managed Identity** | `msi` | ❌ No secrets | ✅ **Recommended** |
| **Service Principal** | `spn` | Yes (client secret) | ⚠️ For cross-tenant |

**NFS v3 does not need authentication** — security is enforced at the network level (VNet restrictions).

### Method 1: Account Key

The simplest but least secure. The storage account key provides full access to the entire storage account.

```bash
kubectl create secret generic azure-secret \
  --from-literal=azurestorageaccountname=mystorageaccount \
  --from-literal=azurestorageaccountkey="youraccountkey..."
```

### Method 2: SAS Token

A **Shared Access Signature (SAS)** provides limited, time-bound access to a container.

```bash
kubectl create secret generic azure-sas-secret \
  --from-literal=azurestorageaccountname=mystorageaccount \
  --from-literal=azurestorageaccountsastoken="?sv=2022-11-02&ss=b&srt=co&sp=rl..."
```

### Method 3: Managed Identity (Recommended)

A **Managed Identity** is an Azure AD identity automatically managed by Azure. No passwords or secrets to store.

See [Section 11](#11-blobfuse-with-managed-identity) for full details.

### Method 4: Service Principal

A **Service Principal** with a client ID and secret. Used for cross-tenant or advanced scenarios.

---

## 11. BlobFuse with Managed Identity

### Why Managed Identity?

- ✅ **No secrets to manage** — No account keys or SAS tokens in Kubernetes Secrets
- ✅ **No rotation needed** — Azure handles token lifecycle
- ✅ **Audit trail** — Azure AD logs all access
- ✅ **Least privilege** — Grant only needed permissions (not full account access)

### How It Works

```
Pod mounts BlobFuse volume
         │
         ▼
BlobFuse CSI driver on the node
         │
         ▼
Requests token from Azure AD using Managed Identity
         │
         ▼
Azure AD returns OAuth token
         │
         ▼
BlobFuse uses token to call Blob Storage REST API
         │
         ▼
Azure Blob Storage validates token and serves data
```

### Setup Steps

**Step 1: Create a User-Assigned Managed Identity**

```bash
az identity create \
  --name blob-access-identity \
  --resource-group myResourceGroup
```

**Step 2: Get the Identity's Client ID and Object ID**

```bash
CLIENT_ID=$(az identity show \
  --name blob-access-identity \
  --resource-group myResourceGroup \
  --query clientId -o tsv)

OBJECT_ID=$(az identity show \
  --name blob-access-identity \
  --resource-group myResourceGroup \
  --query principalId -o tsv)

RESOURCE_ID=$(az identity show \
  --name blob-access-identity \
  --resource-group myResourceGroup \
  --query id -o tsv)

echo "Client ID: $CLIENT_ID"
echo "Object ID: $OBJECT_ID"
echo "Resource ID: $RESOURCE_ID"
```

**Step 3: Grant Storage Blob Data Contributor Role**

```bash
STORAGE_ACCOUNT_ID=$(az storage account show \
  --name mystorageaccount \
  --resource-group myResourceGroup \
  --query id -o tsv)

az role assignment create \
  --assignee-object-id $OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ACCOUNT_ID
```

**Step 4: Assign the Identity to AKS VMSS**

```bash
NODE_RG=$(az aks show \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --query nodeResourceGroup -o tsv)

VMSS_NAME=$(az vmss list \
  --resource-group $NODE_RG \
  --query '[0].name' -o tsv)

az vmss identity assign \
  --resource-group $NODE_RG \
  --name $VMSS_NAME \
  --identities $RESOURCE_ID
```

**Step 5: Create PV with Managed Identity**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-blob-msi
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: azureblob-fuse-premium
  mountOptions:
  - -o allow_other
  - --file-cache-timeout-in-seconds=120
  csi:
    driver: blob.csi.azure.com
    readOnly: false
    volumeHandle: storage4aks013-container01
    volumeAttributes:
      resourceGroup: rg-aks-cluster
      storageAccount: storage4aks013
      containerName: "container01"
      AzureStorageAuthType: msi             # ← Managed Identity
      AzureStorageIdentityClientID: ""       # Fill with Client ID
      AzureStorageIdentityObjectID: ""       # Fill with Object ID
      AzureStorageIdentityResourceID: ""     # Fill with Resource ID
```

**Step 6: Create PVC and Pod**

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-blob-msi
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  volumeName: pv-blob-msi
  storageClassName: azureblob-fuse-premium
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blob-msi-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blob-msi-app
  template:
    metadata:
      labels:
        app: blob-msi-app
    spec:
      containers:
      - name: app
        image: nginx
        volumeMounts:
        - name: blob-vol
          mountPath: /mnt/blob
      volumes:
      - name: blob-vol
        persistentVolumeClaim:
          claimName: pvc-blob-msi
```

---

## 12. Storing Secrets in Azure Key Vault

### Why Key Vault?

Instead of storing storage account keys in Kubernetes Secrets (which are only base64-encoded, not encrypted), you can store them in **Azure Key Vault** — a dedicated, encrypted secret management service.

### PV Configuration with Key Vault

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-blob-keyvault
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: azureblob-fuse-premium
  csi:
    driver: blob.csi.azure.com
    readOnly: false
    volumeHandle: unique-volume-id-keyvault
    volumeAttributes:
      containerName: mycontainer
      storageAccountName: mystorageaccount
      keyVaultURL: "https://mykeyvault.vault.azure.net/"
      keyVaultSecretName: "storage-account-key-secret"
```

**How it works:**
1. BlobFuse CSI driver reads `keyVaultURL` and `keyVaultSecretName`
2. Driver authenticates to Key Vault using AKS managed identity
3. Driver retrieves the storage account key from Key Vault
4. Driver uses the key to mount the blob container
5. No secrets stored in Kubernetes!

---

## 13. Custom StorageClass Parameters

### Full Parameter Reference

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: custom-blob-class
provisioner: blob.csi.azure.com
parameters:
  # Protocol: fuse, fuse2, nfs (default: fuse2)
  protocol: nfs

  # Storage Account SKU
  skuName: Standard_LRS    # Premium_LRS, Standard_GRS, Standard_RAGRS, Standard_ZRS

  # Location (defaults to AKS cluster location)
  location: westeurope

  # Existing resources (static-like behavior)
  resourceGroup: rg-aks-storage
  storageAccount: storageblobaks013
  containerName: "existing-container"

  # Container name prefix (for dynamic provisioning)
  containerNamePrefix: "aks-pvc-blob"

  # Security
  allowBlobPublicAccess: "false"

  # Tags
  tags: "tier=frontend,costcenter=10005"

  # Private endpoint (for private networking)
  server: "accountname.privatelink.blob.core.windows.net"
  networkEndpointType: privateEndpoint

  # ADLS Gen2
  isHnsEnabled: "true"          # Enable hierarchical namespace

volumeBindingMode: Immediate
allowVolumeExpansion: true
reclaimPolicy: Delete           # or Retain
```

### Key Parameters Explained

| Parameter | Values | Description |
|-----------|--------|-------------|
| `protocol` | `fuse`, `fuse2`, `nfs` | Mount protocol (`fuse2` is recommended) |
| `skuName` | `Standard_LRS`, `Premium_LRS`, `Standard_GRS`, `Standard_RAGRS`, `Standard_ZRS` | Storage redundancy |
| `containerNamePrefix` | Any string | Prefix for auto-created container names |
| `allowBlobPublicAccess` | `true`, `false` | Allow anonymous public access |
| `networkEndpointType` | `privateEndpoint` | Use private endpoint for access |
| `isHnsEnabled` | `true`, `false` | Enable Data Lake Storage Gen2 |
| `server` | FQDN | Private endpoint hostname |
| `tags` | `key=value,key2=value2` | Azure tags for created resources |

---

## 14. StatefulSets with Azure Blob Storage

### NFS StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-blob-nfs
  labels:
    app: nginx
spec:
  serviceName: statefulset-blob-nfs
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        volumeMounts:
        - name: persistent-storage
          mountPath: /mnt/blob
  volumeClaimTemplates:
  - metadata:
      name: persistent-storage
    spec:
      accessModes: [ "ReadWriteMany" ]
      storageClassName: azureblob-nfs-premium
      resources:
        requests:
          storage: 100Gi
```

**What this creates:**
```
statefulset-blob-nfs-0  →  PVC: persistent-storage-statefulset-blob-nfs-0  →  Blob Container 0
statefulset-blob-nfs-1  →  PVC: persistent-storage-statefulset-blob-nfs-1  →  Blob Container 1
statefulset-blob-nfs-2  →  PVC: persistent-storage-statefulset-blob-nfs-2  →  Blob Container 2
```

Each pod gets its own blob container, even though Blob supports ReadWriteMany.

### BlobFuse StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-blob-fuse
spec:
  serviceName: statefulset-blob-fuse
  replicas: 2
  selector:
    matchLabels:
      app: data-processor
  template:
    metadata:
      labels:
        app: data-processor
    spec:
      containers:
      - name: processor
        image: myapp:latest
        volumeMounts:
        - name: data-vol
          mountPath: /mnt/data
  volumeClaimTemplates:
  - metadata:
      name: data-vol
    spec:
      accessModes: [ "ReadWriteMany" ]
      storageClassName: azureblob-fuse-premium
      resources:
        requests:
          storage: 500Gi
```

---

## 15. Limitations of Azure Blob Storage in AKS

### Critical Limitations to Know

**1. Not Fully POSIX Compliant**
- ❌ File locking (`flock`, `fcntl`) not supported
- ❌ Hard links not supported
- ❌ Some file permissions may not work as expected
- **Impact:** Cannot run databases (MySQL, PostgreSQL) on Blob storage

**2. Rename/Move is Not Atomic**
- Renaming a file = copy + delete (slow for large files)
- **Impact:** Applications that rely on atomic renames may behave unexpectedly

**3. NFS v3 Specific Limitations**
- Storage account must have NFS v3 enabled at creation (cannot enable later)
- Must be in same/peered VNet as AKS
- Cannot use GRS/GZRS redundancy
- Limited soft delete and snapshot support

**4. BlobFuse Specific Limitations**
- Local file cache uses node disk space
- Higher latency than native filesystem
- Cache invalidation may cause stale reads in multi-node scenarios

**5. Capacity Reporting**
- `df -h` may show incorrect capacity (display issue, not actual limit)
- Blob storage is virtually unlimited; the PVC size is a Kubernetes abstraction

**6. Performance**
- Not suitable for low-latency random I/O (use Azure Disk instead)
- Best for sequential reads, large files, streaming workloads

### What NOT to Use Blob Storage For

| Workload | Use This Instead |
|----------|-----------------|
| Databases (MySQL, PostgreSQL) | Azure Disk (Premium SSD) |
| Low-latency caching (Redis) | Azure Disk |
| Small random I/O (OLTP) | Azure Disk |
| File locking required | Azure Files |
| Windows SMB shares | Azure Files |

---

## 16. Troubleshooting and Best Practices

### Common Issues

#### Issue 1: PVC Stuck in Pending (NFS)

**Cause:** AKS cluster not in same VNet as storage account, or NFS v3 not enabled.

```bash
kubectl describe pvc blob-nfs-pvc
# Events:
#   Warning  ProvisioningFailed  mount failed: nfs: server not responding
```

**Solution:**
- Ensure storage account has NFS v3 enabled
- Ensure AKS cluster is in same/peered VNet
- Ensure AKS cluster identity has `Contributor` role on VNet and NSG

#### Issue 2: BlobFuse Mount Permission Denied

**Cause:** Invalid account key or insufficient RBAC permissions.

```bash
kubectl describe pod blob-app-xxx
# Events:
#   Warning  FailedMount  MountVolume.SetUp failed: rpc error: AuthorizationPermissionMismatch
```

**Solution:**
- Verify Kubernetes Secret has correct account name and key
- For Managed Identity: ensure identity has `Storage Blob Data Contributor` role
- Check storage account firewall rules

#### Issue 3: Stale Data in Multi-Pod BlobFuse

**Cause:** BlobFuse caches files locally. Changes made by Pod A may not be visible to Pod B immediately.

**Solution:**
```yaml
mountOptions:
- --file-cache-timeout-in-seconds=0    # Disable caching for consistency
```

Or use NFS v3 (no caching issues).

### Best Practices

✅ **Use NFS v3 when possible** — Better performance, no auth complexity

✅ **Use Managed Identity** — No secrets to rotate

✅ **Use BlobFuse2** (`protocol: fuse2`) — BlobFuse v1 is deprecated

✅ **Set `reclaimPolicy: Retain`** for production — Prevent accidental data loss

✅ **Use `allowBlobPublicAccess: false`** — Never expose blob containers publicly

✅ **Use Private Endpoints** — Keep blob traffic on the Microsoft backbone

✅ **Monitor with Azure Monitor** — Track Blob storage metrics (transactions, latency)

✅ **Use Azure Data Lake Gen2** for analytics — Set `isHnsEnabled: true`

✅ **Tag resources** — Use `tags` parameter for cost tracking

---

## References & Further Reading

- [Azure Blob Storage CSI Driver on AKS (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/azure-blob-csi)
- [Create Persistent Volumes with Azure Blob Storage (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/create-volume-azure-blob-storage)
- [Blob CSI Driver GitHub](https://github.com/kubernetes-sigs/blob-csi-driver)
- [Blob CSI Driver Parameters](https://github.com/kubernetes-sigs/blob-csi-driver/blob/master/docs/driver-parameters.md)
- [BlobFuse2 Documentation](https://github.com/Azure/azure-storage-fuse)
- [Azure Blob Storage NFS v3 (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/storage/blobs/network-file-system-protocol-support)
- [AKS Storage Best Practices](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-storage)