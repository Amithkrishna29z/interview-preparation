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
Layer 3 — Network       → IP, routing, ICMP
Layer 2 — Data Link     → Ethernet, MAC addresses, switches
Layer 1 — Physical      → Cables, fiber, radio waves
```

**Memory trick**: "All People Seem To Need Data Processing"

### TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| **Connection** | Connection-oriented (3-way handshake) | Connectionless |
| **Reliability** | Guaranteed delivery, ordering | No guarantees |
| **Speed** | Slower (overhead) | Faster |
| **Use cases** | HTTP, HTTPS, SSH, databases | DNS, streaming, gaming, VoIP |

### TCP 3-Way Handshake

```
Client                    Server
  |──── SYN (seq=x) ────────→|   Client wants to connect
  |←─ SYN-ACK (seq=y,ack=x+1)|   Server acknowledges
  |──── ACK (ack=y+1) ──────→|   Client acknowledges
  |═══════ Data Transfer ════|
  |──── FIN ────────────────→|   Teardown (4-way)
```

### Common Ports to Know

| Port | Service | Port | Service |
|---|---|---|---|
| 22 | SSH | 443 | HTTPS |
| 53 | DNS | 3306 | MySQL |
| 80 | HTTP | 5432 | PostgreSQL |
| 25 | SMTP | 6379 | Redis |
| 6443 | Kubernetes API | 9092 | Kafka |

---

## IP Addressing & Subnetting

### CIDR Notation

| CIDR | Subnet Mask | Hosts | Use Case |
|---|---|---|---|
| /16 | 255.255.0.0 | 65,534 | VPC |
| /24 | 255.255.255.0 | 254 | Standard subnet |
| /28 | 255.255.255.240 | 14 | Small subnet |
| /32 | 255.255.255.255 | 1 | Single host |

**Formula**: Hosts = 2^(32-prefix) - 2

### Private IP Ranges (RFC 1918)

```
10.0.0.0/8          → 10.0.0.0 – 10.255.255.255
172.16.0.0/12       → 172.16.0.0 – 172.31.255.255
192.168.0.0/16      → 192.168.0.0 – 192.168.255.255
```

Not routable on the public internet — used internally in VPCs and corporate networks.

### VPC Subnetting Example

```
10.0.0.0/16  (65,536 IPs)
├── 10.0.1.0/24  → Public Subnet AZ-a
├── 10.0.2.0/24  → Public Subnet AZ-b
├── 10.0.10.0/24 → Private Subnet AZ-a
├── 10.0.11.0/24 → Private Subnet AZ-b
└── 10.0.20.0/24 → Database Subnet
```

### NAT (Network Address Translation)

Many private IPs share one public IP using different ports (PAT/Masquerade — the most common type). A NAT Gateway sits at the edge and rewrites the source IP on outbound packets.

---

## DNS (Domain Name System)

### DNS Resolution Process

```
Browser requests: www.example.com

1. Check local cache (browser, OS)
2. Ask Recursive Resolver (ISP / 8.8.8.8)
3. Resolver asks Root DNS → "ask .com TLD"
4. Resolver asks .com TLD → "ask ns1.example.com"
5. Resolver asks ns1.example.com → "93.184.216.34"
6. Resolver caches result (TTL) and returns IP
```

### DNS Record Types

| Record | Purpose | Example |
|---|---|---|
| **A** | IPv4 address | `example.com → 93.184.216.34` |
| **AAAA** | IPv6 address | `example.com → 2606:2800::1` |
| **CNAME** | Alias to another domain | `www → example.com` |
| **MX** | Mail server | `mail.example.com` |
| **TXT** | Arbitrary text (SPF, DKIM) | `"v=spf1 include:gmail.com"` |
| **PTR** | Reverse DNS (IP → hostname) | `34.216.184.93 → example.com` |

### TTL (Time To Live)

Best practice before changing A records:
1. Lower TTL to 60s (wait for old TTL to expire first)
2. Make the DNS change
3. After stable, raise TTL back to 3600s

### DNS in Kubernetes

```
Service "my-service" in namespace "default":
  → my-service.default.svc.cluster.local

Cross-namespace:
  → my-service.production.svc.cluster.local

Resolved by CoreDNS running in kube-system namespace.
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

### HTTP Status Codes

