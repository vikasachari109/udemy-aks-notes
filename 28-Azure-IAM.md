# Azure IAM & Microsoft Entra ID
### The Ultimate Beginner-Friendly Deep Dive

> **What this document covers:** What Identity and Access Management (IAM) is, what Microsoft Entra ID (formerly Azure Active Directory) is, how tenants/directories work, every identity type (Users, Groups, Service Principals, Managed Identities, App Registrations), Azure RBAC vs Entra RBAC, roles and scopes, the 4 fundamental built-in roles, custom roles, **Kubernetes Service Accounts, Kubernetes RBAC (Roles/ClusterRoles/Bindings), AKS Workload Identity, Azure RBAC for Kubernetes authorization**, Conditional Access, Privileged Identity Management, and practical examples — all explained from absolute scratch.

---

## Table of Contents
1. [What is Identity and Access Management (IAM)?](#1-what-is-identity-and-access-management-iam)
2. [What is Microsoft Entra ID?](#2-what-is-microsoft-entra-id)
3. [Tenant, Directory, and Subscription Relationship](#3-tenant-directory-and-subscription-relationship)
4. [Identity Types — Who or What Can Access Resources](#4-identity-types--who-or-what-can-access-resources)
5. [Authentication vs Authorization](#5-authentication-vs-authorization)
6. [Azure RBAC — Role-Based Access Control for Resources](#6-azure-rbac--role-based-access-control-for-resources)
7. [Scopes — Where Roles Apply](#7-scopes--where-roles-apply)
8. [The 4 Fundamental Built-in Roles](#8-the-4-fundamental-built-in-roles)
9. [Service-Specific Built-in Roles](#9-service-specific-built-in-roles)
10. [Entra ID Roles vs Azure RBAC Roles](#10-entra-id-roles-vs-azure-rbac-roles)
11. [Service Principals and App Registrations](#11-service-principals-and-app-registrations)
12. [Managed Identities — System vs User Assigned](#12-managed-identities--system-vs-user-assigned)
13. [Kubernetes Service Accounts](#13-kubernetes-service-accounts)
14. [Kubernetes RBAC — Roles, ClusterRoles, and Bindings](#14-kubernetes-rbac--roles-clusterroles-and-bindings)
15. [AKS Workload Identity — Pods Accessing Azure Resources](#15-aks-workload-identity--pods-accessing-azure-resources)
16. [Azure RBAC for Kubernetes Authorization](#16-azure-rbac-for-kubernetes-authorization)
17. [Custom Roles](#17-custom-roles)
18. [Conditional Access Policies](#18-conditional-access-policies)
19. [Privileged Identity Management (PIM)](#19-privileged-identity-management-pim)
20. [Practical Examples and Best Practices](#20-practical-examples-and-best-practices)

---

## 1. What is Identity and Access Management (IAM)?

### The Core Question

Every time anyone (or any application) tries to access any resource in Azure, the system asks two fundamental questions:

```
Question 1: WHO ARE YOU?            → Authentication (AuthN)
Question 2: WHAT ARE YOU ALLOWED TO DO?  → Authorization (AuthZ)
```

**Identity and Access Management (IAM)** is the framework that answers both questions. It controls:
- **Who** can access resources (users, applications, services)
- **What** they can do (read, write, delete, manage)
- **Which** resources they can access (specific VMs, storage accounts, entire subscriptions)
- **When** and **how** they can access (conditional policies, MFA requirements)

### The Real-World Analogy

Think of Azure like a large corporate office building:

```
┌──────────────────────────────────────────────────────────────┐
│                    Corporate Building (Azure)                │
│                                                              │
│  SECURITY DESK (Entra ID / Authentication)                   │
│  "Show me your badge" → Verifies identity                   │
│                                                              │
│  ACCESS CONTROL SYSTEM (Azure RBAC / Authorization)          │
│  "Your badge grants access to floors 3-5 only"              │
│                                                              │
│  Floor 3: Marketing Department  → You can VIEW files        │
│  Floor 4: Engineering Department → You can EDIT files        │
│  Floor 5: Server Room            → You can MANAGE servers   │
│  Floor 10: CEO Office            → ACCESS DENIED             │
│                                                              │
│  Your BADGE = Your Identity (User, Service Principal, MI)   │
│  Your ACCESS LEVEL = Your Role (Reader, Contributor, Owner) │
│  The FLOOR = The Scope (Subscription, RG, Resource)         │
└──────────────────────────────────────────────────────────────┘
```

### The Three Pillars of IAM

```
┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐
│   IDENTITY      │  │  AUTHENTICATION  │  │  AUTHORIZATION     │
│                 │  │                  │  │                    │
│  WHO are you?   │  │  PROVE it.       │  │  WHAT can you do?  │
│                 │  │                  │  │                    │
│  • User         │  │  • Password      │  │  • Owner           │
│  • Group        │  │  • MFA           │  │  • Contributor     │
│  • App (SPN)    │  │  • Certificate   │  │  • Reader          │
│  • Managed ID   │  │  • Token         │  │  • Custom Role     │
│                 │  │  • Biometrics    │  │                    │
└─────────────────┘  └──────────────────┘  └────────────────────┘
       │                     │                      │
       └─────────────────────┴──────────────────────┘
                             │
                    Microsoft Entra ID
                    handles all three
```

---

## 2. What is Microsoft Entra ID?

### The Name Change

**Microsoft Entra ID** is the new name for **Azure Active Directory (Azure AD)**. Microsoft renamed it in July 2023 to avoid confusion with Windows Server Active Directory (on-premises AD).

| Old Name | New Name | Still Works? |
|----------|----------|-------------|
| Azure Active Directory (Azure AD) | **Microsoft Entra ID** | Yes (in CLI, APIs) |
| Azure AD B2C | Microsoft Entra External ID | Yes |
| Azure AD B2B | Microsoft Entra B2B | Yes |

All CLI commands still use `az ad` (e.g., `az ad user list`, `az ad sp create-for-rbac`).

### What Entra ID Does

**Microsoft Entra ID** is Microsoft's cloud-based identity and access management service. It is the **central brain** that:

1. **Stores identities** — Users, Groups, Applications, Managed Identities
2. **Authenticates** — Validates passwords, MFA, certificates, tokens
3. **Authorizes** — Determines what an identity can access
4. **Issues tokens** — OAuth 2.0 / OpenID Connect tokens for applications
5. **Enables SSO** — Single Sign-On across thousands of SaaS apps
6. **Enforces policies** — Conditional Access, MFA requirements

### Entra ID is NOT Windows Active Directory

| Feature | Windows AD (On-Premises) | Microsoft Entra ID (Cloud) |
|---------|-------------------------|---------------------------|
| **Location** | On your servers | Microsoft's cloud |
| **Protocol** | Kerberos, LDAP, NTLM | OAuth 2.0, OpenID Connect, SAML |
| **Structure** | Forests, Domains, OUs | Flat (Tenants, Users, Groups) |
| **Devices** | Domain-joined PCs | Any device, any location |
| **Use case** | Corporate network | Cloud apps, SaaS, Azure resources |
| **Scale** | Thousands of users | Millions of users |

You can **connect** them using **Entra Connect** (hybrid identity), but they are fundamentally different systems.

---

## 3. Tenant, Directory, and Subscription Relationship

### What is a Tenant?

A **tenant** is a dedicated instance of Microsoft Entra ID that represents your organization. When your company signs up for Azure, Microsoft 365, or any Microsoft cloud service, a tenant is created.

Every tenant has:
- A **unique Tenant ID** (a GUID like `16b3c013-d300-468d-ac64-7eda0820b6d3`)
- A **primary domain** (like `mycompany.onmicrosoft.com`)
- Custom domains (like `mycompany.com` after verification)

### The Hierarchy

```
┌──────────────────────────────────────────────────────────────────┐
│                    Microsoft Entra ID Tenant                     │
│                    (e.g., mycompany.onmicrosoft.com)              │
│                    Tenant ID: 16b3c013-d300-...                  │
│                                                                  │
│    Contains: Users, Groups, App Registrations, Service Principals│
│              Managed Identities, Enterprise Applications         │
│                                                                  │
│    ┌─────────────────────────────────────────────────────────┐   │
│    │  Management Group (optional)                            │   │
│    │  Organizes subscriptions                                │   │
│    │                                                         │   │
│    │    ┌──────────────────────┐  ┌──────────────────────┐  │   │
│    │    │  Subscription A      │  │  Subscription B      │  │   │
│    │    │  (Production)        │  │  (Development)       │  │   │
│    │    │                      │  │                      │  │   │
│    │    │  ┌────────────────┐  │  │  ┌────────────────┐  │  │   │
│    │    │  │ Resource Group │  │  │  │ Resource Group │  │  │   │
│    │    │  │ (rg-prod-web)  │  │  │  │ (rg-dev-api)   │  │  │   │
│    │    │  │                │  │  │  │                │  │  │   │
│    │    │  │ • VM           │  │  │  │ • AKS Cluster  │  │  │   │
│    │    │  │ • Storage      │  │  │  │ • SQL Database │  │  │   │
│    │    │  │ • Key Vault    │  │  │  │ • App Service  │  │  │   │
│    │    │  └────────────────┘  │  │  └────────────────┘  │  │   │
│    │    └──────────────────────┘  └──────────────────────┘  │   │
│    └─────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

**Key relationships:**
- **One tenant** can have **many subscriptions**
- **One subscription** trusts exactly **one tenant**
- **One subscription** has **many resource groups**
- **One resource group** has **many resources**

### The Trust Relationship

```
Tenant (Entra ID)  ←──── trusts ────  Subscription
   │                                       │
   │ Contains identities                   │ Contains resources
   │ (users, groups, apps)                 │ (VMs, storage, AKS)
   │                                       │
   └───── Authentication ─────────────────►│
          "Who is this user?"              │
          "What role do they have?"        │
                                           │
                                  ┌────────▼────────┐
                                  │ Resource Group   │
                                  │  └── Resource    │
                                  └─────────────────┘
```

---

## 4. Identity Types — Who or What Can Access Resources

### Overview of All Identity Types

```
┌──────────────────────────────────────────────────────────────────┐
│                    Security Principals                           │
│            (Things that can be assigned roles)                   │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌────────────┐  │
│  │   User   │  │  Group   │  │   Service    │  │  Managed   │  │
│  │          │  │          │  │  Principal   │  │  Identity  │  │
│  │ A human  │  │ A set of │  │ An app's     │  │ Azure-     │  │
│  │ with     │  │ users or │  │ identity in  │  │ managed    │  │
│  │ email +  │  │ other    │  │ Entra ID     │  │ identity   │  │
│  │ password │  │ groups   │  │ (has client  │  │ (no creds  │  │
│  │          │  │          │  │  secret or   │  │  to manage)│  │
│  │          │  │          │  │  certificate)│  │            │  │
│  └──────────┘  └──────────┘  └──────────────┘  └────────────┘  │
│       │              │              │                 │          │
│       └──────────────┴──────────────┴─────────────────┘          │
│                              │                                   │
│                     Can be assigned                               │
│                     Azure RBAC Roles                              │
│                     at any scope                                  │
└──────────────────────────────────────────────────────────────────┘
```

### User

A **User** is a human identity. Created in Entra ID with email, name, and authentication method.

```bash
# Create a user
az ad user create \
  --display-name "Alice Developer" \
  --user-principal-name alice@mycompany.onmicrosoft.com \
  --password "TempPassword123!"
```

**Types of users:**
- **Member**: Full internal user in your tenant
- **Guest**: External user invited via B2B collaboration (e.g., a contractor)

### Group

A **Group** is a collection of users, service principals, or other groups. Assigning a role to a group gives that role to all members.

```bash
# Create a security group
az ad group create \
  --display-name "AKS-Admins" \
  --mail-nickname "aks-admins"

# Add a user to the group
az ad group member add \
  --group "AKS-Admins" \
  --member-id <user-object-id>
```

**Types:**
- **Security Group**: Used for RBAC role assignments
- **Microsoft 365 Group**: Used for collaboration (Teams, SharePoint)

### Service Principal (SPN)

A **Service Principal** is the identity of an application in your Entra ID tenant. When you register an application, Entra ID creates a service principal that acts as the app's "badge" in that tenant.

```bash
# Create a service principal
az ad sp create-for-rbac \
  --name "my-cicd-pipeline" \
  --role "Contributor" \
  --scopes "/subscriptions/<sub-id>/resourceGroups/my-rg"

# Output:
# {
#   "appId": "9cc6c0d1-...",       ← Client ID
#   "password": "~K~8Q~PA9...",    ← Client Secret
#   "tenant": "16b3c013-..."       ← Tenant ID
# }
```

**Use case:** CI/CD pipelines, automation scripts, External-DNS, Terraform

### Managed Identity

A **Managed Identity** is a special type of service principal that Azure manages automatically. No secrets to store or rotate.

**Two types:**

| Type | Created By | Lifecycle | Use Case |
|------|-----------|-----------|----------|
| **System-Assigned** | Azure (when enabled on a resource) | Tied to the resource (deleted with it) | Single-resource access (e.g., one AKS cluster accessing Key Vault) |
| **User-Assigned** | You (manually) | Independent (survives resource deletion) | Shared across multiple resources |

```bash
# Create user-assigned managed identity
az identity create \
  --name my-aks-identity \
  --resource-group my-rg
```

**Why Managed Identity over Service Principal?**
- ✅ No passwords to manage
- ✅ No secret rotation needed
- ✅ Azure handles token lifecycle
- ✅ Cannot be accidentally exposed in code
- ✅ Recommended by Microsoft for all Azure-hosted workloads

---

## 5. Authentication vs Authorization

### Authentication (AuthN) — "Prove Who You Are"

```
User types email + password
         │
         ▼
Entra ID checks credentials
         │
    ┌────┴────┐
    │  Valid?  │
    ├── YES ──┤──────────────────────────────────────┐
    └── NO ───┘ → 401 Unauthorized                  │
                                                      │
    MFA Required?                                     │
    ├── YES → Send push notification / SMS code      │
    │         User approves → Continue                │
    └── NO  → Continue                                │
                                                      │
    Conditional Access Policy?                         │
    ├── Block if risky sign-in                        │
    ├── Require compliant device                      │
    └── Pass → Issue Token                            │
                                                      │
                                            ┌─────────▼──────────┐
                                            │  OAuth 2.0 Token   │
                                            │  (JWT)              │
                                            │                     │
                                            │  Contains:          │
                                            │  - User ID          │
                                            │  - Tenant ID        │
                                            │  - Expiry time      │
                                            │  - Permissions      │
                                            └─────────────────────┘
```

### Authorization (AuthZ) — "What Are You Allowed to Do?"

```
User makes request: "Create a VM in resource group rg-prod"
         │
         ▼
Azure Resource Manager receives request + token
         │
         ▼
ARM checks: Does this user have a role assignment
that grants Microsoft.Compute/virtualMachines/write
on /subscriptions/xxx/resourceGroups/rg-prod?
         │
    ┌────┴────┐
    │  Match? │
    ├── YES ──┤ → 200 OK (VM created)
    └── NO ───┘ → 403 Forbidden
```

### The Role Assignment Formula

Every authorization decision in Azure follows this formula:

```
WHO  +  WHAT  +  WHERE  =  Role Assignment

WHO   = Security Principal (User, Group, SPN, Managed Identity)
WHAT  = Role Definition (Owner, Contributor, Reader, Custom)
WHERE = Scope (Management Group, Subscription, Resource Group, Resource)
```

**Example:**
```
WHO:   Alice (alice@mycompany.com)
WHAT:  Contributor
WHERE: Resource Group "rg-production"

= Alice can create, update, and delete any resource
  in rg-production, but cannot manage access (no role assignments)
```

---

## 6. Azure RBAC — Role-Based Access Control for Resources

### What is Azure RBAC?

**Azure RBAC** is the authorization system that controls access to **Azure resources** (VMs, Storage, AKS, Key Vault, etc.). It answers: "Can this identity perform this action on this resource?"

### How Azure RBAC Works

```
┌──────────────────────────────────────────────────────────────────┐
│                    Role Assignment                               │
│                                                                  │
│   Security Principal ──── Role Definition ──── Scope            │
│         (WHO)                  (WHAT)           (WHERE)          │
│                                                                  │
│   Example:                                                       │
│   Alice         ────  Contributor      ──── rg-production        │
│   DevOps-Group  ────  AKS Admin        ──── aks-cluster-prod     │
│   CI/CD-SPN     ────  Contributor      ──── Subscription-Prod    │
│   AKS-MI        ────  Key Vault Reader ──── kv-production        │
└──────────────────────────────────────────────────────────────────┘
```

### Role Definition Structure

A role is defined by:

```json
{
  "Name": "Virtual Machine Contributor",
  "Description": "Lets you manage virtual machines, but not access to them.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/*",
    "Microsoft.Network/networkInterfaces/*",
    "Microsoft.Storage/storageAccounts/read"
  ],
  "NotActions": [
    "Microsoft.Authorization/*/write"
  ],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": ["/"]
}
```

**Actions:** Operations allowed on the control plane (managing resources)
- `Microsoft.Compute/virtualMachines/read` — View VMs
- `Microsoft.Compute/virtualMachines/write` — Create/update VMs
- `Microsoft.Compute/virtualMachines/delete` — Delete VMs
- `Microsoft.Compute/virtualMachines/*` — All VM operations

**NotActions:** Operations explicitly denied from the Actions list

**DataActions:** Operations on the data plane (accessing data inside resources)
- `Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read` — Read blobs

**NotDataActions:** Data operations explicitly denied

### Control Plane vs Data Plane

```
┌─────────────────────────────────────────────────────────────┐
│                  Control Plane                               │
│  "Managing the resource itself"                              │
│                                                              │
│  Examples:                                                   │
│  • Create/delete a Storage Account                          │
│  • Create/delete an AKS Cluster                             │
│  • Change VM size                                           │
│  • View resource tags                                       │
│                                                              │
│  Controlled by: Actions / NotActions                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Data Plane                                │
│  "Accessing the data inside the resource"                   │
│                                                              │
│  Examples:                                                   │
│  • Read/write blobs in a Storage Account                    │
│  • Read/write secrets in Key Vault                          │
│  • Query data in a SQL Database                             │
│  • Pull images from Container Registry                      │
│                                                              │
│  Controlled by: DataActions / NotDataActions                │
└─────────────────────────────────────────────────────────────┘
```

**Important:** A **Contributor** can manage a Storage Account (create, delete, configure) but CANNOT read blobs inside it. To read blobs, you need **Storage Blob Data Reader** (a data plane role).

---

## 7. Scopes — Where Roles Apply

### The 4 Scope Levels

Roles can be assigned at 4 levels. Roles **inherit downward** — a role assigned at a higher scope applies to all resources below it.

```
┌──────────────────────────────────────────────────────────────────┐
│  Management Group (Broadest)                                     │
│  /providers/Microsoft.Management/managementGroups/{mg-id}        │
│  Example: "All Production Subscriptions"                         │
│                                                                  │
│    ┌──────────────────────────────────────────────────────────┐  │
│    │  Subscription                                           │  │
│    │  /subscriptions/{sub-id}                                │  │
│    │  Example: "Production Subscription"                     │  │
│    │                                                         │  │
│    │    ┌────────────────────────────────────────────────┐   │  │
│    │    │  Resource Group                                │   │  │
│    │    │  /subscriptions/{sub-id}/resourceGroups/{rg}   │   │  │
│    │    │  Example: "rg-production"                      │   │  │
│    │    │                                                │   │  │
│    │    │    ┌──────────────────────────────────────┐    │   │  │
│    │    │    │  Resource (Narrowest)                │    │   │  │
│    │    │    │  Example: "my-aks-cluster"           │    │   │  │
│    │    │    └──────────────────────────────────────┘    │   │  │
│    │    └────────────────────────────────────────────────┘   │  │
│    └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Inheritance Example

```
Alice is assigned "Reader" at Subscription level
         │
         ├──► Can read ALL resource groups in the subscription
         ├──► Can read ALL resources in ALL resource groups
         └──► Cannot create, update, or delete anything

Bob is assigned "Contributor" at Resource Group "rg-dev"
         │
         ├──► Can create/update/delete resources in rg-dev ONLY
         ├──► Cannot access rg-production
         └──► Cannot manage access (no role assignment permissions)

Carol is assigned "Owner" at a specific AKS Cluster
         │
         ├──► Full control over ONLY that AKS cluster
         ├──► Can grant others access to that cluster
         └──► Cannot access any other resource
```

### Best Practice: Least Privilege

Always assign roles at the **narrowest scope** possible:

| ❌ Too Broad | ✅ Just Right |
|-------------|-------------|
| Owner at Subscription | Contributor at Resource Group |
| Contributor at Subscription | AKS Cluster Admin at AKS Resource |
| Reader at Management Group | Reader at Resource Group |

---

## 8. The 4 Fundamental Built-in Roles

### Overview

Azure has **hundreds** of built-in roles, but only 4 are fundamental. Every other role is a subset of these:

```
┌─────────────────────────────────────────────────────────────┐
│  OWNER                                                      │
│  "Full access + can manage access for others"               │
│  Actions: *                                                 │
│  Can assign roles: YES                                      │
│                                                              │
│  Use: Subscription administrators, resource owners          │
├─────────────────────────────────────────────────────────────┤
│  CONTRIBUTOR                                                │
│  "Full access but CANNOT manage access for others"          │
│  Actions: * (except Microsoft.Authorization/*)              │
│  Can assign roles: NO                                       │
│                                                              │
│  Use: Developers, DevOps engineers                          │
├─────────────────────────────────────────────────────────────┤
│  READER                                                     │
│  "View everything but change nothing"                       │
│  Actions: */read                                            │
│  Can assign roles: NO                                       │
│                                                              │
│  Use: Auditors, monitoring tools, support staff             │
├─────────────────────────────────────────────────────────────┤
│  USER ACCESS ADMINISTRATOR                                  │
│  "Can ONLY manage access (role assignments)"                │
│  Actions: Microsoft.Authorization/*                         │
│  Can assign roles: YES                                      │
│                                                              │
│  Use: IAM administrators who only manage permissions        │
└─────────────────────────────────────────────────────────────┘
```

### Comparison Table

| Permission | Owner | Contributor | Reader | User Access Admin |
|-----------|-------|------------|--------|------------------|
| Read resources | ✅ | ✅ | ✅ | ❌ |
| Create resources | ✅ | ✅ | ❌ | ❌ |
| Delete resources | ✅ | ✅ | ❌ | ❌ |
| Assign roles to others | ✅ | ❌ | ❌ | ✅ |
| Create custom roles | ✅ | ❌ | ❌ | ✅ |

### CLI Examples

```bash
# Assign Reader to a user at subscription scope
az role assignment create \
  --role "Reader" \
  --assignee alice@mycompany.com \
  --scope "/subscriptions/<sub-id>"

# Assign Contributor to a group at resource group scope
az role assignment create \
  --role "Contributor" \
  --assignee-object-id <group-object-id> \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-dev"

# Assign Owner to a service principal at resource scope
az role assignment create \
  --role "Owner" \
  --assignee <spn-app-id> \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-prod/providers/Microsoft.ContainerService/managedClusters/my-aks"

# List all role assignments for a user
az role assignment list --assignee alice@mycompany.com --output table
```

---

## 9. Service-Specific Built-in Roles

### AKS-Specific Roles

| Role | What It Grants |
|------|---------------|
| **Azure Kubernetes Service Cluster Admin Role** | Full admin access to AKS cluster (get admin kubeconfig) |
| **Azure Kubernetes Service Cluster User Role** | Get user kubeconfig (limited by Kubernetes RBAC) |
| **Azure Kubernetes Service Contributor Role** | Manage AKS clusters (create, delete, scale) |
| **Azure Kubernetes Service RBAC Admin** | Manage Kubernetes RBAC within the cluster |
| **Azure Kubernetes Service RBAC Cluster Admin** | Full Kubernetes cluster admin (like cluster-admin) |
| **Azure Kubernetes Service RBAC Reader** | Read-only access to Kubernetes resources |
| **Azure Kubernetes Service RBAC Writer** | Read/write access to Kubernetes resources |

### Storage-Specific Roles

| Role | Plane | What It Grants |
|------|-------|---------------|
| **Storage Account Contributor** | Control | Manage storage accounts (create, delete, configure) |
| **Storage Blob Data Contributor** | Data | Read/write/delete blobs and containers |
| **Storage Blob Data Reader** | Data | Read blobs and containers |
| **Storage Blob Data Owner** | Data | Full access to blobs + manage access |
| **Storage Queue Data Contributor** | Data | Read/write/delete queue messages |

### Key Vault-Specific Roles

| Role | What It Grants |
|------|---------------|
| **Key Vault Administrator** | Full access to all Key Vault data operations |
| **Key Vault Secrets Officer** | Read/write/delete secrets |
| **Key Vault Secrets User** | Read secret values |
| **Key Vault Certificates Officer** | Manage certificates |
| **Key Vault Crypto Officer** | Manage keys |
| **Key Vault Reader** | View Key Vault properties (not secret values) |

### Networking Roles

| Role | What It Grants |
|------|---------------|
| **Network Contributor** | Manage all networking resources |
| **DNS Zone Contributor** | Manage DNS zones and records |
| **Private DNS Zone Contributor** | Manage private DNS zones |

---

## 10. Entra ID Roles vs Azure RBAC Roles

### The Two Separate RBAC Systems

Azure has **two completely separate** role systems that people often confuse:

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   ┌───────────────────────┐    ┌───────────────────────────┐    │
│   │  Microsoft Entra ID   │    │  Azure RBAC               │    │
│   │  Roles                │    │  Roles                     │    │
│   │                       │    │                             │    │
│   │  WHAT: Manage Entra   │    │  WHAT: Manage Azure        │    │
│   │  ID objects (users,   │    │  resources (VMs, storage,  │    │
│   │  groups, apps, etc.)  │    │  AKS, networks, etc.)      │    │
│   │                       │    │                             │    │
│   │  WHERE: Azure Portal  │    │  WHERE: Azure Portal       │    │
│   │  → Entra ID → Roles  │    │  → Any resource →          │    │
│   │  & Administrators     │    │    Access Control (IAM)    │    │
│   │                       │    │                             │    │
│   │  Examples:            │    │  Examples:                  │    │
│   │  • Global Admin       │    │  • Owner                   │    │
│   │  • User Admin         │    │  • Contributor             │    │
│   │  • App Admin          │    │  • Reader                  │    │
│   │  • Groups Admin       │    │  • AKS Admin               │    │
│   └───────────────────────┘    └───────────────────────────┘    │
│                                                                  │
│   These two systems are SEPARATE.                               │
│   An Entra ID Global Admin does NOT automatically                │
│   have Owner access to Azure subscriptions.                      │
│   (They can elevate, but it's not automatic.)                   │
└──────────────────────────────────────────────────────────────────┘
```

### Key Entra ID Roles

| Entra ID Role | What It Controls |
|--------------|-----------------|
| **Global Administrator** | Everything in Entra ID (create users, manage apps, reset passwords, manage all roles) |
| **User Administrator** | Create/delete/modify users and groups |
| **Application Administrator** | Manage all app registrations and enterprise apps |
| **Cloud Application Administrator** | Same as App Admin but cannot manage app proxy |
| **Privileged Role Administrator** | Manage role assignments in Entra ID |
| **Security Administrator** | Read security info + manage security policies |
| **Groups Administrator** | Create and manage groups |
| **Billing Administrator** | Manage billing and subscriptions |

### When to Use Which?

| Task | Which RBAC? |
|------|------------|
| Create a VM in Azure | Azure RBAC (Contributor role) |
| Create a user in Entra ID | Entra ID RBAC (User Administrator) |
| Assign Azure roles to others | Azure RBAC (Owner or User Access Admin) |
| Register an application | Entra ID RBAC (Application Administrator) |
| Read blobs in Storage Account | Azure RBAC (Storage Blob Data Reader) |
| Reset a user's password | Entra ID RBAC (Authentication Administrator) |

---

## 11. Service Principals and App Registrations

### The Relationship

When you register an application in Entra ID, **two objects** are created:

```
App Registration                     Service Principal
(Application Object)                 (Enterprise Application)
┌────────────────────┐              ┌────────────────────┐
│ Global template    │              │ Local instance     │
│                    │              │ in your tenant     │
│ App ID (Client ID) │──creates──►│ Object ID          │
│ Redirect URIs      │              │ Role assignments   │
│ API Permissions     │              │ Consent status     │
│ Certificates/Secrets│              │ Sign-in logs       │
│                    │              │                    │
│ Lives in: Home     │              │ Lives in: Every    │
│ tenant only        │              │ tenant that uses   │
│                    │              │ the app            │
└────────────────────┘              └────────────────────┘
```

**Analogy:**
- **App Registration** = A passport application form (the template)
- **Service Principal** = The actual passport issued (the identity)

### Creating a Service Principal

```bash
# Method 1: Create with contributor role on a resource group
az ad sp create-for-rbac \
  --name "terraform-sp" \
  --role "Contributor" \
  --scopes "/subscriptions/<sub-id>/resourceGroups/rg-infra"

# Output:
# {
#   "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#   "displayName": "terraform-sp",
#   "password": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
#   "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# }

# Method 2: Create without role assignment
az ad sp create-for-rbac --name "my-app" --skip-assignment
```

### Using a Service Principal

```bash
# Login as the service principal
az login --service-principal \
  --username <appId> \
  --password <password> \
  --tenant <tenantId>

# Now all az commands run as this SPN
az group list  # Only sees resources the SPN has access to
```

---

## 12. Managed Identities — System vs User Assigned

### System-Assigned Managed Identity

**Created automatically** when you enable it on an Azure resource. Tied to that resource's lifecycle.

```bash
# Enable system-assigned identity on an AKS cluster
az aks update \
  --name my-aks \
  --resource-group my-rg \
  --enable-managed-identity

# The AKS cluster now has a system-assigned identity
# Get its principal ID
az aks show --name my-aks --resource-group my-rg \
  --query "identity.principalId" -o tsv
```

```
┌────────────────────┐
│   AKS Cluster      │
│   "my-aks"         │
│                    │
│   System Identity: │
│   Principal ID:    │
│   abc123-def4...   │
│                    │
│   If cluster is    │
│   deleted, this    │
│   identity is      │
│   also deleted     │
└────────────────────┘
```

### User-Assigned Managed Identity

**Created separately** as its own Azure resource. Can be shared across multiple resources.

```bash
# Create a user-assigned identity
az identity create \
  --name aks-kv-identity \
  --resource-group my-rg

# Assign it to an AKS cluster
az aks update \
  --name my-aks \
  --resource-group my-rg \
  --assign-identity /subscriptions/<sub-id>/resourceGroups/my-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aks-kv-identity
```

```
┌────────────────────┐     ┌────────────────────┐
│   AKS Cluster 1    │     │   AKS Cluster 2    │
│   "aks-prod"       │     │   "aks-staging"    │
└──────────┬─────────┘     └──────────┬─────────┘
           │                          │
           └──────────┬───────────────┘
                      │ Both use
                      ▼
           ┌────────────────────┐
           │   User-Assigned    │
           │   Managed Identity │
           │   "aks-kv-identity"│
           │                    │
           │   Independent      │
           │   lifecycle        │
           └────────────────────┘
```

### Comparison

| Feature | System-Assigned | User-Assigned |
|---------|----------------|---------------|
| **Creation** | Automatic (enable on resource) | Manual (separate resource) |
| **Lifecycle** | Tied to resource | Independent |
| **Shared** | ❌ One resource only | ✅ Multiple resources |
| **Deletion** | Deleted with resource | Must delete separately |
| **Use case** | Single resource needing access | Shared identity pattern |

---

---

## 13. Kubernetes Service Accounts

### What is a Kubernetes Service Account?

A **Kubernetes Service Account** is an identity that is assigned to a **pod** (not a human user). It allows pods to authenticate with the Kubernetes API server and access cluster resources.

Every namespace in Kubernetes has a **default** service account. If you don't specify one, your pod uses the default service account of its namespace.

```
┌──────────────────────────────────────────────────────────────────┐
│  Kubernetes Identity Types                                       │
│                                                                  │
│  ┌──────────────────────┐    ┌──────────────────────────────┐   │
│  │  Human Users          │    │  Service Accounts             │   │
│  │  (not stored in K8s)  │    │  (Kubernetes-native objects)  │   │
│  │                       │    │                               │   │
│  │  • Authenticated by   │    │  • Created with kubectl       │   │
│  │    Entra ID / OIDC    │    │  • Mounted as tokens in pods  │   │
│  │  • Use kubeconfig     │    │  • Used by pods, controllers  │   │
│  │  • kubectl users      │    │  • Scoped to a namespace      │   │
│  └──────────────────────┘    └──────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### How Service Accounts Work

When a pod starts, Kubernetes automatically:
1. Mounts a **token** at `/var/run/secrets/kubernetes.io/serviceaccount/token`
2. Mounts the **CA certificate** at `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`
3. Mounts the **namespace** at `/var/run/secrets/kubernetes.io/serviceaccount/namespace`

The pod uses this token to authenticate with the Kubernetes API server.

### Creating and Using a Service Account

```yaml
# Step 1: Create a Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  annotations:
    # Used by Workload Identity (see section 15)
    azure.workload.identity/client-id: "<managed-identity-client-id>"
---
# Step 2: Use it in a Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  serviceAccountName: my-app-sa    # ← Assign service account to pod
  containers:
  - name: app
    image: myapp:latest
```

### CLI Commands

```bash
# List service accounts in a namespace
kubectl get serviceaccounts -n production

# Create a service account
kubectl create serviceaccount my-app-sa -n production

# Describe a service account
kubectl describe serviceaccount my-app-sa -n production

# Check which service account a pod uses
kubectl get pod my-app -n production -o jsonpath='{.spec.serviceAccountName}'
```

### The Default Service Account

Every namespace has a `default` service account:

```bash
kubectl get serviceaccount default -n default
# NAME      SECRETS   AGE
# default   0         30d
```

**⚠️ Best Practice:** Don't use the `default` service account for application pods. Create dedicated service accounts with only the permissions needed (least privilege).

### Service Account Tokens (Projected Volumes)

Since Kubernetes 1.22+, AKS uses **bound service account tokens** (projected volumes). These tokens are:
- **Time-limited** (expire after 1 hour by default, auto-refreshed)
- **Audience-bound** (only valid for a specific API server)
- **Object-bound** (tied to the specific pod)

This is much more secure than the old non-expiring tokens.

---

## 14. Kubernetes RBAC — Roles, ClusterRoles, and Bindings

### What is Kubernetes RBAC?

**Kubernetes RBAC** is the native authorization system inside a Kubernetes cluster. It controls what users and service accounts can do **within the cluster** (create pods, read secrets, delete deployments, etc.).

This is **separate from Azure RBAC** — Azure RBAC controls access to Azure resources, Kubernetes RBAC controls access to Kubernetes resources.

### The 4 Kubernetes RBAC Objects

```
┌─────────────────────────────────────────────────────────────────┐
│                 Kubernetes RBAC Objects                          │
│                                                                  │
│  WHAT permissions?              WHO gets them?                  │
│                                                                  │
│  ┌────────────────┐            ┌─────────────────────┐         │
│  │     Role       │────────►  │   RoleBinding        │         │
│  │  (namespace)   │            │   (namespace)        │         │
│  │                │            │   Binds Role to      │         │
│  │  "Can read     │            │   User/Group/SA      │         │
│  │   pods in      │            │   in THIS namespace  │         │
│  │   namespace    │            └─────────────────────┘         │
│  │   dev"         │                                             │
│  └────────────────┘                                             │
│                                                                  │
│  ┌────────────────┐            ┌─────────────────────┐         │
│  │  ClusterRole   │────────►  │ ClusterRoleBinding   │         │
│  │  (cluster-wide)│            │ (cluster-wide)       │         │
│  │                │            │ Binds ClusterRole    │         │
│  │  "Can read     │            │ to User/Group/SA     │         │
│  │   pods in      │            │ in ALL namespaces    │         │
│  │   ALL          │            └─────────────────────┘         │
│  │   namespaces"  │                                             │
│  └────────────────┘                                             │
└─────────────────────────────────────────────────────────────────┘
```

### Role (Namespace-Scoped Permissions)

A **Role** defines permissions within a **single namespace**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production          # Only applies in this namespace
rules:
- apiGroups: [""]                # "" = core API group (pods, services, etc.)
  resources: ["pods"]            # Resource type
  verbs: ["get", "list", "watch"]  # Actions allowed
- apiGroups: [""]
  resources: ["pods/log"]        # Can also read pod logs
  verbs: ["get"]
```

### ClusterRole (Cluster-Wide Permissions)

A **ClusterRole** defines permissions across **all namespaces** or for cluster-scoped resources (nodes, namespaces, PVs):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-admin
rules:
- apiGroups: [""]
  resources: ["pods", "services", "deployments", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
```

### Common Verbs

| Verb | What it allows |
|------|---------------|
| `get` | Read a single resource by name |
| `list` | List all resources of a type |
| `watch` | Stream real-time changes |
| `create` | Create new resources |
| `update` | Modify existing resources |
| `patch` | Partially modify resources |
| `delete` | Delete resources |
| `*` | All verbs (full access) |

### Common API Groups

| API Group | Resources |
|-----------|-----------|
| `""` (core) | pods, services, configmaps, secrets, namespaces, nodes, PVs, PVCs |
| `apps` | deployments, replicasets, statefulsets, daemonsets |
| `batch` | jobs, cronjobs |
| `networking.k8s.io` | networkpolicies, ingresses |
| `rbac.authorization.k8s.io` | roles, clusterroles, rolebindings, clusterrolebindings |

### RoleBinding (Namespace-Scoped Assignment)

A **RoleBinding** grants a Role's permissions to a user, group, or service account **within a specific namespace**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-pod-reader
  namespace: production
subjects:
- kind: User                     # Can be: User, Group, ServiceAccount
  name: developer1@contoso.com   # Entra ID user (with Entra integration)
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: "aks-developers-group-id"  # Entra ID group Object ID
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-app-sa
  namespace: production
roleRef:
  kind: Role                     # Can reference: Role or ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding (Cluster-Wide Assignment)

A **ClusterRoleBinding** grants a ClusterRole's permissions **across all namespaces**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
- kind: Group
  name: "aks-admins-group-object-id"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin              # Built-in: full cluster access
  apiGroup: rbac.authorization.k8s.io
```

### Built-in ClusterRoles

Kubernetes comes with built-in ClusterRoles:

| ClusterRole | Permissions |
|-------------|-------------|
| **cluster-admin** | Full access to everything in the cluster |
| **admin** | Full access within a namespace (when bound via RoleBinding) |
| **edit** | Read/write most resources in a namespace (no RBAC management) |
| **view** | Read-only access to most resources in a namespace (no secrets) |

### Practical Example: Developer Access to One Namespace

```yaml
---
# 1. Create a Role with read/write on most resources
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-full-access
  namespace: dev
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]
---
# 2. Bind it to an Entra ID group
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: dev
subjects:
- kind: Group
  name: "00000000-aaaa-bbbb-cccc-111111111111"  # Entra ID group Object ID
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-full-access
  apiGroup: rbac.authorization.k8s.io
```

**Result:** Members of the Entra ID group can create/edit/delete any resource in the `dev` namespace, but cannot access other namespaces.

### CLI Commands for Kubernetes RBAC

```bash
# List all roles in a namespace
kubectl get roles -n production

# List all cluster roles
kubectl get clusterroles

# List all role bindings
kubectl get rolebindings -n production

# Check what a user can do (impersonation)
kubectl auth can-i create pods --namespace production --as developer1@contoso.com

# Check what a service account can do
kubectl auth can-i list secrets --namespace production --as system:serviceaccount:production:my-app-sa

# Check all permissions for a user
kubectl auth can-i --list --as developer1@contoso.com
```

---

## 15. AKS Workload Identity — Pods Accessing Azure Resources

### The Problem

Your application pod running in AKS needs to access an Azure resource (Key Vault, Storage, SQL Database). How does the pod authenticate?

**Bad approach (hardcoded secrets):**
```yaml
env:
- name: AZURE_CLIENT_SECRET
  value: "MySecretPassword123!"   # ❌ NEVER DO THIS
```

**Better approach (Kubernetes Secret):**
```yaml
env:
- name: AZURE_CLIENT_SECRET
  valueFrom:
    secretKeyRef:
      name: azure-creds
      key: client-secret           # ⚠️ Still a secret to manage/rotate
```

**Best approach (Workload Identity):**
```yaml
serviceAccountName: my-app-sa      # ✅ No secrets at all!
# Pod automatically gets an Entra ID token via federation
```

### What is Workload Identity?

**Microsoft Entra Workload ID** allows Kubernetes service accounts to **federate** with Entra ID managed identities. The pod gets an Entra ID token automatically — no secrets needed.

### How It Works — Step by Step

```
1. AKS cluster has an OIDC issuer URL
   (https://oidc.prod-aks.azure.com/xxxxx)
         │
         ▼
2. You create a Managed Identity in Azure
   and a Federated Identity Credential that trusts
   the AKS OIDC issuer + specific service account
         │
         ▼
3. You create a Kubernetes ServiceAccount
   annotated with the Managed Identity's client ID
         │
         ▼
4. Pod starts with that ServiceAccount
   Kubernetes projects a JWT token into the pod
         │
         ▼
5. Azure Identity SDK in the pod exchanges
   the K8s JWT token for an Entra ID access token
   (via the OIDC federation trust)
         │
         ▼
6. Pod uses the Entra ID token to call
   Azure services (Key Vault, Storage, etc.)
```

### Complete Setup — End to End

**Step 1: Create AKS cluster with OIDC and Workload Identity enabled**

```bash
az aks create \
  --name my-aks \
  --resource-group my-rg \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --generate-ssh-keys

# Get the OIDC issuer URL
AKS_OIDC_ISSUER=$(az aks show --name my-aks --resource-group my-rg \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)
echo $AKS_OIDC_ISSUER
# https://eastus.oic.prod-aks.azure.com/xxxxx/
```

**Step 2: Create a User-Assigned Managed Identity**

```bash
az identity create \
  --name workload-identity-keyvault \
  --resource-group my-rg

# Get client ID
IDENTITY_CLIENT_ID=$(az identity show --name workload-identity-keyvault \
  --resource-group my-rg --query clientId -o tsv)
```

**Step 3: Create Federated Identity Credential**

```bash
az identity federated-credential create \
  --name fed-credential-aks \
  --identity-name workload-identity-keyvault \
  --resource-group my-rg \
  --issuer "$AKS_OIDC_ISSUER" \
  --subject "system:serviceaccount:production:my-app-sa" \
  --audiences "api://AzureADTokenExchange"
```

This tells Entra ID: "Trust tokens from this AKS cluster's OIDC issuer for the service account `my-app-sa` in namespace `production`."

**Step 4: Grant the Managed Identity access to Azure resources**

```bash
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee "$IDENTITY_CLIENT_ID" \
  --scope "/subscriptions/xxx/resourceGroups/my-rg/providers/Microsoft.KeyVault/vaults/my-keyvault"
```

**Step 5: Create Kubernetes ServiceAccount with annotation**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  annotations:
    azure.workload.identity/client-id: "<IDENTITY_CLIENT_ID>"
  labels:
    azure.workload.identity/use: "true"
```

**Step 6: Deploy pod using the ServiceAccount**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
  labels:
    azure.workload.identity/use: "true"  # Required label
spec:
  serviceAccountName: my-app-sa          # Uses the annotated SA
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: AZURE_CLIENT_ID
      value: "<IDENTITY_CLIENT_ID>"
    - name: KEYVAULT_URL
      value: "https://my-keyvault.vault.azure.net/"
```

The pod can now access Key Vault secrets without any passwords or secrets!

### Workload Identity vs Pod-Managed Identity (Deprecated)

| Feature | Workload Identity (Recommended) | Pod-Managed Identity (Deprecated) |
|---------|-------------------------------|----------------------------------|
| **Status** | ✅ GA, actively maintained | ❌ Deprecated Oct 2022 |
| **Mechanism** | OIDC federation | Azure AD pod identity webhook |
| **Secrets** | None needed | None needed |
| **Performance** | Fast (no NMI pod) | Slower (NMI pod overhead) |
| **Cross-cluster** | ✅ Supported | ❌ Not supported |

---

## 16. Azure RBAC for Kubernetes Authorization

### Two Ways to Authorize Inside AKS

AKS supports **two authorization models** for the Kubernetes API:

```
┌─────────────────────────────────────────────────────────────────┐
│  Option 1: Kubernetes RBAC (Traditional)                       │
│                                                                 │
│  • Permissions stored as K8s objects (Role, ClusterRole)       │
│  • Managed with kubectl                                        │
│  • Per-cluster configuration                                   │
│  • Need to configure RBAC in EVERY cluster separately          │
│                                                                 │
│  kubectl create → Role/RoleBinding → User gets access          │
├─────────────────────────────────────────────────────────────────┤
│  Option 2: Azure RBAC for Kubernetes (Modern)                  │
│                                                                 │
│  • Permissions stored as Azure role assignments                │
│  • Managed with Azure Portal / CLI                             │
│  • Can be scoped to subscription/RG (multiple clusters at once)│
│  • Centralized management via Entra ID                         │
│                                                                 │
│  az role assignment create → Azure role → User gets access     │
└─────────────────────────────────────────────────────────────────┘
```

### Enabling Azure RBAC for Kubernetes

```bash
az aks create \
  --name my-aks \
  --resource-group my-rg \
  --enable-aad \
  --enable-azure-rbac \      # ← This enables Azure RBAC for K8s API
  --generate-ssh-keys
```

### Azure Built-in Roles for Kubernetes Data Plane

When Azure RBAC for Kubernetes is enabled, you use **Azure roles** instead of K8s Role/RoleBinding:

| Azure Role | K8s Equivalent | What It Grants |
|-----------|----------------|---------------|
| **Azure Kubernetes Service RBAC Reader** | `view` | Read most K8s resources (no secrets) |
| **Azure Kubernetes Service RBAC Writer** | `edit` | Read/write K8s resources (no RBAC) |
| **Azure Kubernetes Service RBAC Admin** | `admin` | Full access within namespaces + RBAC |
| **Azure Kubernetes Service RBAC Cluster Admin** | `cluster-admin` | Full cluster access |

### Example: Grant Namespace Access via Azure RBAC

```bash
# Get AKS resource ID
AKS_ID=$(az aks show --name my-aks --resource-group my-rg --query id -o tsv)

# Grant "Writer" access to production namespace only
az role assignment create \
  --role "Azure Kubernetes Service RBAC Writer" \
  --assignee developer1@contoso.com \
  --scope "$AKS_ID/namespaces/production"
```

### Kubernetes RBAC vs Azure RBAC — When to Use Which?

| Aspect | Kubernetes RBAC | Azure RBAC for K8s |
|--------|----------------|-------------------|
| **Where configured** | Inside cluster (kubectl) | Azure Portal/CLI |
| **Scope** | Single cluster | Can span multiple clusters |
| **Central management** | ❌ Per cluster | ✅ Via Entra ID |
| **Audit** | K8s audit logs | Azure Activity Log |
| **PIM support** | ❌ No | ✅ Just-in-time elevation |
| **Conditional Access** | ❌ No | ✅ MFA, device compliance |
| **Best for** | Fine-grained K8s-specific rules | Enterprise-scale management |

**Microsoft recommendation:** Use **Azure RBAC for Kubernetes** in enterprise environments for centralized management. Use **Kubernetes RBAC** when you need fine-grained K8s-specific rules or are in a multi-cloud environment.

---

## 17. Custom Roles

### When to Create Custom Roles

Built-in roles might be too broad. Example: You want a role that can restart VMs but NOT delete them.

### Custom Role Definition

```json
{
  "Name": "VM Operator",
  "Description": "Can start, stop, and restart VMs but not delete them.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/powerOff/action"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/<your-subscription-id>"
  ]
}
```

### Creating a Custom Role

```bash
az role definition create --role-definition custom-role.json
```

### Assigning the Custom Role

```bash
az role assignment create \
  --role "VM Operator" \
  --assignee alice@mycompany.com \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-vms"
```

---

## 18. Conditional Access Policies

### What is Conditional Access?

**Conditional Access** is Entra ID's policy engine that adds conditions to authentication decisions. Instead of just "valid password = access granted", you can add rules like:

```
IF user is signing in from an unknown location
  AND is accessing a sensitive app
  AND is NOT using a compliant device
THEN
  REQUIRE MFA + block if MFA fails
```

### Common Policies

| Policy | Condition | Action |
|--------|-----------|--------|
| **Require MFA for admins** | User has admin role | Require MFA |
| **Block risky sign-ins** | Sign-in risk = High | Block access |
| **Require compliant device** | Device not compliant | Block access |
| **Restrict by location** | Not in corporate network | Require MFA |
| **Block legacy auth** | Using legacy protocols | Block access |
| **Session controls** | Accessing sensitive apps | Limit session to 1 hour |

### Conditional Access Flow

```
User attempts to sign in
         │
         ▼
Entra ID authenticates (password, MFA)
         │
         ▼
Conditional Access engine evaluates ALL policies
         │
    ┌────┴─────────────────┐
    │ Policy 1: Admin MFA  │
    │ Match? Is user admin?│
    │ → YES → Require MFA  │
    └────┬─────────────────┘
         │
    ┌────┴──────────────────┐
    │ Policy 2: Location    │
    │ Match? Unknown IP?    │
    │ → YES → Require MFA  │
    └────┬──────────────────┘
         │
    ┌────┴──────────────────┐
    │ Policy 3: Device      │
    │ Match? Non-compliant? │
    │ → YES → Block access  │
    └────┬──────────────────┘
         │
         ▼
    All policies evaluated
    Most restrictive result applies
         │
         ▼
    Access Granted / Denied / MFA Required
```

**Requires:** Entra ID Premium P1 or P2 license.

---

## 19. Privileged Identity Management (PIM)

### What is PIM?

**Privileged Identity Management (PIM)** is a feature that provides **time-limited, just-in-time** access to elevated roles. Instead of permanently assigning the Owner role, you make a user **eligible** — they must activate the role when needed, and it expires after a set time.

### Why PIM?

**Without PIM:**
```
Alice has Owner role permanently
  → If Alice's account is compromised, attacker has Owner forever
  → Alice might accidentally delete resources with her elevated access
  → No audit trail of when Alice used her elevated permissions
```

**With PIM:**
```
Alice is ELIGIBLE for Owner role
  → Alice activates Owner when needed (with justification)
  → Owner access expires after 4 hours
  → Full audit trail of activation
  → Approval can be required before activation
  → Alert sent when elevated role is activated
```

### PIM Workflow

```
1. Admin makes Alice ELIGIBLE for "Contributor" on rg-production
         │
         ▼
2. Alice needs to deploy a new service
         │
         ▼
3. Alice goes to PIM portal and clicks "Activate"
   - Provides justification: "Deploying v2.5 of payment service"
   - Duration: 4 hours
         │
         ▼
4. (Optional) Manager approves the request
         │
         ▼
5. Alice's Contributor role is ACTIVE for 4 hours
         │
         ▼
6. Alice deploys the service
         │
         ▼
7. After 4 hours, the role automatically deactivates
         │
         ▼
8. Full audit trail recorded:
   - Who activated
   - When
   - Why (justification)
   - What they did during activation
```

**Requires:** Entra ID Premium P2 license.

---

## 20. Practical Examples and Best Practices

### Example 1: AKS Team Access Setup

```bash
# Create groups
az ad group create --display-name "AKS-Admins" --mail-nickname "aks-admins"
az ad group create --display-name "AKS-Developers" --mail-nickname "aks-devs"
az ad group create --display-name "AKS-Viewers" --mail-nickname "aks-viewers"

# Assign roles
# Admins: Full control over AKS cluster
az role assignment create \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --assignee-object-id <aks-admins-group-id> \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-prod/providers/Microsoft.ContainerService/managedClusters/aks-prod"

# Developers: Read/write Kubernetes resources
az role assignment create \
  --role "Azure Kubernetes Service RBAC Writer" \
  --assignee-object-id <aks-devs-group-id> \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-prod/providers/Microsoft.ContainerService/managedClusters/aks-prod"

# Viewers: Read-only
az role assignment create \
  --role "Azure Kubernetes Service RBAC Reader" \
  --assignee-object-id <aks-viewers-group-id> \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-prod/providers/Microsoft.ContainerService/managedClusters/aks-prod"
```

### Example 2: Managed Identity Accessing Key Vault

```bash
# Create managed identity
az identity create --name aks-keyvault-identity --resource-group rg-prod

# Get identity's principal ID
PRINCIPAL_ID=$(az identity show --name aks-keyvault-identity \
  --resource-group rg-prod --query principalId -o tsv)

# Grant Key Vault Secrets User role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee-object-id $PRINCIPAL_ID \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-prod/providers/Microsoft.KeyVault/vaults/kv-prod"
```

### Example 3: CI/CD Pipeline Service Principal

```bash
# Create SPN with Contributor on production RG
az ad sp create-for-rbac \
  --name "github-actions-prod" \
  --role "Contributor" \
  --scopes "/subscriptions/<sub-id>/resourceGroups/rg-prod"

# Also grant ACR Pull permission
az role assignment create \
  --role "AcrPull" \
  --assignee <spn-app-id> \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-prod/providers/Microsoft.ContainerRegistry/registries/myacr"
```

### Best Practices

✅ **Use Managed Identities** over Service Principals whenever possible

✅ **Least Privilege** — Assign the narrowest role at the narrowest scope

✅ **Use Groups** — Assign roles to groups, not individual users

✅ **Use PIM** — Make elevated roles eligible, not permanent

✅ **Enable MFA** — For all users, especially admins

✅ **Conditional Access** — Block risky sign-ins, require compliant devices

✅ **Audit regularly** — Review role assignments quarterly

✅ **Avoid Owner at subscription level** — Use Contributor + separate User Access Admin

✅ **Separate control plane and data plane roles** — Contributor ≠ can read your blobs

✅ **Use custom roles** — When built-in roles are too broad

✅ **Never store SPN secrets in code** — Use Key Vault or Managed Identity

✅ **Rotate SPN secrets** — Every 90 days or use certificates

---

## References & Further Reading

- [Microsoft Entra ID Documentation](https://learn.microsoft.com/en-us/entra/identity/)
- [Azure RBAC Overview](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- [Azure Built-in Roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Entra ID Built-in Roles](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference)
- [Azure Roles vs Entra ID Roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/rbac-and-directory-admin-roles)
- [Managed Identities Overview](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)
- [App Registrations and Service Principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)
- [Conditional Access Overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)
- [Privileged Identity Management](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
- [AKS Access and Identity Concepts (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/concepts-identity)
- [Kubernetes RBAC with Entra ID in AKS (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/azure-ad-rbac)
- [AKS Workload Identity (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster)
- [Azure RBAC for Kubernetes Authorization (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/manage-azure-rbac)
- [Kubernetes RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [AKS Identity Best Practices (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-identity)