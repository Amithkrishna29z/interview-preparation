# 🌐 Networking Concepts — Interview Questions

> 🎯 Essential networking knowledge for Full Stack Java Developer interviews!

---

## Table of Contents
1. [OSI Model](#osi-model)
2. [TCP vs UDP](#tcp-vs-udp)
3. [TCP Three-Way Handshake](#tcp-three-way-handshake)
4. [IP Addressing & Subnetting](#ip-addressing--subnetting)
5. [DNS — Domain Name System](#dns--domain-name-system)
6. [HTTP vs HTTPS](#http-vs-https)
7. [Ports & Protocols](#ports--protocols)
8. [Network Devices](#network-devices)
9. [Firewalls & NAT](#firewalls--nat)
10. [WebSockets & Long Polling](#websockets--long-polling)
11. [Load Balancing & CDN](#load-balancing--cdn)
12. [Quick Revision Summary](#quick-revision-summary)

---

## OSI Model

### Q1: What is the OSI Model and what are its 7 layers?

OSI (Open Systems Interconnection) is a conceptual framework that standardizes how network systems communicate. **Memory Trick:** "Please Do Not Throw Sausage Pizza Away"

| Layer | Name | Role | Protocols/Examples |
|-------|------|------|--------------------|
| **7** | **Application** | User-facing services | HTTP, HTTPS, FTP, SMTP, DNS |
| **6** | **Presentation** | Data format, encryption | SSL/TLS, JPEG, JSON |
| **5** | **Session** | Manages sessions | NetBIOS, RPC |
| **4** | **Transport** | End-to-end delivery | TCP, UDP |
| **3** | **Network** | Routing, IP addressing | IP, ICMP, routers |
| **2** | **Data Link** | Frame delivery on same network | Ethernet, MAC, switches |
| **1** | **Physical** | Raw bits | Cables, Wi-Fi, hubs |

**Encapsulation (data sent down the stack):**
```
Layer 7: Application DATA
Layer 4: DATA + TCP/UDP Header  → Segment
Layer 3: Segment + IP Header    → Packet
Layer 2: Packet + MAC Header    → Frame
Layer 1: Frame as raw bits
```

---

### Q2: OSI vs TCP/IP model?

| Feature | OSI (7 layers) | TCP/IP (4 layers) |
|---------|----------------|-------------------|
| Purpose | Conceptual/teaching | Practical/real-world |

```
TCP/IP Layer    ←→  OSI Layers
Application     ←→  Application + Presentation + Session (7,6,5)
Transport       ←→  Transport (4)
Internet        ←→  Network (3)
Network Access  ←→  Data Link + Physical (2,1)
```

---

## TCP vs UDP

### Q3: What is the difference between TCP and UDP?

- **TCP** = phone call (connection established, reliable, in-order)
- **UDP** = postcard (no guarantee it arrives, no order)

| Feature | TCP | UDP |
|---------|-----|-----|
| **Connection** | Connection-oriented (3-way handshake) | Connectionless |
| **Reliability** | ✅ Guaranteed delivery | ❌ No guarantee |
| **Order** | ✅ In order | ❌ May be out of order |
| **Speed** | Slower | ✅ Faster |
| **Header Size** | 20 bytes | 8 bytes |
| **Use Cases** | HTTP/S, FTP, Email, SSH | DNS, video streaming, gaming, VoIP |

```
TCP Flow:
Client ──SYN──────────────→ Server
Client ←──SYN-ACK───────── Server
Client ──ACK──────────────→ Server
[Connection Established — data flows with ACKs]

UDP Flow:
Client ──DATA─────────────→ Server  (no handshake, no ACK)
```

---

## TCP Three-Way Handshake

### Q4: Explain the TCP Three-Way Handshake

Before data is sent, TCP establishes a connection in 3 steps:

```
Step 1 — SYN:     Client → Server: "I want to connect. My seq = X."
Step 2 — SYN-ACK: Server → Client: "OK, I acknowledge X. My seq = Y."
Step 3 — ACK:     Client → Server: "Got it, I acknowledge Y. Let's go!"
```

**TCP Four-Way Termination:**
```
Client ──FIN──→ Server   (done sending)
Client ←──ACK── Server
Client ←──FIN── Server   (server done too)
Client ──ACK──→ Server   (connection closed)
```

---

## IP Addressing & Subnetting

### Q5: IPv4 vs IPv6?

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Length | 32 bits | 128 bits |
| Format | Decimal `192.168.1.1` | Hex `2001:db8::1` |
| Addresses | ~4.3 billion | ~340 undecillion |
| NAT needed? | ✅ Yes | ❌ No |

### Q6: Private vs Public IP?

```
Private (inside home/office network):
  10.0.0.0    – 10.255.255.255
  172.16.0.0  – 172.31.255.255
  192.168.0.0 – 192.168.255.255

Public: assigned by ISP, unique on the internet.
Your router has both: a private IP inside, a public IP facing the internet.
```

### Q7: What is a Subnet Mask?

A subnet mask splits an IP into **network** and **host** parts.

```
IP:     192.168.1.100
Mask:   255.255.255.0  (/24)
→ Network: 192.168.1   (all devices here are on the same network)
→ Host:    .100        (254 usable hosts on this subnet)
```

---

## DNS — Domain Name System

### Q8: How does DNS work?

DNS is the internet's phone book: you know the name (`google.com`), DNS gives you the number (IP).

```
1. Browser cache → 2. OS cache → 3. Recursive Resolver (ISP)
→ 4. Root DNS → 5. TLD server (.com) → 6. Authoritative DNS
→ Returns IP → Resolver caches it → Browser connects
```

**DNS Record Types:**

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Domain → IPv4 | `google.com → 142.250.80.46` |
| **AAAA** | Domain → IPv6 | `google.com → 2607:f8b0::200e` |
| **CNAME** | Alias → domain | `www.google.com → google.com` |
| **MX** | Mail server | `gmail.com → mail.google.com` |
| **TXT** | Text info (SPF, verification) | — |
| **PTR** | Reverse DNS (IP → domain) | — |

---

## HTTP vs HTTPS

### Q9: HTTP vs HTTPS?

| Feature | HTTP | HTTPS |
|---------|------|-------|
| **Port** | 80 | 443 |
| **Encryption** | ❌ Plain text | ✅ TLS encrypted |
| **Certificate** | ❌ Not required | ✅ SSL/TLS cert needed |
| **SEO** | Lower | Higher (Google prefers HTTPS) |

**TLS Handshake (simplified):**
```
1. Client → Server: "Here are TLS versions & ciphers I support."
2. Server → Client: "Here's my SSL Certificate (contains public key)."
3. Client verifies certificate against trusted CA.
4. Client encrypts pre-master secret with server's public key → Server decrypts.
5. Both derive the same symmetric session key → all traffic encrypted.
```

> **CA** = trusted third party that signs certificates (e.g., Let's Encrypt, DigiCert).

---

## Ports & Protocols

### Q10: Important well-known ports?

A port is like an apartment number — the IP gets you to the building, the port gets you to the right service.

| Port | Protocol | Description |
|------|----------|-------------|
| **22** | SSH | Secure remote login |
| **25** | SMTP | Sending email |
| **53** | DNS | Domain Name System |
| **80** | HTTP | Web traffic (unencrypted) |
| **443** | HTTPS | Secure web traffic |
| **3306** | MySQL | MySQL database |
| **5432** | PostgreSQL | PostgreSQL database |
| **6379** | Redis | Redis cache |
| **8080** | HTTP Alt | Default Spring Boot dev port |
| **27017** | MongoDB | MongoDB database |

---

## Network Devices

### Q11: Hub vs Switch vs Router?

| Device | Layer | What it does |
|--------|-------|--------------|
| **Hub** | L1 Physical | Broadcasts to ALL ports |
| **Switch** | L2 Data Link | Sends to specific device via MAC address |
| **Router** | L3 Network | Routes packets between different networks |

### Q12: Forward Proxy vs Reverse Proxy?

**Forward Proxy** (client-side): hides the client's IP, bypasses geo-restrictions, caches responses.

**Reverse Proxy** (server-side): load balancing, SSL termination, hides backend servers.
> Examples: Nginx, Apache, AWS ALB, Cloudflare

---

## Firewalls & NAT

### Q13: What is a Firewall?

A firewall is a security guard that allows or blocks network traffic based on rules.

```
Types:
1. Packet Filtering — checks IP, port, protocol
2. Stateful        — tracks connection state
3. WAF (Web App)   — inspects HTTP, can block SQL injection

Example rules:
ALLOW inbound  TCP 443   (HTTPS in)
ALLOW inbound  TCP 22    (SSH from specific IP)
DENY  inbound  TCP 3306  (block MySQL from internet)
DENY  inbound  any       (block everything else)
```

### Q14: What is NAT?

NAT lets many devices with private IPs share one public IP.

```
Laptop  192.168.1.10 ─┐
Phone   192.168.1.11 ─┤── Router (NAT) ── Public IP: 103.22.45.67 ── Internet
Tablet  192.168.1.12 ─┘

Router replaces private IP with public IP on outbound, reverses on inbound.
```

---

## WebSockets & Long Polling

### Q15: Short Polling vs Long Polling vs WebSockets?

| | Short Polling | Long Polling | WebSocket |
|--|--------------|-------------|-----------|
| How | Client asks repeatedly on interval | Client asks; server holds until data ready | Persistent full-duplex connection |
| Overhead | High (many requests) | Medium | Low after initial handshake |
| Use case | Simple notifications | Chat without WebSocket support | Real-time chat, live dashboards, gaming |

**WebSocket upgrade handshake:**
```
Client → Server: GET /ws HTTP/1.1, Upgrade: websocket
Server → Client: HTTP 101 Switching Protocols
[Now either side can send at any time]
```

**Spring Boot WebSocket example:**
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
        registry.setApplicationDestinationPrefixes("/app");
    }
}

@Controller
public class ChatController {

    @MessageMapping("/chat")
    @SendTo("/topic/messages")
    public Message sendMessage(Message message) {
        return message; // Broadcast to all subscribers
    }
}
```

---

## Load Balancing & CDN

### Q16: What is Load Balancing?

A load balancer distributes incoming traffic across multiple servers so no single server is overwhelmed.

**Algorithms:**
```
Round Robin       — cycle through servers in order
Least Connections — route to server with fewest active connections
IP Hash           — same client IP always hits same server (sticky sessions)
Weighted          — assign traffic % per server
```

**Layer 4 vs Layer 7:**
```
L4 (Transport):   routes by IP + port only — fast
L7 (Application): routes by URL, headers, cookies — smarter
  e.g., /api/users → User Service, /api/orders → Order Service
```

### Q17: What is a CDN?

A CDN caches static assets (images, CSS, JS) at edge servers worldwide so users get content from the nearest location.

```
Benefits: lower latency, less load on origin, DDoS protection, high availability
Examples: Cloudflare, AWS CloudFront, Akamai
```

---

## Quick Revision Summary

### OSI Model — 7 Layers
```
7 - Application   → HTTP, HTTPS, FTP, DNS, SMTP
6 - Presentation  → SSL/TLS, encoding/encryption
5 - Session       → Session management
4 - Transport     → TCP (reliable), UDP (fast)
3 - Network       → IP addressing, routing
2 - Data Link     → MAC addresses, switches
1 - Physical      → Cables, Wi-Fi signals
```

### TCP vs UDP
| | TCP | UDP |
|--|-----|-----|
| Reliable? | ✅ Yes | ❌ No |
| Speed | Slower | ✅ Faster |
| Use | HTTP, SSH, FTP | DNS, streaming, gaming |

### Must-Know Ports
```
22   = SSH
53   = DNS
80   = HTTP
443  = HTTPS
3306 = MySQL
5432 = PostgreSQL
6379 = Redis
8080 = Spring Boot (default dev)
```

### Key Definitions
```
DNS       = Phone book of the internet (domain → IP)
NAT       = Lets many private IPs share one public IP
Firewall  = Security guard for network traffic
Proxy     = Intermediary between client and server
CDN       = Caches content at edge servers worldwide
WebSocket = Persistent, full-duplex, real-time connection
```

### Interview Tips
1. **OSI Model** — Know all 7 layers + which protocols belong to each
2. **TCP vs UDP** — TCP = reliable, UDP = fast; know when to use each
3. **3-Way Handshake** — SYN → SYN-ACK → ACK (memorize this)
4. **HTTPS** — TLS, port 443, certificate validated by CA
5. **DNS** — lookup chain: Browser cache → OS → Resolver → Root → TLD → Authoritative
6. **Ports** — SSH (22), DNS (53), HTTP (80), HTTPS (443), MySQL (3306), PostgreSQL (5432)
7. **Load Balancing** — Round Robin, Least Connections; Layer 4 vs Layer 7
8. **WebSockets** — persistent full-duplex vs HTTP polling; use cases: chat, live data