```
2xx — Success         200 OK, 201 Created, 204 No Content
3xx — Redirect        301 Moved Permanently, 304 Not Modified
4xx — Client Error    400 Bad Request, 401 Unauthorized, 403 Forbidden,
                      404 Not Found, 429 Too Many Requests
5xx — Server Error    500 Internal Server Error, 502 Bad Gateway,
                      503 Service Unavailable, 504 Gateway Timeout
```

### TLS Handshake (HTTPS)

```
Client                         Server
  |─── ClientHello ───────────→|  (TLS version, cipher suites)
  |←── Certificate ────────────|  (server's public cert)
  |  [Client verifies cert against trusted CAs]
  |─── ClientKeyExchange ─────→|  (pre-master secret, encrypted)
  |─── Finished (encrypted) ──→|
  |←── Finished (encrypted) ───|
  |════ Encrypted Application Data ═|
```

### Certificate Chain

```
Root CA (trusted by OS/browser)
  └── Intermediate CA
        └── Server Certificate (example.com)
```

| Type | Validates | Use Case |
|---|---|---|
| **DV** | Domain only | Personal sites, Let's Encrypt |
| **OV** | Domain + organization | Business websites |
| **Wildcard** | `*.example.com` | All subdomains |

---

## Load Balancing

### L4 vs L7 Load Balancers

| Type | OSI Layer | Routes On | Examples |
|---|---|---|---|
| **L4** | Transport | IP + port | AWS NLB |
| **L7** | Application | URL, headers, cookies | AWS ALB, NGINX |

### Load Balancing Algorithms

| Algorithm | Best For |
|---|---|
| **Round Robin** | Equal server capacity |
| **Least Connections** | Varied request duration |
| **IP Hash** | Session persistence |
| **Weighted Round Robin** | Different server capacity |

### Session Persistence

```
Problem: User session on Server A, next request hits Server B → session lost.

Solutions:
1. Sticky sessions: LB routes user to same server (risky if server dies)
2. Centralized session store (Redis): Best practice
3. Stateless + JWT: No server-side session at all
```

### NGINX as Reverse Proxy / Load Balancer

```nginx
upstream backend {
    least_conn;
    server 10.0.1.10:3000 weight=3;
    server 10.0.1.11:3000 weight=1;
    server 10.0.1.12:3000 backup;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## Firewalls & Security Groups

### Security Groups (AWS)

Stateful virtual firewalls at the instance/ENI level. If inbound traffic is allowed, the response is automatically allowed — no explicit outbound rule needed.

```
Inbound Rules:
│ SSH   │ 22  │ TCP │ 10.0.0.0/8 (VPN) │
│ HTTP  │ 80  │ TCP │ 0.0.0.0/0        │
│ HTTPS │ 443 │ TCP │ 0.0.0.0/0        │
│ App   │3000 │ TCP │ sg-alb-id (ALB)  │
```

### Network ACLs (NACLs)

Stateless firewalls at the subnet level. Rules evaluated in number order (first match wins). You must explicitly allow both inbound and outbound, including ephemeral ports (1024-65535) for responses.

### Security Groups vs NACLs

| Feature | Security Group | NACL |
|---|---|---|
| Level | Instance/ENI | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and deny |
| Evaluation | All rules checked | First match wins |

---

## Virtual Private Cloud (VPC)

### VPC Architecture (AWS)

```
Region: us-east-1 — VPC: 10.0.0.0/16
┌─────────────────────────────────────┐
│  AZ: us-east-1a     AZ: us-east-1b  │
│  Public  10.0.1.0   Public 10.0.2.0 │
│  [NAT GW][ALB]      [NAT GW][ALB]   │
│  Private 10.0.10.0  Private10.0.11.0│
│  [App Servers]      [App Servers]   │
│  DB      10.0.20.0  DB    10.0.21.0 │
│  [RDS Primary]      [RDS Replica]   │
│  [Internet Gateway][Route Tables]   │
└─────────────────────────────────────┘
```

### VPC Key Components

| Component | Purpose |
|---|---|
| **Internet Gateway (IGW)** | Allows VPC ↔ internet |
| **NAT Gateway** | Private subnet → internet (outbound only) |
| **Route Table** | Determines where traffic is directed |
| **VPC Peering** | Connect two VPCs privately |
| **Transit Gateway** | Hub-and-spoke for many VPCs + on-prem |
| **VPC Endpoint** | Private connection to AWS services (no internet) |

### Route Tables

```
Public Subnet:
  10.0.0.0/16 → local
  0.0.0.0/0   → igw-xxxx  (Internet Gateway)

