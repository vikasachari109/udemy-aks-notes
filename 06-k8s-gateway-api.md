# Kubernetes Gateway API & Application Gateway for Containers
### A Beginner-Friendly Deep Dive

> **What this document covers:** What Gateway API is, why it was created to replace Ingress, the role-oriented RBAC model, GatewayClass/Gateway/HTTPRoute resources, Application Gateway for Containers (AGC) architecture, ALB Controller components, Associations, Frontends, AGC vs AGIC comparison, deployment strategies, and the critical 2025-2026 Ingress NGINX retirement timeline — all explained from absolute scratch with official Azure documentation.

---

## Table of Contents
1. [The Old Way: Kubernetes Ingress API](#1-the-old-way-kubernetes-ingress-api)
2. [Why Ingress API Had to Evolve](#2-why-ingress-api-had-to-evolve)
3. [Ingress NGINX Retirement Timeline (2025-2026)](#3-ingress-nginx-retirement-timeline-2025-2026)
4. [What is Gateway API?](#4-what-is-gateway-api)
5. [Gateway API Components and Roles](#5-gateway-api-components-and-roles)
6. [The Role-Oriented RBAC Model](#6-the-role-oriented-rbac-model)
7. [Gateway API Resources Explained](#7-gateway-api-resources-explained)
8. [Azure's Load Balancing Portfolio](#8-azures-load-balancing-portfolio)
9. [What is Application Gateway for Containers (AGC)?](#9-what-is-application-gateway-for-containers-agc)
10. [AGC Architecture — Data Plane and Control Plane](#10-agc-architecture--data-plane-and-control-plane)
11. [ALB Controller — The Kubernetes Brain](#11-alb-controller--the-kubernetes-brain)
12. [Associations — Connecting to Your VNet](#12-associations--connecting-to-your-vnet)
13. [Frontends — The Public Entry Points](#13-frontends--the-public-entry-points)
14. [AGC vs Application Gateway Ingress Controller (AGIC)](#14-agc-vs-application-gateway-ingress-controller-agic)
15. [Deployment Strategies: BYO vs Managed by ALB Controller](#15-deployment-strategies-byo-vs-managed-by-alb-controller)
16. [Gateway API Example Configuration](#16-gateway-api-example-configuration)
17. [AGC Features and Benefits](#17-agc-features-and-benefits)
18. [Prerequisites and Requirements](#18-prerequisites-and-requirements)
19. [Migration from Ingress NGINX to AGC](#19-migration-from-ingress-nginx-to-agc)
20. [Troubleshooting Common Issues](#20-troubleshooting-common-issues)
21. [Summary Comparison Tables](#21-summary-comparison-tables)

---

## 1. The Old Way: Kubernetes Ingress API

### What is Kubernetes Ingress?

**Ingress** is a Kubernetes resource (introduced in Kubernetes 1.1, GA in 1.19) that manages external HTTP and HTTPS access to services within a cluster. It acts as a Layer 7 (application layer) router — receiving traffic on a single IP and routing it to different services based on:
- **Hostname** (e.g., `app1.example.com` vs `app2.example.com`)
- **URL path** (e.g., `/api` vs `/web`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - aks-app.westeurope.cloudapp.azure.com
      secretName: tls-secret
  rules:
    - host: aks-app.westeurope.cloudapp.azure.com
      http:
        paths:
          - path: /app1(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: app1-svc
                port:
                  number: 80
          - path: /app2(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: app2-svc
                port:
                  number: 80
```

**What this does:**
- Requests to `http://aks-app.westeurope.cloudapp.azure.com/app1` → routed to `app1-svc:80`
- Requests to `http://aks-app.westeurope.cloudapp.azure.com/app2` → routed to `app2-svc:80`

### Ingress Controller

An **Ingress Controller** is the actual software (a pod or set of pods) that reads Ingress resources and implements the routing rules. The Ingress resource by itself does nothing — it's just a configuration object. The Ingress Controller watches for Ingress resources and configures an underlying proxy (NGINX, HAProxy, Envoy, etc.) to route traffic accordingly.

**Popular Ingress Controllers:**
- NGINX Ingress Controller (community and Microsoft-managed versions)
- Traefik
- HAProxy Ingress
- Application Gateway Ingress Controller (AGIC) — Azure-specific
- Istio Ingress Gateway

**How it works:**
```
Internet → Public IP (Load Balancer)
           │
           ▼
    Ingress Controller Pod
      (NGINX, Traefik, etc.)
           │
           ▼ (reads routing rules from Ingress resource)
           │
    ┌──────┴──────┐
    │             │
Service A      Service B
    │             │
  Pods          Pods
```

---

## 2. Why Ingress API Had to Evolve

The Ingress API worked fine for simple use cases, but it had fundamental limitations that became clear as Kubernetes matured:

### Problem 1: Annotation Hell
Advanced features (SSL/TLS, rate limiting, authentication, traffic splitting, headers, redirects) weren't part of the Ingress spec. Each Ingress Controller implemented them via **annotations** — non-standard key-value metadata.

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/rate-limit: "100"
  nginx.ingress.kubernetes.io/auth-url: "https://auth.example.com"
  # ... dozens more, all NGINX-specific
```

**The Problem:** If you switched from NGINX to Traefik, ALL your annotations broke. There was no standardization. This led to "vendor lock-in" at the Ingress Controller level.

### Problem 2: Single Flat Resource Model
The Ingress resource mixes infrastructure concerns (which load balancer to use, TLS certificates) with application routing (path-based routing). This doesn't map well to real-world organizations where:
- **Platform teams** manage infrastructure (load balancers, TLS, domains)
- **Application teams** define routing rules for their apps

With Ingress, both roles edit the same resource → conflict and confusion.

### Problem 3: No Role Separation (RBAC Challenges)
Because Ingress is a single resource, RBAC becomes difficult. You can't give an application team permission to configure routes WITHOUT also giving them permission to configure TLS certificates or change the load balancer class.

### Problem 4: Limited Expressiveness
Ingress was designed for simple HTTP routing. It doesn't natively support:
- **TCP/UDP routing** (Layer 4)
- **Traffic splitting** (90% to v1, 10% to v2 — canary deployments)
- **Header-based routing** (route based on HTTP headers)
- **gRPC routing**
- **Cross-namespace routing** (one Gateway shared by multiple namespaces)

Ingress Controllers added these features via annotations, but again — non-standard.

---

## 3. Ingress NGINX Retirement Timeline (2025-2026)

### The Announcement (November 2025)

The **Kubernetes SIG Network** and **Security Response Committee** announced that the community **Ingress-NGINX project** (the most widely-used Ingress Controller) would:
- Enter **best-effort maintenance** until **March 2026**
- Receive **no further releases, bug fixes, or security patches** after March 2026

**Why?**
- Small group of volunteers maintaining it for years
- Accumulated serious technical debt from flexible annotation model
- Maintainers couldn't sustain the project anymore

### What This Means for AKS Users

**If you self-installed Ingress NGINX via Helm:**
- You're directly exposed to the March 2026 upstream deadline
- After March 2026, you're on your own for CVEs (security vulnerabilities)

**If you use the AKS Application Routing add-on with NGINX:**
- Microsoft will provide **critical security patches** until **November 2026**
- **No new features, no general bug fixes** — only critical security fixes
- After November 2026, no Azure support

### Microsoft's Recommended Migration Path

> **Important (2025-2026):** AKS is aligning with upstream Kubernetes by moving to **Gateway API** as the long-term standard for ingress and L7 traffic management.

Microsoft recommends:
1. **Application Routing add-on users** → Migrate to **Application Routing with Gateway API** (powered by Istio control plane, available H1 2026)
2. **Self-managed NGINX users** → Migrate to:
   - **Application Gateway for Containers** (Gateway API or Ingress API)
   - **Istio-based service mesh add-on** (Gateway API)
   - Or another actively-maintained Ingress Controller

**Timeline:**
```
November 2025: Ingress-NGINX retirement announced
March 2026: Upstream Ingress-NGINX maintenance ends
November 2026: AKS Application Routing (NGINX) support ends
2026+: Gateway API is the future
```

---

## 4. What is Gateway API?

**Gateway API** is an open-source project managed by the **Kubernetes SIG-Network** community. It is a collection of Kubernetes resources (CRDs) that model service networking in a more expressive, extensible, and role-oriented way than Ingress.

GitHub: `https://github.com/kubernetes-sigs/gateway-api`

**Gateway API is NOT a replacement for Ingress in the sense of deprecation** — Ingress API still works and is still supported. But:
- The Ingress API spec is **frozen** — no new features will be added
- Active development has shifted to Gateway API
- Gateway API went **GA (v1.0)** in October 2023, v1.1 in 2024

### Key Principles of Gateway API

| Principle | What it Means |
|-----------|--------------|
| **Role-oriented** | Different resources for different personas (infrastructure vs app teams) |
| **Expressive** | Native support for advanced features (traffic splitting, header routing, TCP/gRPC) |
| **Extensible** | Standardized extension points instead of ad-hoc annotations |
| **Portable** | Works across cloud providers and implementations |

> **Think of Gateway API as:** "Ingress 2.0 — designed from scratch with lessons learned from 8 years of Ingress usage."

---

## 5. Gateway API Components and Roles

Gateway API splits the monolithic Ingress resource into **three separate resources**, each owned by different roles:

```
┌─────────────────────────────────────────────────────────┐
│          Infrastructure Provider                        │
│  (Platform Team / Cloud Provider / Cluster Admin)       │
│                                                          │
│  Manages: GatewayClass                                  │
│  "What type of gateway infrastructure is available"     │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│          Cluster Operator                               │
│  (Platform Team / SRE)                                  │
│                                                          │
│  Manages: Gateway                                       │
│  "Instantiate a gateway with listeners and policies"    │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│          Application Developer                          │
│  (Dev Team)                                             │
│                                                          │
│  Manages: HTTPRoute, TCPRoute, GRPCRoute                │
│  "Route traffic for MY app to MY service"               │
└─────────────────────────────────────────────────────────┘
```

### The Three Core Resources

**1. GatewayClass** (Infrastructure Provider)
Defines the **type** of gateway infrastructure available. Think of it like a "template" or "flavor" of gateway.

Example: `azure-alb-external` (provided by Application Gateway for Containers)

**2. Gateway** (Cluster Operator)
An **instance** of a GatewayClass. Defines:
- Which GatewayClass to use
- Listeners (ports and protocols to accept traffic on)
- Which namespaces can attach routes to this Gateway

**3. HTTPRoute, TCPRoute, GRPCRoute** (Application Developer)
Routing rules that attach to a Gateway and define:
- Match conditions (hostname, path, headers)
- Backend services to route to
- Optional filters (request/response modification)

---

## 6. The Role-Oriented RBAC Model

### How Separation Works

```
┌─────────────────────────────────────────────────────────┐
│        Platform Team (Cluster Operator)                 │
│                                                          │
│  Creates: Gateway "foo"                                 │
│    gatewayClassName: azure-alb-external                 │
│    domain: foo.example.com                              │
│    TLS certificate: from Azure Key Vault                │
│    listeners: HTTP on port 80, HTTPS on port 443        │
│    allowedRoutes: from namespaces [store, site]         │
└─────────────────┬───────────────────────────────────────┘
                  │
      ┌───────────┼───────────┐
      │                       │
┌─────▼──────────────────┐  ┌─▼──────────────────────────┐
│   Store Team           │  │    Site Team               │
│   (namespace: store)   │  │    (namespace: site)       │
│                        │  │                            │
│ Creates: HTTPRoute     │  │  Creates: HTTPRoute        │
│   parentRef: foo       │  │    parentRef: foo          │
│   hostname: store.*    │  │    hostname: site.*        │
│   routes:              │  │    routes:                 │
│     /store/* → 90% v1  │  │      /site/* → site-svc   │
│                10% v2  │  │                            │
└────────────────────────┘  └────────────────────────────┘
```

**What each team can do:**

| Role | Can Create/Modify | Cannot Touch |
|------|------------------|--------------|
| **Infrastructure Provider** | GatewayClass | Gateway, Routes |
| **Cluster Operator** | Gateway, TLS certs, domains | Routes (unless explicitly allowed) |
| **App Developer** | HTTPRoute in their namespace | Gateway, GatewayClass, other namespaces' routes |

This is enforced via **Kubernetes RBAC** — each team only has permissions for their resources.

---

## 7. Gateway API Resources Explained

### GatewayClass

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: azure-alb-external
spec:
  controllerName: alb.networking.azure.io/alb-controller
```

**What it says:** "There exists a gateway type called `azure-alb-external`, and it's managed by the ALB Controller."

In AKS, when you install Application Gateway for Containers, the ALB Controller automatically creates this GatewayClass for you. The name is always `azure-alb-external`.

### Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-app
  namespace: ns-gateway
  annotations:
    alb.networking.azure.io/alb-id: /subscriptions/.../alb-resource-id
spec:
  gatewayClassName: azure-alb-external
  listeners:
    - name: http-listener
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All  # Allow routes from any namespace
  addresses:
    - type: alb.networking.azure.io/alb-frontend
      value: frontend-app  # Reference to AGC Frontend resource
```

**Breaking this down:**

**`gatewayClassName: azure-alb-external`**
This Gateway uses the Application Gateway for Containers implementation.

**`listeners`**
Defines which ports and protocols this Gateway listens on. Each listener can have:
- `port`: 80, 443, 8080, etc.
- `protocol`: HTTP, HTTPS, TCP, TLS
- `allowedRoutes`: which namespaces can attach HTTPRoutes to this listener

**`allowedRoutes.namespaces.from`**
- `All` — any namespace can attach routes (open)
- `Same` — only routes in the same namespace as the Gateway can attach
- `Selector` — only routes in namespaces matching a label selector

**`addresses`**
Points to the AGC Frontend — the actual public IP / FQDN entry point.

### HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httproute-app
  namespace: ns-app
spec:
  parentRefs:
    - kind: Gateway
      name: gateway-app
      namespace: ns-gateway
  hostnames:
    - "app.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: svc-app
          port: 80
          weight: 90   # 90% of traffic
        - name: svc-app-v2
          port: 80
          weight: 10   # 10% of traffic (canary)
```

**Breaking this down:**

**`parentRefs`**
Attaches this HTTPRoute to the Gateway named `gateway-app` in the `ns-gateway` namespace. The HTTPRoute and Gateway can be in different namespaces!

**`hostnames`**
Only requests with `Host: app.example.com` will match this route.

**`rules.matches`**
Match conditions:
- `path.type: PathPrefix, value: /` — match all paths
- Could also match on headers, query params, HTTP method

**`backendRefs`**
Services to route traffic to:
- `weight: 90` / `weight: 10` — **traffic splitting!** 90% goes to `svc-app`, 10% to `svc-app-v2`. This is native Gateway API — no annotations needed.

---

## 8. Azure's Load Balancing Portfolio

Before diving into Application Gateway for Containers, let's understand where it fits in Azure's load balancing portfolio:

| Product | Layer | Scope | Use Case |
|---------|-------|-------|----------|
| **Azure Load Balancer** (Standard) | L4 (TCP/UDP) | Regional or Global | VM/VMSS/AKS workloads, passthrough LB |
| **Azure Application Gateway** | L7 (HTTP/HTTPS) | Regional | VM/VMSS/Hybrid workloads, WAF |
| **Azure Front Door** | L7 (HTTP/HTTPS) | Global | Multi-region web apps, WAF, CDN |
| **Application Gateway for Containers (AGC)** | L7 (HTTP/HTTPS) | Regional/Global | **AKS/container workloads**, Gateway API, dynamic traffic, WAF |

**Key Differentiator:**
- **Application Gateway** — designed for VM-based workloads, uses classic Ingress API, static configuration
- **Application Gateway for Containers** — designed specifically for Kubernetes, uses Gateway API, dynamic configuration, containerized traffic management

---

## 9. What is Application Gateway for Containers (AGC)?

**Application Gateway for Containers** is Azure's **managed Layer 7 load balancer** for AKS. It is:
- The **evolution** of Application Gateway Ingress Controller (AGIC)
- A **fully managed Azure service** (not a pod you deploy in your cluster)
- An **implementation** of the Kubernetes Gateway API
- **Specifically designed** for containerized workloads in AKS

### Timeline
- **Preview:** 2023
- **GA (General Availability):** Late 2024
- **WAF Support Added:** November 2025

### Architecture Overview

```
                    ┌──────────────────────────┐
                    │       Internet           │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │ Application Gateway for  │
                    │     Containers (AGC)     │
                    │  (Managed Azure Service) │
                    │                          │
                    │  - Frontends (FQDNs)     │
                    │  - Associations (VNet)   │
                    │  - Routing Rules         │
                    └────────────┬─────────────┘
                                 │
                    Delegated Subnet (/24)
                                 │
               ┌─────────────────┼────────────────┐
               │         AKS Cluster VNet          │
               │                                   │
               │  ┌───────────────────────────┐   │
               │  │   ALB Controller Pods     │   │
               │  │ (watches Gateway API CRDs)│   │
               │  └──────────┬────────────────┘   │
               │             │                     │
               │  ┌──────────▼──────────┐         │
               │  │   Pod A      Pod B  │         │
               │  │   (app)      (app)  │         │
               │  └─────────────────────┘         │
               └───────────────────────────────────┘
```

**Two Planes:**

**1. Data Plane (Azure-managed):**
- The AGC resource itself — runs outside your cluster
- Handles actual traffic routing, load balancing, SSL/TLS termination
- Fully managed by Azure (you don't patch or scale it)

**2. Control Plane (In-cluster):**
- ALB Controller — a Kubernetes Deployment running in your cluster
- Watches Gateway API resources (Gateway, HTTPRoute, policies)
- Translates them into configuration for the AGC data plane via Azure APIs

---

## 10. AGC Architecture — Data Plane and Control Plane

### Data Plane — Application Gateway for Containers Resource

The AGC resource lives in Azure (outside your cluster) and consists of:

**AGC Resource (Parent)**
```
/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ServiceNetworking/trafficControllers/{name}
```

**Child Resources:**

**1. Frontends**
Public entry points. Each Frontend gets a unique auto-generated FQDN.
```
Format: *.fz{XX}.alb.azure.com  (where XX is 00-99)
Example: 307f3885631c281453835f0a9a97425.fz13.alb.azure.com
```

You can use this FQDN directly, or create a CNAME record pointing your custom domain to it.

**2. Associations**
Defines the connection into a VNet. Maps 1:1 to a delegated subnet.

**Association Requirements:**
- Subnet must be `/24` or smaller (max 251 usable IPs)
- Subnet must be **delegated** to `Microsoft.ServiceNetworking/trafficControllers`
- Cannot be used for any other resources (VMs, pods, etc.)
- Can be in the same VNet as AKS, or a peered VNet

### Control Plane — ALB Controller

The ALB Controller runs **inside your AKS cluster** as a Kubernetes Deployment. When you enable the AGC add-on or install via Helm, you get:

**Two ALB Controller Pods:**

**1. `alb-controller` pod** (main controller)
- Watches Gateway, HTTPRoute, HealthCheckPolicy, and other CRDs
- Translates them into AGC configuration
- Calls Azure REST APIs to update the AGC resource
- Uses Workload Identity / Managed Identity for authentication

**2. `alb-controller-bootstrap` pod** (CRD installer)
- Responsible for installing Gateway API CRDs
- Runs as an Init Container
- Creates the `azure-alb-external` GatewayClass

```bash
kubectl get pods -n kube-system | grep alb-controller
# alb-controller-764cf9ccdf-hf8v6              1/1     Running   0   31h
# alb-controller-bootstrap-5c6c59c7b8-cspg7    1/1     Running   0   31h
```

**ALB Controller Services:**
```bash
kubectl get svc -n kube-system | grep alb-controller
# alb-controller            ClusterIP   10.0.126.102   <none>   8000/TCP   31h
# alb-controller-bootstrap  ClusterIP   10.0.91.170    <none>   9005/TCP   31h
```

---

## 11. ALB Controller — The Kubernetes Brain

### What Does the ALB Controller Do?

The ALB Controller is the bridge between Kubernetes (your cluster) and Azure (the AGC service). Here's the flow:

```
Developer creates:
  Gateway + HTTPRoute YAML
           │
           ▼
kubectl apply -f gateway.yaml
           │
           ▼
ALB Controller Pod detects new Gateway
           │
           ▼
Reads the Gateway spec:
  - gatewayClassName: azure-alb-external
  - listener: port 80, HTTP
  - frontend reference
           │
           ▼
Calls Azure REST API:
  - Create/Update AGC Frontend
  - Configure listener rules
  - Update routing tables
           │
           ▼
AGC Resource Updated in Azure
           │
           ▼
Traffic starts flowing:
  Internet → AGC Frontend FQDN
           → AGC Routes to Pod IPs
```

### Authentication — Workload Identity

The ALB Controller uses **Azure Workload Identity** (federated Managed Identity) to authenticate with Azure APIs.

**Setup:**
1. Create a User-Assigned Managed Identity
2. Grant it the **AppGW for Containers Configuration Manager** role on the AGC resource group
3. Create a federated credential linking the Managed Identity to the ALB Controller's Kubernetes Service Account
4. ALB Controller uses this identity to call Azure APIs (no secrets needed!)

**Role: AppGW for Containers Configuration Manager**
This is a built-in Azure RBAC role specifically for ALB Controller. It has:
- `Microsoft.ServiceNetworking/trafficControllers/read`
- `Microsoft.ServiceNetworking/trafficControllers/write`
- `Microsoft.ServiceNetworking/trafficControllers/frontends/*`
- `Microsoft.ServiceNetworking/trafficControllers/associations/*`

> **Important:** The **Owner** and **Contributor** roles do NOT have the data action permissions needed by ALB Controller. You MUST use the specific **AppGW for Containers Configuration Manager** role.

### Logging and Debugging

```bash
# View ALB Controller logs
kubectl logs -n kube-system deployment/alb-controller --tail=100

# Example log output:
# {"level":"info","component":"lb-resources-reconciler",
#  "message":"Successfully processed object testinfra/gateway-01"}
# {"level":"info","component":"armclient-logger",
#  "message":"Creating Application Gateway for Containers resource alb-abc123"}
```

---

## 12. Associations — Connecting to Your VNet

### What is an Association?

An **Association** is a child resource of AGC that defines:
- **Where** the AGC data plane connects to (which VNet/Subnet)
- The **entry point** for traffic into your AKS cluster

**Mapping:**
```
1 AGC Resource → Multiple Associations (one per VNet connection)
1 Association → 1 Delegated Subnet
```

### Subnet Delegation

The subnet used by the Association must be **delegated** to `Microsoft.ServiceNetworking/trafficControllers`.

**What delegation means:**
Azure reserves the subnet exclusively for AGC. No VMs, pods, or other resources can use it. AGC deploys proxy components into this subnet to forward traffic to your pod IPs.

**Creating a delegated subnet:**
```bash
az network vnet subnet create \
  --resource-group my-rg \
  --vnet-name aks-vnet \
  --name subnet-alb \
  --address-prefix 10.225.0.0/24 \
  --delegations Microsoft.ServiceNetworking/trafficControllers
```

**Subnet Requirements:**
- CIDR: `/24` or smaller (max 251 usable IPs)
- Cannot overlap with AKS pod CIDR or service CIDR
- Can be in the same VNet as AKS, or a separate peered VNet

### Association in Azure Portal

```
AGC Resource: agwc-alb
  └── Association: association-app
        ├── Name: association-app
        ├── Location: West Europe
        └── Virtual Network Subnet: aks-vnet-27545368/subnet-alb
```

---

## 13. Frontends — The Public Entry Points

### What is a Frontend?

A **Frontend** is a child resource of AGC that defines:
- The **inbound endpoint** where client traffic arrives
- A unique auto-generated **FQDN**

Each Gateway in Kubernetes references a Frontend via the `addresses` field.

### Auto-Generated FQDN

Every Frontend gets a globally unique FQDN managed by Azure:
```
Format: *.fz{XX}.alb.azure.com
Example: 307f3885631c281453835f0a9a97425.fz13.alb.azure.com
```

**How to use it:**
1. **Option A:** Use the FQDN directly (ugly but works)
2. **Option B:** Create a CNAME record pointing your custom domain to it:
   ```
   app.example.com  CNAME  307f3885631c281453835f0a9a97425.fz13.alb.azure.com
   ```

### Frontend Limits
- **Up to 5 Frontends** per AGC resource
- Each Frontend can be referenced by multiple Gateways

### Creating a Frontend (Azure CLI)

```bash
az network alb frontend create \
  --resource-group my-rg \
  --alb-name agwc-alb \
  --frontend-name frontend-app
```

After creation:
```bash
az network alb frontend show \
  --resource-group my-rg \
  --alb-name agwc-alb \
  --frontend-name frontend-app \
  --query fqdn -o tsv

# Output: 307f3885631c281453835f0a9a97425.fz13.alb.azure.com
```

---

## 14. AGC vs Application Gateway Ingress Controller (AGIC)

### The Old Way: AGIC

**Application Gateway Ingress Controller (AGIC)** was the first-generation solution for using Azure Application Gateway with AKS.

```
┌────────────────────────────────────────────────┐
│          AKS Cluster                            │
│                                                 │
│  ┌───────────────────┐                         │
│  │  AGIC Pod         │                         │
│  │  (watches Ingress)│──────┐                  │
│  └───────────────────┘      │                  │
│           │                 │                  │
│  ┌────────▼──────┐          │                  │
│  │  Service/Pods │          │                  │
│  └───────────────┘          │                  │
└─────────────────────────────┼──────────────────┘
                              │
                 Azure Resource Manager API
                              │
             ┌────────────────▼────────────────┐
             │  Azure Application Gateway      │
             │  (Classic App Gateway resource) │
             └─────────────────────────────────┘
```

**How AGIC works:**
- AGIC pod runs in the cluster
- Watches Kubernetes **Ingress** resources
- Calls Azure ARM API to configure the Application Gateway

**Limitations:**
- Uses the classic Application Gateway (designed for VMs)
- Static configuration (slow to update)
- Only supports Ingress API (not Gateway API)
- AGIC pod is in the traffic path (can be a bottleneck)

### The New Way: Application Gateway for Containers (AGC)

```
┌────────────────────────────────────────────────┐
│          AKS Cluster                            │
│                                                 │
│  ┌───────────────────┐                         │
│  │  ALB Controller   │                         │
│  │  (watches Gateway │──────┐                  │
│  │   API resources)  │      │                  │
│  └───────────────────┘      │                  │
│           │                 │                  │
│  ┌────────▼──────┐          │                  │
│  │  Service/Pods │          │                  │
│  └───────────────┘          │                  │
└─────────────────────────────┼──────────────────┘
                              │
                Azure ServiceNetworking API
                              │
       ┌──────────────────────▼──────────────────┐
       │ Application Gateway for Containers      │
       │ (New containerized data plane)          │
       │  - Routes directly to Pod IPs           │
       │  - Dynamic updates                      │
       └─────────────────────────────────────────┘
```

**How AGC works:**
- ALB Controller runs in the cluster
- Watches **Gateway API** resources (Gateway, HTTPRoute)
- Calls Azure ServiceNetworking API to configure AGC
- AGC routes traffic **directly to Pod IPs** (not through Services)

### Comparison Table

| Aspect | AGIC (Old) | AGC (New) |
|--------|-----------|----------|
| **API Support** | Ingress API only | Ingress API + Gateway API |
| **Data Plane** | Classic Application Gateway | New containerized data plane |
| **Traffic Path** | Through AGIC pod + Service | Direct to Pod IPs |
| **Update Speed** | Slow (ARM API) | Fast (ServiceNetworking API) |
| **Target** | VM/VMSS backends | Container backends only |
| **Health Probing** | Service-level | Pod-level |
| **Traffic Management** | Basic | Advanced (splitting, header routing, gRPC) |
| **WAF Support** | Yes | Yes (added Nov 2025) |
| **Managed by** | You (update AGIC pod) | Azure (fully managed) |

---

## 15. Deployment Strategies: BYO vs Managed by ALB Controller

There are **two ways** to deploy Application Gateway for Containers:

### Strategy 1: Bring Your Own (BYO)

**You** create the AGC resource, Frontends, and Association in Azure using CLI/Portal/Bicep/Terraform. The ALB Controller then **references** the resource by ID.

**Workflow:**
```bash
# 1. Create AGC resource
az network alb create \
  --resource-group my-rg \
  --name agwc-alb

# 2. Create Frontend
az network alb frontend create \
  --resource-group my-rg \
  --alb-name agwc-alb \
  --frontend-name frontend-app

# 3. Create delegated subnet
az network vnet subnet create \
  --resource-group my-rg \
  --vnet-name aks-vnet \
  --name subnet-alb \
  --address-prefix 10.225.0.0/24 \
  --delegations Microsoft.ServiceNetworking/trafficControllers

# 4. Create Association
az network alb association create \
  --resource-group my-rg \
  --alb-name agwc-alb \
  --association-name association-app \
  --subnet /subscriptions/.../subnets/subnet-alb

# 5. In Kubernetes, reference the AGC by ID
```

**Gateway YAML:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-app
  annotations:
    alb.networking.azure.io/alb-id: /subscriptions/.../alb-resource-id
spec:
  gatewayClassName: azure-alb-external
  addresses:
    - type: alb.networking.azure.io/alb-frontend
      value: frontend-app
  listeners:
    - name: http
      port: 80
      protocol: HTTP
```

**Pros:**
- Full control over Azure resource lifecycle
- Fits well into existing IaC (Terraform/Bicep) pipelines
- Azure resources persist even if Kubernetes cluster is deleted

**Cons:**
- More manual setup
- You must manage Frontend/Association lifecycle

### Strategy 2: Managed by ALB Controller

You define an **ApplicationLoadBalancer** custom resource in Kubernetes. The ALB Controller creates and manages the Azure resources for you.

**Workflow:**
```yaml
# 1. Create delegated subnet (still manual)
az network vnet subnet create ...

# 2. Create ApplicationLoadBalancer CRD in Kubernetes
apiVersion: alb.networking.azure.io/v1
kind: ApplicationLoadBalancer
metadata:
  name: alb-managed
  namespace: alb-infra
spec:
  associations:
    - /subscriptions/.../subnets/subnet-alb
```

**What the ALB Controller does automatically:**
- Creates the AGC resource in Azure
- Creates a Frontend when you create a Gateway
- Deletes resources when you delete the ApplicationLoadBalancer

**Gateway YAML:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-app
  annotations:
    alb.networking.azure.io/alb-namespace: alb-infra
    alb.networking.azure.io/alb-name: alb-managed
spec:
  gatewayClassName: azure-alb-external
  listeners:
    - name: http
      port: 80
      protocol: HTTP
```

**Pros:**
- Simpler to get started (fewer manual steps)
- Kubernetes-native (resource lifecycle tied to K8s)

**Cons:**
- Azure resource lifecycle tied to Kubernetes resource (delete the CRD = delete AGC)
- Some teams prefer Azure resources managed outside the cluster

---

## 16. Gateway API Example Configuration

### Full Example: Simple HTTP Routing

**Step 1: Gateway (Platform Team)**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-01
  namespace: gateway-infra
  annotations:
    alb.networking.azure.io/alb-id: /subscriptions/.../agwc-alb
spec:
  gatewayClassName: azure-alb-external
  listeners:
    - name: http-listener
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
  addresses:
    - type: alb.networking.azure.io/alb-frontend
      value: frontend-app
```

**Step 2: HTTPRoute (App Team)**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httproute-app
  namespace: app-namespace
spec:
  parentRefs:
    - name: gateway-01
      namespace: gateway-infra
  hostnames:
    - "myapp.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: myapp-svc
          port: 80
```

**Step 3: Service (App Team)**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  namespace: app-namespace
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

**Result:**
```
User browses to: http://myapp.example.com (CNAME → Frontend FQDN)
                           │
                           ▼
       AGC Frontend receives request
                           │
                           ▼
       Gateway "gateway-01" listener matches (port 80, HTTP)
                           │
                           ▼
       HTTPRoute "httproute-app" matches (hostname + path)
                           │
                           ▼
       Traffic routed to Service "myapp-svc:80"
                           │
                           ▼
       AGC directly calls Pod IPs (10.244.x.x:8080)
```

### Advanced Example: Traffic Splitting (Canary Deployment)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httproute-canary
  namespace: app-namespace
spec:
  parentRefs:
    - name: gateway-01
      namespace: gateway-infra
  hostnames:
    - "myapp.example.com"
  rules:
    - backendRefs:
        - name: myapp-svc-v1
          port: 80
          weight: 90    # 90% of traffic to stable version
        - name: myapp-svc-v2
          port: 80
          weight: 10    # 10% of traffic to canary version
```

**No annotations needed!** Traffic splitting is a native Gateway API feature.

---

## 17. AGC Features and Benefits

### What AGC Offers (2024-2025)

| Feature | Description | Available Since |
|---------|-------------|----------------|
| **Gateway API Support** | Native implementation of Kubernetes Gateway API v1 | GA (Late 2024) |
| **Ingress API Support** | Also supports classic Ingress resources for migration | GA |
| **Traffic Splitting** | Weight-based routing for canary/blue-green deployments | GA |
| **Header-based Routing** | Route based on HTTP headers | GA |
| **gRPC Support** | Native gRPC routing | GA |
| **TLS Termination** | SSL/TLS offload with Azure Key Vault integration | GA |
| **WAF (Web Application Firewall)** | OWASP rule sets, custom rules, rate limiting | Nov 2025 (GA) |
| **Auto-scaling** | Elastic scaling based on traffic | GA |
| **Zone Redundancy** | Automatically deployed across Availability Zones | GA |
| **Direct Pod Routing** | Routes to Pod IPs, not Services (faster convergence) | GA |

### Benefits Over Self-Managed Ingress Controllers

**1. Fully Managed Data Plane**
- No pods consuming node resources for ingress
- No patching/upgrading ingress controller pods
- Azure handles all infrastructure scaling

**2. Faster Updates**
- Near real-time configuration propagation
- No pod restarts needed for config changes

**3. Better Observability**
- Azure Monitor integration
- Metrics, logs, and alerts built-in

**4. Enterprise Support**
- Microsoft support SLA
- File tickets via Azure Portal
- Engineering assistance

**5. Native Azure Integration**
- Azure Key Vault for TLS certificates
- Azure Monitor for logs/metrics
- Azure RBAC for access control

---

## 18. Prerequisites and Requirements

### AKS Cluster Requirements

**Network Plugin:**
- ✅ Azure CNI
- ✅ Azure CNI Overlay
- ❌ Kubenet (deprecated, will be retired in 2028)

**Workload Identity:**
- ✅ Must be enabled (for ALB Controller authentication)

**Kubernetes Version:**
- ✅ Supported AKS Kubernetes version (check AKS release notes)

### Regional Availability

AGC is available in **30+ Azure regions** (as of 2025). Check current availability:
```bash
az provider show \
  --namespace Microsoft.ServiceNetworking \
  --query "resourceTypes[?resourceType=='trafficControllers'].locations" \
  -o table
```

### Network Requirements

**Delegated Subnet:**
- CIDR: `/24` or smaller
- Must be delegated to `Microsoft.ServiceNetworking/trafficControllers`
- Cannot contain any other resources
- Can be in AKS VNet or a peered VNet

**Firewall Rules:**
If using Azure Firewall or NSGs, ensure outbound connectivity from:
- ALB Controller pods → Azure ServiceNetworking API (`management.azure.com`)
- ALB subnet → AKS node subnet (for health probes)

---

## 19. Migration from Ingress NGINX to AGC

### Microsoft's Official Migration Utility

Microsoft provides an open-source **Application Gateway for Containers Migration Utility** to help translate Ingress resources to Gateway API:

GitHub: `https://github.com/Azure/Application-Gateway-for-Containers-Migration-Utility`

**What it does:**
- Reads your existing Ingress YAML files
- Translates NGINX-specific annotations to Gateway API resources
- Generates Gateway, HTTPRoute, and AGC policy YAML

**Usage:**
```bash
# Install the utility
# (see GitHub repo for latest installation instructions)

# Analyze existing Ingress resources
./agc-migration analyze --provider nginx --ingress-class nginx ./manifests/

# Generate Gateway API YAML (BYO mode)
./agc-migration files \
  --provider nginx \
  --ingress-class nginx \
  --byo-resource-id $AGC_ID \
  --output-dir ./output \
  ./manifests/*.yaml

# Or generate for Managed mode
./agc-migration files \
  --provider nginx \
  --ingress-class nginx \
  --managed-subnet-id $SUBNET_ID \
  --output-dir ./output \
  ./manifests/*.yaml
```

**Review the generated YAML before applying!** The tool is a starting point, not a perfect 1:1 translation.

### Migration Steps

**1. Set up AGC and ALB Controller**
```bash
# Enable Gateway API on AKS
az aks update \
  --resource-group my-rg \
  --name my-cluster \
  --enable-gateway-api

# Enable AGC add-on
az rest \
  --method put \
  --uri "https://management.azure.com/.../aks-cluster-id?api-version=2025-09-02-preview" \
  --body '{
    "properties": {
      "ingressProfile": {
        "applicationLoadBalancer": { "enabled": true },
        "gatewayAPI": { "installation": "Standard" }
      }
    }
  }'
```

**2. Create AGC Azure Resources (if using BYO)**
```bash
# Create AGC, Frontend, Subnet, Association
# (see Deployment Strategies section)
```

**3. Run Migration Utility**
```bash
./agc-migration files --provider nginx --byo-resource-id $AGC_ID ...
```

**4. Review Generated YAML**
Check:
- Gateway references correct AGC resource ID
- HTTPRoutes reference correct Gateway
- Backend services match your actual service names
- TLS configuration (if using HTTPS)

**5. Apply in Non-Production First**
```bash
kubectl apply -f ./output/gateway.yaml
kubectl apply -f ./output/httproute.yaml
```

**6. Test Traffic**
```bash
# Get Frontend FQDN
az network alb frontend show ... --query fqdn -o tsv

# Test with curl
curl -H "Host: myapp.example.com" http://<frontend-fqdn>
```

**7. Update DNS**
```bash
# Create CNAME pointing your domain to AGC Frontend FQDN
# myapp.example.com CNAME <frontend-fqdn>
```

**8. Monitor and Validate**
```bash
# Check ALB Controller logs
kubectl logs -n kube-system deployment/alb-controller

# Check Gateway status
kubectl get gateway gateway-01 -o yaml

# Check HTTPRoute status
kubectl get httproute httproute-app -o yaml
```

**9. Remove Old Ingress Controller**
Once traffic is flowing through AGC and validated:
```bash
# Delete NGINX Ingress resources
kubectl delete ingress my-ingress

# Uninstall NGINX Ingress Controller
helm uninstall nginx-ingress -n ingress-nginx
```

---

## 20. Troubleshooting Common Issues

### Problem 1: GatewayClass Not Found

**Symptoms:**
```bash
kubectl get gatewayclass
# No resources found
```

**Cause:** ALB Controller bootstrap pod didn't install Gateway API CRDs

**Solution:**
```bash
# Check bootstrap pod logs
kubectl logs -n kube-system deployment/alb-controller-bootstrap

# Verify Gateway API CRDs are installed
kubectl get crds | grep gateway.networking.k8s.io

# If missing, manually install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

### Problem 2: Gateway Not Accepting Routes

**Symptoms:**
```bash
kubectl get gateway gateway-01 -o yaml
# Status: Accepted: False
```

**Cause:** Missing AGC resource ID or incorrect Frontend reference

**Solution:**
```bash
# Check Gateway annotations
kubectl describe gateway gateway-01

# Verify AGC resource exists
az network alb show --resource-group my-rg --name agwc-alb

# Verify Frontend exists
az network alb frontend list --resource-group my-rg --alb-name agwc-alb
```

### Problem 3: HTTPRoute Not Routing Traffic

**Symptoms:**
Gateway is Accepted, but traffic returns 404

**Cause:** HTTPRoute namespace mismatch or `allowedRoutes` blocking attachment

**Solution:**
```bash
# Check HTTPRoute status
kubectl get httproute httproute-app -o yaml

# Check if parentRef matches actual Gateway
# Check if Gateway's allowedRoutes permits this namespace

# Gateway must allow routes from HTTPRoute's namespace:
spec:
  listeners:
    - allowedRoutes:
        namespaces:
          from: All  # or Selector with matching labels
```

### Problem 4: "No Healthy Upstream" Error

**Symptoms:**
```bash
curl http://myapp.example.com
# 503 Service Unavailable
# no healthy upstream
```

**Cause:** AGC cannot reach Pod IPs (network policy, subnet config, or pods not ready)

**Solution:**
```bash
# Check Pod status
kubectl get pods -n app-namespace

# Check Pod IPs
kubectl get pods -n app-namespace -o wide

# Check AGC diagnostic logs (Azure Portal)
# Navigate to: AGC Resource → Diagnostic Settings → Logs

# Verify subnet delegation
az network vnet subnet show \
  --resource-group my-rg \
  --vnet-name aks-vnet \
  --name subnet-alb \
  --query delegations
```

### Problem 5: ALB Controller Authentication Failure

**Symptoms:**
```bash
kubectl logs -n kube-system deployment/alb-controller
# ERROR: failed to create Application Gateway for Containers
# ERROR: POST https://management.azure.com/.../trafficControllers
# 403 Forbidden
```

**Cause:** Managed Identity missing **AppGW for Containers Configuration Manager** role

**Solution:**
```bash
# Get Managed Identity principal ID
IDENTITY_PRINCIPAL_ID=$(az identity show \
  --name azure-alb-identity \
  --resource-group my-rg \
  --query principalId -o tsv)

# Grant role on resource group
az role assignment create \
  --role "faa91807-a699-40c5-ad60-ac7d6fd1e3fd" \  # AppGW for Containers Configuration Manager
  --assignee-object-id $IDENTITY_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --scope /subscriptions/.../resourceGroups/my-rg
```

---

## 21. Summary Comparison Tables

### Ingress API vs Gateway API

| Aspect | Ingress API | Gateway API |
|--------|------------|-------------|
| **Maturity** | GA since 2019 | GA since 2023 |
| **Development Status** | Frozen (no new features) | Active development |
| **Resource Model** | Single flat resource | Multi-resource (GatewayClass, Gateway, Route) |
| **Role Separation** | No | Yes (RBAC-friendly) |
| **Traffic Splitting** | Annotations only | Native support |
| **Header Routing** | Annotations only | Native support |
| **gRPC Support** | Via annotations | Native GRPCRoute |
| **TCP/UDP** | Not supported | Native TCPRoute/UDPRoute |
| **Vendor Portability** | Low (annotation hell) | High (standard API) |

### NGINX Ingress vs AGC

| Aspect | NGINX Ingress | Application Gateway for Containers |
|--------|--------------|-----------------------------------|
| **Deployment** | Pods in your cluster | Managed Azure service (outside cluster) |
| **Resource Usage** | Consumes node CPU/RAM | Zero cluster resources (data plane external) |
| **API Support** | Ingress only | Ingress + Gateway API |
| **Updates** | Manual (Helm upgrade) | Automatic (Azure-managed) |
| **Scaling** | Manual/HPA | Automatic elastic scaling |
| **WAF** | Requires separate deployment | Built-in (Nov 2025) |
| **Support** | Community (retiring March 2026) | Microsoft Enterprise SLA |
| **Cost** | Free (OSS) + node costs | Azure pricing (pay for data processed) |

### AGC vs AGIC

| Feature | AGIC | AGC |
|---------|------|-----|
| **Data Plane** | Classic Application Gateway | New containerized platform |
| **Target Backends** | VMs, VMSS, AKS | AKS only |
| **API Support** | Ingress only | Ingress + Gateway API |
| **Routing Target** | Service ClusterIP | Direct to Pod IPs |
| **Update Speed** | Slow (ARM API) | Fast (ServiceNetworking API) |
| **Management** | You manage App Gateway | Azure manages AGC |
| **Traffic Features** | Basic | Advanced (splitting, headers, gRPC) |

---

## References & Further Reading

- [What is Application Gateway for Containers? (Microsoft Docs)](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/overview)
- [Application Gateway for Containers Components](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/application-gateway-for-containers-components)
- [Quickstart: Deploy AGC ALB Controller (AKS Add-on)](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/quickstart-deploy-application-gateway-for-containers-alb-controller-addon)
- [Kubernetes Gateway API Official Docs](https://gateway-api.sigs.k8s.io/)
- [Gateway API GitHub](https://github.com/kubernetes-sigs/gateway-api)
- [AGC Migration Utility GitHub](https://github.com/Azure/Application-Gateway-for-Containers-Migration-Utility)
- [Ingress NGINX Retirement Announcement](https://blog.aks.azure.com/2025/11/13/ingress-nginx-update)
- [Managed Gateway API Installation (AKS)](https://learn.microsoft.com/en-us/azure/aks/managed-gateway-api)
- [AKS Ingress Concepts](https://learn.microsoft.com/en-us/azure/aks/concepts-network-ingress)