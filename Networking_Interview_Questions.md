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

**Easy Explanation:** OSI (Open Systems Interconnection) is a conceptual framework that standardizes how different network systems communicate. Think of it as a layered cake — each layer has a specific job.

**Memory Trick:** **"Please Do Not Throw Sausage Pizza Away"**

| Layer | Name | Role | Protocols/Examples |
|-------|------|------|--------------------|
| **7** | **Application** | User-facing services | HTTP, HTTPS, FTP, SMTP, DNS |
| **6** | **Presentation** | Data format, encryption, compression | SSL/TLS, JPEG, JSON, XML |
| **5** | **Session** | Manages sessions/connections | NetBIOS, RPC, SQL sessions |
| **4** | **Transport** | End-to-end delivery, error control | TCP, UDP |
| **3** | **Network** | Routing, IP addressing | IP, ICMP, ARP, routers |
| **2** | **Data Link** | Frame delivery on same network | Ethernet, MAC addresses, switches |
| **1** | **Physical** | Raw bits over wire/wireless | Cables, Wi-Fi signals, hubs |

```
Data travels DOWN when sending:
  Application → Presentation → Session → Transport → Network → Data Link → Physical
                                                                      (sent over wire)
Data travels UP when receiving:
  Physical → Data Link → Network → Transport → Session → Presentation → Application
```

**What each layer adds (Encapsulation):**
```
Layer 7: Application DATA
Layer 4: DATA + TCP/UDP Header  → called "Segment"
Layer 3: Segment + IP Header   → called "Packet"
Layer 2: Packet + MAC Header   → called "Frame"
Layer 1: Frame as raw bits     → "Bits"
```

---

### Q2: What is the difference between OSI and TCP/IP model?

| Feature | OSI Model (7 layers) | TCP/IP Model (4 layers) |
|---------|----------------------|--------------------------|
| Layers | 7 | 4 |
| Purpose | Conceptual/teaching | Practical/real-world |
| Used in | Theory | Internet (actual implementation) |

**TCP/IP Model mapping:**
```
TCP/IP Layer          ←→  OSI Layers
──────────────────────────────────────
Application           ←→  Application + Presentation + Session (7,6,5)
Transport             ←→  Transport (4)
Internet              ←→  Network (3)
Network Access        ←→  Data Link + Physical (2,1)
```

---

## TCP vs UDP

### Q3: What is the difference between TCP and UDP?

**Easy Explanation:**
- **TCP** = Like a phone call (connection established, reliable, in-order)
- **UDP** = Like sending a postcard (no guarantee it arrives, no order)

| Feature | TCP | UDP |
|---------|-----|-----|
| **Connection** | Connection-oriented (3-way handshake) | Connectionless |
| **Reliability** | ✅ Guaranteed delivery | ❌ No guarantee |
| **Order** | ✅ Packets arrive in order | ❌ May arrive out of order |
| **Speed** | Slower (overhead for reliability) | ✅ Faster (no overhead) |
| **Error Checking** | ✅ Yes (retransmits lost packets) | Basic checksum only |
| **Flow Control** | ✅ Yes | ❌ No |
| **Header Size** | 20 bytes | 8 bytes |
| **Use Cases** | HTTP, HTTPS, FTP, Email, SSH | DNS, Video streaming, Gaming, VoIP |

**When to use TCP:**
- Web browsing (HTTP/HTTPS)
- File downloads (FTP)
- Email (SMTP, IMAP)
- SSH connections
- Any time data accuracy matters

**When to use UDP:**
- Live video/audio streaming (YouTube Live, Zoom)
- Online gaming (speed > accuracy)
- DNS lookups (quick request-response)
- IoT sensor data

```
TCP Flow:
Client ──SYN──────────────→ Server
Client ←──SYN-ACK───────── Server
Client ──ACK──────────────→ Server
[Connection Established]
Client ──DATA─────────────→ Server
Client ←──ACK─────────────  Server

UDP Flow:
Client ──DATA─────────────→ Server  (no handshake, no ACK)
```

---

## TCP Three-Way Handshake

### Q4: Explain the TCP Three-Way Handshake

**Easy Explanation:** Before TCP sends data, it must establish a connection using a 3-step process — like confirming a phone call is connected before talking.

```
Step 1 — SYN (Synchronize):
  Client → Server: "Hey! I want to connect. My sequence number is X."

Step 2 — SYN-ACK (Synchronize-Acknowledge):
  Server → Client: "OK! I acknowledge X. My sequence number is Y."

Step 3 — ACK (Acknowledge):
  Client → Server: "Got it! I acknowledge Y. Connection is established!"

[Now data can be transferred]
```

