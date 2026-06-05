# Networking Concepts for Cloud & DevOps - Interview Preparation

---

## Table of Contents
1. [Networking Fundamentals](#networking-fundamentals)
2. [IP Addressing & Subnetting](#ip-addressing--subnetting)
3. [DNS (Domain Name System)](#dns-domain-name-system)
4. [HTTP/HTTPS & TLS](#httphttps--tls)
5. [Load Balancing](#load-balancing)
6. [Firewalls & Security Groups](#firewalls--security-groups)
7. [Virtual Private Cloud (VPC)](#virtual-private-cloud-vpc)
8. [VPN & Private Connectivity](#vpn--private-connectivity)
9. [CDN (Content Delivery Network)](#cdn-content-delivery-network)
10. [Kubernetes Networking](#kubernetes-networking)
11. [Service Mesh](#service-mesh)
12. [Network Protocols Reference](#network-protocols-reference)
13. [Common Interview Questions](#common-interview-questions)

---

## Networking Fundamentals

### OSI Model (7 Layers)

```
Layer 7 — Application   → HTTP, HTTPS, DNS, FTP, SMTP, WebSocket
Layer 6 — Presentation  → TLS/SSL, encryption, compression
Layer 5 — Session       → Session management, authentication
Layer 4 — Transport     → TCP, UDP (ports, reliability)
Layer 3 — Network       → IP, routing, ICMP (addresses, routing)
Layer 2 — Data Link     → Ethernet, MAC addresses, switches
Layer 1 — Physical      → Cables, fiber, radio waves
```

**Memory trick**: "All People Seem To Need Data Processing"

### TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| **Connection** | Connection-oriented (3-way handshake) | Connectionless |
| **Reliability** | Guaranteed delivery, ordering | No guarantees |
| **Error checking** | Yes (retransmission) | Basic checksum only |
| **Speed** | Slower (overhead) | Faster |
| **Use cases** | HTTP, HTTPS, SSH, FTP, databases | DNS, streaming, gaming, VoIP |
| **Header size** | 20 bytes | 8 bytes |

### TCP 3-Way Handshake

```
Client                    Server
  |                          |
  |──── SYN (seq=x) ────────→|   Client wants to connect
  |                          |
  |←─ SYN-ACK (seq=y,ack=x+1)|   Server acknowledges
  |                          |
  |──── ACK (ack=y+1) ──────→|   Client acknowledges
  |                          |
  |═══════ Data Transfer ════|
  |                          |
  |──── FIN ────────────────→|   Teardown (4-way)
```

### Common Ports to Know

| Port | Protocol | Service |
|---|---|---|
| 22 | TCP | SSH |
| 25 | TCP | SMTP (email) |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 27017 | TCP | MongoDB |
| 2181 | TCP | ZooKeeper |
| 9092 | TCP | Kafka |
| 2379/2380 | TCP | etcd (Kubernetes) |
| 6443 | TCP | Kubernetes API server |
| 10250 | TCP | kubelet |

---

## IP Addressing & Subnetting

### IPv4 Address Structure

```
192    .  168   .   1   .   10   /24
─────    ─────    ────    ────    ──
Octet 1  Octet 2  Octet 3  Octet 4  Prefix
(8 bits) (8 bits) (8 bits) (8 bits) (subnet mask)
```

### CIDR Notation

| CIDR | Subnet Mask | Hosts | Use Case |
|---|---|---|---|
| /8 | 255.0.0.0 | 16,777,214 | Large ISP |
| /16 | 255.255.0.0 | 65,534 | Large organization / VPC |
| /24 | 255.255.255.0 | 254 | Standard subnet |
| /25 | 255.255.255.128 | 126 | Split subnet |
| /28 | 255.255.255.240 | 14 | Small subnet |
| /30 | 255.255.255.252 | 2 | Point-to-point links |
| /32 | 255.255.255.255 | 1 | Single host |

**Formula**: Hosts = 2^(32-prefix) - 2 (network + broadcast reserved)

### Private IP Ranges (RFC 1918)

```
10.0.0.0/8          → 10.0.0.0 – 10.255.255.255    (Class A)
172.16.0.0/12       → 172.16.0.0 – 172.31.255.255  (Class B)
192.168.0.0/16      → 192.168.0.0 – 192.168.255.255 (Class C)
```

These are **not routable on the public internet** — used internally in VPCs, corporate networks.

### Subnetting Example

**VPC CIDR: 10.0.0.0/16** — Split into subnets:

```
10.0.0.0/16  (65,536 IPs available)
├── 10.0.1.0/24  → Public Subnet AZ-a  (254 hosts)
├── 10.0.2.0/24  → Public Subnet AZ-b  (254 hosts)
├── 10.0.3.0/24  → Public Subnet AZ-c  (254 hosts)
├── 10.0.10.0/24 → Private Subnet AZ-a (254 hosts)
├── 10.0.11.0/24 → Private Subnet AZ-b (254 hosts)
├── 10.0.12.0/24 → Private Subnet AZ-c (254 hosts)
└── 10.0.20.0/24 → Database Subnet AZ-a (254 hosts)
```

### IPv6 Basics

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
────────────────────────────────────────
128-bit address, written as 8 groups of 4 hex digits
Abbreviated: 2001:db8:85a3::8a2e:370:7334

IPv6 has no NAT — every device gets a globally unique address
/64 subnet is standard for a single subnet
```

### NAT (Network Address Translation)

```
Private Network                    Public Internet
┌─────────────────┐               ┌────────────────┐
│ 10.0.1.10       │               │                │
│ 10.0.1.11  ─────┼──→ NAT GW ───┼──→ 54.12.34.56 │
│ 10.0.1.12       │  (translates  │                │
└─────────────────┘   private to  └────────────────┘
                       public IP)
```

**Types**:
- **Static NAT**: One private IP → one public IP (1:1)
- **Dynamic NAT**: Pool of public IPs assigned dynamically
- **PAT/Masquerade**: Many private IPs → one public IP using different ports (most common)

---

## DNS (Domain Name System)

### DNS Resolution Process

```
Browser requests: www.example.com

1. Check local cache (browser, OS)
2. Ask Recursive Resolver (ISP/8.8.8.8)
3. Resolver asks Root DNS server → "I don't know, ask .com TLD"
4. Resolver asks .com TLD → "Ask ns1.example.com"
5. Resolver asks ns1.example.com (authoritative) → "93.184.216.34"
6. Resolver returns IP to client, caches result (TTL)

Total time: 50-200ms (first request), <1ms (cached)
```

### DNS Record Types

| Record | Purpose | Example |
|---|---|---|
| **A** | IPv4 address | `example.com → 93.184.216.34` |
| **AAAA** | IPv6 address | `example.com → 2606:2800::1` |
| **CNAME** | Alias to another domain | `www → example.com` |
| **MX** | Mail server | `example.com MX mail.example.com` |
| **TXT** | Arbitrary text (SPF, DKIM, verification) | `"v=spf1 include:gmail.com"` |
| **NS** | Authoritative name servers | `example.com NS ns1.example.com` |
| **SOA** | Start of Authority (zone info) | TTL, serial number |
| **PTR** | Reverse DNS (IP → hostname) | `34.216.184.93 → example.com` |
| **SRV** | Service location | `_http._tcp.example.com` |
| **CAA** | Certificate Authority Authorization | `0 issue "letsencrypt.org"` |

### TTL (Time To Live)

```
Low TTL (60s):   Frequent DNS lookups, quick propagation, more DNS traffic
High TTL (86400s): Cached longer, less traffic, slow propagation on change

Best practice before changing A records:
1. Lower TTL to 60s (wait for old TTL to expire)
2. Make the DNS change
3. After stabilized, raise TTL back to 3600s
```

### DNS in Kubernetes

```
Service "my-service" in namespace "default":
  → my-service.default.svc.cluster.local

Pod resolution:
  → my-pod.default.pod.cluster.local

Cross-namespace:
  → my-service.production.svc.cluster.local

Resolved by CoreDNS running as a pod in kube-system namespace
```

---

## HTTP/HTTPS & TLS

### HTTP Methods

| Method | Idempotent | Safe | Use Case |
|---|---|---|---|
| GET | Yes | Yes | Read resource |
| POST | No | No | Create resource |
| PUT | Yes | No | Replace resource entirely |
| PATCH | No | No | Partial update |
| DELETE | Yes | No | Delete resource |
| HEAD | Yes | Yes | GET without body (check headers) |
| OPTIONS | Yes | Yes | CORS preflight, capability check |

### HTTP Status Codes

```
1xx — Informational   100 Continue, 101 Switching Protocols
2xx — Success         200 OK, 201 Created, 204 No Content
3xx — Redirect        301 Moved Permanently, 302 Found, 304 Not Modified
4xx — Client Error    400 Bad Request, 401 Unauthorized, 403 Forbidden,
                      404 Not Found, 429 Too Many Requests
5xx — Server Error    500 Internal Server Error, 502 Bad Gateway,
                      503 Service Unavailable, 504 Gateway Timeout
```

### HTTP Headers

```http
# Request headers
GET /api/users HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
Content-Type: application/json
Accept: application/json
User-Agent: Mozilla/5.0...
X-Request-ID: abc-123

# Response headers
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=3600, public
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
```

### TLS Handshake (HTTPS)

```
Client                         Server
  |                                |
  |─── ClientHello ───────────────→|  (TLS version, cipher suites)
  |                                |
  |←── ServerHello ────────────────|  (chosen cipher, session ID)
  |←── Certificate ────────────────|  (server's public cert)
  |←── ServerHelloDone ────────────|
  |                                |
  |  [Client verifies cert against trusted CAs]
  |                                |
  |─── ClientKeyExchange ─────────→|  (pre-master secret, encrypted with server pubkey)
  |─── ChangeCipherSpec ──────────→|
  |─── Finished (encrypted) ──────→|
  |                                |
  |←── ChangeCipherSpec ───────────|
  |←── Finished (encrypted) ───────|
  |                                |
  |════ Encrypted Application Data ═|
```

### TLS Certificates

```
Certificate Chain:
Root CA (trusted by OS/browser)
  └── Intermediate CA
        └── Server Certificate (example.com)

Check certificate:
openssl s_client -connect example.com:443
openssl x509 -in cert.pem -noout -text
```

### Certificate Types

| Type | Validates | Use Case |
|---|---|---|
| **DV (Domain Validated)** | Domain ownership only | Personal sites, Let's Encrypt |
| **OV (Organization Validated)** | Domain + organization | Business websites |
| **EV (Extended Validation)** | Domain + full legal entity | Banks, e-commerce |
| **Wildcard** | `*.example.com` | All subdomains |
| **SAN/Multi-domain** | Multiple domains in one cert | CDNs, multi-domain |

---

## Load Balancing

### Load Balancer Types

| Type | OSI Layer | Operates On | Examples |
|---|---|---|---|
| **L4 (Transport)** | Layer 4 | TCP/UDP (IP + port) | AWS NLB, HAProxy L4 |
| **L7 (Application)** | Layer 7 | HTTP/HTTPS (URL, headers, cookies) | AWS ALB, NGINX, Traefik |

### Load Balancing Algorithms

| Algorithm | How it Works | Best For |
|---|---|---|
| **Round Robin** | Distribute in sequence | Equal server capacity |
| **Weighted Round Robin** | Round Robin with weights | Different server capacity |
| **Least Connections** | Send to server with fewest active connections | Varied request duration |
| **IP Hash** | Hash client IP → always same server | Session persistence |
| **Random** | Pick random server | Simple, stateless apps |
| **Least Response Time** | Pick fastest server | Latency-sensitive apps |

### Health Checks

```
# ALB health check config (AWS)
Health check path:     /health
Protocol:              HTTP
Interval:              30 seconds
Timeout:               5 seconds
Healthy threshold:     2 consecutive successes
Unhealthy threshold:   3 consecutive failures

# Health check response
HTTP 200 = healthy
HTTP 500 / timeout = unhealthy → removed from pool
```

### Session Persistence (Sticky Sessions)

```
Problem: User session stored on Server A, but next request goes to Server B → session lost

Solutions:
1. Sticky sessions (affinity): Load balancer routes user to same server
   → Single point of failure if server dies
   
2. Centralized session store: All servers read/write to Redis
   → Best practice, no affinity needed
   
3. Stateless + JWT: No server-side session, token contains state
   → Best for modern APIs
```

### NGINX as Reverse Proxy / Load Balancer

```nginx
# /etc/nginx/nginx.conf

upstream backend {
    least_conn;                        # Algorithm
    server 10.0.1.10:3000 weight=3;   # Higher weight = more traffic
    server 10.0.1.11:3000 weight=1;
    server 10.0.1.12:3000 backup;     # Only used when others fail

    keepalive 32;                      # Connection pooling
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }

    location /static/ {
        root /var/www;
        expires 1d;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## Firewalls & Security Groups

### Security Groups (AWS)

Security groups are **stateful virtual firewalls** at the instance/ENI level.

```
Inbound Rules:
┌──────────┬────────┬───────────┬─────────────────────┐
│ Type     │ Port   │ Protocol  │ Source              │
├──────────┼────────┼───────────┼─────────────────────┤
│ SSH      │ 22     │ TCP       │ 10.0.0.0/8 (VPN)    │
│ HTTP     │ 80     │ TCP       │ 0.0.0.0/0           │
│ HTTPS    │ 443    │ TCP       │ 0.0.0.0/0           │
│ Custom   │ 3000   │ TCP       │ sg-alb-id (ALB SG)  │
└──────────┴────────┴───────────┴─────────────────────┘

Outbound Rules:
│ All traffic │ All │ All │ 0.0.0.0/0 (default) │
```

**Stateful**: If inbound traffic is allowed, the response is automatically allowed outbound (no need for explicit outbound rules for responses).

### Network ACLs (NACLs)

**Stateless** firewalls at the subnet level.

```
Inbound Rules (evaluated in order, first match wins):
Rule 100: ALLOW TCP 80    from 0.0.0.0/0
Rule 110: ALLOW TCP 443   from 0.0.0.0/0
Rule 120: ALLOW TCP 1024-65535 from 0.0.0.0/0  ← Ephemeral ports for responses
Rule *:   DENY  ALL                              ← Default deny

Outbound Rules:
Rule 100: ALLOW TCP 80    to 0.0.0.0/0
Rule 110: ALLOW TCP 443   to 0.0.0.0/0
Rule 120: ALLOW TCP 1024-65535 to 0.0.0.0/0
Rule *:   DENY  ALL
```

**NACLs vs Security Groups**:

| Feature | Security Group | NACL |
|---|---|---|
| Level | Instance/ENI | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and deny |
| Evaluation | All rules checked | First match wins |
| Default | Deny all | Allow all |

---

## Virtual Private Cloud (VPC)

### VPC Architecture (AWS)

```
Region: us-east-1
┌──────────────────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                                                │
│                                                                  │
│  ┌────────────────────────┐  ┌────────────────────────┐          │
│  │   AZ: us-east-1a       │  │   AZ: us-east-1b       │          │
│  │                        │  │                        │          │
│  │  ┌──────────────────┐  │  │  ┌──────────────────┐  │          │
│  │  │  Public Subnet   │  │  │  │  Public Subnet   │  │          │
│  │  │  10.0.1.0/24     │  │  │  │  10.0.2.0/24     │  │          │
│  │  │  [NAT GW] [ALB]  │  │  │  │  [NAT GW] [ALB]  │  │          │
│  │  └──────────────────┘  │  │  └──────────────────┘  │          │
│  │                        │  │                        │          │
│  │  ┌──────────────────┐  │  │  ┌──────────────────┐  │          │
│  │  │  Private Subnet  │  │  │  │  Private Subnet  │  │          │
│  │  │  10.0.10.0/24    │  │  │  │  10.0.11.0/24    │  │          │
│  │  │  [App Servers]   │  │  │  │  [App Servers]   │  │          │
│  │  └──────────────────┘  │  │  └──────────────────┘  │          │
│  │                        │  │                        │          │
│  │  ┌──────────────────┐  │  │  ┌──────────────────┐  │          │
│  │  │  DB Subnet       │  │  │  │  DB Subnet       │  │          │
│  │  │  10.0.20.0/24    │  │  │  │  10.0.21.0/24    │  │          │
│  │  │  [RDS Primary]   │  │  │  │  [RDS Replica]   │  │          │
│  │  └──────────────────┘  │  │  └──────────────────┘  │          │
│  └────────────────────────┘  └────────────────────────┘          │
│                                                                  │
│  [Internet Gateway]  [Route Tables]  [Security Groups]           │
└──────────────────────────────────────────────────────────────────┘
```

### VPC Key Components

| Component | Purpose |
|---|---|
| **Internet Gateway (IGW)** | Allows VPC to communicate with the internet |
| **NAT Gateway** | Allows private subnet instances to reach internet (one-way) |
| **Route Table** | Rules that determine where network traffic is directed |
| **Subnet** | Segment of VPC IP range in a specific AZ |
| **VPC Peering** | Connect two VPCs (same or different accounts/regions) |
| **Transit Gateway** | Hub-and-spoke for connecting many VPCs and on-prem |
| **VPC Endpoint** | Private connection to AWS services without internet |
| **ENI** | Elastic Network Interface — virtual NIC for EC2 |
| **EIP** | Elastic IP — static public IPv4 address |

### Route Tables

```
Public Subnet Route Table:
Destination     Target
10.0.0.0/16    local         ← VPC-internal traffic
0.0.0.0/0      igw-xxxx      ← All other traffic → Internet Gateway

Private Subnet Route Table:
Destination     Target
10.0.0.0/16    local
0.0.0.0/0      nat-xxxx      ← All other traffic → NAT Gateway

Database Subnet Route Table:
Destination     Target
10.0.0.0/16    local         ← Only internal traffic (no internet)
```

### VPC Peering

```
VPC A (10.0.0.0/16)  ←→  VPC B (172.16.0.0/16)

Requirements:
- Non-overlapping CIDR blocks
- Must add route table entries in both VPCs
- Not transitive (A↔B, B↔C does NOT mean A↔C)
```

### Transit Gateway

```
VPC A    VPC B    VPC C    On-Premises
  │        │        │          │
  └────────┴────────┴──────────┘
              Transit Gateway
              (hub, transitive routing)
```

---

## VPN & Private Connectivity

### Site-to-Site VPN

```
Corporate HQ ──── IPSec VPN Tunnel ──── AWS VPC
(Customer Gateway)                  (Virtual Private Gateway)
10.100.0.0/24                         10.0.0.0/16
```

- Encrypted tunnel over the public internet
- ~100-200ms latency
- Bandwidth limited to internet connection speed

### AWS Direct Connect

```
Corporate HQ ──── Dedicated Line ──── AWS Direct Connect ──── VPC
                  (1Gbps/10Gbps)         Location
```

- Dedicated private connection (not over internet)
- Consistent, low latency (~10ms)
- Higher bandwidth, more expensive
- Use: large data transfers, real-time applications

### PrivateLink / VPC Endpoints

```
Without Endpoint:  EC2 → Internet → S3 (leaves VPC, costs egress $)
With Endpoint:     EC2 → VPC Endpoint → S3 (stays in AWS network, free)

Gateway Endpoints: S3, DynamoDB (route table)
Interface Endpoints: Most other services (ENI in subnet)
```

---

## CDN (Content Delivery Network)

### How a CDN Works

```
User (London) → Request example.com/image.jpg
                    ↓
          CDN PoP (London edge server)
          ├── Cache HIT? → Return immediately (10ms)
          └── Cache MISS? → Fetch from Origin (200ms)
                              Cache for future requests
```

### CDN Benefits

- **Reduced latency**: Serve from geographically close server
- **Reduced origin load**: Most requests served from cache
- **DDoS protection**: Absorb traffic at the edge
- **SSL termination**: TLS handled at edge, HTTP to origin possible

### Cache Control Headers

```http
# Cache-Control options
Cache-Control: max-age=86400           # Cache for 1 day
Cache-Control: public, max-age=3600    # Public caches allowed, 1 hour
Cache-Control: private, max-age=0      # Only browser cache, no CDN
Cache-Control: no-cache                # Validate with server before using
Cache-Control: no-store               # Never cache (sensitive data)
Cache-Control: immutable, max-age=31536000  # Content never changes (hashed assets)

# ETag for validation
ETag: "686897696a7c876b7e"
If-None-Match: "686897696a7c876b7e"  → 304 Not Modified (use cache)
```

### CDN Providers

| Provider | Product | Known For |
|---|---|---|
| **Cloudflare** | Cloudflare CDN | DDoS protection, WAF, Workers |
| **AWS** | CloudFront | AWS integration, Lambda@Edge |
| **Akamai** | Intelligent Edge | Enterprise, largest network |
| **Fastly** | Fastly CDN | Real-time config, edge computing |
| **Google** | Cloud CDN | GCP integration |

---

## Kubernetes Networking

### The Kubernetes Networking Model

Kubernetes requires every pod to have a **unique, routable IP** and that:
1. All pods can communicate with all other pods without NAT
2. All nodes can communicate with pods without NAT
3. The IP a pod sees for itself = the IP others see for it

### Pod-to-Pod Communication

```
Same Node:
Pod A (10.244.1.2) → virtual ethernet (veth) → cbr0 bridge → veth → Pod B (10.244.1.3)

Different Nodes:
Pod A (10.244.1.2) → Node 1 → overlay network (VXLAN/BGP) → Node 2 → Pod B (10.244.2.5)

Overlay networks: Flannel (VXLAN), Calico (BGP/VXLAN), Weave, Cilium (eBPF)
```

### Kubernetes Service Types

```
ClusterIP (default):
  Cluster-internal only
  Client (pod) → ClusterIP (10.96.0.1) → kube-proxy → Pod

NodePort:
  Exposes service on each node's IP at a static port
  External → Node:30080 → ClusterIP → Pod
  Port range: 30000-32767

LoadBalancer:
  Provisions cloud load balancer
  External → Cloud LB (54.x.x.x) → NodePort → ClusterIP → Pod

ExternalName:
  Maps service to DNS name (CNAME)
  my-db-service → rds.amazonaws.com

Headless (ClusterIP: None):
  No load balancing, returns all pod IPs via DNS
  Used for StatefulSets (databases)
```

### Ingress

```
External Client → Ingress Controller (NGINX/Traefik) → Service → Pods

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts: [api.example.com]
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port: { number: 80 }
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port: { number: 80 }
```

### NetworkPolicy

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes: [Ingress]

---
# Allow specific ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
```

### CNI Plugins Comparison

| CNI | Network Model | Key Feature |
|---|---|---|
| **Flannel** | VXLAN overlay | Simple, easy setup |
| **Calico** | BGP (or VXLAN) | High performance, NetworkPolicy support |
| **Cilium** | eBPF | Fastest, L7 policy, observability |
| **Weave** | Mesh overlay | Simple, encrypted by default |
| **AWS VPC CNI** | Native VPC | Pod IPs are VPC IPs (no overlay) |

---

## Service Mesh

### What is a Service Mesh?

A service mesh handles **service-to-service communication** transparently — without application code changes.

```
Without Service Mesh:
Service A ────────────────────────────────── Service B
         (handles: retries, auth, certs, metrics)

With Service Mesh (Sidecar Pattern):
Service A → Proxy (Envoy) ──────── Proxy (Envoy) → Service B
              (handles: retries, mTLS, circuit breaker, observability)
```

### Service Mesh Features

| Feature | Description |
|---|---|
| **mTLS** | Mutual TLS — both sides authenticate, all traffic encrypted |
| **Traffic management** | Canary releases, A/B testing, circuit breaking at mesh level |
| **Observability** | Automatic metrics, traces, and logs for all service communication |
| **Policy** | Rate limiting, access control, quota management |
| **Retry / timeout** | Automatic retries with backoff |

### Istio Architecture

```
Data Plane:
  Service A → Envoy sidecar ────────── Envoy sidecar → Service B
                  ↕                         ↕
Control Plane:
  Istiod (Pilot + Citadel + Galley)
  - Pilot: Service discovery, traffic routing config
  - Citadel: Certificate management, mTLS
  - Galley: Config validation
```

### When to Use a Service Mesh

**Use when**:
- Dozens of microservices with complex communication
- Need mTLS between services
- Need uniform observability across all services
- Advanced traffic management (canary, fault injection)

**Don't use when**:
- Simple applications with few services
- Teams lack operational experience
- Sidecar overhead is a concern (adds ~50MB RAM and ~5ms latency per hop)

---

## Network Protocols Reference

### ICMP (Ping)

```bash
# Test connectivity
ping 8.8.8.8
ping -c 4 google.com      # 4 packets

# Path tracing
traceroute google.com      # Linux
tracert google.com         # Windows

# Diagnose: if ping works but service doesn't → app issue
#            if ping fails → network issue (firewall, routing)
```

### WebSocket

```
HTTP Upgrade → Persistent bidirectional connection

Client:                          Server:
GET /ws HTTP/1.1
Upgrade: websocket          →
Connection: Upgrade
                            ←   HTTP/1.1 101 Switching Protocols
                                Upgrade: websocket

════ Persistent TCP connection ═══
Client → Server: message     →
             ←                   Server → Client: push
```

**Use cases**: Real-time apps (chat, live dashboards, gaming, stock tickers)

### gRPC

```
Protocol: HTTP/2 + Protocol Buffers (binary)

Service definition (proto file):
service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
  rpc ListUsers (Empty) returns (stream UserResponse);   // server streaming
  rpc CreateUsers (stream UserRequest) returns (UserResponse);  // client streaming
}

vs REST:
- Strongly typed (protobuf)
- ~7x smaller payload, ~10x faster
- Code generation for all languages
- Requires HTTP/2
```

### CORS (Cross-Origin Resource Sharing)

```
Browser (origin: https://app.example.com)
     → Request to https://api.other.com

Browser enforces: API must include header:
Access-Control-Allow-Origin: https://app.example.com
(or *)

Preflight (OPTIONS) for non-simple requests:
OPTIONS /api/data HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST

Response:
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400    # Cache preflight for 1 day
```

---

## Common Interview Questions

### Q1: What is the difference between a Security Group and NACL?
**Security Groups** are stateful firewalls at the instance level — you only define inbound rules and responses are automatically allowed. **NACLs** are stateless firewalls at the subnet level — you must explicitly define both inbound and outbound rules. NACLs rules are evaluated in number order (first match wins). Use SGs for most cases; NACLs for an extra layer of subnet-level protection.

### Q2: What is the purpose of a NAT Gateway?
NAT Gateway allows instances in **private subnets** to initiate outbound connections to the internet (e.g., download packages, call APIs) while preventing the internet from initiating connections inbound to them. The response traffic is tracked and allowed back.

### Q3: What is the difference between L4 and L7 load balancers?
**L4 (Transport Layer)**: Routes based on IP address and port only — fast, no inspection of content. Good for TCP/UDP. **L7 (Application Layer)**: Routes based on HTTP content — URL path, headers, host, cookies. Enables path-based routing, sticky sessions, SSL termination. L7 is smarter but adds latency overhead.

### Q4: What is DNS TTL and why does it matter for deployments?
**TTL (Time To Live)** is how long DNS resolvers cache a record. Before making DNS changes (like a migration), lower the TTL to 60s. Wait for the current high TTL to expire (so all resolvers refresh). Make the change. Rollback if needed — with low TTL, it propagates in ~60s instead of hours.

### Q5: What is mTLS and how is it different from regular TLS?
Regular **TLS**: Only the server presents a certificate — client verifies server identity. **mTLS (Mutual TLS)**: Both server and client present certificates — each verifies the other. Used in service meshes (Istio), microservices, and API gateways to ensure only authorized services can communicate.

### Q6: How does Kubernetes service discovery work?
Kubernetes uses **CoreDNS** (a DNS server running as pods) for service discovery. When you create a Service named `my-service` in namespace `production`, CoreDNS creates a DNS entry: `my-service.production.svc.cluster.local`. Any pod can resolve this name to the service's ClusterIP, which kube-proxy then routes to a backend pod.

### Q7: What is VPC peering and what are its limitations?
VPC peering creates a **private network connection** between two VPCs. Traffic stays on AWS backbone, not the internet. Limitations: (1) **Non-transitive** — if A peers with B and B peers with C, A cannot reach C via B. (2) **Non-overlapping CIDRs** required. (3) **Route tables** must be explicitly updated in both VPCs. For many VPCs, use Transit Gateway instead.

### Q8: What is the difference between HTTP/1.1, HTTP/2, and HTTP/3?
**HTTP/1.1**: One request per TCP connection (or pipelining, rarely used), head-of-line blocking. **HTTP/2**: Multiplexing — multiple streams over one TCP connection, header compression (HPACK), server push. **HTTP/3**: Uses QUIC (UDP-based) instead of TCP — eliminates TCP head-of-line blocking, faster connection setup, better on lossy networks.

### Q9: What is a CDN and when should you use one?
A CDN (Content Delivery Network) caches content at **edge locations** geographically close to users. Use when: serving static assets (JS, CSS, images, videos), global user base, need to reduce origin server load, need DDoS protection, or need low latency worldwide. Don't use for: highly personalized content, real-time data, or small single-region apps.

### Q10: What is a service mesh and when do you need one?
A service mesh (like Istio) handles service-to-service networking using **sidecar proxies** — providing mTLS, observability, traffic management, and resilience without application code changes. You need one when: you have many microservices needing consistent security (mTLS) and observability, complex traffic routing requirements, or need to enforce policies uniformly. Not needed for simple, small-scale deployments due to added operational complexity.

---

## Quick Reference — Network Debugging

```bash
# Connectivity
ping -c 4 <host>                      # Basic reachability
traceroute <host>                     # Hop-by-hop path
telnet <host> <port>                  # Test TCP port
nc -zv <host> <port>                  # Netcat port test
curl -v https://example.com           # HTTP with verbose headers

# DNS
nslookup example.com                  # DNS lookup
dig example.com A                     # Detailed DNS query
dig @8.8.8.8 example.com             # Query specific DNS server
dig example.com +trace               # Full DNS resolution trace

# Local network
ip addr / ifconfig                    # Network interfaces
ip route / route -n                   # Routing table
ss -tlnp / netstat -tlnp             # Listening ports

# HTTP debugging
curl -o /dev/null -w "%{http_code}\n%{time_total}\n" https://example.com
curl -H "Host: myapp.com" http://10.0.1.10    # Test with custom host header
openssl s_client -connect example.com:443     # TLS inspection
```

---

*Last updated: 2026-06-05*
