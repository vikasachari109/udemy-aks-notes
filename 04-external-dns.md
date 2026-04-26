# External-DNS for Kubernetes & AKS
### A Beginner-Friendly Deep Dive

> **What this document covers:** What External-DNS is, why you need it, how it works step by step, every resource it creates, how it authenticates to Azure DNS, and how to configure it — all explained from absolute scratch.

---

## Table of Contents
1. [The Problem External-DNS Solves](#1-the-problem-external-dns-solves)
2. [Core Concepts You Must Know First](#2-core-concepts-you-must-know-first)
3. [What is External-DNS?](#3-what-is-external-dns)
4. [How External-DNS Works — Step by Step](#4-how-external-dns-works--step-by-step)
5. [Azure DNS Zone — What It Is and Why We Need It](#5-azure-dns-zone--what-it-is-and-why-we-need-it)
6. [DNS Record Types External-DNS Creates](#6-dns-record-types-external-dns-creates)
7. [How External-DNS Authenticates to Azure](#7-how-external-dns-authenticates-to-azure)
8. [Configuring External-DNS with a Kubernetes Service](#8-configuring-external-dns-with-a-kubernetes-service)
9. [Configuring External-DNS with an Ingress](#9-configuring-external-dns-with-an-ingress)
10. [Reading External-DNS Logs](#10-reading-external-dns-logs)
11. [The DNS Records Created in Azure DNS Zone](#11-the-dns-records-created-in-azure-dns-zone)
12. [Supported DNS Providers](#12-supported-dns-providers)
13. [Domain Filtering](#13-domain-filtering)
14. [Ownership and TXT Records](#14-ownership-and-txt-records)
15. [Full Architecture Diagram](#15-full-architecture-diagram)
16. [Pros, Cons and When to Use](#16-pros-cons-and-when-to-use)

---

## 1. The Problem External-DNS Solves

### The Manual Problem
Imagine you are running 10 microservices in AKS. Each service has a Kubernetes **LoadBalancer** Service or an **Ingress** that gets a public IP address from Azure. You want users to reach these services via friendly domain names:

```
app1.mycompany.com  →  20.30.40.50  (Service A's public IP)
app2.mycompany.com  →  30.30.40.50  (Ingress B's public IP)
api.mycompany.com   →  40.30.40.50  (Service C's public IP)
```

**Without External-DNS**, every time:
- You deploy a new service → you manually log into Azure DNS portal and create an A record
- A service IP changes (e.g., after cluster recreation) → you manually update the DNS record
- You delete a service → you manually delete the DNS record

This is tedious, error-prone, and doesn't scale. If you have 50 services across 3 environments, you're making dozens of manual DNS changes per week.

### The Solution
**External-DNS** is a Kubernetes controller (a pod running inside your cluster) that **automatically watches** your Kubernetes resources (Services, Ingresses) and **automatically creates, updates, and deletes** DNS records in your DNS provider (like Azure DNS) whenever those resources change.

> Think of it as: "A robot that keeps your DNS in sync with your Kubernetes cluster — automatically."

---

## 2. Core Concepts You Must Know First

### Kubernetes Service
A **Service** is a Kubernetes object that exposes a group of pods over a network. Services have three main types:

- **ClusterIP**: Only accessible inside the cluster. Gets a virtual internal IP from the Service CIDR. Other pods use this IP.
- **LoadBalancer**: Gets a real external (public) IP from Azure. Traffic from the internet reaches this IP, and Azure routes it to your pods. This is what External-DNS watches for public-facing apps.
- **NodePort**: Exposes a port on every worker node. Less common for production.

### Kubernetes Ingress
An **Ingress** is a Kubernetes object that defines HTTP/HTTPS routing rules. It acts like a smart router — it receives traffic on a single IP (the Ingress Controller's IP) and routes to different services based on the URL path or hostname.

```
Request → ingress-ip:80
  if host == app1.mycompany.com → forward to service app1-svc:80
  if host == app2.mycompany.com → forward to service app2-svc:80
```

An Ingress has a **hostname** field (like `app2.mycompany.com`). External-DNS reads this hostname and creates a DNS record pointing to the Ingress Controller's public IP.

### Kubernetes Annotation
An **Annotation** is metadata you attach to any Kubernetes object as key-value pairs. It doesn't affect how Kubernetes runs the object — it's just extra information that tools (like External-DNS) can read.

Example annotation:
```yaml
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: app01.houssem.cloud
```
This tells External-DNS: "Create a DNS record for `app01.houssem.cloud` pointing to this Service's public IP."

### ConfigMap
A **ConfigMap** is a Kubernetes object that stores configuration data as key-value pairs. Applications and controllers can read from ConfigMaps. External-DNS itself is configured via command-line arguments and a ConfigMap containing credentials.

### Secret
A **Secret** is like a ConfigMap but for sensitive data (passwords, tokens, API keys). It is base64-encoded. External-DNS uses a Secret (containing Azure credentials) to authenticate with Azure DNS.

### Pod
A **Pod** is the smallest deployable unit in Kubernetes — it wraps one or more containers. External-DNS runs as a pod inside your cluster. It has access to the Kubernetes API (to watch Services and Ingresses) and to Azure DNS (via credentials in a Secret).

### Deployment
A **Deployment** manages pods and ensures they stay running. External-DNS is deployed as a Deployment — if its pod crashes, Kubernetes automatically restarts it.

### Namespace
A **Namespace** is a virtual partition inside a cluster, used to group resources. External-DNS is typically installed in a dedicated namespace like `external-dns`.

### RBAC — Role-Based Access Control
**RBAC** in Kubernetes controls which pods/users can do what. External-DNS needs permission to **read** Services, Ingresses, and Endpoints. You grant this via a Kubernetes **ClusterRole** and **ClusterRoleBinding**.

**RBAC in Azure** controls which identities can modify Azure resources. External-DNS needs the **Contributor** role on your Azure DNS Zone resource group (or the DNS Zone Contributor role) to create/update/delete DNS records.

---

## 3. What is External-DNS?

**External-DNS** is an open-source Kubernetes controller maintained by the Kubernetes SIG-Network community. It is hosted at:
```
https://github.com/kubernetes-sigs/external-dns
```

It bridges **Kubernetes** (where your apps live) and **external DNS providers** (where your domain names live). It does one thing really well:

> **Watch Kubernetes resources → Translate them into DNS records → Keep the DNS provider in sync.**

External-DNS supports over 30 DNS providers including Azure DNS, AWS Route53, Google Cloud DNS, Cloudflare, and many more. In the AKS context, we focus on **Azure DNS** (public) and **Azure Private DNS** (private).

---

## 4. How External-DNS Works — Step by Step

Here is the exact sequence of events when you deploy a new app:

```
Step 1: You deploy a Kubernetes Service of type LoadBalancer
        ↓
Step 2: Azure allocates a Public IP (e.g., 20.73.224.33)
        ↓
Step 3: Kubernetes assigns that Public IP to the Service's
        status.loadBalancer.ingress field
        ↓
Step 4: External-DNS (which is continuously watching all Services
        and Ingresses) detects the new Service with annotation:
        external-dns.alpha.kubernetes.io/hostname: app01.houssem.cloud
        ↓
Step 5: External-DNS reads the Service's Public IP: 20.73.224.33
        ↓
Step 6: External-DNS calls the Azure DNS API using its credentials
        and creates an A record:
        app01.houssem.cloud → 20.73.224.33
        ↓
Step 7: External-DNS also creates a TXT record (ownership marker):
        externaldns-app01  →  TXT  →  "heritage=external-dns..."
        ↓
Step 8: Users can now reach app01.houssem.cloud in their browser
        DNS resolves to 20.73.224.33
        Azure Load Balancer routes to the pod
```

**What happens when you delete the Service?**
External-DNS detects the deletion → calls Azure DNS API → deletes both the A record and the TXT record.

**What happens when the IP changes?**
External-DNS detects the IP change → calls Azure DNS API → updates the A record to the new IP.

---

## 5. Azure DNS Zone — What It Is and Why We Need It

### Azure DNS Zone
An **Azure DNS Zone** is a container for DNS records for a domain. When you own a domain like `mycompany.com`, you create an Azure DNS Zone for it and add records.

**Creating an Azure DNS Zone:**
```bash
az group create --name rg-dns --location westeurope
az network dns zone create \
  --resource-group rg-dns \
  --name mycompany.com
```

This creates a zone with **NS (Name Server) records** that tell the internet where to find your DNS records:
```
NS records for mycompany.com:
  ns1-33.azure-dns.com.
  ns2-33.azure-dns.net.
  ns3-33.azure-dns.org.
  ns4-33.azure-dns.info.
```

**Delegating your domain to Azure DNS:**
You take these NS records and enter them at your domain registrar (e.g., GoDaddy, Namecheap). This tells the internet: "Azure DNS is authoritative for `mycompany.com`."

**From this point forward**, creating A records in your Azure DNS Zone makes those names resolvable worldwide.

### SOA Record
Every DNS zone has an **SOA (Start of Authority)** record. This is auto-created by Azure. It contains:
- The primary name server
- An email address for the admin
- A serial number (increments with every change)
- TTL values for refresh/retry/expire

You rarely modify this directly.

---

## 6. DNS Record Types External-DNS Creates

External-DNS creates two types of records for each managed resource:

### A Record (Address Record)
Maps a hostname to an IPv4 address. This is the main record users need.
```
app01.houssem.cloud  →  A  →  20.73.224.33  (TTL: 300 seconds)
```

### TXT Record (Text Record) — Ownership Marker
External-DNS creates a companion **TXT record** for every A record it manages. This record serves as an **ownership marker** — it proves that External-DNS "owns" this A record and allows it to safely update or delete it later.

External-DNS creates **two TXT records** per resource:
```
externaldns-app01.houssem.cloud   → TXT → "heritage=external-dns,external-dns/owner=default,external-dns/resource=service/default/app01-svc"
externaldns-a-app01.houssem.cloud → TXT → "heritage=external-dns,external-dns/owner=default,external-dns/resource=service/default/app01-svc"
```

**Why two TXT records?** The second one (`externaldns-a-`) is a newer format that allows External-DNS to manage multiple record types (A, AAAA, CNAME) for the same hostname without conflicts.

**Why does ownership matter?** Without ownership tracking, if two instances of External-DNS are running or if you manually created a DNS record with the same name, External-DNS would not know whether it's safe to update or delete it. The TXT record solves this.

---

## 7. How External-DNS Authenticates to Azure

To create DNS records in Azure, External-DNS needs permission to call the Azure DNS REST API. There are two authentication methods:

### Method 1: Service Principal (SPN) with Client Secret
A **Service Principal** is an identity (like a service account) in Azure Active Directory. You give it a **role** on the DNS Zone resource group, and it gets a **Client ID + Client Secret** (essentially a username and password for applications).

**Step 1: Create a Service Principal**
```bash
az ad sp create-for-rbac --name "external-dns-sp"
# Output:
# {
#   "appId": "9cc6c0d1-...",         ← this is the Client ID
#   "password": "~K~8Q~PA9WM...",    ← this is the Client Secret
#   "tenant": "16b3c013-..."         ← your Azure AD Tenant ID
# }
```

**Step 2: Grant it Contributor access on the DNS Zone resource group**
```bash
az role assignment create \
  --role "DNS Zone Contributor" \
  --assignee "9cc6c0d1-..." \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-dns"
```

**Step 3: Create the azure.json config file**
```json
{
  "tenantId": "16b3c013-d300-468d-ac64-7eda0820b6d3",
  "subscriptionId": "82f6d75e-85f4-434a-ab74-5dddd9fa8910",
  "resourceGroup": "rg-azure-dns",
  "aadClientId": "9cc6c0d1-99a3-4d86-9df4-a84df55b8232",
  "aadClientSecret": "YOUR_CLIENT_SECRET"
}
```

**Step 4: Store it as a Kubernetes Secret**
```bash
kubectl create secret generic azure-config-file \
  --namespace external-dns \
  --from-file azure.json
```

External-DNS reads this Secret and uses it to call Azure APIs.

### Method 2: Workload Identity (Recommended for AKS)
A **Workload Identity** is the modern way in AKS to assign Microsoft Entra identities to pods. It uses OIDC federation to allow pods to authenticate to Azure resources without secrets.

**Why is Workload Identity better?**
- No secrets to rotate or accidentally expose
- Credentials never leave Azure
- Better security posture — no `aadClientSecret` in your yaml files
- Integrated with AKS and Microsoft Entra ID

**How to set it up:**
```bash
# Create a User Assigned Managed Identity
az identity create --name external-dns-identity --resource-group my-rg

# Get its Client ID
CLIENT_ID=$(az identity show --name external-dns-identity \
  --resource-group my-rg --query clientId -o tsv)

# Grant it DNS Zone Contributor on the DNS resource group
az role assignment create \
  --role "DNS Zone Contributor" \
  --assignee $CLIENT_ID \
  --scope "/subscriptions/.../resourceGroups/rg-dns"

# In AKS, enable Workload Identity and create federated identity
# (This requires setting up the service account with annotations)
```

For External-DNS, configure the deployment to use Workload Identity by annotating the service account and using the Azure Identity client library.

With Workload Identity, no `azure.json` file is needed; the pod authenticates automatically.

### Azure RBAC Roles for External-DNS
External-DNS only needs to **create, update, and delete DNS records** — not manage the DNS Zone itself. The minimum required role is:

| Role | What it allows |
|------|---------------|
| **DNS Zone Contributor** | Create/update/delete record sets within DNS zones (recommended — least privilege) |
| **Contributor** | Broader — can also create/delete zones themselves (more than needed) |

Always use **least privilege** — give only what's needed.

---

## 8. Configuring External-DNS with a Kubernetes Service

### Service YAML with External-DNS Annotation
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app01-svc
  annotations:
    external-dns.alpha.kubernetes.io/hostname: app01.houssem.cloud
spec:
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer   # MUST be LoadBalancer to get a public IP
```

**What happens:**
1. Kubernetes creates the Service
2. Azure allocates Public IP (e.g., `20.73.224.33`) to the Service
3. External-DNS reads the annotation → target hostname = `app01.houssem.cloud`
4. External-DNS reads the Public IP → `20.73.224.33`
5. External-DNS calls Azure DNS API → creates A record: `app01.houssem.cloud → 20.73.224.33`

### External-DNS Deployment YAML (simplified)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.14.0
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=houssem.cloud
        - --provider=azure
        - --azure-resource-group=rg-azure-dns
        - --txt-owner-id=default
```

**Key flags explained:**

| Flag | What it does |
|------|-------------|
| `--source=service` | Watch Kubernetes Services |
| `--source=ingress` | Watch Kubernetes Ingresses |
| `--domain-filter=houssem.cloud` | Only manage records in this domain (safety guard) |
| `--provider=azure` | Use Azure DNS as the DNS provider |
| `--azure-resource-group=rg-azure-dns` | Resource group containing the DNS Zone |
| `--txt-owner-id=default` | A unique string stamped on TXT ownership records |

---

## 9. Configuring External-DNS with an Ingress

### Ingress YAML
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app02-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: app02.houssem.cloud
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app02-svc
            port:
              number: 80
```

**What happens:**
1. NGINX Ingress Controller exposes a public IP (e.g., `20.31.96.10`)
2. The Ingress object has `host: app02.houssem.cloud`
3. External-DNS reads the Ingress → hostname = `app02.houssem.cloud`, IP = `20.31.96.10`
4. External-DNS creates: `app02.houssem.cloud → A → 20.31.96.10`

**Note**: For Ingresses, External-DNS reads the hostname from the `spec.rules[].host` field — no annotation needed. The annotation is only required for Services (since Services don't have a hostname field by default).

---

## 10. Reading External-DNS Logs

External-DNS produces detailed logs that show exactly what it's doing. Here is a real log output and what each line means:

```bash
kubectl logs external-dns-5fd5797df-xklxn -n external-dns
```

```
level=info msg="Updating A record named 'app01' to '20.73.224.33'
for Azure DNS zone 'houssem.cloud'."
```
→ External-DNS found the Service with IP `20.73.224.33` and is creating/updating the A record.

```
level=info msg="Updating TXT record named 'externaldns-app01' to
'"heritage=externaldns,external-dns/owner=default,
external-dns/resource=service/default/app01-svc"'
for Azure DNS zone 'houssem.cloud'."
```
→ External-DNS is creating the ownership TXT record. The value tells us: this record belongs to External-DNS (`heritage=external-dns`), the owner ID is `default`, and the source Kubernetes resource is the `app01-svc` Service in the `default` namespace.

```
level=info msg="Updating A record named 'app02' to '20.31.96.10'
for Azure DNS zone 'houssem.cloud'."
```
→ External-DNS found the Ingress with hostname `app02.houssem.cloud` and IP `20.31.96.10`.

**Reconciliation Loop**: External-DNS runs its sync loop every **~30 seconds** by default. It compares the desired state (from Kubernetes) with the actual state (in Azure DNS) and makes changes as needed.

---

## 11. The DNS Records Created in Azure DNS Zone

After External-DNS runs, your Azure DNS Zone (`houssem.cloud`) contains:

| Name | Type | TTL | Value |
|------|------|-----|-------|
| `@` | NS | 172800 | `ns1-33.azure-dns.com.`, etc. |
| `@` | SOA | 3600 | Azure DNS admin metadata |
| `app01` | **A** | 300 | `20.86.202.21` ← Service's public IP |
| `app02` | **A** | 300 | `20.73.123.67` ← Ingress IP |
| `externaldns-app01` | **TXT** | 300 | Ownership marker |
| `externaldns-a-app01` | **TXT** | 300 | Ownership marker |
| `externaldns-app02` | **TXT** | 300 | Ownership marker |
| `externaldns-a-app02` | **TXT** | 300 | Ownership marker |

---

## 12. Supported DNS Providers

External-DNS supports over 30 DNS providers. Here are the most popular ones:

| Provider | Description |
|----------|-------------|
| **Azure DNS** | Public DNS zones in Azure |
| **Azure Private DNS** | Private DNS zones in Azure (for private clusters) |
| **AWS Route53** | Amazon's DNS service |
| **Google Cloud DNS** | Google's DNS service |
| **Cloudflare** | Popular CDN/DNS provider |
| **DigitalOcean** | Cloud provider DNS |
| **Hetzner** | European cloud provider |
| **Linode** | Cloud provider |
| **OVH** | European hosting provider |

For AKS, use `azure` or `azure-private-dns`.

---

## 13. Domain Filtering

External-DNS can be configured to only manage records in specific domains using `--domain-filter`. This is a safety feature to prevent External-DNS from accidentally modifying DNS records for domains you don't own.

```bash
# Only manage records in houssem.cloud
--domain-filter=houssem.cloud

# Multiple domains
--domain-filter=houssem.cloud --domain-filter=mycompany.com
```

Without domain filtering, External-DNS would try to create records for any hostname it finds in your cluster, which could fail or cause issues.

---

## 14. Ownership and TXT Records

External-DNS uses TXT records to track which DNS records it "owns". This prevents conflicts when:

- Multiple External-DNS instances are running
- You have manual DNS records with the same name
- External-DNS is redeployed

The TXT record contains metadata like:
```
"heritage=external-dns,external-dns/owner=default,external-dns/resource=service/default/app01-svc"
```

If External-DNS finds a DNS record without its TXT marker, it won't touch it.

---

## 15. Full Architecture Diagram

```
[Kubernetes Cluster]
     ↓
[External-DNS Pod]
     ↓ Watches Services & Ingresses
     ↓
[Azure DNS API]
     ↓
[Azure DNS Zone]
     ↓
[Internet Users]
```

---

## 16. Pros, Cons and When to Use

### Pros
- **Automation**: No more manual DNS changes
- **Reliability**: IPs change automatically updated
- **Scalability**: Works with hundreds of services
- **Multi-provider**: Supports many DNS providers
- **Safety**: Domain filtering and ownership tracking

### Cons
- **Complexity**: Requires Azure permissions and Kubernetes RBAC
- **Cost**: Azure DNS has a small cost per zone
- **Delay**: DNS propagation takes time (TTL)
- **Debugging**: Logs can be verbose

### When to Use
- You have multiple services with public IPs
- You want to automate DNS management
- You're using AKS with Azure DNS
- You have frequent deployments

### When NOT to Use
- Static IPs that never change
- Small number of services (manual is fine)
- Using a different DNS provider not supported by External-DNS

---

- [External-DNS GitHub](https://github.com/kubernetes-sigs/external-dns)
- [Azure DNS Documentation](https://learn.microsoft.com/en-us/azure/dns/)
- [Workload Identity for External-DNS on AKS](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)