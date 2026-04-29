# Kubernetes CoreDNS & Custom Domains in AKS
### A Beginner-Friendly Deep Dive

> **What this document covers:** What DNS is inside a Kubernetes cluster, what CoreDNS is, how every pod resolves names, what the default Kubernetes DNS naming pattern looks like, how to customize CoreDNS with your own domain names, the ConfigMap system AKS uses, the plugin architecture, CoreDNS autoscaling, LocalDNS feature, and the Azure DNS plugin — all explained from absolute scratch with 2024-2025 updates from Microsoft Azure documentation.

---

## Table of Contents
1. [Why Does Kubernetes Need Its Own DNS?](#1-why-does-kubernetes-need-its-own-dns)
2. [How DNS Works Inside a Pod](#2-how-dns-works-inside-a-pod)
3. [The Default Kubernetes DNS Naming Pattern](#3-the-default-kubernetes-dns-naming-pattern)
4. [What is CoreDNS?](#4-what-is-coredns)
5. [CoreDNS Inside AKS — What Runs Where](#5-coredns-inside-aks--what-runs-where)
6. [The Corefile — CoreDNS Configuration Language](#6-the-corefile--coredns-configuration-language)
7. [CoreDNS Plugin Architecture](#7-coredns-plugin-architecture)
8. [The coredns-custom ConfigMap — How AKS Lets You Customize](#8-the-coredns-custom-configmap--how-aks-lets-you-customize)
9. [Custom Domain Names with CoreDNS (The Rewrite Plugin)](#9-custom-domain-names-with-coredns-the-rewrite-plugin)
10. [Forwarding to External DNS Servers](#10-forwarding-to-external-dns-servers)
11. [CoreDNS Autoscaling in AKS](#11-coredns-autoscaling-in-aks)
12. [LocalDNS — Advanced Low-Latency DNS](#12-localdns--advanced-low-latency-dns)
13. [The CoreDNS Azure DNS Plugin](#13-the-coredns-azure-dns-plugin)
14. [Applying and Restarting CoreDNS Changes](#14-applying-and-restarting-coredns-changes)
15. [Verifying CoreDNS Behavior](#15-verifying-coredns-behavior)
16. [Common Use Cases with Full YAML Examples](#16-common-use-cases-with-full-yaml-examples)
17. [Troubleshooting CoreDNS](#17-troubleshooting-coredns)
18. [Summary Cheat Sheet](#18-summary-cheat-sheet)

---

## 1. Why Does Kubernetes Need Its Own DNS?

### The Problem with IP Addresses in Kubernetes

In Kubernetes, pods are ephemeral — they start and stop constantly. Every time a pod is recreated, it gets a **new IP address**. If Service A needed to call Service B directly by IP address, it would break every time Service B's pod restarted.

Kubernetes solves this with **Services** — stable virtual IPs that always point to the right set of pods, no matter how often those pods restart. But even Service IPs can change (when you recreate the Service). So Kubernetes goes one step further: it gives every Service a **stable DNS name**.

Instead of connecting to `10.0.24.138` (which might change), you connect to `nginx.default.svc.cluster.local` (which never changes as long as the Service exists).

**This is why Kubernetes needs its own internal DNS server** — to map stable service names to their current cluster IPs, automatically, in real time.

### What Happened Before CoreDNS?
Before CoreDNS (prior to Kubernetes 1.12), Kubernetes used **kube-dns** — a set of three containers (kubedns, dnsmasq, sidecar) to handle cluster DNS. It was complex and had limitations. CoreDNS replaced it as the default DNS server for Kubernetes in version 1.12 and is now used in all modern AKS clusters.

> **Important (2024-2025 Update):** kube-dns is now **fully deprecated** in AKS. All AKS clusters use CoreDNS. If you have old kube-dns customizations, you must rewrite them for CoreDNS — they are NOT backward compatible.

---

## 2. How DNS Works Inside a Pod

### What Happens When a Pod Makes a DNS Query

Every pod in Kubernetes has a file called `/etc/resolv.conf` injected into it automatically by Kubernetes. This file tells the pod which DNS server to use.

```
# Inside any pod: cat /etc/resolv.conf
nameserver 10.0.0.10        ← IP of the CoreDNS Service (kube-dns)
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Breaking this down:

**`nameserver 10.0.0.10`**
This is the ClusterIP of the `kube-dns` Service in the `kube-system` namespace. All DNS queries from every pod go to this IP, which forwards them to the CoreDNS pods. The `10.0.0.10` value comes from the `--dns-service-ip` parameter you set when creating the AKS cluster (or the default within your Service CIDR).

**`search default.svc.cluster.local svc.cluster.local cluster.local`**
This is the **DNS search list**. It defines suffix domains that Kubernetes automatically appends to short names. If you query just `nginx`, the system tries:
1. `nginx.default.svc.cluster.local` ← appends first search domain
2. `nginx.svc.cluster.local`
3. `nginx.cluster.local`
4. `nginx` (bare name)

This is why you can reach a Service called `nginx` in the same namespace by just typing `nginx` — Kubernetes tries appending search suffixes until it finds a match.

**`options ndots:5`**
If a name has fewer than 5 dots, Kubernetes treats it as a relative name and appends search domains. If it has 5 or more dots, it's treated as an absolute FQDN. 

> **Important (Azure DNS Interaction):** Azure DNS configures a default search domain of `<VNET_ID>.<REGION>.internal.cloudapp.net` in virtual networks using Azure DNS. Combined with Kubernetes' `ndots:5`, this can result in invalid search domain completion queries that cause DNS resolution delays. See [Troubleshooting section](#17-troubleshooting-coredns) for mitigation strategies.

### DNS Query Flow — Full Trace

```
Pod wants to connect to "nginx" (short name)
           │
           ▼
Pod reads /etc/resolv.conf → DNS server is 10.0.0.10
           │
           ▼
Pod sends DNS query: "nginx" to 10.0.0.10
           │
           ▼
CoreDNS pod receives the query
CoreDNS uses the Kubernetes plugin to look up:
  "nginx.default.svc.cluster.local"
           │
           ▼
Found! Returns ClusterIP: 10.0.197.10
           │
           ▼
Pod connects to 10.0.197.10:80
           │
           ▼
kube-proxy routes the connection to an actual pod IP
```

---

## 3. The Default Kubernetes DNS Naming Pattern

Every Kubernetes Service gets a DNS name following this exact pattern:

```
<service-name>.<namespace>.svc.<cluster-domain>
```

Where:
- `<service-name>` = the `metadata.name` of the Service
- `<namespace>` = the namespace the Service lives in
- `svc` = literal string "svc"
- `<cluster-domain>` = always `cluster.local` by default

### Examples

| Service Name | Namespace | Full DNS Name |
|-------------|-----------|---------------|
| `nginx` | `default` | `nginx.default.svc.cluster.local` |
| `postgres` | `databases` | `postgres.databases.svc.cluster.local` |
| `payment-api` | `payments` | `payment-api.payments.svc.cluster.local` |
| `kube-dns` | `kube-system` | `kube-dns.kube-system.svc.cluster.local` |

### Short Names That Also Work

Thanks to the DNS search list in `/etc/resolv.conf`, you can use shorter forms:

```bash
# From a pod in the "default" namespace, all of these work:
nslookup nginx                              # ← searched as nginx.default.svc.cluster.local
nslookup nginx.default                     # ← searched as nginx.default.svc.cluster.local
nslookup nginx.default.svc                 # ← searched as nginx.default.svc.cluster.local
nslookup nginx.default.svc.cluster.local   # ← FQDN, always works from anywhere
```

**Cross-namespace access requires at least `<service>.<namespace>`:**
```bash
# From a pod in "frontend" namespace calling a service in "backend" namespace:
nslookup payment-api.backend               # Works
nslookup payment-api                       # Does NOT work (searches frontend namespace)
```

---

## 4. What is CoreDNS?

**CoreDNS** is an open-source, flexible, extensible DNS server written in Go. It was donated to the **CNCF (Cloud Native Computing Foundation)** — the same organization that governs Kubernetes itself. It graduated from CNCF incubation in 2019.

GitHub: `https://github.com/coredns/coredns`

The key design principle of CoreDNS is **plugins**. Every feature of CoreDNS is a plugin — from answering Kubernetes Service queries, to caching, to forwarding to upstream DNS servers, to logging, to rate limiting. You chain plugins together in a configuration file called the **Corefile**.

### Why CoreDNS for Kubernetes?
- Single binary — simpler than kube-dns's three containers
- Plugin-based — easy to extend and configure
- Handles all DNS record types: A, AAAA, CNAME, SRV, TXT, PTR, MX
- Native Kubernetes integration via the `kubernetes` plugin
- Active maintenance and large community

### CoreDNS Version in AKS (2024-2025)
> **As of 2025:** AKS ships with CoreDNS v1.9.4 (with hotfixes). Microsoft has announced plans to bump CoreDNS to version 1.12 with Kubernetes 1.32 release. AKS does NOT support customizing the CoreDNS version — it's managed by the platform.

---

## 5. CoreDNS Inside AKS — What Runs Where

When you create an AKS cluster, CoreDNS is automatically installed. Let's see exactly what resources exist:

```bash
kubectl get pods,service,configmap -n kube-system -l=k8s-app=kube-dns
```

Output:
```
NAME                                READY   STATUS    RESTARTS   AGE
pod/coredns-77f75ff65d-clfgl        1/1     Running   0          12m
pod/coredns-77f75ff65d-twzz9        1/1     Running   0          11m

NAME                    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
service/kube-dns        ClusterIP   10.0.0.10    <none>        53/UDP,53/TCP    12m

NAME                         DATA   AGE
configmap/coredns             1     12m
configmap/coredns-custom      1     12m
```

### Resource Breakdown

**`pod/coredns-*` (two pods)**
CoreDNS runs as **two replicas** for high availability. If one pod crashes, DNS still works. Both pods handle DNS queries — requests are load-balanced between them.

- They run in the `kube-system` namespace (which is reserved for Kubernetes system components)
- They use very little CPU and RAM
- They have special permissions to read all Kubernetes Services and Endpoints

**`service/kube-dns` (ClusterIP: 10.0.0.10)**
This is the stable Service that all pods use as their DNS server. The IP `10.0.0.10` is what gets written into every pod's `/etc/resolv.conf`.

Notice it has **no External-IP** (`<none>`) — it's a ClusterIP Service, only reachable from inside the cluster. It listens on port **53** for both UDP (standard DNS) and TCP (for large responses).

The name `kube-dns` is historical (kept from the old kube-dns system for compatibility).

**`configmap/coredns` (the main config — DO NOT EDIT directly)**
This ConfigMap contains the base **Corefile** — CoreDNS's main configuration. In AKS, this is **managed by the platform**. You cannot and should not edit it directly.

**`configmap/coredns-custom` (YOUR customization point)**
This is the ConfigMap AKS provides for you to customize CoreDNS behavior safely. AKS's CoreDNS automatically imports entries from this ConfigMap into its running configuration. This is the only ConfigMap you should modify.

---

## 6. The Corefile — CoreDNS Configuration Language

The Corefile is CoreDNS's configuration file. It defines **server blocks** — each block says "for queries matching this domain, use these plugins."

### Reading the Default AKS Corefile

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

The Corefile looks roughly like:

```
.:53 {
    errors
    health {
        lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
        max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
    import custom/*.override
}

import custom/*.server
```

Let's decode every line:

**`.:53 {`**
This is a **server block**. The `.` means "match all domains" (the root domain). Port `53` is the standard DNS port. So this block handles ALL DNS queries on port 53.

**`errors`**
The `errors` plugin: logs error messages to stdout. Useful for debugging failed DNS lookups.

**`health { lameduck 5s }`**
The `health` plugin: exposes an HTTP endpoint at `:8080/health` for liveness probes. `lameduck 5s` means CoreDNS will delay shutdown by 5 seconds to allow in-flight requests to complete (graceful shutdown).

**`ready`**
The `ready` plugin: exposes `:8181/ready` for readiness probes. Kubernetes uses this to know when the CoreDNS pod is ready to accept traffic.

**`kubernetes cluster.local in-addr.arpa ip6.arpa { ... }`**
The `kubernetes` plugin — **the most important one**. This is what makes CoreDNS understand Kubernetes Services and Pods.
- `cluster.local`: serve DNS for the `cluster.local` domain (all Kubernetes service names)
- `in-addr.arpa ip6.arpa`: serve reverse DNS lookups (IP → name) for IPv4 and IPv6
- `pods insecure`: allow pod IP-based DNS (e.g., `10-0-0-1.default.pod.cluster.local`)
- `fallthrough`: if a name in `in-addr.arpa` is not found, pass the query to the next plugin
- `ttl 30`: DNS records for Services get a 30-second TTL

**`prometheus :9153`**
The `prometheus` plugin: exposes metrics for scraping by Prometheus monitoring system (request counts, latency histograms, cache hit rates, etc.).

**`forward . /etc/resolv.conf { max_concurrent 1000 }`**
The `forward` plugin — **the upstream forwarder**. For any query that the `kubernetes` plugin couldn't answer (i.e., external domains like `google.com`), forward it to the DNS servers listed in `/etc/resolv.conf` on the node. On AKS nodes, this file points to the Azure DNS virtual server (`168.63.129.16`), which resolves all public internet names and Azure internal names.

**`cache 30`**
The `cache` plugin: cache DNS responses for 30 seconds. This reduces load on upstream DNS servers and speeds up repeated queries.

**`loop`**
The `loop` plugin: detects and breaks DNS forwarding loops (protection against infinite query loops).

**`reload`**
The `reload` plugin: CoreDNS automatically reloads its configuration when the Corefile changes, without requiring a pod restart.

**`loadbalance`**
The `loadbalance` plugin: when a Service has multiple pod IPs (multiple endpoints), CoreDNS rotates the order of returned IPs across different responses — basic round-robin DNS load balancing.

**`import custom/*.override`**
Imports any `.override` files from the `coredns-custom` ConfigMap. These override default settings (like changing the upstream DNS server).

**`import custom/*.server`**
Imports any `.server` files from the `coredns-custom` ConfigMap. These add NEW server blocks for custom domains.

---

## 7. CoreDNS Plugin Architecture

CoreDNS processes each DNS query through a **chain of plugins**. Think of it like a middleware pipeline:

```
DNS Query arrives
       │
       ▼
┌─────────────┐
│   errors    │ ← Catches and logs errors from downstream plugins
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  kubernetes │ ← Is this a Kubernetes Service name? → YES: answer it
└──────┬──────┘   NO: pass to next plugin
       │
       ▼
┌─────────────┐
│   forward   │ ← Forward to upstream DNS (Azure DNS / 168.63.129.16)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│    cache    │ ← Cache the response before returning it
└──────┬──────┘
       │
       ▼
DNS Response returned to pod
```

Plugins execute in the order they appear in the Corefile. Each plugin can:
- **Answer** the query (and stop the chain)
- **Modify** the query and pass it along
- **Log** or **record metrics** and pass it along
- **Reject** the query with an error

### Key Plugins Reference

| Plugin | Purpose |
|--------|---------|
| `kubernetes` | Resolves Kubernetes Service/Pod names |
| `forward` | Forwards unresolved queries to upstream DNS |
| `cache` | Caches responses to reduce upstream load |
| `rewrite` | Rewrites query names (used for custom domains!) |
| `errors` | Logs errors |
| `health` | Liveness probe endpoint |
| `ready` | Readiness probe endpoint |
| `prometheus` | Metrics endpoint |
| `loop` | Detects forwarding loops |
| `reload` | Hot-reloads config changes |
| `loadbalance` | Round-robin DNS load balancing |
| `hosts` | Serve records from a static hosts file |
| `log` | Logs all DNS queries (useful for debugging) |
| `azure` | Reads records from Azure DNS zones |

> **Important:** All built-in CoreDNS plugins are supported in AKS. However, **add-on/third party plugins** (custom plugins not built into the CoreDNS binary) are NOT supported by AKS.

---

## 8. The coredns-custom ConfigMap — How AKS Lets You Customize

### Why You Can't Edit the Main Corefile
AKS manages the `coredns` ConfigMap as part of its managed service. If you edited it directly:
- AKS updates could overwrite your changes
- A misconfiguration could break DNS for the entire cluster
- There's no rollback mechanism

Instead, AKS gives you the `coredns-custom` ConfigMap as a safe customization layer.

### The Two Types of Customization

Every entry in `coredns-custom` must have a key ending in either `.server` or `.override`:

**`.server` keys** — Add a completely NEW server block for a specific domain
```yaml
data:
  my-domain.server: |         # Key MUST end in .server
    myinternaldomain.com:53 { # Handle queries for this domain
      errors
      cache 30
      forward . 10.0.0.53     # Forward to a custom DNS server
    }
```
Use case: "For queries to `myinternaldomain.com`, use MY DNS server at `10.0.0.53` instead of Azure DNS."

**`.override` keys** — Modify the existing default server block
```yaml
data:
  custom-settings.override: |  # Key MUST end in .override
    forward . 8.8.8.8 8.8.4.4  # Change default upstream to Google DNS
```
Use case: "Change the default upstream DNS from Azure DNS to Google DNS."

> **Naming Convention:** When you create configurations in `coredns-custom`, the names you specify in the data section **MUST** end in `.server` or `.override`. This naming convention is defined in the default AKS CoreDNS ConfigMap.

### Viewing the ConfigMap
```bash
kubectl get configmap coredns-custom -n kube-system -o yaml
```

Initially it's mostly empty — AKS creates it as a placeholder for your customizations.

---

## 9. Custom Domain Names with CoreDNS (The Rewrite Plugin)

### The Goal
By default, your apps are reachable inside the cluster at:
```
nginx.default.svc.cluster.local
```

That's ugly. You want your apps to be reachable at:
```
nginx.default.aks-internal.com
```

Or even just:
```
nginx.aks.com
```

This is exactly what the `rewrite` plugin does — it intercepts queries for your custom domain and rewrites them to the Kubernetes internal DNS format.

### How the Rewrite Plugin Works

The `rewrite stop` directive intercepts a DNS query, transforms it, and sends the transformed query back through the pipeline. The `stop` keyword means: once this rewrite matches, stop trying other rewrite rules.

```
Query: nginx.default.aks-internal.com
          │
          ▼
CoreDNS rewrite plugin:
  name regex (.*)\.aks-internal\.com$ {1}.svc.cluster.local.
  → transforms to: nginx.default.svc.cluster.local.
          │
          ▼
kubernetes plugin: 
  "I know nginx.default.svc.cluster.local! ClusterIP = 10.0.197.10"
          │
          ▼
answer name regex: transforms the RESPONSE name back:
  (.*)\.svc\.cluster\.local\.$ → {1}.aks-internal.com.
  → nginx.default.aks-internal.com in the response

Response returned with:
  Name: nginx.default.aks-internal.com
  Address: 10.0.197.10
```

### The Full ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  internal-custom.override: |
    rewrite stop {
      name regex (.*)\.aks-internal\.com\.$ {1}.svc.cluster.local.
      answer name (.*)\.svc\.cluster\.local\.$ {1}.aks-internal.com.
    }
```

### Breaking Down the Regex

**`name regex (.*)\.aks-internal\.com\.$`**
- `name regex` — match the query name using a regular expression
- `(.*)` — capture group 1: captures everything before `.aks-internal.com`
- `\.aks-internal\.com\.` — the literal domain suffix (backslashes escape the dots)
- `$` — end of string

So `nginx.default.aks-internal.com.` matches, capturing `nginx.default` as group 1.

**`{1}.svc.cluster.local.`**
- `{1}` — substitute capture group 1 (`nginx.default`)
- Result: `nginx.default.svc.cluster.local.`

**`answer name (.*)\.svc\.cluster\.local\.$`**
- This transforms the ANSWER (the response) name back from internal format to your custom domain
- Without this, the DNS response would say `Name: nginx.default.svc.cluster.local` — confusing for clients

### Verification

```bash
# Exec into any running pod
kubectl exec -it nginx -- bash

# Test with the standard Kubernetes name (still works)
nslookup nginx.default.svc.cluster.local
# Server:   10.0.0.10
# Address:  10.0.0.10#53
# Name:     nginx.default.svc.cluster.local
# Address:  10.0.197.10    ← ClusterIP

# Test with the custom domain name (NEW!)
nslookup nginx.default.aks-internal.com
# Server:   10.0.0.10
# Address:  10.0.0.10#53
# Name:     nginx.default.aks-internal.com
# Address:  10.0.197.10    ← Same ClusterIP! The rewrite worked.
```

Both names resolve to the same IP — the standard Kubernetes name still works, AND the custom domain name works too.

---

## 10. Forwarding to External DNS Servers

Sometimes your pods need to resolve names that live on your corporate/on-premises DNS server (e.g., `myserver.corp.internal`). Azure DNS doesn't know about these names. You need to tell CoreDNS: "For `corp.internal` queries, ask this specific DNS server."

### Forwarding a Specific Domain to a Custom DNS Server

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  corp-dns.server: |              # .server extension — adds new server block
    corp.internal:53 {            # Handle queries for corp.internal domain
      errors                      # Log errors
      cache 30                    # Cache responses for 30 seconds
      forward . 10.10.0.5         # Forward to corporate DNS server
    }
```

**What this does:**
- Any pod querying `anything.corp.internal` → CoreDNS forwards to `10.10.0.5`
- All other queries (Google, Azure, Kubernetes services) → handled as usual by the default block

### Forwarding ALL queries to a Custom DNS Server

```yaml
data:
  upstream.override: |            # .override — modifies the default block
    forward . 10.10.0.5           # All external queries go to corporate DNS
```

⚠️ **Warning**: If you change the default forward to a custom DNS server, make sure that server can resolve Azure internal names (like `*.internal.cloudapp.net`). If it can't, your nodes may fail to resolve Azure-specific hostnames. The safe way is to also add a fallback:

```yaml
data:
  upstream.override: |
    forward . 10.10.0.5 168.63.129.16  # Corporate DNS first, Azure DNS as fallback
```

### Forwarding Internal Azure Domain Names
Azure uses special internal domain names like `*.internal.cloudapp.net` for VM name resolution. If you've changed the upstream DNS, you must explicitly forward these back to Azure DNS (`168.63.129.16`):

```yaml
data:
  azure-internal.server: |
    internal.cloudapp.net:53 {
      errors
      cache 30
      forward . 168.63.129.16     # Azure's virtual DNS IP
    }
```

---

## 11. CoreDNS Autoscaling in AKS

### The Challenge
AKS clusters are elastic — they scale up and down based on workload. Sudden spikes in DNS traffic can lead to increased memory consumption by CoreDNS pods, potentially causing Out of Memory (OOM) issues.

### How AKS Handles CoreDNS Autoscaling (2024-2025)

AKS uses **two mechanisms** to autoscale CoreDNS:

**1. Horizontal Pod Autoscaler (HPA) - Cluster Proportional Autoscaler**
CoreDNS pods scale based on the number of **nodes** and **cores** in the cluster. This is controlled by the `coredns-autoscaler` ConfigMap.

Default formula (roughly):
```
replicas = max(2, ceil( nodes * 1/5 + cores * 1/512 ))
```

**2. Vertical Pod Autoscaler (VPA) - Resource Requests/Limits**
VPA automatically adjusts CoreDNS pod CPU and memory requests/limits based on observed usage.

Default resource configuration (as of 2025):

| Resource | Request | Limit | Request-to-Limit Ratio |
|----------|---------|-------|----------------------|
| **CPU** | 500m | 7500m (7.5 cores) | 1:15 |
| **Memory** | 700Mi | 5000Mi (5GB) | ~1:7 |

### Customizing CoreDNS Autoscaling

You can customize the autoscaler behavior by modifying the `coredns-autoscaler` ConfigMap:

```bash
kubectl edit configmap coredns-autoscaler -n kube-system
```

Example customization:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-autoscaler
  namespace: kube-system
data:
  linear: |-
    {
      "coresPerReplica": 256,      # Default: 512
      "nodesPerReplica": 4,        # Default: 5
      "min": 2,                    # Minimum replicas
      "max": 10,                   # Maximum replicas (add this if needed)
      "preventSinglePointFailure": true
    }
```

> **Important:** Simply increasing the number of CoreDNS pods without addressing the root cause of OOM issues might only provide a temporary fix. Always investigate why DNS queries are spiking (application behavior, ndots configuration, etc.).

### Disabling VPA for CoreDNS

If you prefer to manage CoreDNS resources manually:

```yaml
# Set VPA update mode to Off
# Then customize resources in the CoreDNS Deployment
kubectl edit deployment coredns -n kube-system
```

---

## 12. LocalDNS — Advanced Low-Latency DNS

### What is LocalDNS? (New Feature — 2024-2025)

**LocalDNS** is an advanced AKS feature that deploys a DNS proxy **on each node** to provide highly resilient, low-latency DNS resolution. Instead of every pod sending DNS queries across the network to CoreDNS pods, queries are handled locally on the same node.

### How LocalDNS Works

```
Traditional CoreDNS:
Pod → (network hop) → CoreDNS pod → (network hop) → Upstream DNS

With LocalDNS:
Pod → LocalDNS service on same node → Cache or forward to CoreDNS → Upstream DNS
```

Each AKS node runs a **LocalDNS systemd service**. Pods send DNS queries to this local service (`169.254.20.10` by default), which:
1. Checks its local cache
2. If cache miss, forwards to cluster CoreDNS pods via **TCP** (more reliable than UDP)
3. Returns the answer to the pod

### Benefits of LocalDNS

| Benefit | Explanation |
|---------|-------------|
| **Reduced latency** | DNS queries resolved locally without network hops to CoreDNS pods |
| **Avoid conntrack races** | LocalDNS queries use TCP to CoreDNS, avoiding UDP conntrack table exhaustion |
| **Better reliability** | Even if CoreDNS pods are under heavy load, local cache reduces impact |
| **Customizable behavior** | Use `kubeDNSOverrides` and `vnetDNSOverrides` to control DNS routing |

### When to Enable LocalDNS

- Large clusters (100+ nodes)
- High DNS query volumes
- Applications experiencing DNS timeouts
- Clusters with conntrack table exhaustion issues

> **Note:** LocalDNS setup instructions are in the official AKS documentation. This is an advanced feature and requires careful planning for migration from traditional CoreDNS.

---

## 13. The CoreDNS Azure DNS Plugin

CoreDNS has an official plugin specifically for **Azure DNS**: `https://coredns.io/plugins/azure/`

This plugin enables CoreDNS to serve DNS records directly from an **Azure DNS Zone** (public or private). Instead of using External-DNS to sync records, CoreDNS itself reads them directly from Azure.

### What it Supports
The Azure plugin supports ALL DNS record types that Azure DNS supports:
`A`, `AAAA`, `CNAME`, `MX`, `NS`, `PTR`, `SOA`, `SRV`, `TXT`

**Note:** `NS` record type is NOT supported for Azure **Private** DNS (only for public zones).

### Configuration Syntax

```
azure RESOURCE_GROUP:ZONE... {
    tenant TENANT_ID
    client CLIENT_ID
    secret CLIENT_SECRET
    subscription SUBSCRIPTION_ID
    environment ENVIRONMENT
    fallthrough [ZONES...]
    access private
}
```

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `RESOURCE_GROUP:ZONE` | Azure resource group and DNS zone name. Example: `rg-dns:mycompany.com` |
| `tenant` | Azure AD Tenant ID |
| `client` | Service Principal Client ID |
| `secret` | Service Principal Client Secret |
| `subscription` | Azure Subscription ID |
| `environment` | Azure cloud (`AzurePublicCloud`, `AzureChinaCloud`, `AzureUSGovernmentCloud`) |
| `fallthrough` | If a record is not found, pass to next plugin |
| `access private` | Use Azure Private DNS instead of public DNS |

### Example Corefile Block
```
mycompany.com:53 {
    azure rg-dns:mycompany.com {
        tenant 16b3c013-d300-468d-ac64-7eda0820b6d3
        client 9cc6c0d1-99a3-4d86-9df4-a84df55b8232
        secret <REDACTED>
        subscription 82f6d75e-85f4-434a-ab74-5dddd9fa8910
        environment AzurePublicCloud
        fallthrough
    }
    cache 30
    errors
}
```

> **Important Limitation:** The Azure DNS plugin is NOT included by default in AKS CoreDNS builds. To use it, you would need to build a custom CoreDNS image with the plugin included (advanced/unsupported by AKS).

### Azure DNS Plugin vs External-DNS

| Aspect | CoreDNS Azure Plugin | External-DNS |
|--------|---------------------|-------------|
| **What it does** | CoreDNS READS from Azure DNS Zone | Kubernetes controller WRITES to Azure DNS Zone |
| **Direction** | Azure DNS → Pod resolution | Kubernetes Service → Azure DNS record |
| **Use case** | Serve existing Azure DNS records to pods | Auto-create DNS records for Services/Ingresses |
| **Record management** | Read-only (CoreDNS just serves them) | Read+Write (creates/deletes records) |
| **When to use** | You have records in Azure DNS you want pods to resolve | You want Kubernetes to auto-manage DNS records |

They serve different purposes and are often used together.

---

## 14. Applying and Restarting CoreDNS Changes

### Step 1: Apply the ConfigMap
```bash
kubectl apply -f coredns-custom.yaml
```

### Step 2: Restart CoreDNS Pods
The `reload` plugin watches for Corefile changes, but changes to the `coredns-custom` ConfigMap are not always picked up immediately. A manual restart ensures changes take effect right away:

```bash
kubectl rollout restart deployment coredns -n kube-system
```

### Step 3: Wait for Rollout to Complete
```bash
kubectl rollout status deployment coredns -n kube-system
# Output: deployment "coredns" successfully rolled out
```

### Step 4: Verify CoreDNS is Running
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
# NAME                       READY   STATUS    RESTARTS   AGE
# coredns-77f75ff65d-abc12   1/1     Running   0          30s
# coredns-77f75ff65d-def34   1/1     Running   0          30s
```

---

## 15. Verifying CoreDNS Behavior

### Method 1: nslookup from Inside a Pod

```bash
# Create a test pod with DNS utilities
kubectl run test-dns --image=busybox:1.28 --rm -it -- sh

# Inside the pod:
nslookup nginx.default.svc.cluster.local
# Server:    10.0.0.10
# Address:   10.0.0.10#53
# Name:      nginx.default.svc.cluster.local
# Address:   10.0.197.10

# Test custom domain (if configured)
nslookup nginx.default.aks.com
# Server:    10.0.0.10
# Address:   10.0.0.10#53
# Name:      nginx.default.aks.com
# Address:   10.0.197.10
```

### Method 2: Check CoreDNS Logs

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

If you enabled the `log` plugin, every DNS query appears in the logs:
```
[INFO] 10.244.0.5:48392 - "A IN nginx.default.svc.cluster.local. udp 47 false 512" NOERROR qr,aa,rd 106 0.000123456s
```

Breaking this down:
- `10.244.0.5:48392` — pod IP and source port that made the query
- `A IN` — query type (A record) and class (IN = Internet)
- `nginx.default.svc.cluster.local.` — the queried name
- `udp` — protocol
- `NOERROR` — successful response
- `qr,aa,rd` — DNS flags (query/response, authoritative answer, recursion desired)
- `0.000123456s` — query processing time

### Method 3: Describe the CoreDNS Deployment

```bash
kubectl describe deployment coredns -n kube-system
```

Look for the `Args:` section — this shows all the command-line arguments CoreDNS is running with (though most configuration is in the Corefile).

---

## 16. Common Use Cases with Full YAML Examples

### Use Case 1: Custom Internal Domain for All Services

**Goal:** Instead of `myapp.default.svc.cluster.local`, use `myapp.mycompany.internal`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  custom-domain.override: |
    rewrite stop {
      name regex (.*)\.mycompany\.internal\.$ {1}.svc.cluster.local.
      answer name (.*)\.svc\.cluster\.local\.$ {1}.mycompany.internal.
    }
```

**Test:**
```bash
nslookup nginx.default.mycompany.internal
# Returns the ClusterIP of the nginx service in the default namespace
```

### Use Case 2: Forward Corporate Domain to On-Prem DNS

**Goal:** Queries for `*.corp.local` go to corporate DNS at `192.168.1.10`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  corporate-dns.server: |
    corp.local:53 {
      errors
      cache 30
      forward . 192.168.1.10
    }
```

**Test:**
```bash
nslookup fileserver.corp.local
# Forwarded to 192.168.1.10, returns the corporate DNS answer
```

### Use Case 3: Change Default Upstream DNS to Google DNS

**Goal:** All external queries (google.com, github.com) use Google DNS instead of Azure DNS

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  google-dns.override: |
    forward . 8.8.8.8 8.8.4.4
```

**Test:**
```bash
nslookup google.com
# Forwarded to 8.8.8.8, returns Google's public IP
```

### Use Case 4: Add Static DNS Records with the Hosts Plugin

**Goal:** Create a static mapping for `database.local → 10.0.50.100` without creating a Kubernetes Service

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  custom-hosts.server: |
    database.local:53 {
      errors
      hosts {
        10.0.50.100 database.local
        fallthrough
      }
      cache 30
    }
```

**Test:**
```bash
nslookup database.local
# Returns: 10.0.50.100
```

### Use Case 5: Enable Detailed Query Logging (Debugging)

**Goal:** Log every DNS query for troubleshooting

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  logging.override: |
    log  # Logs every query to stdout
```

After applying and restarting, watch the logs:
```bash
kubectl logs -n kube-system -l k8s-app=kube-dns -f
```

You'll see every single DNS query made by every pod in the cluster.

⚠️ **Warning:** This generates A LOT of log data. Only enable temporarily for debugging, then remove it.

---

## 17. Troubleshooting CoreDNS

### Problem 1: DNS Queries Time Out

**Symptoms:**
```bash
nslookup google.com
# Server:    10.0.0.10
# Address:   10.0.0.10#53
# ** server can't find google.com: NXDOMAIN
```

**Common Causes:**
1. CoreDNS pods are not running
2. The upstream DNS server (in `forward` directive) is unreachable
3. Network policy is blocking DNS traffic

**Troubleshooting Steps:**
```bash
# Check CoreDNS pod health
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs for errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100

# Test DNS from the CoreDNS pod itself
kubectl exec -n kube-system coredns-77f75ff65d-abc12 -- nslookup google.com
```

### Problem 2: Custom Domain Not Resolving

**Symptoms:**
```bash
nslookup myapp.custom.com
# ** server can't find myapp.custom.com: NXDOMAIN
```

**Common Causes:**
1. ConfigMap not applied
2. CoreDNS not restarted after ConfigMap change
3. Syntax error in rewrite regex

**Troubleshooting Steps:**
```bash
# Verify the ConfigMap exists and has your changes
kubectl get configmap coredns-custom -n kube-system -o yaml

# Restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system

# Check for CoreDNS startup errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50 | grep -i error
```

### Problem 3: Azure Internal Names Not Resolving (After Changing Upstream DNS)

**Symptoms:**
```bash
# From a pod:
nslookup myvm.internal.cloudapp.net
# ** server can't find myvm.internal.cloudapp.net: NXDOMAIN
```

**Cause:** You changed the default `forward` to a custom DNS server, but that server doesn't know about Azure's internal domain names.

**Solution:** Add a dedicated server block for Azure internal names:
```yaml
data:
  azure-internal.server: |
    internal.cloudapp.net:53 {
      errors
      cache 30
      forward . 168.63.129.16
    }
```

### Problem 4: Slow DNS Resolution

**Symptoms:** DNS queries take 3-5 seconds to resolve

**Common Causes:**
1. Cache disabled or too low TTL
2. Upstream DNS server is slow
3. Too many ndots causing unnecessary search list attempts

**Solutions:**
```yaml
# Increase cache time
data:
  performance.override: |
    cache 300  # Cache for 5 minutes instead of 30 seconds
```

Or adjust `ndots` in pod's DNS config:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "1"  # Reduce from default of 5
  containers:
    - name: app
      image: myapp:latest
```

### Problem 5: Uneven CoreDNS Load Distribution (2024-2025 Issue)

**Symptoms:** One or two CoreDNS pods show significantly higher CPU/memory usage than others

**Cause:** Kubernetes UDP load balancing uses a five-tuple hash. If applications reuse the same source port for DNS queries, all queries route to the same CoreDNS pod.

**Solution (From Microsoft Docs):**
- Adjust application DNS connection handling
- Reduce DNS connection Time to Live (TTL)
- Randomize connection creation
- Consider enabling **LocalDNS** for better distribution

### Problem 6: Invalid Search Domain Queries (Azure VNet + ndots Issue)

**Symptoms:** DNS resolution delays, invalid queries in CoreDNS logs

**Cause:** Azure configures `<VNET_ID>.<REGION>.internal.cloudapp.net` as a search domain. Combined with Kubernetes' `ndots:5`, this results in invalid completion queries (e.g., `mcr.microsoft.com.vnetid.region.internal.cloudapp.net`).

**CoreDNS Mitigation (Built-in):**
AKS CoreDNS already includes a filter that rejects invalid queries with more than 7 labels:

```
# Query with 6 labels: aks12345.myvnetid.myregion.internal.cloudapp.net ← accepted
# Query with 8 labels: mcr.microsoft.com.vnetid.region.internal.cloudapp.net ← rejected
```

**Custom Override (if needed):**
```yaml
data:
  override-block.server: |
    internal.cloudapp.net:53 {
      errors
      cache 30
      forward . 168.63.129.16
    }
```

---

## 18. Summary Cheat Sheet

### Quick Reference Table

| Task | ConfigMap Key Type | Example |
|------|-------------------|---------|
| Custom domain rewrite | `.override` | Rewrite `*.mycompany.com` → `*.svc.cluster.local` |
| Forward specific domain | `.server` | `corp.local` queries → forward to `10.0.0.5` |
| Change default upstream DNS | `.override` | `forward . 8.8.8.8` |
| Add static DNS records | `.server` | `hosts { 10.0.0.1 myhost.local }` |
| Enable query logging | `.override` | `log` |

### ConfigMap Template

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  # Use .override to modify the default server block
  my-override.override: |
    forward . 8.8.8.8
  
  # Use .server to add a new server block for a specific domain
  my-domain.server: |
    example.com:53 {
      errors
      cache 30
      forward . 10.0.0.5
    }
```

### Essential Commands

```bash
# View current CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# View custom config
kubectl get configmap coredns-custom -n kube-system -o yaml

# Apply changes
kubectl apply -f coredns-custom.yaml

# Restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system

# Watch restart progress
kubectl rollout status deployment coredns -n kube-system

# View CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Test DNS from a pod
kubectl run test-dns --image=busybox:1.28 --rm -it -- nslookup nginx.default.svc.cluster.local
```

### DNS Naming Patterns

| Pattern | Example | When to Use |
|---------|---------|-------------|
| Full Kubernetes DNS | `nginx.default.svc.cluster.local` | Cross-namespace, absolute reference |
| Namespace shorthand | `nginx.default` | From any namespace |
| Service name only | `nginx` | Same namespace only |
| Custom domain | `nginx.mycompany.internal` | After rewrite rule configured |

---

## References & Further Reading

- [CoreDNS Official Documentation](https://coredns.io/)
- [CoreDNS Plugins](https://coredns.io/plugins/)
- [Customize CoreDNS in AKS (Microsoft Docs)](https://learn.microsoft.com/en-us/azure/aks/coredns-custom)
- [Troubleshoot CoreDNS in AKS](https://learn.microsoft.com/en-us/azure/aks/coredns-troubleshoot)
- [CoreDNS Autoscaling in AKS](https://learn.microsoft.com/en-us/azure/aks/coredns-autoscale)
- [DNS in AKS (Overview)](https://learn.microsoft.com/en-us/azure/aks/dns-concepts)
- [LocalDNS in AKS](https://learn.microsoft.com/en-us/azure/aks/dns-concepts)
- [Kubernetes DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [CoreDNS Azure Plugin](https://coredns.io/plugins/azure/)
- [CoreDNS GitHub Repository](https://github.com/coredns/coredns)