**Real Example:**
```
Client                    Server
  |                          |
  |—— SYN (seq=100) ────────→|
  |                          |
  |←── SYN-ACK (seq=200, ack=101) ──|
  |                          |
  |—— ACK (ack=201) ────────→|
  |                          |
  |===  Connection Open  ====|
  |                          |
  |—— HTTP GET /index.html ─→|
  |←── HTTP 200 OK ──────────|
```

**TCP Four-Way Termination (Connection Close):**
```
Client ──FIN──→ Server   (I'm done sending)
Client ←──ACK── Server   (OK, noted)
Client ←──FIN── Server   (I'm done too)
Client ──ACK──→ Server   (OK, bye!)
[Connection Closed]
```

---

## IP Addressing & Subnetting

### Q5: What is an IP address? Difference between IPv4 and IPv6?

**IPv4:**
```
Format: 4 octets separated by dots
Example: 192.168.1.100
Range: 0.0.0.0 to 255.255.255.255
Total addresses: ~4.3 billion (2^32)
```

**IPv6:**
```
Format: 8 groups of 4 hex digits separated by colons
Example: 2001:0db8:85a3:0000:0000:8a2e:0370:7334
Total addresses: ~340 undecillion (2^128)
Reason: IPv4 addresses are exhausted
```

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Length | 32 bits | 128 bits |
| Format | Decimal (192.168.1.1) | Hexadecimal (2001:db8::1) |
| Addresses | ~4.3 billion | ~340 undecillion |
| Header Size | 20 bytes | 40 bytes |
| NAT needed? | ✅ Yes (address shortage) | ❌ No |

### Q6: What is the difference between Private and Public IP?

```
Private IP (used inside your home/office network):
  10.0.0.0    – 10.255.255.255     (Class A)
  172.16.0.0  – 172.31.255.255     (Class B)
  192.168.0.0 – 192.168.255.255    (Class C)

Public IP (used on the internet, unique globally):
  Everything else (assigned by ISP)

Example:
  Your laptop:         192.168.1.5   (private — only your router knows this)
  Your router:         192.168.1.1   (private inside) + 103.22.45.67 (public outside)
  Google's server:     142.250.80.46 (public)
```

### Q7: What is a Subnet Mask?

**Easy Explanation:** A subnet mask divides an IP address into the **network part** and the **host part**.

```
IP Address:   192.168.1.100
Subnet Mask:  255.255.255.0  (or /24 in CIDR notation)

Network part: 192.168.1   (first 24 bits)
Host part:    .100         (last 8 bits)

This means:
  - All devices with 192.168.1.x are on the same network
  - Can have 254 hosts (2^8 - 2 = 254, subtract network/broadcast)
```

---

## DNS — Domain Name System

### Q8: How does DNS work?

**Easy Explanation:** DNS is like a phone book for the internet. You know the name (google.com), DNS tells you the number (IP address).

```
You type: www.google.com in browser

Step 1: Check browser cache (DNS result stored?)
Step 2: Check OS cache (/etc/hosts file)
Step 3: Ask your Recursive Resolver (usually your ISP's DNS server)
Step 4: Recursive Resolver asks Root DNS Server (knows who handles .com)
Step 5: Root says: "Ask the .com TLD server"
Step 6: TLD server says: "Ask Google's Authoritative DNS server"
Step 7: Google's DNS says: "www.google.com = 142.250.80.46"
Step 8: Recursive Resolver caches the result + returns IP to browser
Step 9: Browser connects to 142.250.80.46
```

**DNS Record Types:**

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Domain → IPv4 address | `google.com → 142.250.80.46` |
| **AAAA** | Domain → IPv6 address | `google.com → 2607:f8b0::200e` |
| **CNAME** | Alias → another domain | `www.google.com → google.com` |
| **MX** | Mail server for domain | `gmail.com → mail.google.com` |
| **TXT** | Text info (SPF, verification) | Various verification strings |
| **NS** | Name servers for domain | `google.com → ns1.google.com` |
| **PTR** | Reverse DNS (IP → domain) | `142.250.80.46 → google.com` |

---

## HTTP vs HTTPS

### Q9: What is the difference between HTTP and HTTPS?

| Feature | HTTP | HTTPS |
|---------|------|-------|
| **Port** | 80 | 443 |
| **Encryption** | ❌ Plain text | ✅ Encrypted (TLS/SSL) |
| **Security** | ❌ Data can be intercepted | ✅ Secure |
| **Certificate** | ❌ Not required | ✅ SSL/TLS certificate needed |
| **Speed** | Slightly faster | Slightly slower (encryption overhead) |
| **SEO** | Lower ranking | Higher ranking (Google prefers HTTPS) |
| **Use case** | Internal networks only | Any public website |

