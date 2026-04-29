# 📘 Doc 08 — Migrating AKS OutboundType: Load Balancer → NAT Gateway

> **Goal:** Understand *every supported migration path* between AKS outbound types, *why* they exist or are blocked, and walk through the most common migration (LB → userAssignedNATGateway) with every step explained in depth.

---

## 🧭 Table of Contents

1. [Why Would You Migrate?](#1-why-would-you-migrate)
2. [Two VNet Modes — Why They Matter for Migration](#2-two-vnet-modes--why-they-matter-for-migration)
3. [Full Migration Matrix — Managed VNet](#3-full-migration-matrix--managed-vnet)
4. [Full Migration Matrix — BYO VNet](#4-full-migration-matrix--byo-vnet)
5. [Reading the Matrix — Which Path Do You Need?](#5-reading-the-matrix--which-path-do-you-need)
6. [Why Some Paths Are Blocked](#6-why-some-paths-are-blocked)
7. [Step-by-Step: loadBalancer → userAssignedNATGateway (BYO VNet)](#7-step-by-step-loadbalancer--userassignednatgateway-byo-vnet)
8. [Step-by-Step: loadBalancer → managedNATGateway (Managed VNet)](#8-step-by-step-loadbalancer--managednatgateway-managed-vnet)
9. [What Happens During the Migration](#9-what-happens-during-the-migration)
10. [Impact on Existing Workloads](#10-impact-on-existing-workloads)
11. [Post-Migration Verification](#11-post-migration-verification)
12. [Pre-Migration Checklist](#12-pre-migration-checklist)
13. [Common Mistakes and How to Avoid Them](#13-common-mistakes-and-how-to-avoid-them)
14. [Rollback — Can You Undo a Migration?](#14-rollback--can-you-undo-a-migration)
15. [Azure CLI Version Requirement](#15-azure-cli-version-requirement)

---

## 1. Why Would You Migrate?

You started with the default AKS setup (`loadBalancer` egress) and now something has changed:

| Trigger | Details |
|---------|---------|
| **SNAT exhaustion** | Pods are getting connection failures to external services. Azure Monitor shows `SNATConnectionCount` with `State=Failed` spiking. |
| **Scaling pain** | Your cluster grew. With 200+ nodes, each node only gets 256 SNAT ports from the LB. Not enough for even moderate workloads. |
| **IP allowlisting** | A partner requires you to send traffic from a specific, stable IP. With LB you have one IP, but it might change if the LB is recreated. NAT Gateway gives you more explicit control. |
| **Separation of concerns** | Your team wants egress (outbound) fully separated from ingress (inbound). With LB they share a resource. With NAT Gateway they are completely independent. |
| **Cost optimization** | Running 5 LB IPs for SNAT scale is more expensive and less efficient than one NAT Gateway with 1 IP. |
| **Enterprise architecture adoption** | Adopting Azure Landing Zone or Hub & Spoke? You need UDR (Firewall egress). |

---

## 2. Two VNet Modes — Why They Matter for Migration

Every AKS cluster uses one of two VNet modes, and this determines which migration paths are available to you.

### Mode A: AKS-Managed VNet

When you create a cluster **without** specifying `--vnet-subnet-id`, AKS creates its own VNet and subnet inside the node resource group (`MC_*`).

```bash
az aks create -g my-rg -n my-cluster  # No --vnet-subnet-id → AKS manages VNet
```

- AKS owns and manages all networking resources
- You have limited direct access to the VNet/subnet (it's in AKS's resource group)
- Migration paths: `loadBalancer` ↔ `managedNATGateway` ↔ `managedNATGatewayV2`

### Mode B: BYO VNet (Bring Your Own)

When you pre-create a VNet and pass its subnet ID at cluster creation:

```bash
az aks create -g my-rg -n my-cluster --vnet-subnet-id $SUBNET_ID  # BYO VNet
```

- You own and manage the VNet, subnet, route tables, and NAT Gateways
- Full control over networking infrastructure
- Migration paths: `loadBalancer` ↔ `userAssignedNATGateway` ↔ `userDefinedRouting`

> **How to check which mode you're in:**
> ```bash
> az aks show -g $RG -n $NAME --query 'agentPoolProfiles[0].vnetSubnetId'
> # If output is null → AKS-managed VNet (Mode A)
> # If output is a subnet resource ID → BYO VNet (Mode B)
> ```

---

## 3. Full Migration Matrix — Managed VNet

When AKS manages your VNet, migration is only supported within the "managed" tier:

| From → To | `loadBalancer` | `managedNATGateway` | `managedNATGatewayV2` |
|-----------|---------------|---------------------|----------------------|
| **loadBalancer** | N/A | ✅ Supported | ✅ Supported |
| **managedNATGateway** | ✅ Supported | N/A | ✅ Supported |
| **managedNATGatewayV2** | ✅ Supported | ✅ Supported | N/A |

### Commands for Managed VNet Migrations

```bash
# loadBalancer → managedNATGateway
az aks update -g $RG -n $NAME \
  --outbound-type managedNATGateway \
  --nat-gateway-managed-outbound-ip-count 2

# managedNATGateway → loadBalancer
az aks update -g $RG -n $NAME \
  --outbound-type loadBalancer \
  --load-balancer-managed-outbound-ip-count 1

# managedNATGateway → managedNATGatewayV2 (Preview)
az aks update -g $RG -n $NAME \
  --outbound-type managedNATGatewayV2 \
  --nat-gateway-managed-outbound-ip-count 2
```

---

## 4. Full Migration Matrix — BYO VNet

When you manage your own VNet, migration is only supported within the "user-managed" tier:

| From → To | `loadBalancer` | `userAssignedNATGateway` | `userDefinedRouting` |
|-----------|---------------|--------------------------|---------------------|
| **loadBalancer** | N/A | ✅ Supported | ✅ Supported |
| **userAssignedNATGateway** | ✅ Supported | N/A | ✅ Supported |
| **userDefinedRouting** | ✅ Supported | ✅ Supported | N/A |

### Commands for BYO VNet Migrations

```bash
# loadBalancer → userAssignedNATGateway
# (You must first create the NAT GW and attach it to the subnet)
az aks update -g $RG -n $NAME \
  --outbound-type userAssignedNATGateway

# userAssignedNATGateway → loadBalancer
az aks update -g $RG -n $NAME \
  --outbound-type loadBalancer \
  --load-balancer-managed-outbound-ip-count 1

# loadBalancer → userDefinedRouting
# (You must first create the route table with 0.0.0.0/0 → NVA)
az aks update -g $RG -n $NAME \
  --outbound-type userDefinedRouting

# userDefinedRouting → userAssignedNATGateway
az aks update -g $RG -n $NAME \
  --outbound-type userAssignedNATGateway
```

---

## 5. Reading the Matrix — Which Path Do You Need?

Use this decision tree:

```
What is your current outbound type?
│
├─ loadBalancer
│   ├─ AKS-managed VNet? → Can migrate to: managedNATGateway, managedNATGatewayV2
│   └─ BYO VNet? → Can migrate to: userAssignedNATGateway, userDefinedRouting
│
├─ managedNATGateway
│   ├─ AKS-managed VNet? → Can migrate to: loadBalancer, managedNATGatewayV2
│   └─ BYO VNet? → ❌ NOT SUPPORTED (managedNATGateway is only for managed VNets)
│
├─ userAssignedNATGateway
│   ├─ BYO VNet? → Can migrate to: loadBalancer, userDefinedRouting
│   └─ AKS-managed VNet? → ❌ NOT SUPPORTED (userAssigned requires BYO VNet)
│
└─ userDefinedRouting
    └─ BYO VNet? → Can migrate to: loadBalancer, userAssignedNATGateway
```

---

## 6. Why Some Paths Are Blocked

Understanding why certain migrations are blocked helps you architect better from the start.

### Why `managedNATGateway → userAssignedNATGateway` is Blocked

`managedNATGateway` only works with AKS-managed VNets. `userAssignedNATGateway` requires BYO VNet. They are in different "tiers" (managed vs user-managed). Since the VNet type is set at cluster creation and cannot change, migration between these tiers is impossible.

### Why Mixing Managed and User-Managed Is Always Blocked

AKS cannot safely transfer ownership of networking resources between "AKS manages it" and "you manage it" mid-cluster-lifecycle. The resources are in different resource groups with different ownership semantics.

### Why `managedNATGateway → loadBalancer` IS Supported (Managed VNet)

Both are within the "AKS-managed" tier. AKS can:
1. Delete the NAT Gateway and its IPs
2. Create a Load Balancer and its IPs
3. Update the subnet configuration

All within resources it already owns.

### The Golden Rule

```
Managed VNet cluster:  only migrate between managed outbound types
BYO VNet cluster:      only migrate between user-managed outbound types
```

---

## 7. Step-by-Step: loadBalancer → userAssignedNATGateway (BYO VNet)

This is the most common real-world migration — upgrading a production BYO VNet cluster from the default Load Balancer to a NAT Gateway for scalable egress.

### Prerequisites Check

```bash
# 1. Confirm Azure CLI version ≥ 2.56
az version --query '"azure-cli"' -o tsv
# Must be 2.56 or higher. If not: az upgrade

# 2. Confirm this is a BYO VNet cluster
az aks show -g $RG -n $NAME \
  --query 'agentPoolProfiles[0].vnetSubnetId' -o tsv
# Must return a subnet resource ID (not null)

# 3. Note the current outbound IP (so you know what to update in allowlists)
az aks show -g $RG -n $NAME \
  --query 'networkProfile.loadBalancerProfile.effectiveOutboundIPs[].id' -o tsv | \
  xargs -I{} az network public-ip show --ids {} --query ipAddress -o tsv
# Save this IP — you'll need to replace it in any firewall rules or partner allowlists
```

### Step 1: Create a Standard Public IP for the NAT Gateway

```bash
az network public-ip create \
  --resource-group $RG \
  --name pip-natgateway \
  --sku standard \
  --allocation-method Static \
  --location $LOCATION
```

**What this does:**
- Creates an Azure Public IP resource with **Standard SKU**
- `--allocation-method Static` ensures this IP never changes as long as the resource exists
- This will be the IP that all pod egress traffic appears to come from
- Cost: approximately $3–4/month per IP

**Why Standard SKU is mandatory:**
- NAT Gateway ONLY supports Standard SKU Public IPs
- Basic SKU Public IPs are incompatible with NAT Gateway and are being deprecated across Azure
- Standard IPs also support zone-redundancy and higher availability

**Get the IP address for later use:**
```bash
NEW_EGRESS_IP=$(az network public-ip show \
  --resource-group $RG \
  --name pip-natgateway \
  --query ipAddress -o tsv)
echo "New egress IP will be: $NEW_EGRESS_IP"
# Save this for updating allowlists at external services
```

---

### Step 2: Create the NAT Gateway Resource

```bash
az network nat gateway create \
  --resource-group $RG \
  --name nat-gateway \
  --public-ip-addresses pip-natgateway \
  --idle-timeout 4 \
  --location $LOCATION
```

**What this does:**
- Creates an Azure NAT Gateway resource
- Attaches the public IP created in Step 1
- Sets idle timeout to 4 minutes (connections with no data flow for 4 minutes are closed and the SNAT port is released)
- The NAT Gateway is now created but NOT yet attached to any subnet — it doesn't do anything yet

**Idle timeout explained:**
- `4 minutes` (minimum, recommended for HTTP workloads): Ports get recycled quickly. Good for short-lived HTTP requests where connections close after each request.
- `10–30 minutes`: Good for persistent connections (WebSockets, database connection pools) where the connection stays open between uses.
- `120 minutes` (maximum): Very long timeout. Ports are held for a long time even when no data flows.

**Optional: Add more public IPs later**
```bash
# Add a second IP for more SNAT capacity
az network public-ip create -g $RG -n pip-natgateway-2 --sku standard
az network nat gateway update \
  -g $RG -n nat-gateway \
  --public-ip-addresses pip-natgateway pip-natgateway-2
```

---

### Step 3: Attach the NAT Gateway to the AKS Subnet

This is the critical networking step. Attaching the NAT Gateway to the subnet is what makes it take effect.

```bash
az network vnet subnet update \
  --resource-group $RG \
  --vnet-name vnet-aks \
  --name subnet-aks \
  --nat-gateway nat-gateway
```

**What this does:**
- Modifies the subnet's configuration to route outbound internet traffic through the NAT Gateway
- At this exact moment, any new connections from VMs/pods in this subnet start using the NAT Gateway's IP
- Existing connections from the Load Balancer continue briefly until they time out or close
- This is the step that causes **brief outage** (~seconds) as Azure updates routing

**Why attach BEFORE updating AKS?**
Because the NAT Gateway is a subnet-level Azure networking construct, independent of AKS. You want the subnet ready before telling AKS to acknowledge it. This ensures no gap where AKS removes the LB before the NAT GW is handling traffic.

**Verify the attachment:**
```bash
az network vnet subnet show \
  -g $RG --vnet-name vnet-aks --name subnet-aks \
  --query 'natGateway.id' -o tsv
# Should return the NAT Gateway resource ID
```

---

### Step 4: Update AKS to Acknowledge the New Outbound Type

```bash
az aks update \
  --resource-group $RG \
  --name aks-cluster \
  --outbound-type userAssignedNATGateway
```

**What this does:**
- Tells AKS that the outbound type is now `userAssignedNATGateway`
- AKS removes the outbound rule from the existing Load Balancer (the LB's outbound IP is no longer used for egress)
- AKS **deletes the egress Public IP** that was associated with the Load Balancer's outbound rule
- AKS **deletes the Load Balancer** itself (the `kubernetes` LB) — if no ingress services were using it
- Updates cluster metadata to reflect the new egress configuration
- **Takes approximately 6 minutes** (subnet reconfiguration and AKS control plane updates)

**What happens to ingress (public LoadBalancer services)?**
- If you had services of type `LoadBalancer` (ingress), their Load Balancers and IPs are **NOT affected**
- Only the egress-specific LB and IPs are cleaned up
- Ingress services continue working as before

**IMPORTANT: Update Allowlists Before This Step**  
Once AKS runs this update, the old egress IP (LB IP) is deleted. If external services only allow traffic from the old IP, your pods will lose access to those services immediately after this step. Update allowlists BEFORE running this command:

```bash
# At external services: add $NEW_EGRESS_IP to allowlist
# Then run:
az aks update -g $RG -n $NAME --outbound-type userAssignedNATGateway
# Then optionally: remove old LB IP from external service allowlists
```

---

## 8. Step-by-Step: loadBalancer → managedNATGateway (Managed VNet)

If AKS manages your VNet (simpler path — AKS does the heavy lifting):

```bash
# Single command — AKS handles everything
az aks update \
  --resource-group $RG \
  --name $NAME \
  --outbound-type managedNATGateway \
  --nat-gateway-managed-outbound-ip-count 2
```

**What AKS does automatically:**
1. Creates 2 new Standard Public IPs
2. Creates a NAT Gateway resource
3. Attaches the NAT Gateway to the AKS subnet
4. Removes the Load Balancer's outbound rule
5. Deletes the old egress Public IP
6. Deletes the old `kubernetes` Load Balancer (if no ingress services exist)
7. Updates cluster metadata

**Duration:** 6–15 minutes depending on cluster size

You don't need the 4-step manual process from Section 7 — AKS handles the subnet update and resource creation automatically.

---

## 9. What Happens During the Migration

Here's a precise timeline of what occurs between issuing `az aks update` and completion:

```
T+0:00  az aks update command submitted to Azure
T+0:30  AKS control plane begins processing
T+1:00  Subnet update begins — NAT Gateway associates with subnet
T+1:15  Brief routing disruption (~5-15 seconds): pods cannot egress
        (New connections fail; existing TCP connections may drop)
T+1:30  NAT Gateway is active — new connections use NAT GW IP
T+2:00  AKS updates Load Balancer — removes outbound rules
T+3:00  Old egress Public IP is disassociated from LB
T+4:00  AKS deletes old egress Public IP (it's now gone)
T+4:30  AKS deletes kubernetes Load Balancer (if no ingress services)
T+6:00  Migration complete, cluster reflects new outbound type
```

**Key insight:** The actual traffic disruption is very brief (seconds). Most of the 6-minute window is AKS cleaning up old resources and updating state.

---

## 10. Impact on Existing Workloads

| Workload type | Impact during migration | Impact after migration |
|---------------|------------------------|----------------------|
| Pods making HTTP calls to internet | Brief interruption (~5–15 sec) | Works normally, different egress IP |
| Pods connected to internal services (VNet) | **No impact** — internal traffic doesn't use SNAT | No impact |
| Kubernetes Services type LoadBalancer (ingress) | No impact — these aren't removed | No impact |
| Kubernetes Services type ClusterIP / NodePort | No impact | No impact |
| Long-running TCP connections to internet | May drop during routing change | Works normally after |
| CronJobs / batch workloads | If running during migration window, may fail their outbound call | Works normally after |

### Recommendation: Plan the Migration Window

- Migrate during **low-traffic periods** (nights/weekends)
- Inform your team about the ~15-second egress interruption
- Pause any CronJobs that make external calls during the window
- Have a rollback plan ready (see Section 14)

---

## 11. Post-Migration Verification

After the migration completes, verify everything is working:

### Check 1: Confirm New OutboundType in AKS

```bash
az aks show -g $RG -n $NAME \
  --query 'networkProfile.outboundType' -o tsv
# Expected: userAssignedNATGateway (or managedNATGateway)
```

### Check 2: Verify Egress IP Changed

```bash
# Deploy a test pod
kubectl run egress-test --image=curlimages/curl --restart=Never \
  --command -- curl -s ifconfig.me

kubectl logs egress-test
# Should show the NAT Gateway's public IP, NOT the old LB IP
```

### Check 3: Confirm Old LB is Gone (if no ingress services)

```bash
# List Load Balancers in node resource group
NODE_RG=$(az aks show -g $RG -n $NAME \
  --query 'nodeResourceGroup' -o tsv)

az network lb list -g $NODE_RG --query '[].name' -o tsv
# If no public services exist, this should be empty
# If public services exist, the LB for those services should appear
```

### Check 4: Verify NAT Gateway Is Active

```bash
# List NAT Gateways in node resource group (managedNATGateway) or your RG (userAssigned)
az network nat gateway list -g $NODE_RG --query '[].{name:name, state:provisioningState}' -o table
# Should show your NAT Gateway with state: Succeeded
```

### Check 5: Test External Connectivity from a Pod

```bash
kubectl exec egress-test -- curl -s https://api.github.com/zen
# If this returns any text, external HTTPS connectivity works
```

### Check 6: Update External Allowlists

```bash
# Get the new NAT Gateway public IPs
az network nat gateway show \
  -g $NODE_RG -n <natgw-name> \
  --query 'publicIpAddresses[].id' -o tsv | \
  while read ip_id; do
    az network public-ip show --ids $ip_id --query ipAddress -o tsv
  done
```

Give these IPs to any external partner/service that was allowlisting your old LB IP.

---

## 12. Pre-Migration Checklist

Use this checklist before running any migration command:

```
□ Azure CLI is version 2.56 or higher
  Command: az version --query '"azure-cli"' -o tsv

□ You know your current outbound type
  Command: az aks show -g $RG -n $NAME --query 'networkProfile.outboundType' -o tsv

□ You know whether you have a BYO VNet or AKS-managed VNet
  Command: az aks show -g $RG -n $NAME --query 'agentPoolProfiles[0].vnetSubnetId' -o tsv

□ You've identified the current egress IP(s) to notify partners
  Command: Check LB public IPs in node resource group

□ You've checked which external services/firewalls allowlist your current IP

□ You've prepared the new NAT Gateway IPs to share with those services

□ You've scheduled the migration in a low-traffic window

□ You've paused any CronJobs that make external calls

□ You have a rollback plan (see Section 14)

□ You've notified relevant team members about the brief disruption

□ For BYO VNet migrations:
  □ New Public IP is created with Standard SKU
  □ NAT Gateway is created and attached to the subnet
  □ Subnet attachment is verified
```

---

## 13. Common Mistakes and How to Avoid Them

### ❌ Mistake 1: Using Basic SKU Public IP

```bash
# WRONG: Creates a Basic SKU IP — will fail with NAT Gateway
az network public-ip create -g $RG -n pip-nat  # Missing --sku standard!

# CORRECT: Must explicitly use Standard SKU
az network public-ip create -g $RG -n pip-nat --sku standard
```

**Error you'll see if you do this:**
`The referenced public IP address /subscriptions/.../pip-nat of NAT gateway ... is not in the correct SKU (Standard).`

---

### ❌ Mistake 2: Trying `loadBalancer → managedNATGateway` on BYO VNet

```bash
# WRONG on BYO VNet:
az aks update -g $RG -n $NAME --outbound-type managedNATGateway

# Error: managedNATGateway is incompatible with custom virtual networks
# CORRECT on BYO VNet:
az aks update -g $RG -n $NAME --outbound-type userAssignedNATGateway
```

---

### ❌ Mistake 3: Forgetting to Attach NAT Gateway to Subnet (BYO VNet)

If you create the NAT Gateway but forget to attach it to the subnet before running `az aks update`:

```bash
# Step you might skip:
az network vnet subnet update \
  -g $RG --vnet-name vnet-aks --name subnet-aks \
  --nat-gateway nat-gateway  ← Don't forget this!

# Then running az aks update will appear to succeed, but pods
# won't have internet connectivity because no NAT GW is on the subnet
```

**How to fix:** Attach the NAT Gateway to the subnet after the fact — connectivity will restore.

---

### ❌ Mistake 4: Not Updating External Allowlists Before Migration

If you migrate and the old LB IP is deleted, any external service that was allowlisting that IP will now block your pods. The fix requires:
1. Get the new NAT Gateway IP(s)
2. Contact the external service to add the new IP(s) to their allowlist
3. Pods may be blocked from that service for hours/days while waiting for the allowlist update

**Prevention:** Get the new IP BEFORE migrating and update allowlists proactively.

---

### ❌ Mistake 5: Not Using Azure CLI 2.56+

Migration commands require Azure CLI 2.56 or higher. Older versions may silently ignore parameters or give confusing errors.

```bash
az upgrade  # Update Azure CLI to latest version
```

---

## 14. Rollback — Can You Undo a Migration?

Yes, for most paths. Refer back to the migration matrices (Sections 3 & 4).

### Rolling back `userAssignedNATGateway → loadBalancer` (BYO VNet)

```bash
# Step 1: Tell AKS to go back to loadBalancer
az aks update \
  --resource-group $RG \
  --name $NAME \
  --outbound-type loadBalancer \
  --load-balancer-managed-outbound-ip-count 1

# Step 2: Optionally detach NAT Gateway from subnet
az network vnet subnet update \
  -g $RG --vnet-name vnet-aks --name subnet-aks \
  --nat-gateway ""  # empty string = remove NAT GW

# Step 3: Optionally delete NAT Gateway if no longer needed
az network nat gateway delete -g $RG -n nat-gateway
az network public-ip delete -g $RG -n pip-natgateway
```

### Rolling back `managedNATGateway → loadBalancer` (Managed VNet)

```bash
az aks update \
  --resource-group $RG \
  --name $NAME \
  --outbound-type loadBalancer \
  --load-balancer-managed-outbound-ip-count 1
```

AKS handles all resource cleanup automatically.

### Important: New Egress IP After Rollback

Rolling back creates a **new** Load Balancer with a **new** public IP. This is NOT the same IP you had before the migration. Update allowlists accordingly.

---

## 15. Azure CLI Version Requirement

Microsoft requires **Azure CLI version 2.56 or higher** to migrate outbound types. This requirement exists because the migration feature was added in the `aks` CLI module in version 2.56.

```bash
# Check current version
az version

# Upgrade if needed
az upgrade

# Verify
az version --query '"azure-cli"' -o tsv
# Must be 2.56.0 or higher
```

If you use Terraform or ARM templates instead of Azure CLI, ensure you're using a provider version that supports the `outboundType` update operation (Azure provider ≥ 3.x for Terraform).

---

## 📋 Full Command Reference

```bash
# ── MANAGED VNET MIGRATIONS ──────────────────────────────────────

# loadBalancer → managedNATGateway
az aks update -g $RG -n $NAME \
  --outbound-type managedNATGateway \
  --nat-gateway-managed-outbound-ip-count 2

# managedNATGateway → loadBalancer
az aks update -g $RG -n $NAME \
  --outbound-type loadBalancer \
  --load-balancer-managed-outbound-ip-count 1

# ── BYO VNET MIGRATIONS ──────────────────────────────────────────

# PREP: Create NAT GW resources
az network public-ip create -g $RG -n pip-natgateway --sku standard
az network nat gateway create -g $RG -n nat-gateway \
  --public-ip-addresses pip-natgateway
az network vnet subnet update -g $RG --vnet-name vnet-aks \
  --name subnet-aks --nat-gateway nat-gateway

# loadBalancer → userAssignedNATGateway
az aks update -g $RG -n $NAME \
  --outbound-type userAssignedNATGateway

# userAssignedNATGateway → loadBalancer
az aks update -g $RG -n $NAME \
  --outbound-type loadBalancer \
  --load-balancer-managed-outbound-ip-count 1

# loadBalancer → userDefinedRouting
# (Requires route table with 0.0.0.0/0 → NVA already on subnet)
az aks update -g $RG -n $NAME \
  --outbound-type userDefinedRouting

# userDefinedRouting → userAssignedNATGateway
az aks update -g $RG -n $NAME \
  --outbound-type userAssignedNATGateway
```

---

## 📚 Official References

- [Supported Migration Paths](https://learn.microsoft.com/en-us/azure/aks/egress-outboundtype#supported-migration-paths-for-byo-vnet)
- [NAT Gateway with AKS](https://learn.microsoft.com/en-us/azure/aks/nat-gateway)
- [Azure Standard Public IP](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-addresses#sku)
- [Azure NAT Gateway Resource](https://learn.microsoft.com/en-us/azure/virtual-network/nat-gateway/natgateway-resource)