Private Subnet:
  10.0.0.0/16 → local
  0.0.0.0/0   → nat-xxxx  (NAT Gateway)

Database Subnet:
  10.0.0.0/16 → local     (no internet route)
```

### VPC Peering

- Non-overlapping CIDR blocks required
- Route table entries must be added in both VPCs
- **Not transitive**: A↔B, B↔C does NOT mean A↔C
- For many VPCs, use Transit Gateway instead

---

## VPN & Private Connectivity

### Site-to-Site VPN vs Direct Connect

| | Site-to-Site VPN | Direct Connect |
|---|---|---|
| Path | IPSec over public internet | Dedicated private line |
| Latency | ~100-200ms | ~10ms |
| Cost | Low | High |
| Use case | Small/medium workloads | Large transfers, real-time apps |

### PrivateLink / VPC Endpoints

```
Without Endpoint:  EC2 → Internet → S3  (egress cost, leaves VPC)
With Endpoint:     EC2 → VPC Endpoint → S3  (free, stays on AWS network)

Gateway Endpoints:  S3, DynamoDB
Interface Endpoints: Most other AWS services
```

---

## CDN (Content Delivery Network)

### How a CDN Works

User in London requests `example.com/image.jpg` → hits nearest CDN edge (London). Cache HIT: returns in ~10ms. Cache MISS: fetches from origin (~200ms), caches for future requests.

### CDN Benefits

- **Reduced latency**: Content served from nearby edge server
- **Reduced origin load**: Most requests served from cache
- **DDoS protection**: Absorbs traffic at the edge
- **SSL termination**: TLS handled at edge

### Cache Control Headers

```http
Cache-Control: max-age=86400            # Cache 1 day
Cache-Control: no-store                 # Never cache (sensitive data)
Cache-Control: immutable, max-age=31536000  # Hashed assets — never changes

# Validation with ETag
ETag: "686897696a7c876b7e"
If-None-Match: "686897696a7c876b7e"  →  304 Not Modified
```

---

## Kubernetes Networking

### The Kubernetes Networking Model

Every pod gets a unique, routable IP. Pods communicate with each other without NAT, regardless of which node they're on.

### Kubernetes Service Types

```
ClusterIP (default):  internal only — Client pod → ClusterIP → Pod
NodePort:             external → Node:30080 → ClusterIP → Pod
LoadBalancer:         external → Cloud LB → NodePort → Pod
Headless:             no load balancing; DNS returns all pod IPs (used for StatefulSets)
ExternalName:         maps service to external DNS name (CNAME)
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
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
          service: { name: api-v1, port: { number: 80 } }
      - path: /v2
        pathType: Prefix
        backend:
          service: { name: api-v2, port: { number: 80 } }
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
# Allow only api pods to reach database on 5432
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

### CNI Plugins

| CNI | Key Feature |
|---|---|
| **Flannel** | Simple, VXLAN overlay |
| **Calico** | High performance, NetworkPolicy support |
| **Cilium** | Fastest (eBPF), L7 policy |
| **AWS VPC CNI** | Pod IPs are real VPC IPs (no overlay) |

---

## Service Mesh

### What is a Service Mesh?

A service mesh handles service-to-service communication via **sidecar proxies** (Envoy) — adding mTLS, retries, and observability without changing application code.

```
Without: Service A ──────────────────────── Service B
With:    Service A → Envoy ──── Envoy → Service B
                     (mTLS, retries, metrics)
```

### Service Mesh Features

| Feature | Description |
|---|---|
| **mTLS** | Both services authenticate; all traffic encrypted |
| **Traffic management** | Canary releases, circuit breaking, A/B testing |
| **Observability** | Automatic metrics, traces, logs for all service calls |
| **Retry / timeout** | Automatic retries with backoff |

**Use when**: many microservices needing consistent mTLS and observability.
**Skip when**: simple apps with few services — adds ~50MB RAM and ~5ms latency per hop.

---

## Network Protocols Reference

### WebSocket

HTTP upgrades to a persistent bidirectional TCP connection. Used for real-time apps: chat, live dashboards, gaming, stock tickers.