**How HTTPS/TLS Handshake works:**
```
1. Client → Server: "Hello! Here are TLS versions & cipher suites I support."
2. Server → Client: "Hello! Here's my SSL Certificate (contains public key)."
3. Client: Verifies certificate is valid (signed by trusted CA).
4. Client → Server: Encrypts a "pre-master secret" using server's public key.
5. Server: Decrypts using private key. Both now have the same session key.
6. Both: Generate symmetric session key.
7. All further communication is encrypted with this session key.
```

**Certificate Authority (CA):**
```
CA = trusted third party that signs SSL certificates
Examples: Let's Encrypt (free), DigiCert, Comodo
Browser trusts the certificate because it's signed by a known CA.
```

---

## Ports & Protocols

### Q10: What are the important well-known ports?

**Easy Explanation:** A port is like an apartment number in a building. The IP gets you to the building (server), the port gets you to the right apartment (service).

| Port | Protocol | Description |
|------|----------|-------------|
| **20/21** | FTP | File Transfer Protocol (data/control) |
| **22** | SSH | Secure Shell (remote login) |
| **23** | Telnet | Insecure remote login (avoid!) |
| **25** | SMTP | Sending email |
| **53** | DNS | Domain Name System |
| **67/68** | DHCP | Dynamic Host Configuration Protocol |
| **80** | HTTP | Web traffic (unencrypted) |
| **110** | POP3 | Receiving email (older) |
| **143** | IMAP | Receiving email (modern) |
| **443** | HTTPS | Secure web traffic |
| **3306** | MySQL | MySQL database |
| **5432** | PostgreSQL | PostgreSQL database |
| **6379** | Redis | Redis cache |
| **8080** | HTTP Alt | Common dev server port |
| **27017** | MongoDB | MongoDB database |

**Spring Boot defaults:**
```
Spring Boot app:    8080
MySQL:              3306
PostgreSQL:         5432
Redis:              6379
MongoDB:            27017
```

---

## Network Devices

### Q11: What is the difference between a Hub, Switch, and Router?

| Device | Layer | Intelligence | What it does |
|--------|-------|-------------|--------------|
| **Hub** | Layer 1 (Physical) | None | Broadcasts data to ALL ports (dumb) |
| **Switch** | Layer 2 (Data Link) | MAC addresses | Sends data to SPECIFIC device using MAC |
| **Router** | Layer 3 (Network) | IP addresses | Routes data between different networks |

```
Hub:    PC-A sends data → Hub broadcasts to ALL PCs (A, B, C, D)
                         (B, C, D get unnecessary data)

Switch: PC-A sends to PC-B → Switch sends ONLY to PC-B
                              (C, D don't receive it)

Router: Home network (192.168.1.x) ↔ Internet (public IPs)
        Routes packets between different networks
```

### Q12: What is a Proxy Server?

**Forward Proxy** (client-side):
```
Client → Proxy → Internet

Uses:
- Hide client's real IP
- Bypass geo-restrictions
- Cache responses
- Content filtering (corporate networks block sites)
```

**Reverse Proxy** (server-side):
```
Internet → Reverse Proxy → Backend Servers

Uses:
- Load balancing (distribute traffic to multiple servers)
- SSL termination (handle HTTPS at proxy, HTTP to backend)
- Caching
- Security (hides backend servers)

Examples: Nginx, Apache, AWS ALB, Cloudflare
```

---

## Firewalls & NAT

### Q13: What is a Firewall?

**Easy Explanation:** A firewall is a security guard that decides which network traffic is allowed in or out based on rules.

```
Types:
1. Packet Filtering — Checks IP, port, protocol (simple, fast)
2. Stateful — Tracks connection state (knows if packet belongs to existing connection)
3. Application (WAF) — Inspects application data (HTTP, can block SQL injection)

Example Firewall Rules:
ALLOW  inbound  TCP port 443     (HTTPS traffic in)
ALLOW  inbound  TCP port 22      (SSH from specific IP only)
DENY   inbound  TCP port 3306    (block MySQL from internet)
ALLOW  outbound any              (allow all outbound traffic)
DENY   inbound  any              (block everything else)
```

### Q14: What is NAT (Network Address Translation)?

**Easy Explanation:** NAT allows many devices with private IPs to share one public IP when accessing the internet.

```
Home Network:
  Laptop:  192.168.1.10  ─┐
  Phone:   192.168.1.11  ─┤── Router (NAT) ── Public IP: 103.22.45.67 ── Internet
  Tablet:  192.168.1.12  ─┘

When Laptop sends request to google.com:
  Router replaces 192.168.1.10 → 103.22.45.67
  Google sees 103.22.45.67 (not your private IP)
  Response comes back to 103.22.45.67
  Router translates back to 192.168.1.10
```

---

## WebSockets & Long Polling

### Q15: What is the difference between WebSockets, Long Polling, and Short Polling?

**Short Polling:**
```
Client asks "any new messages?" every 5 seconds
Server: "No" / "No" / "No" / "Yes, here's the message"

Problem: Lots of unnecessary requests, wasteful
```

