# DevOps + Networking Complete Interview Preparation Guide

> **Target Audience:** Beginners to Advanced | **Goal:** Interview Readiness + Deep Conceptual Understanding

---

## Table of Contents

1. [Networking Fundamentals](#1-networking-fundamentals)
2. [OSI Model & TCP/IP](#2-osi-model--tcpip)
3. [DNS, HTTP & HTTPS](#3-dns-http--https)
4. [Routing & Switching](#4-routing--switching)
5. [Linux for DevOps](#5-linux-for-devops)
6. [Git & Version Control](#6-git--version-control)
7. [Docker & Container Networking](#7-docker--container-networking)
8. [Kubernetes Networking](#8-kubernetes-networking)
9. [CI/CD Pipelines](#9-cicd-pipelines)
10. [Terraform & Infrastructure as Code](#10-terraform--infrastructure-as-code)
11. [Ansible & Configuration Management](#11-ansible--configuration-management)
12. [Monitoring: Prometheus & Grafana](#12-monitoring-prometheus--grafana)
13. [Cloud Networking: VPC, Subnets, Security Groups](#13-cloud-networking-vpc-subnets-security-groups)
14. [Load Balancers, NAT & VPN](#14-load-balancers-nat--vpn)
15. [Real-World DevOps Networking Scenarios](#15-real-world-devops-networking-scenarios)
16. [Mock Interview: 25 Mixed Questions](#16-mock-interview-25-mixed-questions)
17. [30-Day Mastery Roadmap](#17-30-day-mastery-roadmap)

---

## 1. Networking Fundamentals

### What is a Network?

A **network** is a collection of devices (computers, servers, phones) connected together to share data and resources. The internet is the world's largest network — billions of devices communicating using agreed-upon rules called **protocols**.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **IP Address** | Unique address for each device (like a postal address) |
| **MAC Address** | Hardware-level address burned into your network card |
| **Subnet** | A logical division of a network |
| **Port** | Virtual endpoint for a specific service (HTTP=80, HTTPS=443, SSH=22) |
| **Protocol** | Rules governing how data is sent/received |
| **Bandwidth** | Maximum data transfer rate |
| **Latency** | Time delay in data transmission |
| **Packet** | A unit of data transmitted across a network |

### Basic Network Flow (Client → Server)

```
┌─────────────┐                                      ┌─────────────┐
│   Client    │                                      │   Server    │
│ 192.168.1.5 │                                      │ 10.0.0.100  │
└──────┬──────┘                                      └──────┬──────┘
       │                                                     │
       │  1. DNS Lookup: "what is google.com's IP?"          │
       │──────────────────────────────────────────►          │
       │  2. DNS Response: "142.250.80.46"                   │
       │◄──────────────────────────────────────────          │
       │                                                     │
       │  3. TCP Handshake (SYN → SYN-ACK → ACK)            │
       │◄────────────────────────────────────────►          │
       │                                                     │
       │  4. HTTP GET /index.html                            │
       │──────────────────────────────────────────►          │
       │                                                     │
       │  5. HTTP 200 OK + HTML Content                      │
       │◄──────────────────────────────────────────          │
       │                                                     │
       │  6. TCP FIN (connection close)                      │
       │◄────────────────────────────────────────►          │
└─────────────┘                                      └─────────────┘
```

### Interview Q&A

**Q1. What is the difference between a private IP and a public IP?**
> **A:** A **private IP** is used within a local network (LAN) and is not routable on the internet. Common ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. A **public IP** is globally unique and routable on the internet. Your home router has one public IP but assigns private IPs to all devices inside — this is accomplished via NAT (Network Address Translation).

**Q2. What is the difference between TCP and UDP?**
> **A:**
> - **TCP (Transmission Control Protocol):** Connection-oriented, guarantees delivery, ordered packets, flow control. Used for HTTP, HTTPS, SSH, FTP.
> - **UDP (User Datagram Protocol):** Connectionless, no delivery guarantee, faster, lower overhead. Used for DNS, video streaming, VoIP, gaming.
> **Rule of thumb:** Use TCP when accuracy matters; use UDP when speed matters.

**Q3. What is a subnet mask and why is it needed?**
> **A:** A subnet mask separates the **network** portion from the **host** portion of an IP address. Example: IP `192.168.1.10` with mask `255.255.255.0` (or `/24`) means:
> - Network: `192.168.1.0`
> - Host range: `192.168.1.1` to `192.168.1.254`
> - Broadcast: `192.168.1.255`
> Subnet masks allow routers to determine if a destination is on the local network or needs to be forwarded elsewhere.

**Q4. Explain CIDR notation.**
> **A:** CIDR (Classless Inter-Domain Routing) notation represents IP ranges compactly. `/24` means 24 bits for the network, leaving 8 bits for hosts = 256 addresses (254 usable). `/16` = 65,536 addresses. `/32` = a single host. In AWS, VPCs commonly use `/16` and subnets use `/24`.

**Q5. What happens when you type an IP address in a browser?**
> **A:** The browser skips DNS (since it's already an IP), performs a TCP three-way handshake with the server on port 80 (HTTP) or 443 (HTTPS), sends an HTTP request, and receives the response. If HTTPS, a TLS handshake occurs before the HTTP exchange.

### Real-World Use Case

> **DevOps Context:** When a Kubernetes pod can't reach an external service, a DevOps engineer checks: Is the IP private or public? Is the pod's network policy blocking egress? Does the node's subnet route to the internet? Is there a NAT gateway? This fundamental understanding of IP addresses and subnets is essential daily.

---

## 2. OSI Model & TCP/IP

### The OSI Model (Open Systems Interconnection)

The OSI model is a conceptual framework with **7 layers** that standardizes network communication. Think of it as a stack — each layer serves the one above it.

```
┌───────────────────────────────────────────────────────────────────┐
│  Layer 7 - APPLICATION    │ HTTP, HTTPS, FTP, SMTP, DNS, SSH      │
├───────────────────────────────────────────────────────────────────┤
│  Layer 6 - PRESENTATION   │ SSL/TLS, Encryption, Compression      │
├───────────────────────────────────────────────────────────────────┤
│  Layer 5 - SESSION        │ Session management, Authentication    │
├───────────────────────────────────────────────────────────────────┤
│  Layer 4 - TRANSPORT      │ TCP, UDP, Ports, Segmentation         │
├───────────────────────────────────────────────────────────────────┤
│  Layer 3 - NETWORK        │ IP, ICMP, Routing, IP Addressing      │
├───────────────────────────────────────────────────────────────────┤
│  Layer 2 - DATA LINK      │ MAC Addresses, Ethernet, Switches     │
├───────────────────────────────────────────────────────────────────┤
│  Layer 1 - PHYSICAL       │ Cables, NICs, Fiber, Wifi signals     │
└───────────────────────────────────────────────────────────────────┘

Memory Aid: "Please Do Not Throw Sausage Pizza Away" (bottom to top)
            Physical → Data Link → Network → Transport → Session → Presentation → Application
```

### TCP/IP Model (Practical 4-Layer Model)

```
┌──────────────────────────────────────────┐
│  Application Layer    (OSI 5+6+7)        │
├──────────────────────────────────────────┤
│  Transport Layer      (OSI Layer 4)      │
├──────────────────────────────────────────┤
│  Internet Layer       (OSI Layer 3)      │
├──────────────────────────────────────────┤
│  Network Access Layer (OSI 1+2)          │
└──────────────────────────────────────────┘
```

### Data Encapsulation Flow

```
Application Layer:    [  DATA  ]
Transport Layer:      [ TCP HDR |  DATA  ]           (called Segment)
Network Layer:        [ IP HDR | TCP HDR | DATA ]    (called Packet)
Data Link Layer:      [ ETH HDR | IP HDR | TCP HDR | DATA | ETH FCS ]  (called Frame)
Physical Layer:       01001010 10110101 ...           (Bits on wire)
```

### TCP Three-Way Handshake

```
Client                          Server
  │                               │
  │──────── SYN (seq=x) ─────────►│   "I want to connect"
  │                               │
  │◄─── SYN-ACK (seq=y, ack=x+1)──│   "OK, I acknowledge"
  │                               │
  │──── ACK (ack=y+1) ───────────►│   "Acknowledged, connection open"
  │                               │
  │    [Data Exchange Begins]      │
  │                               │
  │──────── FIN ─────────────────►│   "I'm done"
  │◄──────── FIN-ACK ─────────────│
```

### Interview Q&A

**Q1. What layer does a switch operate at, and what about a router?**
> **A:** A **switch** operates at Layer 2 (Data Link) — it uses MAC addresses to forward frames within a local network. A **router** operates at Layer 3 (Network) — it uses IP addresses to route packets between networks. Modern "Layer 3 switches" can do both: they switch within VLANs and route between them.

**Q2. What is the difference between OSI and TCP/IP models?**
> **A:** OSI is a theoretical 7-layer framework used for education and troubleshooting. TCP/IP is the practical 4-layer model that the internet actually runs on. OSI's Session, Presentation, and Application layers are combined into the TCP/IP Application layer.

**Q3. At which OSI layer does a firewall typically operate?**
> **A:** Traditional firewalls work at **Layer 3/4** (IP addresses and ports). Modern **stateful firewalls** track connection state at Layer 4. **Application-layer firewalls (WAF)** operate at Layer 7 and can inspect HTTP content, block SQL injection, etc. AWS Security Groups are Layer 4 stateful firewalls.

**Q4. What is the purpose of the TCP sequence number?**
> **A:** Sequence numbers allow the receiver to reassemble out-of-order packets and detect missing packets. Each byte sent has a sequence number. The receiver sends ACKs indicating the next expected byte. This is how TCP guarantees ordered, reliable delivery.

**Q5. Explain what happens during a TCP connection teardown.**
> **A:** A 4-way FIN handshake: Client sends FIN → Server ACKs → Server sends FIN → Client ACKs. The connection enters a `TIME_WAIT` state (2× MSL = ~60 seconds) to handle any delayed packets. This is why a server port might not be immediately reusable after closing.

### Real-World Use Case

> **DevOps Scenario:** A load balancer health check is failing. The DevOps engineer uses `tcpdump -i eth0 port 80` (Layer 4) and `curl -v` (Layer 7) to diagnose whether the TCP connection is being established (Layer 4 OK) but the HTTP response is failing (Layer 7 issue), narrowing down whether it's a network or application problem.

---

## 3. DNS, HTTP & HTTPS

### How DNS Works

DNS (Domain Name System) translates human-readable names like `app.example.com` into IP addresses.

```
Browser             Local Cache / OS       Resolver          Root NS      TLD NS (.com)    Auth NS
   │                      │                    │                 │               │              │
   │─ "app.example.com?"─►│                    │                 │               │              │
   │◄─ Cache Miss ─────────│                    │                 │               │              │
   │                       │                    │                 │               │              │
   │──────────────────────────"app.example.com?"──────────────────────────────────────────────► │
   │                                            │                 │               │              │
   │                                            │─ "Where is .com?"──────────────►│             │
   │                                            │◄─ "Ask this TLD NS"─────────────│             │
   │                                            │                                 │              │
   │                                            │─── "Where is example.com?" ────►│             │
   │                                            │◄─── "Ask Auth NS at x.x.x.x" ──│             │
   │                                            │                                              │
   │                                            │─────────── "app.example.com?" ──────────────►│
   │                                            │◄─────────── "IP: 93.184.216.34" ─────────────│
   │◄───────────────────────"IP: 93.184.216.34"──│
```

### Common DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Maps hostname to IPv4 | `app.example.com → 93.184.216.34` |
| **AAAA** | Maps hostname to IPv6 | `app.example.com → 2606:2800::` |
| **CNAME** | Alias to another hostname | `www → app.example.com` |
| **MX** | Mail server | `example.com → mail.example.com` |
| **TXT** | Text data (SPF, DKIM, ownership) | Used for email validation |
| **NS** | Nameserver for zone | `example.com → ns1.route53.aws` |
| **PTR** | Reverse DNS (IP → name) | Used in email anti-spam |
| **SRV** | Service location (port+protocol) | Used by Kubernetes, SIP |

### HTTP vs HTTPS

```
HTTP (Port 80):
Client ──── Plaintext request ────► Server
Client ◄─── Plaintext response ─── Server
(Anyone on the network can read this traffic!)

HTTPS (Port 443):
Client ──── TLS Handshake ─────────► Server
           (exchange certs + keys)
Client ──── Encrypted request ─────► Server
Client ◄─── Encrypted response ─── Server
(Traffic is encrypted end-to-end)
```

### TLS Handshake (Simplified)

```
Client                                    Server
  │                                         │
  │──── ClientHello (TLS version, ciphers)─►│
  │◄─── ServerHello (chosen cipher)─────────│
  │◄─── Certificate (public key)────────────│
  │                                         │
  │  [Client verifies cert against CA]      │
  │                                         │
  │──── ClientKeyExchange ─────────────────►│
  │  (encrypted with server's public key)   │
  │                                         │
  │◄────────── Both sides derive session key ─
  │                                         │
  │ ════ All further data is encrypted ════ │
```

### HTTP Status Codes

| Range | Meaning | Key Examples |
|-------|---------|--------------|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Permanent, 302 Temporary, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| 5xx | Server Error | 500 Internal Error, 502 Bad Gateway, 503 Unavailable, 504 Timeout |

### Interview Q&A

**Q1. What is DNS TTL and why does it matter in DevOps?**
> **A:** TTL (Time-To-Live) determines how long a DNS record is cached before re-querying. Low TTL (60s) = faster propagation during deployments/failovers, but more DNS queries. High TTL (86400s = 1 day) = fewer queries, but changes take longer to propagate. **DevOps tip:** Lower TTL before a major deployment or IP change, then restore it after.

**Q2. What is the difference between HTTP/1.1, HTTP/2, and HTTP/3?**
> **A:**
> - **HTTP/1.1:** One request per TCP connection (can use pipelining, but head-of-line blocking). Text-based headers.
> - **HTTP/2:** Multiplexing (multiple requests over one TCP connection), binary protocol, header compression (HPACK). Major performance improvement.
> - **HTTP/3:** Uses QUIC (UDP-based), eliminates TCP head-of-line blocking, built-in TLS 1.3, better for mobile/lossy networks.

**Q3. How does HTTPS prevent man-in-the-middle attacks?**
> **A:** HTTPS uses TLS certificates signed by trusted Certificate Authorities (CAs). When your browser connects, it verifies the server's certificate chain up to a trusted root CA. If a MITM attacker intercepts and presents their own certificate, the browser will show a warning because that cert isn't signed by a trusted CA. Certificate pinning provides even stronger guarantees.

**Q4. What is a wildcard certificate?**
> **A:** A wildcard cert (e.g., `*.example.com`) covers all first-level subdomains (`api.example.com`, `app.example.com`) but NOT nested ones (`api.dev.example.com`). In Kubernetes, wildcard certs on an ingress controller can serve multiple services under one domain.

**Q5. How does DNS-based load balancing work?**
> **A:** A single DNS name can resolve to multiple IP addresses (round-robin DNS). Each query may return a different IP. However, this is a crude form of load balancing — it doesn't account for server health or real traffic. True load balancing uses dedicated hardware/software (like AWS ALB) that distributes connections based on health checks and load.

### Real-World Use Case

> **Scenario:** After a Kubernetes ingress controller IP changes, users can't reach the app for 30 minutes. Root cause: DNS TTL was set to 3600s (1 hour). The old IP was cached. Fix: Set TTL to 60s before planned changes, verify propagation with `dig +short app.example.com @8.8.8.8`, then restore TTL post-migration.

---

## 4. Routing & Switching

### Switching (Layer 2)

A **switch** maintains a **MAC address table** mapping MAC addresses to physical ports. When a frame arrives, it looks up the destination MAC:
- **Known MAC:** Forward to specific port only
- **Unknown MAC:** Flood to all ports except the source
- **Broadcast MAC (FF:FF:FF:FF:FF:FF):** Always flood

### Routing (Layer 3)

A **router** uses a **routing table** to decide where to forward packets.

```
Routing Table (simplified):
┌────────────────┬────────────┬──────────────┬──────────┐
│ Destination    │ Subnet     │ Next Hop     │ Interface│
├────────────────┼────────────┼──────────────┼──────────┤
│ 192.168.1.0    │ /24        │ directly con │ eth0     │
│ 10.0.0.0       │ /8         │ 192.168.1.1  │ eth0     │
│ 0.0.0.0        │ /0         │ 203.0.113.1  │ eth1     │ ← Default route
└────────────────┴────────────┴──────────────┴──────────┘

Longest prefix match wins:
- Packet to 192.168.1.50 → matches /24 (more specific) → eth0
- Packet to 10.5.5.5 → matches /8 → via 192.168.1.1
- Packet to 8.8.8.8 → matches /0 (default) → via internet gateway
```

### VLANs (Virtual LANs)

VLANs logically segment a network on a single physical switch. VLAN 10 for Dev, VLAN 20 for Prod — traffic can't cross unless a router (or Layer 3 switch) explicitly routes between them.

### Interview Q&A

**Q1. What is the difference between static routing and dynamic routing?**
> **A:** **Static routing** — Routes manually configured by an admin. Simple, predictable, zero overhead. Doesn't adapt to failures. Used for small networks or specific policy routes. **Dynamic routing** — Routers exchange information using protocols (BGP, OSPF, RIP) and automatically compute best paths. Adapts to failures. Used in large networks and the internet. **BGP** is what makes internet routing work between ISPs.

**Q2. What is ARP and why is it important?**
> **A:** ARP (Address Resolution Protocol) resolves an IP address to a MAC address within a local network. When a device needs to send a packet to `192.168.1.1`, it broadcasts "Who has 192.168.1.1?" The device with that IP replies with its MAC address. The sender caches this (ARP cache). **ARP poisoning** is a MITM attack where a malicious device sends fake ARP replies to redirect traffic.

**Q3. What is OSPF and how does it work?**
> **A:** OSPF (Open Shortest Path First) is a link-state routing protocol used within an organization (IGP). Each router builds a complete map of the network (LSDB — Link State Database) and computes the shortest path using Dijkstra's algorithm. All routers in an area have an identical LSDB, ensuring consistent routing decisions. It's fast to converge and loop-free.

**Q4. Explain BGP (Border Gateway Protocol).**
> **A:** BGP is the protocol of the internet — it routes between autonomous systems (AS), like between different ISPs or cloud providers. It's path-vector based: routers advertise their known routes with the AS-path (list of ASes traversed). BGP selects routes based on policies (local preference, AS-path length, MED, etc.), not just shortest path. AWS uses BGP for Direct Connect and VPN connections.

**Q5. What is the purpose of a default gateway?**
> **A:** The default gateway is the router address configured on a host. When a host needs to send traffic to a destination not on its local subnet, it sends the packet to the default gateway, which then routes it further. In cloud environments, every subnet has an implicit router at the first IP (e.g., `10.0.1.1` for subnet `10.0.1.0/24`).

### Real-World Use Case

> **Scenario:** A Kubernetes node can reach pods on the same node but not pods on other nodes. Investigation reveals the host routing table doesn't have routes for the pod CIDRs on other nodes. Fix: Verify the CNI plugin (Flannel/Calico) has correctly programmed routes, or check if BGP peering (used by Calico) between nodes is established.

---

## 5. Linux for DevOps

### Essential Networking Commands

```bash
# View network interfaces
ip addr show                    # Modern way
ifconfig                        # Older way

# View routing table
ip route show
route -n                        # Older way

# Check connectivity
ping 8.8.8.8                   # ICMP echo
traceroute 8.8.8.8             # Path tracing
mtr 8.8.8.8                    # Combined ping + traceroute (live)

# DNS lookup
dig app.example.com            # Full DNS query info
dig +short app.example.com     # Just the IP
nslookup app.example.com       # Simpler DNS query
host app.example.com           # Quick lookup

# Port and socket info
ss -tulnp                      # Show listening ports (modern)
netstat -tulnp                 # Older equivalent
lsof -i :8080                  # What's using port 8080?

# Packet capture
tcpdump -i eth0 port 80        # Capture HTTP traffic
tcpdump -i any -w dump.pcap    # Capture all to file
wireshark                      # GUI packet analysis

# HTTP testing
curl -v https://api.example.com          # Verbose HTTP
curl -I https://api.example.com          # Headers only
wget --spider https://api.example.com    # Check URL
```

### File System & Process Commands

```bash
# Files
ls -la              # List with permissions
chmod 755 file      # Change permissions (rwxr-xr-x)
chown user:group f  # Change owner
find / -name "*.log" -mtime +7   # Find logs older than 7 days
tail -f /var/log/app.log         # Follow log in real-time
grep -r "ERROR" /var/log/        # Search for errors
awk '{print $2}' file            # Print second column
sed 's/old/new/g' file           # Replace text

# Processes
ps aux              # All running processes
top / htop          # Interactive process viewer
kill -9 <PID>       # Force kill process
systemctl status nginx           # Service status
journalctl -u nginx -f           # Service logs

# Disk / Memory
df -h               # Disk usage
du -sh /var/log/*   # Directory sizes
free -h             # Memory usage
```

### Interview Q&A

**Q1. What is the difference between `ps aux` and `top`?**
> **A:** `ps aux` gives a static snapshot of all running processes at that instant. `top` provides a live, updating view sorted by CPU usage. `htop` is a friendlier interactive version of `top`. For scripting and alerting, `ps aux | grep <process>` is common.

**Q2. Explain Linux file permissions (755, 644, etc.).**
> **A:** Permissions are three sets of rwx for Owner, Group, Others. `755` = `rwxr-xr-x` (owner: full, group+others: read+execute). `644` = `rw-r--r--` (owner: read+write, others: read only). Directories need execute bit to be "entered." SSH private keys must be `600` (owner read+write only) or SSH will refuse to use them.

**Q3. How do you troubleshoot a service that's not starting?**
> **A:** Systematic approach: (1) `systemctl status <service>` — see the error. (2) `journalctl -u <service> -n 50` — last 50 log lines. (3) Run the binary directly to see raw errors. (4) Check config file syntax (e.g., `nginx -t`). (5) Check port conflicts with `ss -tulnp`. (6) Check file permissions on config/log directories.

**Q4. What are `stdin`, `stdout`, and `stderr`?**
> **A:** Standard streams in Linux. `stdin` (fd 0): input, usually keyboard. `stdout` (fd 1): normal output. `stderr` (fd 2): error output. Redirections: `> file` redirects stdout, `2> file` redirects stderr, `2>&1` merges stderr into stdout, `&>` redirects both. In Docker: app logs should go to stdout/stderr so they can be collected by the container runtime.

**Q5. What is `iptables` and how is it used in containerization?**
> **A:** `iptables` is the Linux kernel's firewall/packet filter. Rules are organized in tables (filter, nat, mangle) and chains (INPUT, OUTPUT, FORWARD, PREROUTING, POSTROUTING). Docker and Kubernetes heavily use iptables for:
> - Container NAT (mapping container ports to host ports)
> - Service IP routing (kube-proxy writes iptables rules for ClusterIP Services)
> - Network policy enforcement

### Real-World Use Case

> **Scenario:** A microservice intermittently fails to connect to the database. Diagnosis: `ss -tulnp | grep 5432` shows the DB port is listening. `tcpdump -i eth0 host <db-ip> port 5432` shows SYN packets going out but no SYN-ACK coming back. Conclusion: Security group or firewall is dropping the connection. Fix: Add inbound rule allowing port 5432 from the app's IP range.

---

## 6. Git & Version Control

### Core Git Concepts

```
Working Directory → Staging Area → Local Repo → Remote Repo
       │                │               │              │
   (edit files)   (git add)       (git commit)   (git push)
```

### Git Branching Strategy (GitFlow)

```
main ──────────────────────────────────────────────────────► (production)
  │                                                    ▲
  │  develop ─────────────────────────────────────────┤
  │     │         │            │                       │
  │     │  feature/login  feature/api    release/1.0  │
  │     │         │            │              │        │
  │     └─────────┘            └──────────────┘        │
  │                                                    │
  └────────────── hotfix/critical-bug ─────────────────┘
```

### Interview Q&A

**Q1. What is the difference between `git merge` and `git rebase`?**
> **A:** `git merge` creates a new merge commit joining two branch histories — preserves the exact history of when branches diverged. `git rebase` moves your branch's commits on top of the target branch, creating a linear history as if the branching never happened. **Rule:** Rebase for local/feature branches to keep clean history. Merge for integrating into shared branches. Never rebase public/shared branches (it rewrites history and breaks others' work).

**Q2. How do you handle a broken main branch in production?**
> **A:** (1) `git revert <commit-hash>` to create a new commit that undoes the bad one (safe, preserves history). (2) If it's truly urgent and the commit is fresh: `git reset --hard HEAD~1` + force push (dangerous if others pulled). (3) Best practice: Use a CI gate — main should never be directly pushable without a passing PR + review.

**Q3. What is a Git hook and how is it used in DevOps?**
> **A:** Git hooks are scripts that run automatically at certain points (pre-commit, pre-push, post-merge, etc.). DevOps uses: `pre-commit` for running linters/formatters, `pre-push` for running unit tests, `post-receive` on a server to trigger deployments. Tools like Husky manage hooks in Node.js projects. They're local by default (not pushed with code), but shared via tooling.

**Q4. Explain the difference between `git pull` and `git fetch`.**
> **A:** `git fetch` downloads changes from remote but does NOT update your working branch. `git pull` = `git fetch` + `git merge` (or rebase). **Best practice:** Use `git fetch` + review changes + `git merge/rebase` manually for safer updates, especially in production-related branches.

**Q5. How do you resolve a merge conflict?**
> **A:** (1) Git marks conflicted files with `<<<<<<<`, `=======`, `>>>>>>>` markers. (2) Open the file, understand both versions, edit to the correct state, remove markers. (3) `git add <file>` to mark resolved. (4) `git commit` to complete the merge. Prevention: Short-lived branches, frequent integration (CI), code ownership to minimize overlapping changes.

### Real-World Use Case

> **Scenario:** A CI/CD pipeline runs on every merge to `main`. A developer accidentally pushed a hardcoded secret. Fix: `git revert` the commit immediately, rotate the secret (assume it's compromised), use `git-secrets` or `truffleHog` in pre-commit hooks going forward, and enable branch protection requiring PR reviews.

---

## 7. Docker & Container Networking

### What is Docker?

Docker packages an application and all its dependencies into a **container** — a lightweight, isolated, portable unit that runs the same on any system.

```
Without Docker:             With Docker:
┌────────────┐              ┌──────────────────────────┐
│ App needs  │              │ Container:               │
│ Python 3.8 │              │  - App code              │
│ + lib v1   │              │  - Python 3.8            │
│ + lib v2   │    ──────►   │  - All dependencies      │
│            │              │  - Identical everywhere  │
│ "Works on  │              └──────────────────────────┘
│ my machine"│              "Works everywhere"
└────────────┘
```

### Docker Network Types

```
┌────────────────────────────────────────────────────────────────────┐
│                         Docker Host                                │
│                                                                    │
│  Bridge Network (default):          Host Network:                  │
│  ┌──────────┐  ┌──────────┐         ┌──────────┐                  │
│  │Container │  │Container │         │Container │                  │
│  │  A       │  │  B       │         │ (shares  │                  │
│  │172.17.0.2│  │172.17.0.3│         │ host     │                  │
│  └────┬─────┘  └────┬─────┘         │ network) │                  │
│       │             │               └──────────┘                  │
│  ┌────┴─────────────┴────┐                                        │
│  │    docker0 bridge     │  172.17.0.1                            │
│  └──────────┬────────────┘                                        │
│             │                                                      │
│       ┌─────┴──────┐                                              │
│       │  iptables  │  NAT: 172.17.0.x → Host IP                  │
│       └─────┬──────┘                                              │
│             │                                                      │
│      Host IP: 10.0.0.5                                            │
└─────────────┼──────────────────────────────────────────────────────┘
              │
          Internet
```

### Docker Network Types Summary

| Network | Description | Use Case |
|---------|-------------|----------|
| **bridge** | Default; isolated virtual network | Single host, multi-container |
| **host** | Container shares host network stack | Max performance, no isolation |
| **none** | No networking | Batch jobs, security-sensitive |
| **overlay** | Spans multiple Docker hosts | Docker Swarm |
| **macvlan** | Container gets real MAC/IP | Legacy apps needing direct L2 |

### Port Mapping

```
Host Port 8080 ──────► Container Port 80

docker run -p 8080:80 nginx
     iptables rule: traffic to host:8080 → container:172.17.0.2:80
```

### Container-to-Container Communication

```bash
# Same custom network: use container name as hostname
docker network create myapp-net
docker run --network myapp-net --name db postgres
docker run --network myapp-net --name app myapp
# Inside 'app' container: connect to 'db:5432' ← Works!

# Docker Compose (auto-creates network):
services:
  app:
    depends_on: [db]
  db:
    image: postgres
# 'app' can reach 'db' via hostname 'db'
```

### Interview Q&A

**Q1. What is the difference between a Docker image and a container?**
> **A:** An **image** is a read-only template (a blueprint) — it's the packaged filesystem layers with instructions. A **container** is a running instance of an image — it's the image + a writable layer + isolated process. Multiple containers can run from the same image simultaneously. Stopping a container doesn't delete it; `docker rm` does. Images are stored in registries (Docker Hub, ECR, GCR).

**Q2. How does Docker networking work internally?**
> **A:** Docker creates a virtual bridge (`docker0`) on the host. Each container gets a virtual ethernet pair (veth pair) — one end inside the container (`eth0`), one end connected to the bridge on the host. The bridge acts as a virtual switch. Docker uses iptables for NAT (MASQUERADE) so containers can reach the internet using the host's IP. Port mappings are DNAT rules in iptables.

**Q3. What is the difference between COPY and ADD in a Dockerfile?**
> **A:** `COPY` is simple — copies files/dirs from host to container. `ADD` does everything `COPY` does PLUS: auto-extracts TAR archives, and can fetch remote URLs. **Best practice:** Always use `COPY` unless you specifically need TAR extraction. `ADD` with URLs is an antipattern (no caching, security concerns) — use `RUN curl` instead.

**Q4. How do you make Docker containers secure?**
> **A:** (1) Use minimal base images (Alpine, distroless). (2) Run as non-root user (`USER appuser` in Dockerfile). (3) Drop Linux capabilities (`--cap-drop=ALL`). (4) Read-only filesystem (`--read-only`). (5) Scan images with Trivy or Snyk. (6) Don't store secrets in images (use env vars, secrets managers). (7) Use Docker Content Trust for image signing. (8) Limit resource usage (`--memory`, `--cpu`).

**Q5. What is a multi-stage Docker build and why use it?**
> **A:** Multi-stage builds use multiple `FROM` instructions. Early stages handle compilation/building; the final stage copies only the artifacts needed to run. This keeps production images tiny:

```dockerfile
# Stage 1: Build
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

# Stage 2: Runtime (tiny!)
FROM alpine:3.18
COPY --from=builder /app/myapp /myapp
CMD ["/myapp"]
# Result: ~10MB instead of ~800MB
```

### Real-World Use Case

> **Scenario:** A containerized app is slow and consuming too much memory on the host. Investigation: `docker stats` shows memory thrashing. Fix: (1) Add `--memory=512m` limit. (2) Rebuild image with multi-stage (reduced from 1.2GB to 120MB). (3) Use `--network=host` for the specific performance-sensitive component. (4) Profile the app with `docker exec -it <container> sh` + `/usr/bin/time -v ./app`.

---

## 8. Kubernetes Networking

### Kubernetes Networking Model

**Fundamental rule:** Every Pod gets its own IP. Pods can communicate with each other across nodes without NAT.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                        │
│                                                                  │
│  Node 1 (10.0.1.10)           Node 2 (10.0.1.11)                │
│  ┌──────────────────────┐     ┌──────────────────────┐           │
│  │  Pod A    Pod B      │     │  Pod C    Pod D      │           │
│  │ 10.244.1.2  .3       │     │ 10.244.2.2  .3       │           │
│  │   │         │        │     │   │         │        │           │
│  │   └────┬────┘        │     │   └────┬────┘        │           │
│  │    [cbr0/cni0]       │     │    [cbr0/cni0]       │           │
│  │    bridge            │     │    bridge            │           │
│  └──────────┬───────────┘     └──────────┬───────────┘           │
│             │                            │                       │
│         [flannel/calico tunnel or routes between nodes]          │
│             │                            │                       │
│             └────────────────────────────┘                       │
│                  Pod A can reach Pod C directly                  │
│                  using Pod C's IP (10.244.2.2)                   │
└─────────────────────────────────────────────────────────────────┘
```

### Kubernetes Service Types

```
ClusterIP (Default):
  ┌─────────┐       ClusterIP: 10.96.50.100      ┌─────────┐
  │ Pod (A) │─────────────────────────────────── │ Pod (B) │
  └─────────┘          kube-proxy (iptables)      └─────────┘
  Internal only, stable virtual IP

NodePort:
  External ─────► Node:30080 ──► ClusterIP ──► Pod
  Accessible from outside cluster on each node's IP

LoadBalancer:
  Internet ──► Cloud LB IP ──► NodePort ──► ClusterIP ──► Pod
  Cloud-provided external load balancer

Ingress:
  Internet ──► Ingress Controller (nginx/traefik) ──► Services ──► Pods
              Routes by hostname/path:
              api.example.com    ──► api-service
              app.example.com    ──► app-service
              app.example.com/v2 ──► app-v2-service
```

### DNS in Kubernetes

```
Service DNS format: <service>.<namespace>.svc.cluster.local

Examples:
  db-service.default.svc.cluster.local   → 10.96.0.100
  api-service.prod.svc.cluster.local     → 10.96.0.200

Pod DNS: <pod-ip-dashes>.<namespace>.pod.cluster.local
  10-244-1-5.default.pod.cluster.local

CoreDNS serves all DNS resolution within the cluster.
```

### Network Policies

```yaml
# Allow ingress to pod 'app' only from pod 'frontend' on port 8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

### Interview Q&A

**Q1. What is the difference between a ClusterIP, NodePort, and LoadBalancer service?**
> **A:** **ClusterIP:** Internal only, stable virtual IP accessible within the cluster. Used for service-to-service communication. **NodePort:** Exposes the service on a static port (30000-32767) on every node's IP. Usable externally but requires knowing node IPs — not production-friendly. **LoadBalancer:** Provisions a cloud load balancer (AWS ELB, GCP LB) with a public IP. The standard way to expose services to the internet in production.

**Q2. How does kube-proxy work?**
> **A:** `kube-proxy` runs on every node and watches the API server for Service/Endpoint changes. In **iptables mode** (most common), it programs iptables DNAT rules: traffic to ClusterIP is randomly distributed among healthy pod IPs using iptables `--probability` rules. In **IPVS mode**, it uses Linux IPVS (IP Virtual Server) for better performance at scale (1000+ services). **eBPF-based CNIs** (Cilium) can bypass kube-proxy entirely for higher performance.

**Q3. What is a CNI plugin and name some examples?**
> **A:** CNI (Container Network Interface) is a specification for how networking is configured for pods. When a pod starts, Kubernetes calls the CNI plugin to: assign IP, set up routing, configure firewall rules. Popular CNIs: **Flannel** (simple, VXLAN overlay), **Calico** (BGP-based, supports Network Policies, high performance), **Weave** (encrypted mesh), **Cilium** (eBPF-based, advanced observability and policy). Cloud providers have their own: **AWS VPC CNI** assigns real VPC IPs to pods.

**Q4. Explain how Kubernetes DNS resolution works for a service call.**
> **A:** When pod A calls `http://db-service/query`, the DNS resolver (configured to use CoreDNS at the cluster DNS IP) receives the query. CoreDNS checks: is this a known service? Yes → return the ClusterIP. Pod A sends HTTP to ClusterIP. iptables on the node intercepts and DNAT's the packet to one of the healthy pod IPs backing `db-service`. The response returns through the same path. The application sees only the ClusterIP.

**Q5. What is an Ingress controller and how does it differ from a LoadBalancer service?**
> **A:** A **LoadBalancer** service creates one cloud LB per service (expensive, no path routing). An **Ingress** is a Kubernetes resource that defines HTTP(S) routing rules. An **Ingress Controller** (nginx, Traefik, AWS ALB Ingress Controller) implements those rules — it's a single load balancer that routes to multiple services based on hostname and path. Benefits: One LB for N services (cost efficient), SSL termination, path-based routing, auth, rate limiting.

### Real-World Use Case

> **Scenario:** Pods in namespace `prod` are able to connect to the database service in namespace `prod`, but a recently-added `debug` namespace pod is also connecting. Fix: Implement a NetworkPolicy on the `db` pods selecting only pods from `prod` namespace. Without NetworkPolicy, Kubernetes allows all pod-to-pod traffic by default. Apply `default-deny` policies first, then explicitly allow required traffic.

---

## 9. CI/CD Pipelines

### What is CI/CD?

**CI (Continuous Integration):** Automatically build and test code on every commit.
**CD (Continuous Delivery):** Automatically prepare release-ready artifacts; human approves deployment.
**CD (Continuous Deployment):** Automatically deploy to production on every passing build.

### CI/CD Pipeline with Networking Touchpoints

```
Developer pushes code
       │
       ▼
┌─────────────────┐
│   Git Remote    │ (GitHub/GitLab/Bitbucket)
│  Webhook fires  │ ──────────────────────────────────────────────►
└─────────────────┘                                               │
                                                                  ▼
                                                    ┌─────────────────────┐
                                                    │  CI Server          │
                                                    │  (Jenkins/GitHub    │
                                                    │   Actions/GitLab CI)│
                                                    └──────────┬──────────┘
                                                               │
                            ┌──────────────────────────────────┘
                            │
               ┌────────────▼───────────────────────────────────────┐
               │              Pipeline Stages                        │
               │                                                     │
               │  1. Checkout → [git clone via HTTPS/SSH]            │
               │         │                                           │
               │  2. Build → [compile, npm install, maven build]     │
               │         │                                           │
               │  3. Test → [unit, integration, coverage]            │
               │         │                                           │
               │  4. Security Scan → [Trivy, Snyk, SonarQube]       │
               │         │                                           │
               │  5. Docker Build → [docker build + push to ECR]    │
               │         │   [Network: pulls base img, pushes to    │
               │         │    registry via HTTPS port 443]          │
               │         │                                           │
               │  6. Deploy to Staging → [kubectl apply / Helm]      │
               │         │   [Network: API server call to K8s]      │
               │         │                                           │
               │  7. Integration Test → [curl staging endpoints]    │
               │         │                                           │
               │  8. Manual Approval Gate ──────────────────────►   │
               │         │                  [Slack/Email notify]     │
               │         │                                           │
               │  9. Deploy to Production → [rolling update]        │
               │         │   [Network: K8s pulls image from ECR]    │
               │         │                                           │
               │  10. Smoke Test + Monitor [Prometheus alerts]      │
               └────────────────────────────────────────────────────┘
```

### Jenkins Pipeline (Declarative Example)

```groovy
pipeline {
    agent any
    environment {
        IMAGE = "myapp:${BUILD_NUMBER}"
        REGISTRY = "123456789.dkr.ecr.us-east-1.amazonaws.com"
    }
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t $IMAGE .'
            }
        }
        stage('Test') {
            steps {
                sh 'docker run $IMAGE pytest tests/'
            }
        }
        stage('Push') {
            steps {
                sh 'aws ecr get-login-password | docker login --username AWS $REGISTRY'
                sh 'docker tag $IMAGE $REGISTRY/$IMAGE'
                sh 'docker push $REGISTRY/$IMAGE'
            }
        }
        stage('Deploy') {
            steps {
                sh 'kubectl set image deployment/myapp myapp=$REGISTRY/$IMAGE'
                sh 'kubectl rollout status deployment/myapp'
            }
        }
    }
    post {
        failure { slackSend message: "Build FAILED: ${env.JOB_NAME}" }
        success { slackSend message: "Deployed: ${env.BUILD_URL}" }
    }
}
```

### Interview Q&A

**Q1. What is the difference between CI and CD?**
> **A:** CI (Continuous Integration) is the practice of frequently merging code changes and automatically building and testing. It detects integration issues early. CD is an extension: after CI succeeds, code is automatically delivered to a staging environment (Continuous Delivery) or even production (Continuous Deployment). CI is about code quality confidence; CD is about deployment speed and reliability.

**Q2. What networking access does a CI/CD pipeline typically need?**
> **A:** Outbound: Pull base Docker images (Docker Hub, ECR), download dependencies (npm registry, Maven Central, PyPI), push built images to a registry, communicate with deployment targets (Kubernetes API server, SSH to servers). Inbound: Receive webhooks from Git (GitHub → Jenkins). Internal: Access to test databases, staging environments, secrets managers (Vault, AWS Secrets Manager). **Security:** CI agents should be in a private subnet with controlled egress via NAT gateway, with explicit security group rules.

**Q3. What is a blue-green deployment?**
> **A:** Two identical environments exist simultaneously — "Blue" (current production) and "Green" (new version). After Green passes all tests, the load balancer switches traffic to Green instantly. If problems arise, traffic switches back to Blue immediately (near-zero-downtime rollback). Kubernetes can implement this with two Deployments and a Service that switches selector labels.

**Q4. What is a canary deployment?**
> **A:** New version is deployed to a small percentage of traffic (e.g., 5%) while the old version handles the rest. Metrics (error rate, latency) are monitored. If healthy, the percentage gradually increases to 100%. If errors spike, the canary is rolled back. Tools: Argo Rollouts, Flagger (with Prometheus), AWS CodeDeploy canary, Istio traffic splitting.

**Q5. How do you handle secrets in a CI/CD pipeline?**
> **A:** Never hardcode secrets in code or Dockerfiles. Options: (1) **CI platform secrets** (GitHub Actions Secrets, Jenkins Credentials) — injected as env vars at runtime. (2) **Vault by HashiCorp** — dynamic secrets with short TTLs. (3) **AWS Secrets Manager / Parameter Store** — IAM-authenticated access. (4) **Kubernetes Secrets** — mounted as env vars or files in pods. (5) **Sealed Secrets** — encrypted Kubernetes secrets safely storable in Git. At runtime, the app fetches its own secrets via the secrets manager SDK.

### Real-World Use Case

> **Scenario:** A CI pipeline is building Docker images but the `docker push` step fails with "connection timeout" to the ECR registry. Root cause: The CI server is in a private subnet. iptables allows outbound HTTPS, but no NAT gateway is configured. Fix: Add a NAT gateway to the subnet's route table, or use a VPC endpoint for ECR (`com.amazonaws.region.ecr.api`) for private, internal access to the registry.

---

## 10. Terraform & Infrastructure as Code

### What is Terraform?

Terraform is an IaC (Infrastructure as Code) tool by HashiCorp that provisions infrastructure (servers, networks, databases) using declarative configuration files.

### How Terraform Works

```
terraform init    ──► Download providers (AWS, Azure, GCP plugins)
terraform plan    ──► Show what will change (dry run)
terraform apply   ──► Create/modify resources
terraform destroy ──► Delete all managed resources

State Flow:
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│   .tf files  │───►│ terraform.tfstate│◄──►│  Real Cloud  │
│ (desired)    │    │  (current known) │    │ Infrastructure│
└──────────────┘    └──────────────────┘    └──────────────┘
```

### Networking Example: VPC + Subnet + EC2

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "production-vpc" }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  map_public_ip_on_launch = true
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

# Route Table: Public traffic → IGW
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

# Security Group: Allow HTTP + SSH
resource "aws_security_group" "web" {
  vpc_id = aws_vpc.main.id
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Interview Q&A

**Q1. What is Terraform state and why is it important?**
> **A:** Terraform state (`terraform.tfstate`) is a JSON file that maps your configuration to real resources. It tracks resource IDs, dependencies, and metadata. It's critical because Terraform uses it to determine what needs to change (diff between desired state and current state). **In production:** Always use **remote state** (S3 + DynamoDB for AWS, Terraform Cloud) with state locking to prevent concurrent modifications. Never edit state manually.

**Q2. What is the difference between `terraform plan` and `terraform apply`?**
> **A:** `terraform plan` is a dry run — it shows exactly what actions (create, modify, destroy) will be taken without making any changes. Always run `plan` before `apply`. `terraform apply` executes those changes against the real infrastructure. In CI/CD: `plan` on PR, `apply` only after approval and merge to main.

**Q3. How do you handle sensitive values in Terraform?**
> **A:** Mark variables as `sensitive = true` (hidden from CLI output). Don't put secrets in `.tf` files — use environment variables (`TF_VAR_db_password`) or data sources reading from Secrets Manager. State file may still contain sensitive values in plaintext — ensure state backend is encrypted (S3 with SSE) and access-controlled.

**Q4. What are Terraform modules?**
> **A:** Modules are reusable packages of Terraform configurations. Like functions in programming — accept inputs (variables), produce outputs. A `vpc` module might accept a CIDR block and environment name, and create the VPC, subnets, IGW, and route tables. Teams build module registries for standardized, approved infrastructure patterns. The Terraform Registry has community modules for common patterns (e.g., `terraform-aws-modules/vpc`).

**Q5. What is a Terraform workspace?**
> **A:** Workspaces allow multiple state files from the same configuration — used to manage multiple environments (dev, staging, prod) with one codebase. `terraform workspace new staging` creates a `staging` workspace with its own state. However, for truly isolated environments (separate AWS accounts), use separate directories/repos per environment rather than workspaces.

### Real-World Use Case

> **Scenario:** Two engineers both run `terraform apply` simultaneously on production, causing state corruption and duplicated resources. Fix: Implement **state locking** with DynamoDB (for AWS S3 backend) — Terraform acquires a lock before applying, preventing concurrent runs. Add this to the backend config: `dynamodb_table = "terraform-lock"`. Also enforce Terraform runs only through CI/CD, never from local machines in production.

---

## 11. Ansible & Configuration Management

### What is Ansible?

Ansible automates configuration management, application deployment, and task automation using **YAML playbooks** executed over SSH (agentless — no software installed on managed nodes).

### Ansible Architecture

```
┌───────────────────────────────────────────────────────┐
│                  Control Node                          │
│                                                       │
│  Inventory       Playbooks          Roles             │
│  (hosts.ini)     (tasks.yml)        (reusable)        │
│                                                       │
│           ansible-playbook deploy.yml                 │
└──────────────────┬────────────────────────────────────┘
                   │ SSH (port 22) / WinRM (Windows)
         ┌─────────┼─────────┐
         ▼         ▼         ▼
    ┌─────────┐ ┌───────┐ ┌───────┐
    │ web-01  │ │web-02 │ │db-01  │
    │(target) │ │(target│ │(target│
    └─────────┘ └───────┘ └───────┘
       Managed Nodes (no agent needed)
```

### Sample Playbook: Install + Configure Nginx

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true          # Run as sudo
  vars:
    nginx_port: 80

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Deploy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart Nginx

    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

### Interview Q&A

**Q1. What is the difference between Ansible and Terraform?**
> **A:** **Terraform** provisions infrastructure (create VMs, VPCs, databases). It's declarative and manages infrastructure state. **Ansible** configures software on existing systems (install packages, deploy apps, manage files). It's procedural (runs tasks in order). They complement each other: Terraform creates the server, Ansible configures it. Both can overlap (Ansible has cloud modules; Terraform has provisioners), but the distinction in responsibility makes for cleaner architecture.

**Q2. What are Ansible roles and why use them?**
> **A:** Roles are a structured way to organize playbooks into reusable units with a standard directory structure (`tasks/`, `handlers/`, `templates/`, `vars/`, `defaults/`, `files/`). A `nginx` role can be shared across projects. Ansible Galaxy hosts community roles. Using roles: promotes reuse, enforces standards, makes large playbooks manageable.

**Q3. How does Ansible ensure idempotency?**
> **A:** Idempotency means running the same playbook multiple times produces the same result without unintended changes. Ansible modules are designed to check current state before acting: `apt: name=nginx state=present` checks if nginx is installed; if yes, it does nothing. **Be careful with `shell:` and `command:` modules** — they always run and are NOT idempotent by default. Use `creates:` parameter or `when:` conditions to add idempotency manually.

**Q4. What is Ansible Vault?**
> **A:** Ansible Vault encrypts sensitive data (passwords, API keys) in playbooks and variable files using AES-256. `ansible-vault encrypt secrets.yml` encrypts the file. During playbook runs, `--ask-vault-pass` or `--vault-password-file` decrypts at runtime. Vault passwords themselves should be stored in a secrets manager (Vault, AWS Secrets Manager) and injected into the CI/CD environment.

**Q5. How would you use Ansible to deploy an app with zero downtime?**
> **A:** Use rolling updates with `serial:` directive (update one server at a time), combined with load balancer drain: (1) Remove server from load balancer (`delegate_to: load_balancer`). (2) Wait for connections to drain. (3) Deploy new version. (4) Health check. (5) Add back to load balancer. (6) Repeat for next server. Or use `serial: "25%"` to update 25% of servers at a time.

### Real-World Use Case

> **Scenario:** After provisioning 50 new EC2 instances with Terraform, the team needs to configure all of them with the same software stack. An Ansible playbook runs against the dynamic inventory (using `aws_ec2` inventory plugin that auto-discovers instances by tag), installs the application, configures monitoring agents, and sets up log shipping — all in one run, taking minutes instead of hours of manual work.

---

## 12. Monitoring: Prometheus & Grafana

### Monitoring Stack Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                       Application Layer                          │
│   App (metrics at /metrics)   Node Exporter   kube-state-metrics│
└──────────────────────┬──────────────────────────────────────────┘
                       │ Scrape (HTTP pull every 15s)
                       ▼
              ┌─────────────────┐
              │   Prometheus    │ ← Stores time-series data
              │   (TSDB)        │   Evaluates alert rules
              └────────┬────────┘
                       │
           ┌───────────┴──────────────┐
           ▼                          ▼
    ┌─────────────┐          ┌──────────────────┐
    │  Grafana    │          │  Alertmanager    │
    │  (dashboards│          │  (routes alerts  │
    │   & graphs) │          │  to Slack/Email) │
    └─────────────┘          └──────────────────┘
```

### Key Prometheus Concepts

| Concept | Description |
|---------|-------------|
| **Metric** | A named time-series data point |
| **Labels** | Key-value pairs for dimensions (`{env="prod", region="us-east-1"}`) |
| **Scrape** | Prometheus pulls metrics from `/metrics` endpoints |
| **Counter** | Always increases (requests_total, errors_total) |
| **Gauge** | Can go up or down (memory_usage, active_connections) |
| **Histogram** | Samples observations into buckets (request latency) |
| **PromQL** | Prometheus Query Language |

### PromQL Examples

```
# Request rate (5-min window)
rate(http_requests_total[5m])

# Error rate percentage
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Memory usage per pod
container_memory_usage_bytes{namespace="prod"}

# CPU throttling
rate(container_cpu_throttled_seconds_total[5m])
```

### Interview Q&A

**Q1. What is the difference between Prometheus pull model and push model?**
> **A:** **Pull model (Prometheus default):** Prometheus actively scrapes `/metrics` endpoints on a schedule. Advantages: Centralized control, can detect if a target is down (no scrape = alert). **Push model (Graphite, InfluxDB with Telegraf):** Agents push metrics to the collector. Advantages: Works behind firewalls, short-lived jobs (use Prometheus Pushgateway for this). Prometheus also supports push via Pushgateway for batch/scheduled jobs.

**Q2. Explain the difference between a counter and a gauge metric.**
> **A:** A **counter** only goes up (or resets on restart) — examples: total HTTP requests, total errors, bytes sent. You use `rate()` to get meaningful data from counters. A **gauge** is a snapshot value that can go up or down — examples: current memory usage, number of active connections, temperature. Use gauges for current-state metrics; counters for cumulative events.

**Q3. What is a "golden signal" in monitoring?**
> **A:** Google SRE defined four golden signals: **Latency** (how long requests take), **Traffic** (requests per second), **Errors** (error rate), and **Saturation** (how "full" is the service — CPU, memory, disk). Monitoring these four signals for every service gives you comprehensive visibility. In Prometheus: `rate()` for traffic and errors, `histogram_quantile()` for latency, various resource metrics for saturation.

**Q4. How do you set up an alert for high error rate in Prometheus?**

```yaml
# alerting_rules.yml
groups:
- name: api_alerts
  rules:
  - alert: HighErrorRate
    expr: |
      rate(http_requests_total{status=~"5.."}[5m])
      /
      rate(http_requests_total[5m]) > 0.05
    for: 2m    # Must be true for 2 minutes before firing
    labels:
      severity: critical
    annotations:
      summary: "High error rate on {{ $labels.service }}"
      description: "Error rate is {{ $value | humanizePercentage }}"
```

**Q5. What is the difference between Prometheus and Grafana?**
> **A:** **Prometheus** is a time-series database and monitoring system — it collects, stores, and queries metrics. It has basic graphing but is not a dashboarding tool. **Grafana** is a visualization platform — it connects to Prometheus (and 50+ other data sources) and creates rich, shareable dashboards. They are complementary: Prometheus is the backend; Grafana is the frontend. Grafana also supports alerting, but Prometheus Alertmanager handles routing/deduplication.

### Real-World Use Case

> **Scenario:** During peak hours, the checkout service latency spikes to 5 seconds. With Grafana dashboards showing the P95 latency histogram alongside pod CPU/memory metrics, the team identifies that the DB connection pool is exhausted (saturated). An alert fires (via Alertmanager → PagerDuty) before users flood customer support. Fix: Increase connection pool size and add a database read replica, monitored via the same stack.

---

## 13. Cloud Networking: VPC, Subnets, Security Groups

### VPC Architecture (AWS)

```
┌──────────────────────────────────────────────────────────────────────┐
│                         AWS Region (us-east-1)                        │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    VPC: 10.0.0.0/16                            │  │
│  │                                                                │  │
│  │  Availability Zone A (us-east-1a)  AZ B (us-east-1b)          │  │
│  │  ┌──────────────────────────┐   ┌──────────────────────────┐  │  │
│  │  │  Public Subnet           │   │  Public Subnet           │  │  │
│  │  │  10.0.1.0/24             │   │  10.0.2.0/24             │  │  │
│  │  │  [ALB] [Bastion Host]    │   │  [ALB standby]           │  │  │
│  │  └──────────┬───────────────┘   └──────────┬───────────────┘  │  │
│  │             │                              │                   │  │
│  │  ┌──────────▼───────────────┐   ┌──────────▼───────────────┐  │  │
│  │  │  Private Subnet          │   │  Private Subnet          │  │  │
│  │  │  10.0.3.0/24             │   │  10.0.4.0/24             │  │  │
│  │  │  [App Servers / EKS]     │   │  [App Servers / EKS]     │  │  │
│  │  └──────────┬───────────────┘   └──────────┬───────────────┘  │  │
│  │             │                              │                   │  │
│  │  ┌──────────▼───────────────┐   ┌──────────▼───────────────┐  │  │
│  │  │  Data Subnet             │   │  Data Subnet             │  │  │
│  │  │  10.0.5.0/24             │   │  10.0.6.0/24             │  │  │
│  │  │  [RDS, ElastiCache]      │   │  [RDS standby]           │  │  │
│  │  └──────────────────────────┘   └──────────────────────────┘  │  │
│  │                                                                │  │
│  │  [Internet Gateway]    [NAT Gateway in Public Subnet]          │  │
│  │  [Route Tables]        [VPC Endpoints]                        │  │
│  └────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘

Internet Traffic Flow:
User → Internet Gateway → ALB (public subnet) → App Server (private subnet) → RDS (data subnet)

Outbound from Private Subnet:
App Server → NAT Gateway (public subnet) → Internet Gateway → Internet
```

### Security Groups vs NACLs

| Feature | Security Group | Network ACL |
|---------|---------------|-------------|
| Level | Instance level | Subnet level |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules evaluated | Rules evaluated in order (number) |
| Default | Deny all inbound, allow all outbound | Allow all |

### Interview Q&A

**Q1. What is the difference between a public subnet and a private subnet in AWS?**
> **A:** A **public subnet** has a route to the Internet Gateway (0.0.0.0/0 → IGW) and resources can have public IPs — internet traffic flows directly in/out. A **private subnet** has no direct route to the internet; resources have only private IPs. Outbound internet access from private subnets goes through a **NAT Gateway** (in a public subnet) which masquerades private IPs. Inbound internet access to private resources goes through a load balancer in the public subnet.

**Q2. Explain security groups and when you'd use them.**
> **A:** Security groups are instance-level virtual firewalls. They're **stateful** — if you allow inbound port 80, the return traffic is automatically allowed (no need for explicit outbound rule for responses). Rules can reference other security groups (not just CIDRs) — e.g., "allow port 5432 from security group `app-sg`" — this dynamically allows the IP of any instance using `app-sg`, which is powerful for auto-scaling.

**Q3. What is a VPC peering connection vs. a Transit Gateway?**
> **A:** **VPC Peering:** One-to-one connection between two VPCs (can be cross-account or cross-region). Simple but doesn't scale — 5 VPCs need 10 peering connections (N*(N-1)/2), and peering is NOT transitive. **Transit Gateway:** Acts as a hub — all VPCs connect to it (hub-and-spoke model). Peering is transitive through TGW. Scales to 1000s of VPCs, supports BGP, multicast, and centralized routing policies. Use TGW for large, complex multi-VPC architectures.

**Q4. What is a VPC endpoint and why use it?**
> **A:** A VPC endpoint allows private connectivity to AWS services (S3, DynamoDB, ECR, etc.) without traffic leaving the AWS network (no internet, no NAT gateway). **Interface endpoints** create ENIs in your subnet using PrivateLink. **Gateway endpoints** (S3, DynamoDB only) add entries to route tables. Benefits: Security (traffic stays private), cost savings (no NAT gateway data processing charges for S3/DynamoDB), lower latency.

**Q5. How would you design a secure, highly-available VPC?**
> **A:** (1) Multi-AZ: Subnets in at least 2 (preferably 3) AZs. (2) Three tiers: Public (ALB, NAT GW), Private (app), Data (RDS, cache). (3) Security groups with least privilege (app-sg only allows traffic from alb-sg). (4) NACLs as additional layer. (5) Flow Logs to CloudWatch or S3 for audit. (6) VPC endpoints for AWS service access. (7) No direct internet to app or data tiers. (8) Bastion host or Systems Manager Session Manager for admin access (no public SSH).

### Real-World Use Case

> **Scenario:** An EKS cluster's pods can't pull images from ECR, causing deployment failures. Diagnosis: Pods are in a private subnet and can't reach the internet. The team adds a VPC endpoint for ECR (`com.amazonaws.us-east-1.ecr.dkr` and `com.amazonaws.us-east-1.ecr.api`) and an S3 gateway endpoint (ECR uses S3 for image layers), eliminating the need for NAT gateway for this traffic — saving ~$30/month in data transfer costs.

---

## 14. Load Balancers, NAT & VPN

### Load Balancer Architecture

```
                        Internet
                           │
                    ┌──────▼──────┐
                    │  DNS:       │
                    │ api.app.com │───► ALB IP (Elastic)
                    └──────┬──────┘
                           │
                    ┌──────▼──────────────────┐
                    │   Application Load      │
                    │   Balancer (Layer 7)    │
                    │                         │
                    │  Listener: HTTPS 443    │
                    │  SSL Termination        │
                    │  Rules:                 │
                    │  /api/* → api-tg        │
                    │  /app/* → app-tg        │
                    │  Health Checks          │
                    └────────┬────────────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
     ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
     │  App Server │  │  App Server │  │  App Server │
     │  10.0.3.10  │  │  10.0.3.11  │  │  10.0.4.10  │
     │  (AZ-a)     │  │  (AZ-a)     │  │  (AZ-b)     │
     └─────────────┘  └─────────────┘  └─────────────┘
         Target Group: app-tg (round-robin with health checks)
```

### Types of AWS Load Balancers

| Type | Layer | Use Case |
|------|-------|----------|
| **ALB** | 7 (HTTP/HTTPS) | Web apps, microservices, WebSocket, path routing |
| **NLB** | 4 (TCP/UDP/TLS) | Ultra-low latency, static IP, non-HTTP protocols |
| **CLB** | 4+7 (Legacy) | Legacy EC2-classic apps (avoid for new designs) |
| **GWLB** | 3 | Security appliances (firewalls, IDS/IPS) |

### NAT (Network Address Translation)

```
Private Subnet            NAT Gateway               Internet
  App Server              (Public Subnet)
  10.0.3.10    ─────►   Elastic IP: 3.4.5.6  ─────►  api.github.com

Outbound:  Source: 10.0.3.10 → NAT translates to 3.4.5.6 → Internet
Inbound:   Response returns to 3.4.5.6 → NAT de-translates to 10.0.3.10

Internet-initiated connections to 10.0.3.10: BLOCKED (NAT is one-way)
```

### VPN Architecture

```
On-Premises Network                        AWS VPC
10.1.0.0/16                                10.0.0.0/16
                                           
┌─────────────────┐   IPSec VPN Tunnel    ┌─────────────────┐
│  VPN Appliance  │◄────────────────────►│  Virtual Private │
│  (firewall/     │   Encrypted over      │  Gateway (VGW)  │
│   router)       │   the internet        │                  │
└─────────────────┘                       └─────────────────┘
       │                                         │
  Corporate HQ                            Private Resources
  (servers, workstations)                 (RDS, EC2, EKS)

AWS Direct Connect (alternative):
┌─────────────────┐   Dedicated Fiber    ┌─────────────────┐
│  On-Premises    │◄────────────────────►│  AWS Direct     │
│  Data Center    │   No internet!        │  Connect        │
└─────────────────┘   Private, fast       └─────────────────┘
```

### Interview Q&A

**Q1. What is the difference between an ALB and NLB?**
> **A:** **ALB (Layer 7):** Understands HTTP — can route based on URL paths, hostnames, headers, query strings. Supports WebSocket, HTTP/2, gRPC. Has WAF integration. Has a variable IP (DNS-based). **NLB (Layer 4):** Works with any TCP/UDP protocol. Extremely low latency (<1ms). Has static Elastic IPs (useful for IP whitelisting). Preserves client IP. Use ALB for web apps; NLB for gaming, IoT, financial trading, or non-HTTP workloads.

**Q2. What is sticky sessions and when would you use it?**
> **A:** Sticky sessions (session affinity) use a cookie to ensure the same client always routes to the same backend server. Needed when session state is stored locally on the server (not in a shared store). **Problem:** A server failure loses that user's session. **Better approach:** Store session in Redis/DynamoDB, allowing any server to handle any request (stateless architecture). Use sticky sessions only as a bridge for legacy apps that can't be made stateless quickly.

**Q3. Explain how NAT works and the difference between SNAT and DNAT.**
> **A:** **SNAT (Source NAT):** Changes the source IP of outgoing packets — what AWS NAT Gateway does. Private server `10.0.3.10` connects out; NAT changes source to `3.4.5.6` (Elastic IP). **DNAT (Destination NAT):** Changes the destination IP of incoming packets — used in port forwarding and load balancing. When ALB receives traffic on `3.4.5.6:443`, DNAT sends it to `10.0.3.10:8080`. Docker's port mapping (`-p 8080:80`) is DNAT implemented with iptables.

**Q4. What is AWS Direct Connect vs. VPN?**
> **A:** **VPN:** Encrypted tunnel over the public internet. Easy to set up (hours). Lower cost. Variable latency (shared internet). Good for most use cases. **Direct Connect:** Dedicated private fiber connection from your data center to AWS. Consistent low latency (bypasses internet). High bandwidth (up to 100Gbps). Expensive ($$$) and takes weeks to provision. Use for large data transfers, latency-sensitive workloads (financial), or compliance requirements.

**Q5. How do you ensure high availability for a NAT Gateway?**
> **A:** NAT Gateways are **per-AZ**. If you have subnets in 3 AZs, create 3 NAT Gateways — one per AZ — and configure each AZ's private subnet route table to use its own NAT Gateway. This prevents cross-AZ dependency: if AZ-a's NAT Gateway fails, AZ-b and AZ-c continue independently. Using a single NAT Gateway (common in dev environments to save cost) means a single point of failure and cross-AZ data transfer charges.

### Real-World Use Case

> **Scenario:** An ALB health check is marking all backend instances as unhealthy, taking down the service. Investigation: Health check path `/health` returns 200, but ALB expects the response in `<5s` and the endpoint is connecting to a slow DB before responding. Fix: (1) Create a lightweight `/health` endpoint that returns 200 immediately without DB check. (2) Separate deep health check (`/health/deep`) for operational monitoring. (3) Set health check timeout to 10s as a short-term fix.

---

## 15. Real-World DevOps Networking Scenarios

### Scenario 1: Microservices Cannot Communicate in Kubernetes

**Problem:** `frontend` pods cannot reach `backend` service in Kubernetes.

```
Debugging Steps:
1. kubectl exec -it frontend-pod -- curl backend-service:8080
   → "Connection refused"

2. kubectl get svc backend-service
   → Check if service exists and correct port

3. kubectl get endpoints backend-service
   → If empty: no healthy pods match selector
   → Check pod labels: kubectl get pods --show-labels

4. kubectl describe pod backend-pod
   → Check Readiness probe status

5. kubectl get networkpolicy
   → Is there a NetworkPolicy blocking the connection?

6. kubectl exec -it frontend-pod -- nslookup backend-service
   → DNS resolution working?

Resolution: Pod labels didn't match Service selector.
Fix: Update deployment label to match service selector.
```

### Scenario 2: CI/CD Pipeline Deployment Failure

**Problem:** Pipeline fails at Kubernetes deployment step with "unable to connect to API server."

```
Root Cause Analysis:
Jenkins agent → Kubernetes API Server (port 6443)

Check 1: kubectl config view → Is the right cluster configured?
Check 2: nc -zv k8s-api.example.com 6443 → Can we reach the port?
Check 3: Security group on control plane: does it allow 6443 from Jenkins?
Check 4: kubectl auth can-i create deployment → RBAC permissions?

Resolution: Jenkins agent was in a new subnet not covered by the
control plane security group rule. Add Jenkins subnet CIDR to 
the security group inbound rule for port 6443.
```

### Scenario 3: High Latency in Production Microservices

**Problem:** P99 latency spikes from 200ms to 3s during peak hours.

```
Investigation Path:
Grafana: latency spike correlates with traffic surge

→ Check: DB connection pool (gauge: db_pool_active)
   → Pool saturated at 100/100 connections

→ Check: Which service is holding connections longest?
   → SQL query taking 2.8s (missing index)

→ Check: Network latency (mtr to DB host)
   → <1ms, network is fine

Fix:
1. Add index on frequently-queried column (immediate)
2. Increase connection pool (stop-gap)
3. Add read replica (long-term scaling)
4. Alert on pool saturation > 80%
```

### Scenario 4: VPC Connectivity Between Microservices Across Accounts

**Problem:** App in Account A (VPC-A) cannot reach data service in Account B (VPC-B).

```
Architecture Choice:
Option 1: VPC Peering
  - Create peering connection between VPC-A and VPC-B
  - Add routes in both route tables
  - Update security groups to allow cross-account CIDR
  - Works, but doesn't scale

Option 2: AWS PrivateLink (Recommended)
  - Create NLB in Account B
  - Create VPC Endpoint Service pointing to NLB
  - Account A creates Interface Endpoint
  - App in Account A connects to endpoint (as if it's local)
  - Secure: only specific services exposed, not full VPC peering

Implementation:
Account B: aws elbv2 create-load-balancer (NLB)
           aws ec2 create-vpc-endpoint-service
Account A: aws ec2 create-vpc-endpoint (Interface)
```

### Scenario 5: Docker Container Networking Debug

**Problem:** Container A can't reach Container B, both on the same host.

```bash
# Check they're on the same network
docker network inspect bridge
# → Container A on 172.17.0.2, Container B on 172.17.0.3

# Test connectivity
docker exec container-a ping 172.17.0.3  # By IP → works
docker exec container-a ping container-b  # By name → fails!

# Root cause: Default bridge doesn't support DNS resolution
# Fix: Use a custom bridge network
docker network create mynet
docker run --network mynet --name db postgres
docker run --network mynet --name app myapp
# Now: docker exec app ping db → works (DNS via Docker embedded DNS)
```

---

## 16. Mock Interview: 25 Mixed Questions

### Networking (1–8)

**Q1.** A user says "the website is slow." Walk me through your complete troubleshooting approach from the client side to the backend.
> **Model Answer:** Check DNS resolution (`dig`), test network latency (`ping`, `mtr`), check HTTP response time (`curl -w "%{time_total}\n"`), check server load (CPU, memory, disk I/O), check application logs for slow queries, use Grafana to correlate with metrics — isolate whether it's network, server, or application/database.

**Q2.** Explain what happens in the first 500ms after a user types `https://www.google.com` and presses Enter.
> DNS lookup (~10-100ms) → TCP connect (~50ms for SYN-ACK) → TLS handshake (~100ms) → HTTP GET (~50-200ms RTT) → HTML parsed → Additional DNS + connections for resources.

**Q3.** What is CORS and how does it work?
> Cross-Origin Resource Sharing. Browser security policy: JavaScript on `app.com` cannot call `api.other.com` by default. The browser sends a preflight OPTIONS request; the server responds with `Access-Control-Allow-Origin` header. The browser only proceeds if the origin is allowed. Configured in the API server (or API gateway/load balancer).

**Q4.** How does BGP affect cloud deployments and AWS Direct Connect?
> BGP advertises routes between your on-premises AS and AWS. Direct Connect uses BGP to advertise your on-premises CIDRs to AWS and AWS VPC CIDRs to you. You can use BGP communities to control routing preferences (prefer Direct Connect over VPN). Route filters prevent advertising overly broad prefixes.

**Q5.** What is MTU and how does it affect containerized applications?
> MTU (Maximum Transmission Unit) is the largest packet size a network can carry (Ethernet: 1500 bytes). VPNs and overlay networks (VXLAN) add headers, reducing effective MTU to ~1450 bytes. If containers send 1500-byte packets over a VXLAN-based CNI, packets get fragmented or dropped (depending on DF bit), causing mysterious connection hangs. Fix: Set container MTU to match the overlay network's effective MTU.

**Q6.** Explain TCP connection states you'd see with `ss -s` on a busy server.
> ESTABLISHED: Active connections. TIME_WAIT: Closed connections waiting (2×MSL, ~60s). CLOSE_WAIT: Remote closed; local hasn't closed yet (application bug if too many). SYN_RECV: Mid-handshake (SYN flood attack if thousands here). FIN_WAIT: Local initiated close, waiting for remote. Large TIME_WAIT count is normal on busy servers; CLOSE_WAIT buildup indicates an application bug.

**Q7.** What is anycast and how does Cloudflare use it?
> Anycast assigns the same IP address to multiple servers in different locations. BGP routing sends traffic to the nearest (fewest hops) server. Cloudflare announces the same IPs from 200+ data centers globally — your request automatically goes to the nearest Cloudflare PoP. This enables low-latency CDN, DDoS mitigation (attack traffic is distributed), and resilience.

**Q8.** What's the difference between a Layer 4 and Layer 7 health check?
> Layer 4: Just establishes a TCP connection — confirms the server is up and the port is listening. Fast, low overhead. Layer 7: Makes an actual HTTP request to a specific path and checks the response code (and optionally body). Confirms the application itself is healthy, not just that the server is running. Use L7 health checks for web applications in production.

### Docker & Kubernetes (9–15)

**Q9.** Explain how DNS works inside Kubernetes for inter-service communication.
> CoreDNS runs as a Deployment in `kube-system`. Each pod's `/etc/resolv.conf` points to CoreDNS ClusterIP. `myservice.mynamespace.svc.cluster.local` resolves to the service's ClusterIP. The service selector matches pod labels, and kube-proxy maintains iptables rules to DNAT ClusterIP traffic to pod IPs.

**Q10.** A pod is in `CrashLoopBackOff`. How do you diagnose and fix it?
> `kubectl describe pod <name>` — check Events for OOM, config errors, probe failures. `kubectl logs <pod> --previous` — logs from the crashed container. `kubectl exec -it <pod> -- sh` — inspect runtime state. Common causes: wrong env vars, missing config/secret, crashed on startup exception, OOM (check `limits.memory`), bad liveness probe.

**Q11.** What is the role of `etcd` in Kubernetes networking?
> `etcd` is Kubernetes' distributed key-value store — the single source of truth. All cluster state (pods, services, endpoints, configmaps, network policies) is stored in etcd. When a Service is created, the API server writes to etcd; kube-proxy watches etcd (via API server) and updates iptables. CoreDNS watches for Service/Endpoint changes via etcd. etcd availability is critical — losing etcd means losing the ability to make any cluster state changes.

**Q12.** How do you expose a Kubernetes app to the internet with TLS?
> (1) Create an Ingress resource with TLS secret reference. (2) Use cert-manager with Let's Encrypt to auto-provision and renew certificates. (3) The Ingress controller (nginx/traefik) handles TLS termination. (4) Traffic flows: User → DNS → Ingress Controller LB IP → Ingress rules → ClusterIP Service → Pod. Alternatively, use ACM certificates with AWS ALB Ingress Controller.

**Q13.** What is a DaemonSet and give a networking use case?
> A DaemonSet ensures one pod runs on every node (or selected nodes). Networking use cases: CNI plugins (Calico, Flannel node agents), kube-proxy, monitoring agents (Prometheus Node Exporter), log collectors (Fluentd/Filebeat), network security agents.

**Q14.** How does Kubernetes handle pod-to-pod traffic on different nodes with Flannel vs. Calico?
> **Flannel (VXLAN mode):** Encapsulates pod packets in UDP/VXLAN packets with node IPs as outer headers. This adds ~50 bytes overhead and requires UDP 8472 open between nodes. **Calico (BGP mode):** Each node peers with others (or a route reflector) via BGP and advertises its pod CIDR. No encapsulation — routing is at Layer 3 with native performance. Calico also supports eBPF dataplane for even higher performance and NetworkPolicy enforcement.

**Q15.** Explain what happens when `kubectl apply -f deployment.yaml` is run.
> `kubectl` sends the YAML to the API server → API server validates and stores in etcd → Deployment Controller detects the new Deployment → creates ReplicaSet → Scheduler assigns pods to nodes → Kubelet on each node pulls the image (from registry) → Container runtime creates the container → CNI plugin assigns IP and sets up networking → Probes pass → Pod marked Ready → Service Endpoints updated → kube-proxy updates iptables.

### CI/CD & Infrastructure (16–20)

**Q16.** What is GitOps and how does it differ from traditional CI/CD?
> In GitOps, Git is the single source of truth for both infrastructure and application state. An agent (ArgoCD, Flux) continuously reconciles the cluster state with what's in Git. Differences: In traditional CI/CD, a pipeline pushes changes to the cluster. In GitOps, the cluster pulls changes from Git. Benefits: Auditability (Git history = deployment history), easy rollback (revert a commit), declarative drift detection.

**Q17.** How would you set up a secure, minimal Jenkins agent in AWS?
> EC2 instance in a private subnet. IAM role with least-privilege (only ECR push, specific S3 bucket access). Security group: inbound port 22 from Jenkins master's SG only (or use SSM Session Manager — no port 22 needed). Outbound: HTTPS to package registries, Docker Hub, ECR. No public IP. Use NAT gateway for outbound. Harden OS: disable root login, enable auditd, apply patches via SSM Patch Manager.

**Q18.** Describe a Terraform deployment pipeline for an AWS environment.
> (1) PR: `terraform fmt` + `terraform validate` (syntax check) + `tflint` (linting) + `checkov` (security scan) in CI. (2) PR comment with `terraform plan` output (using Atlantis or GitHub Actions). (3) Merge to main: `terraform plan` again to confirm no drift. (4) Manual approval gate. (5) `terraform apply` with remote state locked in S3+DynamoDB. (6) Post-apply: Tag the commit, notify Slack, run smoke tests.

**Q19.** How do you handle database migrations in a CI/CD pipeline with zero downtime?
> Use an expand-contract pattern: (1) **Expand:** Add new column (nullable, backward compatible) — both old and new app can run. (2) **Migrate:** Backfill data. (3) **Contract:** Deploy new app version using new column. (4) **Cleanup:** Remove old column once old version is retired. Tools: Flyway, Liquibase for migration versioning. Never deploy code and DB changes atomically if they're breaking changes.

**Q20.** What networking considerations exist for a multi-region Kubernetes deployment?
> Global load balancing (AWS Route 53 Latency routing, Cloudflare). Cross-region service discovery (possibly a service mesh). Data replication latency (eventual consistency). Independent control planes per region (avoid cross-region API server dependency). Consistent CIDRs (no overlapping pod/service CIDRs). Inter-region communication via VPC peering, Transit Gateway, or PrivateLink. Compliance: data residency requirements may restrict cross-region data flow.

### Advanced & Scenario (21–25)

**Q21.** A production pod is experiencing packet loss. How would you diagnose?
> Layer by layer: (1) `ping` from pod to destination — loss rate? (2) `mtr pod-ip destination-ip` — which hop drops? (3) `tcpdump` on both pod's interface and the node's interface — packets leaving pod but not arriving at node (CNI bug)? (4) Check network interfaces for errors: `ip -s link`. (5) Check iptables rules: `iptables -L -n -v` for drops. (6) Check kernel: `dmesg | grep -i drop`. (7) Check security groups and NACLs for stateless drops.

**Q22.** How would you implement service mesh in Kubernetes and what networking problems does it solve?
> Install Istio (or Linkerd). It injects a sidecar proxy (Envoy) into every pod. All traffic goes through the sidecar. Problems solved: (1) **Mutual TLS** — automatic encryption between all services. (2) **Traffic management** — canary deployments, circuit breaking, retries, timeouts at the network level without code changes. (3) **Observability** — L7 metrics, distributed tracing, service topology maps automatically. (4) **Authorization policy** — service-to-service authz beyond NetworkPolicy.

**Q23.** Explain eBPF and how it's used in modern Kubernetes networking.
> eBPF (extended Berkeley Packet Filter) allows running sandboxed programs in the Linux kernel without changing kernel source or loading kernel modules. In networking: eBPF programs attach to network interfaces and process/modify packets directly in the kernel, bypassing iptables completely. Cilium uses eBPF to implement service routing, NetworkPolicy, and load balancing with significantly lower overhead than iptables (especially at scale — iptables is O(N) per rule; eBPF hash maps are O(1)).

**Q24.** How do you debug a "502 Bad Gateway" from an NGINX ingress controller?
> 502 = NGINX couldn't get a valid response from the upstream (your pod). Steps: (1) Check if the upstream pod is running and healthy: `kubectl get pods`. (2) Check if the Service endpoints exist: `kubectl get endpoints`. (3) Exec into the ingress pod and `curl <pod-ip>:<port>` directly. (4) Check ingress controller logs: `kubectl logs <ingress-pod>`. (5) Check if the upstream is accepting connections (wrong port in service spec?). (6) Check if the app is crashing after receiving the request (OOM, exception). (7) Check upstream connection timeout config.

**Q25.** Design a networking architecture for a globally distributed, highly available e-commerce platform.
> Multi-region active-active (or active-standby): Cloudflare for global CDN + DDoS protection + global load balancing with latency routing. Per region: VPC with 3 AZs, ALB in public subnets, EKS cluster in private subnets, RDS Aurora Global Database (multi-region replication), ElastiCache for session storage. Microservices: Istio service mesh for mTLS and traffic management. Database writes go to primary region; reads to nearest replica. Failover: Route 53 health checks + failover routing. Security: WAF on ALB, VPC endpoints for AWS services, GuardDuty for threat detection, CloudTrail for audit.

---

## 17. 30-Day Mastery Roadmap

### Week 1: Networking Foundations (Days 1–7)

| Day | Topic | Action |
|-----|-------|--------|
| 1 | OSI & TCP/IP | Draw the model from memory. Use Wireshark on local traffic. |
| 2 | IP Addressing & Subnetting | Practice subnetting (subnettingpractice.com). Learn CIDR. |
| 3 | DNS deep dive | Use `dig` to trace DNS resolution. Set up your own DNS server locally. |
| 4 | HTTP/HTTPS & TLS | Use `curl -v` on various sites. Inspect TLS certs. |
| 5 | Routing & Switching | Set up VMs with VirtualBox, configure static routes. |
| 6 | Linux networking | Master `ip`, `ss`, `tcpdump`, `iptables` commands. |
| 7 | Review + Quiz | Write 10 Q&As from memory. Teach concepts to someone else. |

### Week 2: Containers & Orchestration (Days 8–14)

| Day | Topic | Action |
|-----|-------|--------|
| 8 | Docker basics | Install Docker. Build, run, stop containers. |
| 9 | Docker networking | Create bridge networks. Debug container-to-container comm. |
| 10 | Docker Compose | Write a 3-tier app (nginx + app + db) in Docker Compose. |
| 11 | Kubernetes basics | Install kind/minikube. Deploy first app. |
| 12 | K8s networking | Create Services (all types), trace traffic with `tcpdump`. |
| 13 | K8s Ingress + NetworkPolicy | Set up nginx ingress. Write and test NetworkPolicies. |
| 14 | Review + Project | Deploy a full stack app on Kubernetes with Ingress + TLS. |

### Week 3: DevOps Tools & CI/CD (Days 15–21)

| Day | Topic | Action |
|-----|-------|--------|
| 15 | Git advanced | Practice rebasing, cherry-picking, merge conflict resolution. |
| 16 | Jenkins / GitHub Actions | Create a CI pipeline that builds and tests a Docker image. |
| 17 | CI/CD + K8s deployment | Extend pipeline to deploy to Kubernetes. |
| 18 | Terraform basics | Provision AWS VPC + EC2 with Terraform. |
| 19 | Terraform advanced | Write a reusable VPC module. Implement remote state. |
| 20 | Ansible | Write a playbook to configure a web server. |
| 21 | Review + Project | End-to-end IaC: Terraform (infra) + Ansible (config) + CI deploy. |

### Week 4: Cloud, Monitoring & Integration (Days 22–30)

| Day | Topic | Action |
|-----|-------|--------|
| 22 | AWS VPC deep dive | Build the 3-tier VPC architecture by hand in AWS console. |
| 23 | AWS ALB + WAF | Configure path-based routing, health checks, WAF rules. |
| 24 | AWS Security Groups + NACLs | Practice least-privilege SG configs. |
| 25 | Prometheus + Grafana | Install on Kubernetes. Create dashboards for golden signals. |
| 26 | Alerting + SLOs | Configure Alertmanager. Define SLIs/SLOs for a service. |
| 27 | Real-world scenarios | Debug 5 intentionally broken scenarios in your lab. |
| 28 | Mock interviews | Answer 10 Q&As verbally (time yourself). |
| 29 | Weak areas | Revisit topics you struggled with. |
| 30 | Final review | Read through this guide. Teach the key concepts. You're ready. |

### Recommended Resources

| Category | Resource |
|----------|----------|
| **Networking** | "Computer Networking: Top-Down Approach" (Kurose & Ross) |
| **Linux** | "The Linux Command Line" (William Shotts) - free online |
| **Docker** | Docker official documentation + Play with Docker labs |
| **Kubernetes** | Kubernetes.io tutorials + KodeKloud labs |
| **AWS Networking** | AWS Networking Specialty exam guide (free on AWS) |
| **Terraform** | HashiCorp Learn (developer.hashicorp.com) |
| **Practice** | killer.sh, KodeKloud, Katacoda scenarios |
| **Interview Prep** | DevOps Roadmap (roadmap.sh/devops) |

### Key Principles to Remember

```
1. MENTAL MODEL FIRST
   Understand WHY before HOW.
   If you understand WHY a packet is dropped,
   you can debug any network issue.

2. TRACE THE PACKET
   For any networking problem: follow the packet
   Client → DNS → TCP → TLS → LB → Backend → Response
   Something failed at one of these steps.

3. LAYERS SAVE TIME
   Use the OSI model as a debugging framework:
   Physical → Link → Network → Transport → Application
   Eliminate one layer at a time.

4. EVERYTHING IS NETWORKING
   Kubernetes services, Docker bridges, VPC routing,
   CI/CD webhooks — they're all networking at their core.

5. TEST WITH MINIMAL TOOLS
   curl, ping, telnet, nc, dig, tcpdump —
   these 7 tools can diagnose 90% of networking issues.
```

---

*Guide Version 1.0 | Last Updated: May 2026*
*Coverage: OSI/TCP/IP | DNS/HTTP | Routing | Linux | Git | Docker | Kubernetes | CI/CD | Terraform | Ansible | Prometheus | AWS VPC | Load Balancers | Mock Interview | 30-Day Roadmap*
