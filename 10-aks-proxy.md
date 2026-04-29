# 📘 Doc 10 — AKS with HTTP Proxy: Complete Deep Dive

> **Goal:** Understand what an HTTP Proxy is at every level — why it exists, how AKS integrates with it, how environment variables are injected, how SSL inspection works, how to bypass the proxy per-pod, how to update proxy settings, and how to set up a real proxy for testing.

---

## 🧭 Table of Contents

1. [What Is an HTTP Proxy?](#1-what-is-an-http-proxy)
2. [Why Would You Use HTTP Proxy with AKS?](#2-why-would-you-use-http-proxy-with-aks)
3. [HTTP Proxy vs Other Egress Methods](#3-http-proxy-vs-other-egress-methods)
4. [AKS HTTP Proxy Architecture](#4-aks-http-proxy-architecture)
5. [The Proxy Configuration JSON File](#5-the-proxy-configuration-json-file)
6. [SSL/TLS Inspection and the `trustedCA` Field](#6-ssltls-inspection-and-the-trustedca-field)
7. [Creating a New Cluster with HTTP Proxy](#7-creating-a-new-cluster-with-http-proxy)
8. [How AKS Injects Proxy Settings — The Mutating Webhook](#8-how-aks-injects-proxy-settings--the-mutating-webhook)
9. [Environment Variables Injected Into Pods](#9-environment-variables-injected-into-pods)
10. [AKS-Managed No-Proxy Entries](#10-aks-managed-no-proxy-entries)
11. [Bypassing the Proxy for Specific Pods](#11-bypassing-the-proxy-for-specific-pods)
12. [Updating Proxy Settings on an Existing Cluster](#12-updating-proxy-settings-on-an-existing-cluster)
13. [Proxy Settings at the Node Level (containerd)](#13-proxy-settings-at-the-node-level-containerd)
14. [What Cannot Be Changed After Cluster Creation](#14-what-cannot-be-changed-after-cluster-creation)
15. [UDR Firewall vs HTTP Proxy — Side by Side](#15-udr-firewall-vs-http-proxy--side-by-side)
16. [Setting Up mitmproxy for Testing](#16-setting-up-mitmproxy-for-testing)
17. [Generating a Custom CA Certificate for the Proxy](#17-generating-a-custom-ca-certificate-for-the-proxy)
18. [Troubleshooting Proxy Issues](#18-troubleshooting-proxy-issues)
19. [Common Application Behaviors with Proxy](#19-common-application-behaviors-with-proxy)

---

## 1. What Is an HTTP Proxy?

An **HTTP proxy** (also called a "web proxy" or "forward proxy") is a server that sits between your application and the internet. Instead of your application connecting directly to a destination, it connects to the proxy, which then forwards the request on behalf of your application.

### Without a Proxy

```
AKS Pod ─────────────────────────────────────────────► api.github.com
         Direct TCP connection
         Pod's egress IP visible to GitHub
```

### With a Proxy

```
AKS Pod ──► HTTP Proxy Server ────────────────────────► api.github.com
            Pod connects here   Proxy connects here
            (proxy private IP)  (proxy's public IP)
```

### The Proxy as an Intermediary

When your application has proxy environment variables set (e.g., `http_proxy=http://10.0.0.4:8080`), it works like this for HTTP:

1. Your app reads the `http_proxy` variable
2. Instead of resolving `api.github.com` and connecting directly, your app connects to `10.0.0.4:8080`
3. Your app sends an HTTP request with header `Host: api.github.com` to the proxy
4. The proxy reads the `Host` header, resolves `api.github.com`, connects to it, and forwards the request
5. The proxy receives the response and forwards it back to your app

For HTTPS (via the `https_proxy` variable):
- Your app connects to the proxy with an HTTP `CONNECT` tunnel request: `CONNECT api.github.com:443 HTTP/1.1`
- The proxy establishes a TCP tunnel to `api.github.com:443`
- Your app then performs TLS handshake through the tunnel
- The proxy just passes encrypted bytes back and forth (it can't see HTTPS content unless doing SSL inspection)

---

## 2. Why Would You Use HTTP Proxy with AKS?

HTTP proxies serve specific needs that network-level solutions (NAT Gateway, Load Balancer) can't address:

### Reason 1: Corporate IT Policy

Many large organizations require ALL internet traffic from internal networks to pass through a centrally managed proxy server. This is a decades-old IT policy enforced through network controls. When AKS is deployed in such environments, pods MUST use the proxy — otherwise their traffic is blocked at the network level.

### Reason 2: Compliance and Audit Logging

HTTP proxies can log every HTTP/HTTPS request with:
- Full URL (not just IP/port — the actual path like `/api/v1/charge`)
- HTTP method (GET, POST, etc.)
- Response codes
- Bytes transferred
- Timestamp
- Source IP (which pod made the request)

This level of logging is required in regulated industries (PCI-DSS for payments, HIPAA for healthcare, etc.).

### Reason 3: SSL/TLS Inspection (Deep Packet Inspection)

A corporate proxy can **decrypt and re-encrypt HTTPS traffic** to inspect its contents. This is called SSL inspection, TLS interception, or "man-in-the-middle" proxy:

1. App connects to proxy, establishes TLS with **proxy's certificate**
2. Proxy decrypts the traffic and inspects it (scans for malware, data exfiltration, policy violations)
3. Proxy establishes a NEW TLS connection to the actual destination
4. Proxy re-encrypts and forwards

NAT Gateway and Azure Firewall cannot do this. Only application-layer proxies can.

### Reason 4: Caching

Proxies like Squid cache HTTP responses. If 100 pods pull the same Docker layer or the same npm package, the proxy downloads it once and serves it from cache for the other 99. This saves bandwidth and speeds up deployments.

### Reason 5: Authentication to External Services

Some enterprises route all traffic through a proxy that adds authentication headers to outbound requests. Pods don't need individual credentials — the proxy handles authentication centrally.

---

## 3. HTTP Proxy vs Other Egress Methods

### Functional Comparison

| Feature | Load Balancer | NAT Gateway | Azure Firewall (UDR) | HTTP Proxy |
|---------|--------------|-------------|---------------------|------------|
| Network layer | L4 (TCP/UDP) | L4 (TCP/UDP) | L3–L7 | L7 (HTTP/S) |
| Sees HTTP URLs | ❌ | ❌ | ✅ (FQDN filter) | ✅ (full URL) |
| SSL inspection | ❌ | ❌ | ✅ | ✅ |
| HTTP caching | ❌ | ❌ | ❌ | ✅ |
| Applies to | ALL traffic | ALL traffic | ALL traffic | Only apps that read `http_proxy` |
| Works for non-HTTP | ✅ | ✅ | ✅ | ❌ (HTTP/HTTPS only) |
| Per-request logging | ❌ | ❌ | ✅ | ✅ (full URL logs) |
| Enterprise compliance | Sometimes | Sometimes | Usually | Always |
| AKS configures nodes | N/A | N/A | N/A | ✅ (env vars, CA cert) |

### The Critical Difference: Application Layer vs Network Layer

**Azure Firewall (UDR)** works at the network level. It intercepts all packets regardless of what application sent them. Even if your app doesn't know about the firewall, all its traffic goes through it.

**HTTP Proxy** works at the application level. It relies on the application reading `http_proxy` environment variables and voluntarily sending traffic through the proxy. Applications that don't respect these variables (some poorly written custom apps, or apps using raw TCP sockets) will bypass the proxy completely.

This is why enterprises that want guaranteed proxy enforcement use BOTH: UDR forces all network traffic through the firewall, and the firewall is configured to only allow traffic from the proxy IP to the internet. Any application that tries to bypass the proxy gets blocked by the firewall.

---

## 4. AKS HTTP Proxy Architecture

Here is the complete architecture when AKS is configured with an HTTP proxy:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           AKS Cluster (VNet A)                          │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │                            Node                                 │     │
│  │                                                                  │     │
│  │   ┌────────────────────┐    ┌────────────────────┐             │     │
│  │   │      Pod A          │    │      Pod B          │             │     │
│  │   │                     │    │ (no-proxy annotated)│             │     │
│  │   │ http_proxy=...      │    │ No proxy env vars   │             │     │
│  │   │ https_proxy=...     │    │                     │             │     │
│  │   └────────┬────────────┘    └──────────┬──────────┘             │     │
│  │            │                             │                        │     │
│  │   All HTTP/S traffic                     │ No-proxy destinations │     │
│  │   goes to Proxy                          ▼                        │     │
│  └────────────┼─────────────────────────── LB → Public IP ──► Internet│     │
│               │ (via VNet peering)                                    │     │
└───────────────┼───────────────────────────────────────────────────────┘     │
                │                                                               │
                ▼                                                               │
┌─────────────────────────────────────────────┐                                │
│             Proxy VNet (VNet B)              │                                │
│                                              │                                │
│   ┌─────────────────────────────────────┐   │                                │
│   │  HTTP Proxy Server (e.g., mitmproxy) │   │                                │
│   │  - Logs all requests                 │   │─────────────► Internet         │
│   │  - SSL inspection                    │   │  (Proxy's public IP)           │
│   │  - Enforces allowed domains          │   │                                │
│   └─────────────────────────────────────┘   │                                │
│                                              │                                │
└──────────────────────────────────────────────┘                                │
                                                                                │
VNet A ←→ VNet B: Connected via VNet Peering
```

**Two egress paths:**
1. **Via Proxy (default for HTTP/S):** Pod reads `http_proxy` → connects to proxy private IP → proxy fetches and returns → proxy's public IP visible to internet
2. **Bypassing Proxy (no-proxy or annotated pods):** Pod connects directly → through cluster's Load Balancer / NAT Gateway → LB/NAT Gateway's public IP visible

---

## 5. The Proxy Configuration JSON File

AKS proxy configuration is provided as a JSON file at cluster creation time. Here is every field with full explanation:

```json
{
  "httpProxy": "http://20.73.245.90:8080/",
  "httpsProxy": "https://20.73.245.90:8080/",
  "noProxy": [
    "localhost",
    "127.0.0.1",
    "docker.io",
    "docker.com"
  ],
  "trustedCA": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t..."
}
```

### Field: `httpProxy`

**Type:** String (URL)  
**Required:** Yes (if using proxy)  
**URL scheme:** Must start with `http://` (NOT `https://`)  

The proxy server URL used for outgoing **HTTP** (port 80) connections. Format: `http://[user:password@]hostname:port/`

Why `http://` for the httpProxy URL even if the proxy itself is accessed over HTTPS? This is historical convention from how proxy environment variables work in Unix. The `httpProxy` URL specifies HOW to reach the proxy server (via HTTP), not the protocol of the upstream connections being proxied.

Example values:
```json
"httpProxy": "http://proxy.corp.internal:8080/"
"httpProxy": "http://username:password@proxy.corp.internal:8080/"
"httpProxy": "http://10.0.0.4:3128/"
```

### Field: `httpsProxy`

**Type:** String (URL)  
**Required:** Yes (if using proxy for HTTPS)  
**URL scheme:** Can be `http://` or `https://`  

The proxy server URL used for outgoing **HTTPS** (port 443) connections. If this field is omitted, `httpProxy` value is used for both HTTP and HTTPS.

Most proxies listen on the same port for both HTTP and HTTPS CONNECT tunnel requests, but some proxies use different ports or have different endpoints.

### Field: `noProxy`

**Type:** Array of strings  
**Required:** No (but important!)  

List of destinations that should **bypass** the proxy. Requests to these destinations go directly without going through the proxy server.

Supported formats:
- Domain names: `"docker.com"` (matches `docker.com` and all subdomains)
- Subdomains: `"*.microsoft.com"` (all subdomains of microsoft.com)
- IP addresses: `"192.168.1.100"` (exact match)
- CIDR ranges: `"10.0.0.0/8"` (all IPs in the range)
- Specific ports: `"192.168.1.100:8080"`
- Localhost: `"localhost"`, `"127.0.0.1"`

**Critical:** AKS automatically adds required system entries to `noProxy` (see Section 10), but you should explicitly add:
- Your VNet CIDR ranges (internal traffic shouldn't go through proxy)
- Azure services your pods use (Key Vault, Storage, etc.) if you want direct access
- Internal DNS/service names

### Field: `trustedCA`

**Type:** String (Base64-encoded PEM certificate)  
**Required:** Only when the proxy does SSL inspection  
**Format:** PEM certificate, base64-encoded, no whitespace or newlines  
**Limit:** Up to 20 certificates in a single base64 block  

The Certificate Authority (CA) certificate for the proxy. Required when the proxy performs SSL/TLS inspection (decrypts and re-encrypts HTTPS traffic). Without this, pods will see SSL certificate errors when accessing HTTPS sites through the proxy.

**Important for Go-based components:** The certificate MUST use **Subject Alternative Names (SANs)** instead of Common Name (CN). Kubernetes system components (kubelet, kube-proxy, etc.) are written in Go, and Go's TLS library doesn't accept certificates with only CN — it requires SANs.

---

## 6. SSL/TLS Inspection and the `trustedCA` Field

This is one of the trickier aspects to understand. Let's go deep.

### Normal HTTPS Connection (No Proxy)

```
App → TLS Handshake → api.github.com
       ↑ App verifies GitHub's certificate (signed by a trusted public CA)
       ↑ Connection is encrypted end-to-end
```

### HTTPS Through a Proxy WITHOUT SSL Inspection

```
App → HTTP CONNECT api.github.com:443 → Proxy
      ← 200 Connection Established ←
App → TLS Handshake → [through Proxy tunnel] → api.github.com
      ↑ The proxy is just a dumb pipe — it passes encrypted bytes
      ↑ App still verifies GitHub's real certificate
      ↑ Proxy cannot see the encrypted content
```

The proxy can log that a connection was made to `api.github.com:443`, but cannot see what URL path, request headers, or response body was exchanged.

### HTTPS Through a Proxy WITH SSL Inspection (Man-in-the-Middle)

```
App → HTTP CONNECT api.github.com:443 → Proxy
      ← 200 Connection Established ←
App → TLS Handshake → PROXY (not GitHub!)
      ↑ Proxy presents a FAKE certificate for api.github.com
      ↑ Fake certificate is signed by the PROXY's CA
      ↑ If the app TRUSTS the proxy's CA → handshake succeeds
      ↑ Proxy can now see ALL decrypted content!

Proxy → NEW TLS Handshake → api.github.com (real)
      ↑ Proxy uses GitHub's real certificate for this connection
      
Proxy: Decrypts, inspects, potentially modifies, re-encrypts, forwards
```

**This is why `trustedCA` is required for SSL inspection:**
- Without it, the app sees an untrusted certificate (signed by an unknown CA) → SSL error
- With `trustedCA` installed, the app trusts the proxy's fake certificate → connection succeeds

### Getting the CA Certificate as Base64

```bash
# Get the proxy CA certificate
# (Method depends on your proxy software)

# For mitmproxy, it generates a cert at ~/.mitmproxy/mitmproxy-ca-cert.pem

# Encode it as base64 (no line breaks)
base64 -w 0 proxy-ca-cert.pem
# or on macOS:
base64 -i proxy-ca-cert.pem

# OR generate your own CA certificate for the proxy:
openssl genrsa -out cert.key 2048
openssl req -new -x509 -key cert.key -out cert.crt \
  -days 3650 \
  -subj "/CN=Corporate Proxy CA" \
  -addext "subjectAltName=DNS:proxy.corp.internal"
# ^ The -addext line adds the SAN required for Kubernetes Go components

TRUSTED_CA=$(base64 -w 0 cert.crt)
echo $TRUSTED_CA  # This value goes in "trustedCA" in your JSON
```

### Adding Multiple CA Certificates

If you need to trust multiple CAs (e.g., the proxy CA + your internal PKI CA), concatenate the PEM certificates and encode together:

```bash
cat proxy-ca.pem internal-ca.pem > combined-ca.pem
base64 -w 0 combined-ca.pem
```

The `trustedCA` field accepts up to 20 concatenated PEM certificates in a single base64-encoded block.

---

## 7. Creating a New Cluster with HTTP Proxy

### Step 1: Create the Configuration File

```json
{
  "httpProxy": "http://20.73.245.90:8080/",
  "httpsProxy": "https://20.73.245.90:8080/",
  "noProxy": [
    "localhost",
    "127.0.0.1",
    ".svc.cluster.local",
    ".svc",
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "docker.io",
    "docker.com",
    ".azurecr.io",
    ".blob.core.windows.net",
    ".vault.azure.net"
  ],
  "trustedCA": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t..."
}
```

Save as `aks-proxy-config.json`.

### Step 2: Create the AKS Cluster

```bash
az aks create \
  --name aks \
  --resource-group rg-aks \
  --http-proxy-config aks-proxy-config.json \
  --generate-ssh-keys
```

AKS will:
1. Configure each node's system proxy settings (modifying `/etc/environment`, systemd services)
2. Configure containerd (the container runtime) to use the proxy for image pulls
3. Install the `trustedCA` certificate in the system trust store on each node
4. Set up the mutating admission webhook to inject proxy env vars into pods

### Verifying Configuration

```bash
# Check that proxy env vars are set on a pod
kubectl run nginx --image=nginx
kubectl exec nginx -- env | grep -i proxy

# Expected output:
# http_proxy=http://20.73.245.90:8080/
# HTTP_PROXY=http://20.73.245.90:8080/
# https_proxy=https://20.73.245.90:8080/
# HTTPS_PROXY=https://20.73.245.90:8080/
# no_proxy=localhost,127.0.0.1,...
# NO_PROXY=localhost,127.0.0.1,...
```

```bash
# Verify egress goes through proxy
kubectl exec nginx -- curl -s ifconf.me
# Should return the PROXY's public IP, not the AKS cluster's public IP
```

---

## 8. How AKS Injects Proxy Settings — The Mutating Webhook

The mechanism AKS uses to inject proxy environment variables into pods is a **Kubernetes Mutating Admission Webhook**.

### What Is a Mutating Admission Webhook?

When you submit a pod specification to Kubernetes (via `kubectl apply`, `kubectl run`, or from a Deployment controller), the request goes through the Kubernetes API server's admission chain:

```
kubectl apply pod.yaml
      ↓
Kubernetes API Server
      ↓
[Authentication] → [Authorization] → [Admission Control]
                                          ↓
                               Mutating Admission Webhooks
                               (can modify the pod spec!)
                                          ↓
                               Validating Admission Webhooks
                               (can reject the pod spec)
                                          ↓
                               Pod stored in etcd and scheduled
```

**AKS installs a mutating webhook** that intercepts every pod creation request and adds the proxy environment variables to every container in the pod spec.

### What The Webhook Does

Before the webhook:
```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    # No env vars
```

After the webhook mutates it:
```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: http_proxy
      value: "http://20.73.245.90:8080/"
    - name: HTTP_PROXY
      value: "http://20.73.245.90:8080/"
    - name: https_proxy
      value: "https://20.73.245.90:8080/"
    - name: HTTPS_PROXY
      value: "https://20.73.245.90:8080/"
    - name: no_proxy
      value: "localhost,127.0.0.1,..."
    - name: NO_PROXY
      value: "localhost,127.0.0.1,..."
```

The application never knows the proxy env vars came from AKS — it just sees them as regular environment variables.

### Why Both Lowercase and Uppercase?

The convention for proxy environment variables in Linux/Unix is:
- **Lowercase** (`http_proxy`, `https_proxy`, `no_proxy`): The original Unix convention
- **Uppercase** (`HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`): Windows convention, also common on Linux

**Different applications check different cases.** Some check only lowercase, some only uppercase, some check both. By setting both, AKS maximizes compatibility across all applications.

---

## 9. Environment Variables Injected Into Pods

Here is the complete output of checking proxy env vars inside a pod:

```bash
kubectl exec -it nginx -- env | grep -i proxy

# Output:
http_proxy=http://10.0.0.4:8080/
HTTP_PROXY=http://10.0.0.4:8080/

https_proxy=https://10.0.0.4:8080/
HTTPS_PROXY=https://10.0.0.4:8080/

no_proxy=localhost,aks8v0n0swv.hcp.westeurope.azmk8s.io,10.10.0.0/24,10.0.0.0/16,169.254.169.254,docker.com,127.0.0.1,docker.io,konnectivity,10.10.0.0/16,168.63.129.16
NO_PROXY=localhost,aks8v0n0swv.hcp.westeurope.azmk8s.io,10.10.0.0/24,10.0.0.0/16,169.254.169.254,docker.com,127.0.0.1,docker.io,konnectivity,10.10.0.0/16,168.63.129.16
```

### The Proxy IP Is Private (10.0.0.4)

Notice the proxy address is `10.0.0.4` — a **private VNet IP**, not the proxy server's public IP. This means:
- The proxy server is inside Azure (in a VNet peered with the AKS VNet)
- Pod traffic to the proxy goes via private VNet routing (no internet hop)
- The proxy then egresses to the internet from its own public IP

This is the typical enterprise architecture: the proxy server is inside your corporate network, reachable via private IPs.

---

## 10. AKS-Managed No-Proxy Entries

AKS automatically adds critical system entries to the `no_proxy` list, REGARDLESS of what you specify. These entries are added to protect cluster functionality:

| Entry | Why It Must Bypass the Proxy |
|-------|------------------------------|
| `169.254.169.254` | Azure Instance Metadata Service (IMDS). VMs use this to get tokens, instance info, etc. Must not go through proxy. |
| `168.63.129.16` | Azure Wire Server / Platform DNS. Used for health monitoring and Azure platform services. Must not go through proxy. |
| `konnectivity` | Internal Kubernetes service for API server connectivity. Internal traffic, must bypass. |
| `aks8v0n0swv.hcp.westeurope.azmk8s.io` | The AKS API server FQDN. Control plane communication. Must not go through proxy. |
| `10.x.x.x/x` (VNet CIDR) | Your VNet's IP range. Pod-to-pod, pod-to-node, pod-to-service communication is all private and must bypass the proxy. |
| `10.x.x.x/x` (Service CIDR) | Kubernetes service IPs. ClusterIP services are internal — must bypass proxy. |

### Adding Your Own No-Proxy Entries

Beyond AKS-managed entries, add entries for:
- Azure services accessed via private endpoints (Storage, Key Vault, SQL)
- Internal microservices that pods call (internal FQDNs)
- VNet CIDR ranges
- Any partner APIs that should use the direct egress path

```json
"noProxy": [
  "localhost",
  "127.0.0.1",
  ".mycompany.internal",
  ".svc.cluster.local",
  ".svc",
  "10.0.0.0/8",
  "172.16.0.0/12",
  "192.168.0.0/16",
  ".blob.core.windows.net",
  ".vault.azure.net",
  ".azurecr.io",
  ".database.windows.net"
]
```

---

## 11. Bypassing the Proxy for Specific Pods

Sometimes individual pods need to bypass the proxy — for example:
- A monitoring agent that must use the direct cluster egress IP
- A pod that connects only to internal services (no need for proxy)
- A privileged pod for network testing

### The Bypass Annotation

Add the annotation `kubernetes.azure.com/no-http-proxy-vars: "true"` to the pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-noproxy
  annotations:
    "kubernetes.azure.com/no-http-proxy-vars": "true"   # ← This annotation
spec:
  containers:
  - image: nginx
    name: nginx
```

### What Happens With This Annotation

The AKS mutating webhook checks for this annotation. When present:
- The webhook does NOT inject `http_proxy`, `https_proxy`, `no_proxy` env vars
- The pod has NO proxy environment variables
- All connections from this pod go directly through the cluster's normal egress (Load Balancer or NAT Gateway)

```bash
kubectl apply -f nginx-noproxy.yaml

# Check env vars — proxy vars should be absent
kubectl exec nginx-noproxy -- env | grep -i proxy
# (No output — no proxy env vars)

# Check egress IP — should be LB/NAT Gateway IP, NOT proxy IP
kubectl exec nginx-noproxy -- curl -s ifconf.me
# Output: 4.245.123.106  ← AKS Load Balancer IP (NOT the proxy)
```

### For Deployments

Apply the annotation in the pod template:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: direct-egress-app
spec:
  template:
    metadata:
      annotations:
        "kubernetes.azure.com/no-http-proxy-vars": "true"
    spec:
      containers:
      - name: app
        image: myapp:latest
```

---

## 12. Updating Proxy Settings on an Existing Cluster

You can update proxy settings on an existing cluster that was created WITH proxy enabled:

```bash
# Create updated config file (e.g., new CA cert, updated noProxy list)
az aks update \
  --name aks \
  --resource-group rg-aks \
  --http-proxy-config aks-proxy-config-updated.json
```

### What Gets Updated and When

| What | When it takes effect |
|------|---------------------|
| `httpProxy` / `httpsProxy` | Pods must be **restarted** to pick up new env vars |
| `noProxy` | Pods must be **restarted** to pick up new env vars |
| `trustedCA` | Injected into new pods immediately; existing pods need restart |
| Node-level settings (containerd, system) | Only after **node image upgrade** |

### Mechanism: Mutating Admission Webhook

The proxy env vars are injected at pod CREATION time by the mutating webhook. Existing running pods already have the old env vars baked in. To pick up new values:

```bash
# Restart a specific deployment
kubectl rollout restart deployment/<deployment-name>

# Restart all deployments in a namespace
kubectl get deployments -n <namespace> -o name | \
  xargs -I{} kubectl rollout restart {}

# Or delete individual pods (Kubernetes recreates them)
kubectl delete pods --all -n <namespace>
```

### What You CANNOT Do

You **cannot enable HTTP proxy on an existing cluster** that was created WITHOUT proxy settings. This is because enabling the proxy requires:
1. Installing the CA certificate in the node OS trust store
2. Configuring `containerd` (the container runtime) to use the proxy for image pulls
3. These changes require node-level bootstrapping that can only happen at node creation time

If you need to enable the proxy on an existing cluster, you must create a new cluster.

However, you CAN **update** proxy settings (`httpProxy`, `httpsProxy`, `noProxy`, `trustedCA`) on a cluster that was already created with proxy enabled.

---

## 13. Proxy Settings at the Node Level (containerd)

Beyond injecting env vars into pods, AKS also configures the **node-level** components to use the proxy:

### What Gets Configured at the Node Level

1. **containerd** (container runtime): Proxy settings so that `containerd` uses the proxy when pulling container images from Docker Hub, MCR, or any registry

2. **System services:** The OS-level proxy environment variables in `/etc/environment` or equivalent, so system tools and `apt`/`yum` use the proxy for package updates

3. **CA Certificate installation:** The `trustedCA` certificate is installed in the OS system trust store (`/etc/ssl/certs/` on Ubuntu, etc.) and in the `containerd` trust store

### Why Node-Level Proxy Matters for Image Pulls

When a pod is scheduled on a node, `containerd` on that node pulls the container image. If `containerd` is NOT configured to use the proxy:
- The node tries to connect directly to Docker Hub / MCR
- In a fully restricted network where direct internet is blocked, the pull fails
- Pod goes into `ErrImagePull` / `ImagePullBackOff` state

With the proxy configured at the node level, `containerd` sends image pull requests through the proxy, which can reach Docker Hub.

### When Node-Level Changes Take Effect

For components under Kubernetes (like containerd and the node OS itself), proxy settings changes from `az aks update` won't take effect until a **node image upgrade** is performed:

```bash
# Upgrade node images to apply new proxy settings to node-level components
az aks upgrade \
  --resource-group $RG \
  --name $NAME \
  --node-image-only    # Only upgrade node images, not Kubernetes version
```

During a node image upgrade, nodes are recycled one by one with the new image (which includes the updated proxy settings). This is a zero-downtime operation when done with the default surge upgrade settings.

---

## 14. What Cannot Be Changed After Cluster Creation

This is a critical constraint to plan around:

| Setting | Can be updated after creation? | Notes |
|---------|-------------------------------|-------|
| `httpProxy` URL | ✅ YES | Pods must restart; node upgrade needed for containerd |
| `httpsProxy` URL | ✅ YES | Same as above |
| `noProxy` list | ✅ YES | Same as above |
| `trustedCA` | ✅ YES | Specifically allowed for CA certificate rotation |
| Enable proxy on non-proxy cluster | ❌ NO | Cannot enable proxy on existing cluster without proxy |
| Disable proxy completely | ❌ NO | Once enabled, the proxy feature cannot be removed |

### Planning Implication

If you think you'll need HTTP proxy, configure it at cluster creation time. You cannot add it later. Plan your proxy architecture before creating production clusters.

---

## 15. UDR Firewall vs HTTP Proxy — Side by Side

These two approaches are often confused or compared. Here's a direct comparison:

### Architecture

```
UDR / Firewall:
  Pod → Node → Route Table (0.0.0.0/0 → Firewall) → Azure Firewall → Internet
  ↑ Works at NETWORK level. Application unaware. ALL traffic forced through.

HTTP Proxy:
  Pod (http_proxy=...) → [reads env var] → Proxy Server → Internet
  ↑ Works at APPLICATION level. App must read and respect the env var.
  Direct traffic (bypassed apps) → LB / NAT GW → Internet
```

### Key Behavioral Differences

| Scenario | UDR/Firewall | HTTP Proxy |
|----------|-------------|-----------|
| App ignores proxy settings | Firewall intercepts anyway | Traffic bypasses proxy ⚠️ |
| UDP traffic (DNS, NTP) | Controlled by firewall | Proxy doesn't intercept UDP ⚠️ |
| Custom TCP protocols | Controlled at L4 by firewall | Proxy can't handle (not HTTP) ⚠️ |
| Network change needed? | Yes (route table changes) | No (only env vars) |
| Infra team involvement | High | Lower |
| Visibility into HTTP requests | URL-level (FQDN) | Full URL + headers |

### When to Choose HTTP Proxy

- Organization already has a proxy infrastructure (Squid, Zscaler, Blue Coat, etc.)
- You need to cache Docker image layers to speed up deployments
- You need SSL inspection at HTTP request granularity
- Changing network routing is difficult in your organization
- Your AKS pods ONLY make HTTP/HTTPS calls (no custom TCP protocols)

### When to Choose UDR/Firewall

- You need guaranteed enforcement (apps that don't respect `http_proxy` are blocked)
- You have UDP traffic to control (DNS, NTP, etc.)
- You have non-HTTP protocols (custom game servers, streaming protocols, database native protocols)
- You're adopting Azure Landing Zone / Hub & Spoke architecture
- Central network team manages all egress

### Using Both Together

In highly secure environments, organizations often use both:

```
AKS Pod
  ↓ (reads http_proxy env var)
Proxy Server (Zscaler, Squid, etc.)
  ↓ (outbound from proxy VM/container)
Azure Firewall (only allows traffic from proxy IP)
  ↓
Internet
```

This ensures:
- Apps that respect proxy → inspected by proxy → then through firewall
- Apps that DON'T respect proxy → direct to internet → blocked by firewall (only proxy IP allowed)
- Guaranteed enforcement: ALL HTTP/S eventually goes through proxy

---

## 16. Setting Up mitmproxy for Testing

**mitmproxy** is an open-source, interactive HTTP proxy used in the demo. It's perfect for learning because it provides a web UI where you can see every request your pods make in real time.

### Infrastructure Setup

The demo creates these resources:

| Resource | Type | Purpose |
|----------|------|---------|
| `vm-linux-mitmproxy` | Virtual Machine | Runs mitmproxy software |
| `pip-vm-proxy` | Public IP | Proxy VM's public IP (for admin SSH) |
| `nic-vm-proxy` | Network Interface | VM's NIC in proxy VNet |
| `nsg-vm-proxy` | Network Security Group | Firewall rules for the proxy VM |
| `vnet-proxy` | Virtual Network | Proxy VM's private network |
| `vnet-aks` | Virtual Network | AKS cluster's network |
| `aks-cluster` | AKS | The Kubernetes cluster |

**VNet Peering:** `vnet-aks` ↔ `vnet-proxy` are peered, allowing the AKS pods to reach the mitmproxy VM via private IP.

### Deploying mitmproxy

```bash
# SSH into the proxy VM
ssh user@<pip-vm-proxy-ip>

# Install mitmproxy
pip3 install mitmproxy

# Start mitmproxy in transparent mode on port 8080
mitmproxy --mode regular --listen-port 8080

# Or start mitmweb for a browser-based UI
mitmweb --mode regular --listen-port 8080 --web-port 8081
```

### Getting the mitmproxy CA Certificate

mitmproxy automatically generates a CA certificate when first started:

```bash
# On the proxy VM, the cert is at:
cat ~/.mitmproxy/mitmproxy-ca-cert.pem

# Encode it for the AKS trustedCA field:
base64 -w 0 ~/.mitmproxy/mitmproxy-ca-cert.pem
```

---

## 17. Generating a Custom CA Certificate for the Proxy

For enterprise scenarios, you create your own CA certificate (not use the proxy's auto-generated one). This gives you control over the certificate lifecycle and integrates with your PKI.

```bash
# Step 1: Generate a private key (2048-bit RSA)
openssl genrsa -out cert.key 2048

# Step 2: Create a Certificate Signing Request (CSR)
openssl req -new -key cert.key -out cert.csr \
  -subj "/C=US/O=My Corporation/CN=Corporate Proxy CA"

# Step 3: Self-sign the CSR as a CA certificate (10 years validity)
openssl x509 -req -in cert.csr -signkey cert.key \
  -out cert.crt \
  -days 3650 \
  -extensions v3_ca \
  -extfile <(cat <<EOF
[v3_ca]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical,CA:true
keyUsage = critical,digitalSignature,cRLSign,keyCertSign
subjectAltName = DNS:proxy.corp.internal,IP:10.0.0.4
EOF
)

# CRITICAL: Add subjectAltName (SAN) — required for Kubernetes Go components
# Without SAN, kubelet and other Go-based components will reject the certificate

# Step 4: Encode for AKS
TRUSTED_CA=$(base64 -w 0 cert.crt)
echo $TRUSTED_CA

# Step 5: Import cert.key and cert.crt into your proxy software
# (Process varies by proxy software — see your proxy's documentation)
```

---

## 18. Troubleshooting Proxy Issues

### Problem: Pod Connects to Internet Directly, Not Through Proxy

**Symptom:** `curl ifconf.me` returns the LB/NAT GW IP, not the proxy IP.

**Diagnosis:**
```bash
kubectl exec <pod-name> -- env | grep -i proxy
```

**Possible causes:**
- Pod was created before proxy was configured (solution: restart pod)
- Pod has the bypass annotation `kubernetes.azure.com/no-http-proxy-vars: "true"` (intentional)
- The application doesn't read proxy env vars (application-specific issue)
- The proxy is listed in `noProxy` for that domain

---

### Problem: SSL Certificate Error When Accessing HTTPS Sites

**Symptom:** `curl: (60) SSL certificate problem: unable to get local issuer certificate`

**Diagnosis:**
```bash
kubectl exec <pod-name> -- curl -v https://api.github.com 2>&1 | grep -A5 "SSL certificate"
```

**Cause:** The proxy is doing SSL inspection but the pod doesn't trust the proxy's CA certificate.

**Fix 1:** Verify `trustedCA` is in the proxy config and the cluster was created with it.  
**Fix 2:** For testing only, use `-k` flag to skip verification (NOT for production).  
**Fix 3:** Confirm the CA cert has SAN fields (not just CN).

---

### Problem: Image Pull Fails (`ErrImagePull`)

**Symptom:** Pods stuck in `ErrImagePull` state.

**Diagnosis:**
```bash
kubectl describe pod <pod-name> | grep -A5 "Failed"
# Look for: "Error response from daemon: Get https://registry-1.docker.io/..."
```

**Possible causes:**
- Node-level containerd proxy not configured (cluster was created without proxy and proxy was added later)
- The registry FQDN is in `noProxy` but the node can't reach it directly
- The `trustedCA` isn't installed in containerd's trust store

**Fix:** Perform a node image upgrade to apply proxy settings to the node level:
```bash
az aks upgrade -g $RG -n $NAME --node-image-only
```

---

### Problem: Internal Services Slow or Failing Through Proxy

**Symptom:** Pod-to-pod communication or pod-to-ClusterIP-service is slow or fails.

**Cause:** Internal IPs/FQDNs aren't in `noProxy`, so internal traffic goes through the proxy unnecessarily.

**Fix:** Add internal CIDRs and service domains to `noProxy`:
```json
"noProxy": [
  ".svc.cluster.local",
  ".svc",
  "10.0.0.0/8",
  "172.16.0.0/12"
]
```

---

## 19. Common Application Behaviors with Proxy

Not all applications automatically respect proxy environment variables. Here's what to expect:

| Language/Runtime | Proxy env var support | Notes |
|-----------------|----------------------|-------|
| curl | ✅ Yes | Reads `http_proxy`, `https_proxy`, `no_proxy` |
| wget | ✅ Yes | Same |
| Python (requests lib) | ✅ Yes | `requests` reads env vars automatically |
| Python (urllib) | ✅ Yes | Built-in support |
| Node.js (axios, node-fetch) | ⚠️ Partial | Need `global-agent` or `node-global-proxy` package |
| Node.js (native http) | ❌ No | Must implement proxy support manually |
| Go (net/http) | ✅ Yes | Built-in PROXY env var support since Go 1.x |
| Java (Apache HttpClient) | ⚠️ Partial | JVM proxy settings (`-Dhttp.proxyHost=...`) differ from env vars |
| Java (OkHttp) | ⚠️ Partial | Proxy must be explicitly configured |
| .NET (HttpClient) | ✅ Yes | Reads `http_proxy` on Linux |

### For Applications That Don't Support Proxy Env Vars

Option 1: Configure the proxy in application code explicitly.

Option 2: Use a sidecar proxy (like Envoy or a transparent proxy) that intercepts all traffic at the pod level.

Option 3: Use UDR/Firewall instead of HTTP proxy (works at network level, application unaware).

---

## 📋 Quick Reference

```bash
# ── CREATE ──────────────────────────────────────────────────────

az aks create -n aks -g rg-aks \
  --http-proxy-config aks-proxy-config.json \
  --generate-ssh-keys

# ── UPDATE PROXY SETTINGS ────────────────────────────────────────

az aks update -n aks -g rg-aks \
  --http-proxy-config aks-proxy-config-updated.json

# ── VERIFY IN POD ────────────────────────────────────────────────

# Check proxy env vars are injected
kubectl exec <pod> -- env | grep -i proxy

# Check egress IP (should be proxy's public IP)
kubectl exec <pod> -- curl -s ifconf.me

# ── BYPASS PROXY FOR SPECIFIC POD ───────────────────────────────

# Add annotation to pod spec:
# annotations:
#   "kubernetes.azure.com/no-http-proxy-vars": "true"

# ── FORCE NODE RELOAD OF PROXY SETTINGS ─────────────────────────

az aks upgrade -g $RG -n $NAME --node-image-only

# ── RESTART PODS TO PICK UP NEW PROXY ENV VARS ──────────────────

kubectl rollout restart deployment/<name>
```

---

## 📚 Official References

- [Configure AKS nodes with HTTP proxy](https://learn.microsoft.com/en-us/azure/aks/http-proxy)
- [GitHub demo: AKS egress proxy](https://github.com/HoussemDellai/docker-kubernetes-course/tree/main/67_egress_proxy)
- [mitmproxy documentation](https://mitmproxy.org/)
- [Kubernetes Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)