```
Client → GET /ws HTTP/1.1, Upgrade: websocket
Server → 101 Switching Protocols
════ Persistent TCP connection — both sides can push messages ════
```

### gRPC

HTTP/2 + Protocol Buffers (binary). ~7x smaller payload and ~10x faster than REST. Strongly typed with code generation. Requires HTTP/2.

```protobuf
service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
  rpc ListUsers (Empty) returns (stream UserResponse);  // server streaming
}
```

### CORS (Cross-Origin Resource Sharing)

Browser blocks cross-origin requests unless the server responds with the correct headers. For non-simple requests (e.g. POST with JSON), the browser first sends an OPTIONS preflight.

```http
# Server response required:
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

---

## Common Interview Questions

### Q1: What is the difference between a Security Group and NACL?
Security Groups are **stateful** instance-level firewalls — inbound allow automatically permits the response. NACLs are **stateless** subnet-level firewalls — you must explicitly allow both directions, including ephemeral ports. Use Security Groups for most cases; NACLs add a subnet-level layer of defense.

### Q2: What is the purpose of a NAT Gateway?
It lets private-subnet instances make outbound internet requests (e.g., download packages, call APIs) while blocking inbound connections from the internet. Traffic flows out through the NAT Gateway's public IP; responses are tracked and returned.

### Q3: What is the difference between L4 and L7 load balancers?
L4 routes based on IP and port only — fast, no content inspection (AWS NLB). L7 routes based on HTTP content — URL path, headers, cookies — enabling path-based routing and SSL termination (AWS ALB, NGINX). L7 is more flexible but adds latency.

### Q4: What is DNS TTL and why does it matter for deployments?
TTL controls how long resolvers cache a record. Before a DNS change, lower TTL to 60s and wait for the old TTL to expire. Then make the change — propagation takes ~60s. Afterward, raise TTL back to avoid excessive DNS traffic.

### Q5: What is mTLS and how is it different from regular TLS?
Regular TLS: only the server presents a certificate. mTLS (Mutual TLS): both client and server present certificates and verify each other. Used in service meshes and microservice-to-microservice calls to ensure only authorized services can communicate.

### Q6: How does Kubernetes service discovery work?
CoreDNS runs as pods in `kube-system` and creates DNS entries for every Service. `my-service` in namespace `production` resolves to `my-service.production.svc.cluster.local`, which maps to the Service's ClusterIP. kube-proxy then forwards traffic to a backend pod.

### Q7: What is VPC peering and what are its limitations?
VPC peering creates a private AWS-backbone connection between two VPCs. Key limitations: (1) non-transitive — A↔B and B↔C does not mean A can reach C; (2) CIDRs must not overlap; (3) route tables must be updated in both VPCs. Use Transit Gateway for many VPCs.

### Q8: What is the difference between HTTP/1.1, HTTP/2, and HTTP/3?
HTTP/1.1: one request per connection, head-of-line blocking. HTTP/2: multiplexing (multiple streams over one TCP connection), header compression. HTTP/3: uses QUIC (UDP-based) — eliminates TCP head-of-line blocking and has faster connection setup on lossy networks.

### Q9: What is a CDN and when should you use one?
A CDN caches content at edge locations near users. Use it for static assets (JS, CSS, images), global audiences, DDoS protection, or reducing origin load. Skip it for highly personalized content or small single-region apps.

### Q10: What is a service mesh and when do you need one?
A service mesh (e.g., Istio) uses sidecar proxies to provide mTLS, observability, and traffic management across all services without code changes. Use it when you have many microservices needing consistent security and observability. Avoid it for small deployments — it adds significant operational overhead.

---

## Quick Reference — Network Debugging

```bash
# Connectivity
ping -c 4 <host>                      # Basic reachability
traceroute <host>                     # Hop-by-hop path
nc -zv <host> <port>                  # Test TCP port
curl -v https://example.com           # HTTP with verbose headers

# DNS
dig example.com A                     # DNS query
dig @8.8.8.8 example.com             # Query specific DNS server
dig example.com +trace               # Full resolution trace

# Local network
ip addr / ifconfig                    # Network interfaces
ip route / route -n                   # Routing table
ss -tlnp                              # Listening ports

# TLS
openssl s_client -connect example.com:443
```

---

*Last updated: 2026-06-18*