**Long Polling:**
```
Client asks "any new messages?" → Server holds the request open
Server: waits... waits... [new message arrives] → sends response
Client immediately sends next request
Better but still HTTP overhead
```

**WebSockets:**
```
Client and Server establish a persistent, full-duplex connection
Either side can send data at any time — no request needed

HTTP Upgrade handshake:
Client → Server: GET /ws HTTP/1.1, Upgrade: websocket
Server → Client: HTTP 101 Switching Protocols

Now both can send at any time:
Client → Server: "Hello"
Server → Client: "New message for you!"
Server → Client: "Another message!"
```

**Java WebSocket with Spring Boot:**
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
        return message;  // Broadcast to all subscribers
    }
}
```

**Use Cases:**
| Technology | Use Case |
|-----------|----------|
| Short Polling | Simple notifications (acceptable delay) |
| Long Polling | Chat apps without WebSocket support |
| WebSocket | Real-time chat, live dashboards, gaming, stock prices |

---

## Load Balancing & CDN

### Q16: What is Load Balancing?

**Easy Explanation:** A load balancer distributes incoming traffic across multiple servers so no single server gets overwhelmed.

```
Without Load Balancer:
  All 10,000 users → Server 1  (overloaded, crashes)

With Load Balancer:
  10,000 users → Load Balancer → Server 1 (3,333 users)
                              → Server 2 (3,333 users)
                              → Server 3 (3,334 users)
```

**Load Balancing Algorithms:**
```
1. Round Robin:       Request 1→S1, Request 2→S2, Request 3→S3, Request 4→S1...
2. Least Connections: Route to server with fewest active connections
3. IP Hash:           Same client IP always goes to same server (sticky sessions)
4. Weighted:          Server 1 gets 50%, Server 2 gets 30%, Server 3 gets 20%
5. Random:            Random server selection
```

**Layer 4 vs Layer 7 Load Balancer:**
```
Layer 4 (Transport): Routes based on IP + Port only (fast, no content inspection)
Layer 7 (Application): Routes based on URL, headers, cookies (smarter, slower)

Example L7:
  /api/users  → User Service servers
  /api/orders → Order Service servers
  /static/*   → CDN
```

### Q17: What is a CDN (Content Delivery Network)?

```
Problem: User in India requests images from server in USA → slow!

CDN Solution:
  Copy static assets (images, CSS, JS, videos) to servers worldwide
  User in India gets content from nearest CDN node (e.g., Singapore)
  → Much faster!

CDN Benefits:
  1. Reduced latency (content served from nearest edge server)
  2. Reduced load on origin server
  3. DDoS protection
  4. High availability

Examples: Cloudflare, AWS CloudFront, Akamai, Fastly
```

---

## Quick Revision Summary

### 🔑 OSI Model — 7 Layers (Top to Bottom)
```
7 - Application   → HTTP, HTTPS, FTP, DNS, SMTP
6 - Presentation  → SSL/TLS, encoding/encryption
5 - Session       → Session management
4 - Transport     → TCP (reliable), UDP (fast)
3 - Network       → IP addressing, routing
2 - Data Link     → MAC addresses, switches
1 - Physical      → Cables, Wi-Fi signals
```

### 🔑 TCP vs UDP At a Glance
| | TCP | UDP |
|--|-----|-----|
| Reliable? | ✅ Yes | ❌ No |
| Speed | Slower | ✅ Faster |
| Use | HTTP, SSH, FTP | DNS, streaming, gaming |

### 🔑 Must-Know Ports
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

### 🔑 HTTP vs HTTPS
```
HTTP  = Port 80, plain text, insecure
HTTPS = Port 443, encrypted via TLS, secure
```

### 🔑 Key Definitions
```
DNS      = Phone book of the internet (domain → IP)
NAT      = Lets many private IPs share one public IP
Firewall = Security guard for network traffic
Proxy    = Intermediary between client and server
CDN      = Caches content at edge servers worldwide
WebSocket= Persistent, full-duplex, real-time connection
```

### 📝 Interview Tips
1. **OSI Model** — Know all 7 layers + which protocols belong to each
2. **TCP vs UDP** — Know when to use each; TCP = reliable, UDP = fast
3. **3-Way Handshake** — SYN → SYN-ACK → ACK (memorize this)
4. **HTTPS** — Know it uses TLS, port 443, and how certificate validation works
5. **DNS** — Know the lookup chain: Browser cache → OS → Resolver → Root → TLD → Authoritative
6. **Ports** — MySQL (3306), PostgreSQL (5432), HTTP (80), HTTPS (443), SSH (22)
7. **Load Balancing** — Know Round Robin, Least Connections, and what Layer 4 vs Layer 7 means
8. **WebSockets** — Know the difference from HTTP polling, key use cases (chat, live data)
