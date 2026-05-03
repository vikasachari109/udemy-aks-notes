# Exposing AKS Applications with Azure API Management (APIM)
### The Ultimate Beginner-Friendly Deep Dive

> **What this document covers:** What Azure API Management is, why you need it for AKS workloads, how APIM works with microservices, the external vs internal deployment modes, VNet integration, end-to-end architecture with NGINX Ingress, subscription keys, policies, and complete setup workflows — all explained from absolute scratch.

---

## Table of Contents
1. [What is Azure API Management (APIM)?](#1-what-is-azure-api-management-apim)
2. [Why Use APIM with AKS?](#2-why-use-apim-with-aks)
3. [APIM Core Concepts Explained](#3-apim-core-concepts-explained)
4. [APIM Tiers and Pricing](#4-apim-tiers-and-pricing)
5. [APIM Deployment Modes: External vs Internal](#5-apim-deployment-modes-external-vs-internal)
6. [Architecture: APIM + AKS + NGINX Ingress](#6-architecture-apim--aks--nginx-ingress)
7. [How Traffic Flows Through APIM to AKS](#7-how-traffic-flows-through-apim-to-aks)
8. [VNet Integration and Private Connectivity](#8-vnet-integration-and-private-connectivity)
9. [Setting Up NGINX Ingress Controller in AKS](#9-setting-up-nginx-ingress-controller-in-aks)
10. [Configuring APIM to Connect to AKS](#10-configuring-apim-to-connect-to-aks)
11. [Subscription Keys and API Security](#11-subscription-keys-and-api-security)
12. [APIM Policies Explained](#12-apim-policies-explained)
13. [End-to-End TLS/SSL Configuration](#13-end-to-end-tlsssl-configuration)
14. [Advanced Pattern: App Gateway + APIM + AKS](#14-advanced-pattern-app-gateway--apim--aks)
15. [Monitoring and Troubleshooting](#15-monitoring-and-troubleshooting)
16. [Best Practices and Recommendations](#16-best-practices-and-recommendations)

---

## 1. What is Azure API Management (APIM)?

### The Elevator Pitch
**Azure API Management (APIM)** is a fully managed service that acts as a **facade** (front door) for your backend APIs. It sits between your API consumers (mobile apps, web apps, partners) and your backend services (microservices running in AKS, Azure Functions, on-premises APIs, third-party APIs).

Think of APIM as a **smart gateway** that:
- Accepts API requests from clients
- Enforces security policies (API keys, OAuth, JWT validation)
- Transforms requests and responses (add headers, change payloads)
- Throttles/rate-limits requests
- Caches responses
- Routes requests to the correct backend
- Logs and monitors all traffic

### The Problem APIM Solves

Imagine you have 20 microservices running in AKS. Each one exposes an API endpoint. Without APIM:

1. **Security nightmare**: Every microservice must implement its own authentication, authorization, rate limiting
2. **No central control**: Changing a security policy means updating 20 services
3. **Poor developer experience**: External developers need to learn 20 different API endpoints, authentication methods
4. **No visibility**: You can't see who's calling what, how often, or when errors occur
5. **Versioning hell**: How do you roll out V2 of an API without breaking V1 clients?

**With APIM**, all these cross-cutting concerns move to one central place. Your microservices focus on business logic. APIM handles everything else.

### The Three Main Components of APIM

```
┌─────────────────────────────────────────────────────┐
│         Azure API Management (APIM)                 │
│                                                     │
│  ┌────────────────┐  ┌────────────────┐  ┌────────┐│
│  │ Developer       │  │ API Gateway    │  │ Mgmt   ││
│  │ Portal          │  │ (Runtime)      │  │ Portal ││
│  │                 │  │                │  │        ││
│  │ Discover APIs   │  │ Request        │  │ Config ││
│  │ Get API keys    │  │ handling       │  │ APIs   ││
│  │ Read docs       │  │ Policy exec.   │  │ Users  ││
│  │ Test APIs       │  │ Auth/Authz     │  │ Monitor││
│  └────────────────┘  └────────────────┘  └────────┘│
└─────────────────────────────────────────────────────┘
```

**1. Developer Portal**
A self-service website where external/internal developers can:
- Discover available APIs
- Read API documentation
- Subscribe to APIs and get subscription keys
- Test APIs interactively
- View usage analytics

**2. API Gateway (The Runtime Engine)**
The actual proxy that receives all API calls. It:
- Validates API keys/tokens
- Executes policies (rate limiting, transformation, caching)
- Routes requests to backend services
- Returns responses to clients

**3. Management Portal (Azure Portal)**
The admin interface where you:
- Define APIs (endpoints, operations, request/response schemas)
- Configure policies
- Manage users and subscriptions
- View logs and metrics

---

## 2. Why Use APIM with AKS?

### The Microservices Challenge

When you deploy microservices to AKS, you typically have:
- 10-100+ services
- Each service has its own endpoints
- Services communicate with each other
- External clients (web/mobile apps) need to call these services

**Without APIM**, you expose each service directly:
```
Mobile App → Service A (exposed via LoadBalancer)
Web App    → Service B (exposed via LoadBalancer)
Partner    → Service C (exposed via Ingress)
```

**Problems:**
- Each service needs public IP or Ingress rule
- Authentication logic duplicated in every service
- No unified rate limiting
- No API versioning strategy
- No developer portal
- No centralized logging

**With APIM**, you expose ONE API Gateway:
```
Mobile App  ┐
Web App     ├─→ APIM Gateway ─→ AKS (all services private)
Partner API ┘
```

**Benefits:**
- ✅ Single public endpoint
- ✅ Centralized security (API keys, OAuth, JWT)
- ✅ API versioning and lifecycle management
- ✅ Rate limiting and quotas
- ✅ Request/response transformation
- ✅ Caching
- ✅ Developer portal for API discovery
- ✅ Unified monitoring and analytics

---

## 3. APIM Core Concepts Explained

### API
An **API** in APIM is a collection of operations (endpoints). Example:
```
User API:
  GET  /users        - List all users
  POST /users        - Create user
  GET  /users/{id}   - Get user by ID
  PUT  /users/{id}   - Update user
```

### Operation
An **operation** is a single HTTP method + path combination. Each operation can have its own policies.

### Product
A **product** is a bundle of APIs that you publish to developers. Products have:
- **Visibility**: Public (anyone can see) or Private (requires admin approval)
- **Subscription requirement**: Does a developer need an API key?
- **Usage quotas**: How many calls per hour/day?

Example products:
- **Starter**: Free tier, 1000 calls/day, access to basic APIs
- **Premium**: Paid tier, 100,000 calls/day, access to all APIs

### Subscription
A **subscription** is a relationship between a user/app and a product. When you subscribe, you get:
- **Primary key**: A secret string used in API requests
- **Secondary key**: For key rotation without downtime

Requests include the key in a header:
```
Ocp-Apim-Subscription-Key: your-subscription-key-here
```

### Policy
A **policy** is an XML configuration that executes during request/response processing. Policies can:
- Validate JWT tokens
- Rate-limit requests
- Transform JSON to XML
- Cache responses
- Add CORS headers
- Rewrite URLs

Policies run in 4 stages:
```
1. INBOUND   → Before request reaches backend
2. BACKEND   → Controls how backend is called
3. OUTBOUND  → After backend responds, before client receives
4. ON-ERROR  → If an error occurs
```

### Backend
A **backend** is the actual service that implements the API logic. In the AKS context, this is:
- A Kubernetes Service (ClusterIP or LoadBalancer)
- Exposed via NGINX Ingress Controller
- Could be HTTP or HTTPS

---

## 4. APIM Tiers and Pricing

Azure API Management has **5 tiers**:

| Tier | VNet Support | Use Case | Price Approx. |
|------|-------------|----------|---------------|
| **Consumption** | ❌ No | Serverless, pay-per-call, auto-scale | $0.035 per 10K calls |
| **Basic** | ❌ No | Dev/test, low traffic | ~$50/month |
| **Standard** | ❌ No | Production, moderate traffic | ~$750/month |
| **Premium** | ✅ Yes (External + Internal) | Enterprise, multi-region, VNet injection | ~$2,800/month |
| **Developer** | ✅ Yes (External only) | Non-production, testing VNet integration | ~$50/month |

### VNet Integration Requirement

**⚠️ Critical**: To connect APIM to a **private AKS cluster** (where services are only accessible inside a VNet), you MUST use:
- **Developer tier** (for testing)
- **Premium tier** (for production)

Basic and Standard tiers **cannot** be injected into a VNet, so they can only call AKS services exposed via **public** LoadBalancer IPs or **public** Ingress controllers.

---

## 5. APIM Deployment Modes: External vs Internal

When you deploy APIM into a VNet, you choose a mode:

### External Mode

```
                  Internet
                     │
                     ▼
         ┌───────────────────────┐
         │  APIM Gateway         │
         │  Public IP: x.x.x.x   │ ← Accessible from internet
         │  Private IP: 10.0.1.5 │ ← Inside VNet
         └──────────┬────────────┘
                    │ VNet
         ┌──────────▼────────────┐
         │  AKS Private Services │
         │  (No public IPs)      │
         └───────────────────────┘
```

**External mode** means:
- APIM Gateway gets BOTH a public IP and a private IP
- API Gateway is accessible from the internet via public IP
- API Gateway can reach private AKS services via private IP
- Developer Portal is also publicly accessible

**When to use:**
- You want external clients (mobile apps, partners, web apps) to call your APIs
- Your AKS services are private (no public IPs)
- You want APIM to protect your private backend

### Internal Mode

```
                  Internet
                     │
                     ▼
         ┌───────────────────────┐
         │  App Gateway / AFD    │ ← Public endpoint
         └──────────┬────────────┘
                    │ VNet
         ┌──────────▼────────────┐
         │  APIM Gateway         │
         │  NO Public IP         │ ← Only private IP: 10.0.1.5
         │  Private IP only      │ ← Accessible only inside VNet
         └──────────┬────────────┘
                    │
         ┌──────────▼────────────┐
         │  AKS Private Services │
         └───────────────────────┘
```

**Internal mode** means:
- APIM Gateway has ONLY a private IP (e.g., `10.0.1.5`)
- API Gateway is NOT accessible from the internet directly
- You must put another service in front (App Gateway, Front Door, VPN) to reach APIM
- Developer Portal is also private (only accessible inside VNet or via VPN)

**When to use:**
- You want maximum security (no direct internet exposure)
- You use App Gateway or Front Door as the public entry point (with WAF)
- Internal APIs only (employee portals, B2B partners via VPN)

---

## 6. Architecture: APIM + AKS + NGINX Ingress

### The Full Picture

```
┌──────────────────────────────────────────────────────────────┐
│                        Internet                              │
│                           │                                  │
│                           ▼                                  │
│              ┌─────────────────────────┐                     │
│              │  Azure API Management   │                     │
│              │  (External Mode)        │                     │
│              │  Public IP: 20.x.x.x    │                     │
│              │  Private IP: 10.0.2.10  │                     │
│              └────────────┬────────────┘                     │
│                           │                                  │
│                           │ HTTPS (over VNet)                │
│                           │                                  │
│              ┌────────────▼────────────┐                     │
│              │  NGINX Ingress         │                     │
│              │  Controller            │                     │
│              │  Private IP: 10.0.3.50 │                     │
│              │  (Internal LB)         │                     │
│              └────────────┬────────────┘                     │
│                           │                                  │
│              ┌────────────┴────────────┐                     │
│              │                         │                     │
│         ┌────▼─────┐            ┌─────▼──────┐              │
│         │ Service A│            │ Service B  │              │
│         │ Pod IPs  │            │ Pod IPs    │              │
│         └──────────┘            └────────────┘              │
│                                                              │
│                 AKS Cluster (Private)                        │
└──────────────────────────────────────────────────────────────┘
```

### Component Breakdown

**1. Client (Mobile/Web App)**
Makes API calls to APIM's public endpoint:
```
GET https://api.mycompany.com/users
Ocp-Apim-Subscription-Key: abc123...
```

**2. Azure API Management Gateway**
- Receives the request on public IP `20.x.x.x`
- Validates the subscription key
- Executes inbound policies (rate limit, transform, cache check)
- Forwards request to backend: `https://aks-ingress.mycompany.internal/users`

**3. NGINX Ingress Controller**
- Runs inside AKS as a Deployment (pods)
- Exposed via an **Internal Load Balancer** (private IP `10.0.3.50`)
- Receives request from APIM over the VNet
- Routes to the correct Service based on hostname/path

**4. Kubernetes Service**
- A ClusterIP Service (e.g., `user-service.default.svc.cluster.local`)
- Routes traffic to backend pods

**5. Backend Pods**
- The actual microservice containers
- Process the request and return response

**Response flows back:**
```
Pod → Service → Ingress → APIM (outbound policies) → Client
```

---

## 7. How Traffic Flows Through APIM to AKS

### Request Flow — Step by Step

```
STEP 1: Client sends request
  →  POST https://api.mycompany.com/users
     Headers:
       Ocp-Apim-Subscription-Key: abc123xyz
       Content-Type: application/json
     Body:
       { "name": "Alice", "email": "alice@example.com" }

STEP 2: APIM receives request on public IP
  →  DNS resolves api.mycompany.com to 20.50.100.200 (APIM public IP)
  →  Request hits APIM Gateway

STEP 3: APIM validates subscription key
  →  Checks if "abc123xyz" is valid
  →  Checks if the key is for a product that includes the Users API
  →  If invalid → Return 401 Unauthorized

STEP 4: APIM executes INBOUND policies
  →  Rate limit check: Has this key exceeded 1000 calls/hour?
  →  Transform: Add custom header "X-Source: APIM"
  →  Cache check: Is this response cached? (If yes, return from cache)

STEP 5: APIM forwards to backend
  →  Backend URL configured in APIM: https://10.0.3.50/api/users
        (This is the NGINX Ingress internal IP)
  →  APIM makes HTTPS request over the VNet to 10.0.3.50

STEP 6: NGINX Ingress Controller receives request
  →  Ingress resource matches: host=api-internal.mycompany.com, path=/api/users
  →  Routes to Service: user-service.default.svc.cluster.local:80

STEP 7: Kubernetes Service routes to Pod
  →  Service selects a healthy pod (round-robin or least connections)
  →  Forwards request to pod IP: 10.244.2.15:8080

STEP 8: Pod processes request
  →  Microservice creates user in database
  →  Returns response:
       HTTP/1.1 201 Created
       { "id": "12345", "name": "Alice", "email": "alice@example.com" }

STEP 9: Response flows back through Ingress
  →  Pod → Service → Ingress Controller

STEP 10: Ingress returns response to APIM

STEP 11: APIM executes OUTBOUND policies
  →  Transform response: Remove sensitive fields
  →  Cache response for 60 seconds
  →  Add header: "X-Powered-By: Azure APIM"

STEP 12: APIM returns response to client
  →  HTTP/1.1 201 Created
     { "id": "12345", "name": "Alice", "email": "alice@example.com" }
```

---

## 8. VNet Integration and Private Connectivity

### Network Prerequisites

For APIM to reach AKS services privately, both must be in the same VNet or in peered VNets.

**Typical Setup:**

```
VNet: 10.0.0.0/16
  ├── Subnet: APIM-Subnet (10.0.2.0/24)      ← APIM injected here
  ├── Subnet: AKS-System-Subnet (10.0.3.0/24) ← AKS system nodes
  ├── Subnet: AKS-User-Subnet (10.0.4.0/24)   ← AKS user nodes
  └── Subnet: Ingress-Subnet (10.0.5.0/24)    ← Internal LB for Ingress
```

### APIM Subnet Requirements

When you deploy APIM into a VNet (Premium/Developer tier), you must:

1. **Create a dedicated subnet** for APIM (minimum `/27` — 32 IPs, but `/24` recommended)
2. **Delegate the subnet** to `Microsoft.ApiManagement/service`
3. **Network Security Group (NSG)** rules:
   - Allow inbound 443 from Internet (for API Gateway)
   - Allow inbound 3443 from ApiManagement service tag (management endpoint)
   - Allow outbound to Storage, SQL, EventHub, Azure Monitor

### Service Principal Permissions

APIM's managed identity or service principal needs:
- **Network Contributor** role on the subnet it's injected into
- **Reader** role on the VNet

---

## 9. Setting Up NGINX Ingress Controller in AKS

### Why NGINX Ingress?

NGINX Ingress Controller acts as the **entry point into AKS** for HTTP/HTTPS traffic. It:
- Listens on a LoadBalancer IP (public or private)
- Routes incoming requests to the correct Kubernetes Service based on hostname/path
- Handles TLS termination
- Supports advanced routing (rewrites, redirects, authentication)

### Installation — Managed (AKS App Routing Add-on)

AKS offers a **managed NGINX Ingress** via the App Routing add-on:

```bash
# Enable App Routing addon
az aks enable-addons \
  --resource-group my-rg \
  --name my-aks-cluster \
  --addons app-routing

# This installs:
# - NGINX Ingress Controller
# - External-DNS (optional)
# - Cert-Manager (optional)
```

### Installation — Unmanaged (Helm)

For more control, install NGINX via Helm:

```bash
# Add NGINX Helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX with INTERNAL Load Balancer
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"="true" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal-subnet"="ingress-subnet"
```

**Key annotation:**
```yaml
service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```
This creates an **Internal Load Balancer** with a private IP instead of a public IP.

### Verify Ingress Controller

```bash
kubectl get svc -n ingress-nginx
# NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)
# nginx-ingress-controller             LoadBalancer   10.0.100.50    10.0.5.10     80:30080/TCP,443:30443/TCP
```

The `EXTERNAL-IP` is now a **private IP** (`10.0.5.10`) — this is the IP APIM will call.

---

## 10. Configuring APIM to Connect to AKS

### Step 1: Deploy a Sample Application

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-api
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-api
  template:
    metadata:
      labels:
        app: user-api
    spec:
      containers:
      - name: api
        image: myacr.azurecr.io/user-api:v1
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: user-api-service
  namespace: default
spec:
  selector:
    app: user-api
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP  # Internal only
```

```bash
kubectl apply -f deployment.yaml
```

### Step 2: Create Ingress Resource

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: user-api-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: api-internal.mycompany.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-api-service
            port:
              number: 80
```

```bash
kubectl apply -f ingress.yaml
```

### Step 3: Configure APIM API

**In Azure Portal:**

1. Navigate to your APIM instance
2. **APIs** → **+ Add API** → **HTTP**
3. **Display name**: User API
4. **Name**: user-api
5. **Web service URL**: `https://10.0.5.10` (NGINX Ingress private IP)
   - OR use a custom domain if you've configured DNS
6. **API URL suffix**: `users`

**Add an operation:**
- **Display name**: Get Users
- **URL**: `GET /`
- **Description**: Retrieve all users

**Backend configuration:**
- Click on the GET operation
- Go to **Backend** section
- Set **HTTP(s) endpoint** to `https://10.0.5.10/users`

### Step 4: Test the API from APIM

1. Go to **APIs** → **User API** → **Test** tab
2. Select the GET operation
3. Click **Send**

You should see a response from your AKS-hosted API!

### Step 5: Add API to a Product

1. Go to **Products** → **+ Add**
2. **Display name**: Standard
3. **Id**: standard
4. **Description**: Standard API access
5. **Requires subscription**: Yes
6. **APIs**: Select "User API"
7. Click **Create**

---

## 11. Subscription Keys and API Security

### How Subscription Keys Work

When a user subscribes to a product, APIM generates two keys:
- **Primary key**: `abc123def456...`
- **Secondary key**: `xyz789ghi012...`

**API calls must include this key:**

```bash
curl -X GET https://api.mycompany.com/users \
  -H "Ocp-Apim-Subscription-Key: abc123def456..."
```

**Without the key:**
```
HTTP/1.1 401 Unauthorized
{ "error": "Access denied due to missing subscription key" }
```

### Key Rotation

To rotate keys without downtime:
1. Regenerate the **secondary key**
2. Update client applications to use the new secondary key
3. Once all clients are updated, regenerate the **primary key**
4. Update remaining clients

### Other Authentication Methods

APIM supports:
- **OAuth 2.0** (validate access tokens)
- **JWT validation** (validate JSON Web Tokens)
- **Client certificates** (mutual TLS)
- **IP filtering** (whitelist specific IPs)
- **Basic authentication** (username/password)

---

## 12. APIM Policies Explained

### Policy Structure

Policies are XML configurations that run at different stages:

```xml
<policies>
  <inbound>
    <!-- Runs BEFORE request is sent to backend -->
  </inbound>
  <backend>
    <!-- Controls HOW backend is called -->
  </backend>
  <outbound>
    <!-- Runs AFTER backend responds -->
  </outbound>
  <on-error>
    <!-- Runs if an error occurs -->
  </on-error>
</policies>
```

### Common Policy Examples

#### Rate Limiting (Throttling)

```xml
<inbound>
  <rate-limit calls="100" renewal-period="60" />
  <!-- Max 100 calls per 60 seconds -->
</inbound>
```

#### JWT Validation

```xml
<inbound>
  <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
    <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration" />
    <required-claims>
      <claim name="aud" match="all">
        <value>api://myapp</value>
      </claim>
    </required-claims>
  </validate-jwt>
</inbound>
```

#### CORS

```xml
<inbound>
  <cors allow-credentials="true">
    <allowed-origins>
      <origin>https://myapp.com</origin>
    </allowed-origins>
    <allowed-methods>
      <method>GET</method>
      <method>POST</method>
    </allowed-methods>
  </cors>
</inbound>
```

#### Caching

```xml
<inbound>
  <cache-lookup vary-by-developer="false" vary-by-developer-groups="false" />
</inbound>
<outbound>
  <cache-store duration="3600" />  <!-- Cache for 1 hour -->
</outbound>
```

#### Request Transformation

```xml
<inbound>
  <set-header name="X-Source" exists-action="override">
    <value>APIM</value>
  </set-header>
  <rewrite-uri template="/api/v2{uri}" />
</inbound>
```

---

## 13. End-to-End TLS/SSL Configuration

### Why End-to-End TLS?

**End-to-end TLS** means:
```
Client → [HTTPS] → APIM → [HTTPS] → NGINX Ingress → [HTTP/HTTPS] → Pod
```

All hops are encrypted. This prevents:
- Man-in-the-middle attacks
- Eavesdropping on VNet traffic
- Compliance violations (PCI-DSS, HIPAA require encryption in transit)

### TLS Certificate Requirements

You need **two certificates**:

1. **APIM Custom Domain Certificate**
   - Domain: `api.mycompany.com`
   - Used by clients to connect to APIM

2. **NGINX Ingress Certificate**
   - Domain: `api-internal.mycompany.com`
   - Used by APIM to connect to NGINX

### Step 1: Store Certificates in Azure Key Vault

```bash
# Create Key Vault
az keyvault create \
  --name my-keyvault \
  --resource-group my-rg \
  --location westeurope

# Upload APIM certificate
az keyvault certificate import \
  --vault-name my-keyvault \
  --name apim-cert \
  --file apim-cert.pfx \
  --password "cert-password"

# Upload Ingress certificate
az keyvault certificate import \
  --vault-name my-keyvault \
  --name ingress-cert \
  --file ingress-cert.pfx \
  --password "cert-password"
```

### Step 2: Configure APIM Custom Domain

```bash
az apim update \
  --name my-apim \
  --resource-group my-rg \
  --set \
    hostnameConfigurations[0].type=Proxy \
    hostnameConfigurations[0].hostName=api.mycompany.com \
    hostnameConfigurations[0].keyVaultId="https://my-keyvault.vault.azure.net/secrets/apim-cert"
```

### Step 3: Configure NGINX Ingress with TLS

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ingress-tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: user-api-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api-internal.mycompany.com
    secretName: ingress-tls-secret
  rules:
  - host: api-internal.mycompany.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-api-service
            port:
              number: 80
```

### Step 4: Configure APIM to Use HTTPS Backend

In APIM API settings:
- Backend URL: `https://api-internal.mycompany.com/users`

Add policy to validate backend certificate:

```xml
<inbound>
  <set-backend-service base-url="https://api-internal.mycompany.com" />
</inbound>
```

---

## 14. Advanced Pattern: App Gateway + APIM + AKS

### The Three-Tier Architecture

```
Internet
   │
   ▼
┌──────────────────────┐
│ App Gateway (WAF)    │ ← Public IP, WAF protection
│ Public IP: 20.x.x.x  │
└──────────┬───────────┘
           │ VNet (HTTPS)
           ▼
┌──────────────────────┐
│ APIM (Internal Mode) │ ← Private IP only, API policies
│ Private IP: 10.0.2.5 │
└──────────┬───────────┘
           │ VNet (HTTPS)
           ▼
┌──────────────────────┐
│ NGINX Ingress        │ ← Internal LB, routing
│ Private IP: 10.0.5.10│
└──────────┬───────────┘
           │
           ▼
     AKS Pods (Services)
```

### Why This Pattern?

**App Gateway** provides:
- **WAF (Web Application Firewall)** — protects against OWASP Top 10, SQL injection, XSS
- **SSL offloading** — handles TLS termination
- **Public endpoint** — internet-facing

**APIM** provides:
- **API management** — versioning, policies, developer portal
- **Authentication** — API keys, OAuth, JWT
- **Transformation** — request/response modification

**NGINX Ingress** provides:
- **Kubernetes-native routing** — maps to Services
- **Path-based routing** — `/users` → user-service, `/orders` → order-service

### Traffic Flow

```
Client → App Gateway (WAF check, TLS) 
  → APIM (API key validation, rate limit, transform)
    → NGINX Ingress (route to service)
      → Pod
```

---

## 15. Monitoring and Troubleshooting

### APIM Metrics to Monitor

In Azure Monitor, track:
- **Total Gateway Requests** — overall traffic volume
- **Successful Gateway Requests** — healthy traffic
- **Failed Gateway Requests** — errors (4xx, 5xx)
- **Requests Latency** — response time (P50, P95, P99)
- **Capacity** — how close you are to hitting scale limits

### Common Issues

#### Issue 1: 401 Unauthorized

**Symptom:**
```
{ "error": "Access denied due to missing subscription key" }
```

**Causes:**
- Subscription key not included in request
- Key is invalid/expired
- API not added to the subscribed product

**Solution:**
Check header:
```bash
curl -H "Ocp-Apim-Subscription-Key: your-key-here" ...
```

#### Issue 2: 502 Bad Gateway

**Symptom:**
```
502 Bad Gateway
The server was acting as a gateway or proxy and received an invalid response from the upstream server.
```

**Causes:**
- APIM can't reach AKS backend
- NGINX Ingress is down
- Backend service is down

**Solution:**
```bash
# Check NGINX Ingress is running
kubectl get pods -n ingress-nginx

# Check service is healthy
kubectl get svc user-api-service

# Test direct connection from APIM to Ingress
# (exec into APIM's VNet or use a test VM)
curl https://10.0.5.10/users
```

#### Issue 3: Slow Response Times

**Symptom:** Requests take 5-10 seconds

**Causes:**
- Backend pods are slow
- No caching enabled
- Network latency

**Solution:**
- Add caching policy
- Scale backend pods
- Check pod resource limits (CPU/memory)

---

## 16. Best Practices and Recommendations

### Security

✅ **Use Premium tier in production** — VNet injection, multi-region support, higher SLA

✅ **Enable HTTPS everywhere** — Client→APIM→NGINX→Pod (end-to-end TLS)

✅ **Use Managed Identity** — For Key Vault access, avoid storing secrets in code

✅ **Implement rate limiting** — Protect backend from overload

✅ **Validate JWT tokens** — For OAuth/OIDC flows

✅ **Use Internal Mode with App Gateway** — Maximum security (APIM not directly exposed)

### Performance

✅ **Enable caching** — Cache GET responses for 60-300 seconds

✅ **Use multiple regions** (Premium tier) — Deploy APIM in multiple Azure regions for low latency

✅ **Monitor capacity** — Scale APIM units before hitting 80% capacity

### Operations

✅ **Use products to organize APIs** — Group related APIs into products (Starter, Premium, Internal)

✅ **Version your APIs** — Use version sets (`/v1/users`, `/v2/users`)

✅ **Enable Application Insights** — Send logs and metrics to App Insights for deep analysis

✅ **Test policies in the Azure Portal** — Use the Test tab before deploying

✅ **Document APIs** — Use OpenAPI (Swagger) specs, publish to Developer Portal

---

## References & Further Reading

- [Azure API Management Documentation (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/api-management/)
- [Use API Management with AKS Microservices (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/api-management/api-management-kubernetes)
- [APIM VNet Integration (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet)
- [APIM Policies Reference](https://learn.microsoft.com/en-us/azure/api-management/api-management-policies)
- [NGINX Ingress Controller Documentation](https://kubernetes.github.io/ingress-nginx/)