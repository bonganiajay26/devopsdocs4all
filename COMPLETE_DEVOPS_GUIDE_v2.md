# Complete DevOps Learning System
### Principal DevOps Architect | SRE | Platform Engineer | Interview Coach
### Beginner → Expert | 20+ Years Production Experience

---

## Table of Contents

| Phase | Topic | Key Tools |
|-------|-------|-----------|
| 1 | Linux & Networking | OS internals, TCP/IP, DNS, SSH, iptables |
| 2 | Git & Version Control | Git internals, GitHub Flow, GitOps |
| 3 | CI/CD | Jenkins, GitHub Actions, GitLab CI, ArgoCD |
| 4 | Docker & Containers | Docker internals, namespaces, cgroups, OCI |
| 5 | Kubernetes | kube-apiserver, etcd, scheduler, networking, Istio |
| 6 | Infrastructure as Code | Terraform, Ansible, Helm, Pulumi |
| 7 | Monitoring & Observability | Prometheus, Grafana, ELK, Loki, Jaeger, OTel |
| 8 | Cloud & Security | AWS, IAM, Vault, Zero Trust, DevSecOps |

---

## How to Use This Guide

1. **Beginners**: Start at Phase 1, read everything top-to-bottom
2. **Intermediate**: Jump to your weak area, use interview questions to self-test
3. **Interview Prep**: Each phase ends with Q&A from beginner → advanced
4. **Production Reference**: Use troubleshooting sections for real incidents

---

# Phase 1: Linux & Networking — Complete Deep Dive

---

## Table of Contents
1. [Linux Internals](#1-linux-internals)
2. [Process Management](#2-process-management)
3. [Memory Management](#3-memory-management)
4. [File System Internals](#4-file-system-internals)
5. [Networking — TCP/IP Deep Dive](#5-networking--tcpip-deep-dive)
6. [DNS — Complete Internals](#6-dns--complete-internals)
7. [HTTP / HTTPS Protocol](#7-http--https-protocol)
8. [SSH — Internals & Security](#8-ssh--internals--security)
9. [Firewalls & iptables](#9-firewalls--iptables)
10. [System Troubleshooting](#10-system-troubleshooting)
11. [Interview Questions — Phase 1](#11-interview-questions--phase-1)

---

## 1. Linux Internals

### What is Linux?

Linux is an open-source, Unix-like operating system kernel written by Linus Torvalds in 1991. It forms the foundation of almost all DevOps infrastructure — servers, containers, cloud VMs, embedded systems.

**Why Linux for DevOps?**
- Stable, predictable, open-source
- Superior process isolation (used by Docker, Kubernetes)
- Excellent networking stack
- Free and extensively documented
- Runs on everything: bare metal, VMs, containers, cloud

### Linux Kernel Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     USER SPACE                              │
│  Applications: nginx, postgres, python, docker, kubectl      │
│  System Libraries: glibc, libpthread, libssl                │
│  System Calls Interface (syscall API)                       │
├─────────────────────────────────────────────────────────────┤
│                     KERNEL SPACE                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Process  │  │ Memory   │  │  File    │  │ Network  │   │
│  │ Manager  │  │ Manager  │  │ System   │  │  Stack   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                Device Drivers                        │   │
│  └──────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                     HARDWARE                                │
│  CPU  |  RAM  |  Disk  |  NIC  |  GPU                      │
└─────────────────────────────────────────────────────────────┘
```

### System Call Flow (How User Space Talks to Kernel)

```
nginx makes read() syscall
        │
        ▼
   glibc wrapper
        │
        ▼
   syscall instruction (x86: int 0x80 or syscall)
        │
        ▼
   CPU switches to kernel mode (Ring 0)
        │
        ▼
   Kernel sys_read() handler
        │
        ▼
   VFS (Virtual File System) layer
        │
        ▼
   ext4 / xfs driver
        │
        ▼
   Block device driver
        │
        ▼
   Data returned → CPU switches back to user mode
```

**Important System Calls in DevOps:**

| Syscall | Purpose | DevOps Relevance |
|---------|---------|-----------------|
| `fork()` | Create child process | Container init, process spawning |
| `exec()` | Replace process image | Shell execution |
| `clone()` | Create thread/process with flags | Docker namespaces use this |
| `mmap()` | Memory-mapped files | Java heap, shared memory |
| `epoll()` | Event notification | nginx, Node.js async I/O |
| `socket()` | Create network socket | Every network connection |
| `ioctl()` | Device control | Network interface config |
| `unshare()` | Disassociate namespaces | Docker uses this |

---

## 2. Process Management

### What is a Process?

A **process** is an instance of a running program. It has:
- A unique **PID** (Process ID)
- Virtual address space (code, stack, heap, libraries)
- File descriptors (open files, sockets)
- A parent process (PPID)
- CPU registers, priority, scheduling state

### Process Lifecycle

```
              fork()
[Parent] ────────────► [Child - RUNNABLE]
                              │
                         exec() → loads program
                              │
                         [RUNNING] ◄──── CPU scheduler
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         [SLEEPING]      [STOPPED]       [ZOMBIE]
         waiting I/O     SIGSTOP        exited, parent
                                        hasn't wait()ed
                              │
                         wait()/waitpid()
                              │
                         [REMOVED from table]
```

### Process States

| State | Code | Meaning |
|-------|------|---------|
| Running | R | On CPU or in run queue |
| Sleeping (interruptible) | S | Waiting for event, can be interrupted |
| Sleeping (uninterruptible) | D | Waiting for I/O (disk), cannot be killed! |
| Zombie | Z | Finished but parent hasn't called wait() |
| Stopped | T | Paused via SIGSTOP |
| Dead | X | Being removed |

> **Production Issue:** Processes stuck in **D state** usually indicate disk I/O hang or NFS mount issues. You can't kill them with `kill -9`. Requires fixing the underlying I/O.

### Process Hierarchy

```
systemd (PID 1)
├── sshd (PID 234)
│   └── sshd: user@pts/0 (PID 1001)
│       └── bash (PID 1002)
│           └── top (PID 1003)
├── dockerd (PID 567)
│   └── containerd (PID 568)
│       └── nginx container (PID 900)
│           └── nginx worker (PID 901)
└── cron (PID 789)
```

### Essential Process Commands

```bash
# View all processes
ps aux
ps -ef

# Process tree
pstree -p

# Real-time process monitor
top
htop

# Find process by name
pgrep nginx
pidof nginx

# Process resource usage
cat /proc/1234/status        # Process info
cat /proc/1234/cmdline       # Command that started it
cat /proc/1234/fd/           # Open file descriptors
cat /proc/1234/net/tcp       # Network connections
ls -la /proc/1234/maps       # Memory maps

# Kill signals
kill -15 PID    # SIGTERM - graceful shutdown (default)
kill -9 PID     # SIGKILL - force kill (no cleanup)
kill -1 PID     # SIGHUP  - reload config (nginx uses this)
kill -19 PID    # SIGSTOP - pause process
kill -18 PID    # SIGCONT - resume process

# Niceness (priority: -20 highest, 19 lowest)
nice -n 10 python script.py      # start with low priority
renice 5 -p 1234                 # change running process priority
```

### How the Linux Scheduler Works

Linux uses **CFS (Completely Fair Scheduler)** since kernel 2.6.23:

```
All runnable processes compete in a red-black tree
ordered by "virtual runtime" (vruntime)

Process with LOWEST vruntime → gets CPU next
│
▼
CPU quantum (typically 4ms by default)
│
▼
Process runs, vruntime increases
│
▼
Re-inserted into tree at new position
│
▼
Next process with lowest vruntime runs
```

**Key scheduler concepts for DevOps:**
- `nice` values adjust scheduling weight, not time slices
- Real-time processes (`SCHED_FIFO`, `SCHED_RR`) preempt normal processes
- CPU pinning (`taskset`, `numactl`) important for latency-sensitive apps
- `cgroups cpu` subsystem can enforce CPU limits (what Docker uses)

---

## 3. Memory Management

### Linux Memory Layout (Per Process)

```
High Address (0xFFFFFFFF)
┌─────────────────────┐
│   Kernel Space      │  ← kernel, not accessible in user mode
├─────────────────────┤ 0xC0000000 (32-bit) / 128TB (64-bit)
│   Stack             │  ← grows downward, function call frames
│   (grows ↓)         │
├─────────────────────┤
│   (gap)             │
├─────────────────────┤
│   Memory Mapped     │  ← mmap(), shared libraries, anonymous maps
│   Files/libs        │
├─────────────────────┤
│   Heap              │  ← malloc/free, grows upward
│   (grows ↑)         │
├─────────────────────┤
│   BSS Segment       │  ← uninitialized globals
├─────────────────────┤
│   Data Segment      │  ← initialized globals
├─────────────────────┤
│   Text Segment      │  ← program code (read-only)
└─────────────────────┘
Low Address (0x00000000)
```

### Virtual Memory and Page Tables

Every process has its own **virtual address space**. The kernel maps virtual pages to physical pages through **page tables**.

```
Virtual Address (process)           Physical RAM
┌─────────────────┐                ┌─────────────────┐
│ 0x7fff000 Stack │ ───page table─►│ Physical Page 5  │
│ 0x7ffe000       │                │                  │
│ ...             │                ├─────────────────┤
│ 0x0804800 Text  │ ───page table─►│ Physical Page 12 │
└─────────────────┘                └─────────────────┘
     Process A                          RAM
```

**Page Fault:** When a process accesses a virtual page not yet in RAM:
1. CPU raises page fault exception
2. Kernel page fault handler runs
3. Kernel allocates physical page, updates page table
4. Process continues

### Memory Types You'll Monitor

```bash
# Full memory info
free -h
cat /proc/meminfo

# Key fields:
# MemTotal   - total physical RAM
# MemFree    - completely unused RAM
# MemAvailable - RAM available without swapping (more useful than Free)
# Buffers    - kernel buffer cache (raw disk blocks)
# Cached     - page cache (file contents)
# SwapTotal/SwapFree - swap space
# Dirty      - pages modified but not yet written to disk
```

### OOM Killer (Out of Memory)

When system runs out of memory, the **OOM Killer** terminates processes:

```
RAM exhausted
     │
     ▼
OOM Killer activates
     │
     ▼
Scores each process (oom_score: 0-1000)
Higher score = more likely to be killed
     │
     ▼
Process with highest score is killed
     │
     ▼
Memory freed → system continues

# Check oom_score
cat /proc/1234/oom_score

# Protect process from OOM killer
echo -1000 > /proc/1234/oom_score_adj

# Check dmesg for OOM kills
dmesg | grep -i "oom\|killed process"
```

**Production Issue:** Java applications frequently get OOM-killed because:
1. JVM heap is reserved but not all used
2. Off-heap memory (metaspace, thread stacks) not accounted for
3. Kubernetes sets memory limits that don't account for JVM overhead

**Fix:** Set `-XX:MaxRAMPercentage=75` and ensure container `memory limit > JVM heap`.

---

## 4. File System Internals

### Linux Virtual File System (VFS)

Linux uses a **VFS layer** to provide a uniform interface for all file systems:

```
Application
    │ open("/etc/nginx/nginx.conf")
    ▼
VFS Layer (virtual file system)
    │
    ├── ext4 driver (/dev/sda1)
    ├── xfs driver  (/dev/sdb1)
    ├── tmpfs       (RAM-based, /tmp)
    ├── procfs      (/proc - process info)
    ├── sysfs       (/sys - kernel objects)
    ├── cgroup fs   (/sys/fs/cgroup)
    └── overlayfs   (Docker layers)
```

### Inodes

Every file/directory has an **inode** (index node) containing metadata:

```bash
# View inode info
stat /etc/nginx/nginx.conf
# Output:
# File: /etc/nginx/nginx.conf
# Size: 2416      Blocks: 8      IO Block: 4096   regular file
# Inode: 524289   Links: 1
# Access: 2024-01-15 10:23:45
# Modify: 2024-01-10 09:12:33
# Change: 2024-01-10 09:12:33

# View inode number
ls -i /etc/nginx/nginx.conf

# Check inode usage (can run out of inodes!)
df -i
```

> **Production Issue:** "No space left on device" even when `df -h` shows free space → check `df -i`, inodes may be exhausted. Common on log servers with millions of small files.

### Important Linux Directories for DevOps

```
/proc/          - Virtual FS: process and kernel info
  /proc/sys/    - Kernel parameters (sysctl)
  /proc/net/    - Network statistics
/sys/           - Kernel object hierarchy
  /sys/fs/cgroup/ - Control groups (used by Docker, K8s)
/var/log/       - System logs
  /var/log/syslog or /var/log/messages
  /var/log/auth.log
  /var/log/kern.log
/etc/           - Configuration files
/tmp/           - Temporary files (tmpfs - in RAM)
/run/           - Runtime data (PIDs, sockets)
/dev/           - Device files
  /dev/null     - Discard
  /dev/zero     - Zero bytes
  /dev/random   - Random bytes
  /dev/sda      - First SATA disk
  /dev/nvme0n1  - NVMe disk
```

### File Permissions Deep Dive

```
-rwxr-xr--  1  nginx  www-data  2416  Jan 10 file.conf
│└┬┘└┬┘└┬┘
│ │  │  └── Other permissions: r-- (read only)
│ │  └───── Group permissions: r-x (read+execute)
│ └──────── Owner permissions: rwx (read+write+execute)
└────────── File type: - (regular), d (dir), l (symlink),
                        b (block device), c (char device)

Octal:
r=4, w=2, x=1
rwx = 7, rw- = 6, r-x = 5, r-- = 4

Special bits:
4000 = SUID (run as owner, e.g. sudo, passwd)
2000 = SGID (run as group / inherit group for dirs)
1000 = Sticky bit (only owner can delete, /tmp)

# Examples
chmod 755 /usr/local/bin/myapp   # rwxr-xr-x
chmod 644 /etc/app/config        # rw-r--r--
chmod 600 ~/.ssh/id_rsa          # rw------- (private key!)
chmod 700 ~/.ssh                 # rwx------

# Ownership
chown nginx:www-data /var/log/nginx/
chown -R app:app /opt/myapp/
```

### Linux File Descriptor Limits (Critical for DevOps)

```bash
# System-wide max open files
cat /proc/sys/fs/file-max

# Per-process limit
ulimit -n          # Soft limit (current process)
ulimit -Hn         # Hard limit

# Check open files for a process
lsof -p 1234 | wc -l
ls /proc/1234/fd | wc -l

# Permanently set higher limits (for nginx, postgres)
# /etc/security/limits.conf:
nginx soft nofile 65536
nginx hard nofile 65536

# Or systemd service:
# [Service]
# LimitNOFILE=65536
```

> **Production Issue:** nginx/Node.js under load: "too many open files" error. Fix: raise `ulimit -n` and set in systemd unit file.

---

## 5. Networking — TCP/IP Deep Dive

### The TCP/IP Model vs OSI Model

```
OSI Model          TCP/IP Model       Protocols
┌────────────┐     ┌────────────┐
│ Application│     │ Application│  HTTP, HTTPS, DNS, SSH, SMTP, FTP
├────────────┤     │            │
│Presentation│     │            │  TLS/SSL, gRPC encoding
├────────────┤     │            │
│  Session   │     │            │  TLS sessions, WebSockets
├────────────┤─────├────────────┤
│ Transport  │     │ Transport  │  TCP, UDP
├────────────┤─────├────────────┤
│  Network   │     │  Internet  │  IP, ICMP, IPsec
├────────────┤─────├────────────┤
│  Data Link │     │  Network   │  Ethernet, WiFi, ARP
├────────────┤     │  Access    │
│  Physical  │     │            │  Cables, fiber, radio
└────────────┘     └────────────┘
```

### IP Addressing — Complete Reference

```
IPv4 Address: 192.168.1.100/24
              └──────┬──────┘ └─┬─┘
                     │          └── CIDR notation (prefix length)
                     └─────────────── 32-bit address

Network Classes (legacy, replaced by CIDR):
Class A: 1.0.0.0    - 126.255.255.255   /8   (16M hosts)
Class B: 128.0.0.0  - 191.255.255.255   /16  (65K hosts)
Class C: 192.0.0.0  - 223.255.255.255   /24  (254 hosts)

Private (RFC 1918) - not routable on Internet:
10.0.0.0/8         (10.x.x.x)
172.16.0.0/12      (172.16.x.x - 172.31.x.x)
192.168.0.0/16     (192.168.x.x)

Special:
127.0.0.1/8        Loopback (localhost)
0.0.0.0/0          Default route (all traffic)
169.254.0.0/16     Link-local (APIPA, AWS metadata)
255.255.255.255    Broadcast

CIDR Subnet Examples:
/32  = 1 host       (single IP, e.g., host route)
/31  = 2 hosts      (point-to-point links)
/30  = 4 hosts      (2 usable + network + broadcast)
/29  = 8 hosts      (6 usable)
/28  = 16 hosts     (14 usable)
/27  = 32 hosts     (30 usable)
/26  = 64 hosts     (62 usable)
/25  = 128 hosts    (126 usable)
/24  = 256 hosts    (254 usable)  ← most common LAN
/23  = 512 hosts    (510 usable)
/22  = 1024 hosts   (1022 usable)
/20  = 4096 hosts   ← common Kubernetes pod CIDR
/16  = 65536 hosts  ← VPC CIDR
/8   = 16M hosts
```

### TCP — Three-Way Handshake

```
Client                          Server
  │                               │
  │──── SYN (seq=x) ─────────────►│  Client initiates
  │                               │  Server allocates resources
  │◄─── SYN-ACK (seq=y, ack=x+1)─│  Server responds
  │                               │
  │──── ACK (ack=y+1) ───────────►│  Connection established
  │                               │
  │═══════ DATA TRANSFER ═════════│
  │                               │
  │──── FIN ─────────────────────►│  Client closes
  │◄─── ACK ──────────────────────│
  │◄─── FIN ──────────────────────│  Server closes
  │──── ACK ─────────────────────►│
  │                               │
  [TIME_WAIT ~60s]                │

TCP Flags:
SYN = Synchronize (start connection)
ACK = Acknowledge (confirm receipt)
FIN = Finish (close connection)
RST = Reset (hard close, error)
PSH = Push (send data immediately)
URG = Urgent
```

### TCP State Machine

```
           LISTEN (server waiting)
               │
         SYN received
               │
         SYN-ACK sent
               │
           SYN_RCVD
               │
           ACK received
               │
          ESTABLISHED ◄──── active connections here
          /         \
       FIN_WAIT_1  CLOSE_WAIT
          │              │
       FIN_WAIT_2   LAST_ACK
          │              │
       TIME_WAIT    CLOSED
          │
       CLOSED (after 2*MSL ≈ 60s)
```

**Why TIME_WAIT matters for DevOps:**
```bash
# Too many TIME_WAIT → can't make new connections!
ss -s | grep TIME-WAIT

# Fix (tune carefully in production):
# /etc/sysctl.conf
net.ipv4.tcp_tw_reuse = 1          # Reuse TIME_WAIT sockets
net.ipv4.ip_local_port_range = 1024 65535  # More ephemeral ports
net.ipv4.tcp_fin_timeout = 15      # Reduce from default 60s

sysctl -p  # Apply changes
```

### TCP vs UDP Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented (handshake) | Connectionless |
| Reliability | Guaranteed delivery, retransmit | Best-effort, no retransmit |
| Order | Ordered packets | No ordering |
| Speed | Slower (overhead) | Faster |
| Flow control | Yes | No |
| Use cases | HTTP, SSH, DB | DNS, DHCP, video streaming |
| Header size | 20+ bytes | 8 bytes |

**DevOps relevance:**
- DNS uses UDP port 53 (falls back to TCP for large responses)
- Kubernetes etcd uses TCP (reliability critical)
- Prometheus scrapes use HTTP/TCP
- Syslog can use UDP (fast logging, some loss acceptable)
- Modern: QUIC protocol (HTTP/3) = UDP + reliability at application layer

### Networking Commands — Complete Reference

```bash
# Interface information
ip addr show                    # Modern: IP addresses
ip link show                    # Interface status
ifconfig                        # Legacy
ip -s link show eth0            # Statistics

# Routing
ip route show                   # Routing table
ip route get 8.8.8.8            # Which route used for IP
route -n                        # Legacy routing table

# Connections
ss -tlnp                        # TCP listening sockets + PID
ss -tulnp                       # TCP+UDP listening
ss -s                           # Summary
netstat -tlnp                   # Legacy (deprecated)

# DNS
dig google.com                  # Full DNS query
dig google.com +short           # Just the answer
dig @8.8.8.8 google.com        # Query specific nameserver
dig -x 8.8.8.8                  # Reverse DNS lookup
nslookup google.com             # Simple lookup
host google.com                 # Simple lookup

# Connectivity testing
ping -c 4 8.8.8.8               # ICMP ping
traceroute 8.8.8.8              # Hop-by-hop path
tracepath 8.8.8.8               # Similar, no root needed
mtr 8.8.8.8                     # Continuous traceroute (best tool!)
curl -v https://api.example.com # Full HTTP request debug
curl -w "%{time_total}" url     # Timing breakdown

# Packet capture
tcpdump -i eth0                 # Capture all traffic
tcpdump -i eth0 port 80         # HTTP only
tcpdump -i eth0 host 10.0.0.5   # From/to specific host
tcpdump -i eth0 -w capture.pcap # Write to file for Wireshark

# Network performance
iperf3 -s                       # Server mode
iperf3 -c server-ip             # Test bandwidth
ss -i                           # TCP stats per connection

# ARP (layer 2 - MAC mapping)
arp -n                          # ARP table
ip neigh show                   # Modern ARP table

# Firewall rules
iptables -L -n -v               # List rules
iptables -L -n -v -t nat        # NAT table
nft list ruleset                # nftables (modern)
```

### Network Troubleshooting Flow

```
User reports: "Can't connect to the application"
                        │
                        ▼
              Is it DNS or IP issue?
              dig app.example.com → resolves?
                        │
          ┌─────────────┴──────────────┐
         YES                          NO
          │                            │
    ping IP → reachable?          Check /etc/resolv.conf
          │                       Check DNS server
     ┌────┴────┐                  Check firewall on DNS port 53
    YES        NO
     │          │
  Port open?  Routing issue?     
  telnet/nc   ip route get       
     │         traceroute        
  ┌──┴──┐                        
 YES    NO                       
  │      │                       
App    Firewall/                  
issue  Security Group            
  │
Check app logs
Check app port binding (ss -tlnp)
Check app error logs (/var/log/)
```

---

## 6. DNS — Complete Internals

### What is DNS?

**Domain Name System (DNS)** is a distributed, hierarchical, globally distributed database that maps human-readable domain names to IP addresses (and other records).

**Why DNS is critical in DevOps:**
- Kubernetes uses DNS for service discovery (CoreDNS)
- Load balancers are reached via DNS
- Microservices communicate via DNS names
- Deployment strategies (blue/green) use DNS switching

### DNS Hierarchy

```
Root DNS Servers (13 clusters, globally anycast)
"." (dot)
        │
        ├── .com (TLD - Top Level Domain)
        │       │
        │       ├── google.com (Authoritative)
        │       │       │
        │       │       ├── www → 142.250.x.x
        │       │       └── mail → 209.85.x.x
        │       │
        │       └── amazon.com (Authoritative)
        │
        ├── .org
        ├── .net
        ├── .io
        └── .in
```

### Full DNS Resolution Flow

```
Browser requests: www.google.com
        │
        ▼
1. Check local DNS cache (browser cache)
        │ not found
        ▼
2. Check OS cache (/etc/hosts, nscd)
        │ not found
        ▼
3. OS asks Recursive Resolver (your ISP or 8.8.8.8)
   [set in /etc/resolv.conf or DHCP]
        │
        ▼
4. Recursive Resolver checks its cache
        │ not found
        ▼
5. Recursive Resolver asks Root Server
   "Who knows about .com?"
        │
        ▼
6. Root Server: "Ask a.gtld-servers.net"
        │
        ▼
7. Recursive Resolver asks TLD Server (a.gtld-servers.net)
   "Who knows about google.com?"
        │
        ▼
8. TLD Server: "Ask ns1.google.com"
        │
        ▼
9. Recursive Resolver asks Authoritative Server (ns1.google.com)
   "What is the IP of www.google.com?"
        │
        ▼
10. Authoritative: "142.250.x.x" (with TTL)
        │
        ▼
11. Recursive Resolver caches result + returns to OS
        │
        ▼
12. OS returns to browser, caches locally
        │
        ▼
Browser connects to 142.250.x.x:443
```

### DNS Record Types

```
A       → IPv4 address mapping
          www.example.com → 93.184.216.34

AAAA    → IPv6 address mapping
          www.example.com → 2606:2800:220:1:248:1893:25c8:1946

CNAME   → Alias to another name (cannot be on apex/root domain)
          blog.example.com → example.github.io

MX      → Mail exchange server (with priority)
          example.com MX 10 mail.example.com

TXT     → Arbitrary text (SPF, DKIM, domain verification)
          example.com TXT "v=spf1 include:_spf.google.com ~all"

NS      → Name server for the zone
          example.com NS ns1.example.com

PTR     → Reverse DNS (IP → name)
          34.216.184.93.in-addr.arpa → www.example.com

SRV     → Service location (host + port)
          _https._tcp.example.com SRV 10 5 443 www.example.com
          Used by: Kubernetes, SIP, XMPP

CAA     → Certificate Authority Authorization
          example.com CAA 0 issue "letsencrypt.org"

SOA     → Start of Authority (zone metadata)
          Serial, refresh, retry, expiry

TTL     → Time To Live (caching duration in seconds)
          Low TTL (60s) = fast propagation, more DNS load
          High TTL (86400s) = less DNS load, slow changes
```

### DNS in Kubernetes (CoreDNS)

```
Pod in Namespace "default" queries: mysql.default.svc.cluster.local
                │
                ▼
        /etc/resolv.conf in Pod:
        nameserver 10.96.0.10  (CoreDNS ClusterIP)
        search default.svc.cluster.local svc.cluster.local cluster.local
                │
                ▼
        CoreDNS receives query
                │
                ▼
        CoreDNS checks its internal records
        (populated from Kubernetes API server)
                │
                ▼
        Returns: ClusterIP of mysql Service (e.g., 10.100.50.25)
                │
                ▼
        Pod connects to 10.100.50.25:3306
        (kube-proxy routes to actual Pod IPs)

DNS search domains enable short names:
"mysql" → tries mysql.default.svc.cluster.local → found!
```

---

## 7. HTTP / HTTPS Protocol

### HTTP Request/Response Lifecycle

```
Browser: GET https://api.example.com/users
                        │
                        ▼
1. DNS Resolution → 93.184.216.34
                        │
                        ▼
2. TCP SYN to 93.184.216.34:443
   TCP SYN-ACK
   TCP ACK
                        │
                        ▼
3. TLS Handshake (HTTPS only):
   → ClientHello (TLS version, cipher suites, random)
   ← ServerHello (chosen cipher, certificate, random)
   → Client verifies certificate against CA
   → Key exchange (ECDHE)
   → Both sides derive session keys
   → Finished messages (encrypted)
                        │
                        ▼
4. HTTP Request (encrypted over TLS):
GET /users HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJ...
Accept: application/json
User-Agent: Mozilla/5.0
                        │
                        ▼
5. HTTP Response:
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 1234
Cache-Control: max-age=300
X-Request-ID: abc-123

{"users": [...]}
                        │
                        ▼
6. Connection kept alive (Keep-Alive) or closed
```

### HTTP Status Codes — Complete Reference

```
1xx - Informational
  100 Continue
  101 Switching Protocols (WebSocket upgrade)

2xx - Success
  200 OK
  201 Created (POST created resource)
  202 Accepted (async processing)
  204 No Content (DELETE success)

3xx - Redirection
  301 Moved Permanently (update bookmarks/caches)
  302 Found (temporary redirect)
  304 Not Modified (use cached version)
  307 Temporary Redirect (preserve method)
  308 Permanent Redirect (preserve method)

4xx - Client Errors
  400 Bad Request (malformed request)
  401 Unauthorized (authentication required)
  403 Forbidden (authenticated but not authorized)
  404 Not Found
  405 Method Not Allowed
  408 Request Timeout
  409 Conflict (resource state conflict)
  429 Too Many Requests (rate limiting)
  431 Request Header Fields Too Large

5xx - Server Errors
  500 Internal Server Error (generic server error)
  502 Bad Gateway (upstream error, nginx → app)
  503 Service Unavailable (overloaded or down)
  504 Gateway Timeout (upstream too slow)
```

### TLS Handshake Deep Dive

```
Client                              Server
  │                                   │
  │── ClientHello ──────────────────►│
  │   TLS 1.3, cipher suites,        │
  │   client random, key share       │
  │                                   │
  │◄── ServerHello + Certificate ────│
  │    + EncryptedExtensions         │
  │    + CertificateVerify           │
  │    + Finished                    │
  │                                   │
  │    Client verifies cert:          │
  │    - Check signature chain        │
  │    - Verify against trusted CAs   │
  │    - Check hostname match         │
  │    - Check expiry                 │
  │                                   │
  │── Finished (encrypted) ─────────►│
  │                                   │
  │═══════ Application Data ══════════│
  │   (encrypted with session keys)   │

Key Exchange: ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)
  - Perfect Forward Secrecy (PFS)
  - Even if private key compromised, past sessions safe
  
Session keys derived from:
  - Client random
  - Server random  
  - Pre-master secret (from key exchange)
```

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Transport | TCP | TCP | QUIC (UDP) |
| Multiplexing | No (HOL blocking) | Yes (streams) | Yes (better) |
| Header compression | None | HPACK | QPACK |
| Server push | No | Yes | Yes |
| TLS required | No | Yes (practical) | Always |
| Head-of-line blocking | Per connection | Per connection | Eliminated |
| Connection setup | TCP+TLS (2 RTT) | TCP+TLS (2 RTT) | 0-1 RTT |

---

## 8. SSH — Internals & Security

### SSH Architecture

```
SSH Client                          SSH Server (sshd)
     │                                     │
     │ ── TCP SYN ───────────────────────►│ :22
     │ ◄─ TCP SYN-ACK ─────────────────── │
     │ ── TCP ACK ───────────────────────►│
     │                                     │
     │ ═══ Protocol negotiation ═══════════│
     │  SSH-2.0-OpenSSH_8.9               │
     │  Kex algorithms, ciphers, MACs      │
     │                                     │
     │ ═══ Key Exchange (ECDH/DH) ═════════│
     │  Client gets server host key        │
     │  Verify against known_hosts         │
     │  Session key derived                │
     │                                     │
     │ ═══ Authentication ══════════════════│
     │  Method 1: Password                 │
     │  Method 2: Public Key ← preferred   │
     │  Method 3: Keyboard-interactive     │
     │  Method 4: Certificate              │
     │                                     │
     │ ═══ Session (encrypted channels) ═══│
     │  shell, exec, sftp, forwarding      │
```

### SSH Key Authentication Flow

```
On Client (setup):
ssh-keygen -t ed25519 -C "user@host"
# Creates: ~/.ssh/id_ed25519 (private) + id_ed25519.pub (public)

Deploy public key to server:
ssh-copy-id user@server   # Appends to ~/.ssh/authorized_keys on server

Authentication flow:
Client                              Server
  │                                   │
  │── "I want to auth as 'ubuntu'    │
  │    using key ed25519"            │
  │── Sends public key ────────────►│
  │                                   │ Check ~/.ssh/authorized_keys
  │                                   │ Key found!
  │                                   │ Generate random challenge
  │◄── Encrypted challenge ──────────│ (encrypted with client's pubkey)
  │                                   │
  │  Decrypt with private key         │
  │  Sign challenge + session ID      │
  │── Send signature ──────────────►│
  │                                   │ Verify signature with public key
  │                                   │ Signature valid!
  │◄── Auth success ─────────────────│
```

### SSH Configuration Best Practices

```bash
# /etc/ssh/sshd_config (server)
Port 2222                          # Non-standard port (minor security)
PermitRootLogin no                 # Never allow root SSH
PasswordAuthentication no          # Key-only auth
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3                     # Limit brute force
ClientAliveInterval 300            # Timeout idle connections
ClientAliveCountMax 2
AllowUsers deploy ubuntu           # Whitelist users
Protocol 2                         # SSH v2 only
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-256,hmac-sha2-512
KexAlgorithms curve25519-sha256,ecdh-sha2-nistp521

# ~/.ssh/config (client)
Host prod-bastion
    HostName 54.1.2.3
    User ubuntu
    IdentityFile ~/.ssh/prod_key
    ServerAliveInterval 60

Host prod-internal-*
    User app
    IdentityFile ~/.ssh/prod_key
    ProxyJump prod-bastion    # Jump through bastion!
    StrictHostKeyChecking yes
```

### SSH Tunneling (Port Forwarding)

```bash
# Local port forwarding
# Access remote DB (3306) locally via port 5432
ssh -L 5432:db.internal:3306 bastion.example.com
# Now: mysql -h 127.0.0.1 -P 5432 works!

# Remote port forwarding (expose local service remotely)
ssh -R 8080:localhost:3000 server.example.com
# On server: curl localhost:8080 → reaches your local port 3000

# Dynamic SOCKS proxy (turn SSH into VPN-like)
ssh -D 1080 bastion.example.com
# Configure browser to use SOCKS5 proxy 127.0.0.1:1080

# Jump host / Bastion pattern
ssh -J bastion.example.com private-server.internal
```

---

## 9. Firewalls & iptables

### iptables Architecture

```
Incoming Packet
        │
        ▼
   PREROUTING chain (nat table)
   [DNAT - Destination NAT]
        │
        ▼
   Routing Decision ──────── Is destination local?
        │                           │
       YES                         NO → FORWARD chain
        │
        ▼
   INPUT chain (filter table)
   [Allow/Drop/Reject]
        │
        ▼
   Local Process (application)
        │
        ▼
   OUTPUT chain (filter table)
        │
        ▼
   POSTROUTING chain (nat table)
   [SNAT - Source NAT / MASQUERADE]
        │
        ▼
   Out the interface
```

### iptables Tables and Chains

```
Tables:
  filter  - Default: INPUT, OUTPUT, FORWARD (allow/block)
  nat     - PREROUTING, POSTROUTING, OUTPUT (address translation)
  mangle  - Modify packet headers (QoS, ToS marking)
  raw     - Connection tracking exemptions
  security- SELinux marking

Chains (user-defined chains possible):
  INPUT      - Incoming to local process
  OUTPUT     - Outgoing from local process
  FORWARD    - Passing through (router)
  PREROUTING - Before routing decision
  POSTROUTING- After routing decision

Actions (targets):
  ACCEPT  - Allow packet through
  DROP    - Silently discard
  REJECT  - Discard + send error back
  LOG     - Log to syslog and continue
  DNAT    - Change destination address
  SNAT    - Change source address
  MASQUERADE - Dynamic SNAT (for DHCP IPs)
```

### Practical iptables Commands

```bash
# View rules
iptables -L -n -v --line-numbers          # filter table
iptables -t nat -L -n -v                  # nat table

# Allow rules
iptables -A INPUT -p tcp --dport 80 -j ACCEPT      # Allow HTTP
iptables -A INPUT -p tcp --dport 443 -j ACCEPT     # Allow HTTPS
iptables -A INPUT -p tcp --dport 22 -j ACCEPT      # Allow SSH

# Allow established connections (crucial!)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Drop everything else (default deny - put LAST!)
iptables -A INPUT -j DROP

# Source IP restrictions
iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 5432 -j ACCEPT  # Postgres internal only
iptables -A INPUT -s 0/0 -p tcp --dport 5432 -j DROP

# Rate limiting (DDoS mitigation)
iptables -A INPUT -p tcp --dport 80 -m limit --limit 100/minute --limit-burst 200 -j ACCEPT

# MASQUERADE (Docker/K8s uses this for pod → external traffic)
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 -o eth0 -j MASQUERADE

# Port forwarding (DNAT)
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.0.5:8080

# Save and restore
iptables-save > /etc/iptables/rules.v4
iptables-restore < /etc/iptables/rules.v4

# nftables (modern replacement)
nft list ruleset
nft add rule inet filter input tcp dport 80 accept
```

### Cloud Security Groups vs iptables

```
AWS Security Groups:
- Stateful (return traffic automatically allowed)
- Applied at ENI (network interface) level
- Rules: protocol, port range, source CIDR/SG
- No explicit deny rules (implicit deny all)
- Changes apply immediately

iptables:
- Stateless by default (need ESTABLISHED,RELATED rule)
- Applied at OS level
- More powerful/complex
- Explicit rules with explicit drop
- Rules processed in order, first match wins

Both work together in cloud:
Internet → Security Group → Network ACL → iptables → Application
```

---

## 10. System Troubleshooting

### CPU Troubleshooting

```bash
# High CPU - find culprit
top -b -n 1 | head -20           # Snapshot
ps aux --sort=-%cpu | head -10   # Top CPU processes

# Per-core CPU usage
mpstat -P ALL 1 5

# CPU profile (what is code spending time on)
perf top                          # Requires root + perf package
perf record -g -p PID             # Profile specific process
perf report

# Context switches (high = scheduler pressure)
vmstat 1 5
pidstat -w 1                      # Per-process context switches

# Load average explained:
# 0.50 0.75 1.00
# └──── 1min avg
#       └──── 5min avg
#             └──── 15min avg
# If load avg > number of CPU cores → system overloaded

nproc                             # Number of CPUs
cat /proc/cpuinfo | grep processor | wc -l
```

### Memory Troubleshooting

```bash
# Memory overview
free -h
vmstat -s | head -20

# Find memory hungry processes
ps aux --sort=-%mem | head -10
smem -r | head -10                # More accurate (accounts for shared)

# Memory per process
cat /proc/PID/status | grep -E "VmRSS|VmPeak|VmSize"
# VmRSS = Resident Set Size (actual RAM used)
# VmSize = Virtual Size (may be much larger)

# Page cache and buffer
cat /proc/meminfo | grep -E "Cached|Buffers|Dirty|Writeback"

# Drop page cache (careful in production!)
echo 1 > /proc/sys/vm/drop_caches  # drop page cache
echo 2 > /proc/sys/vm/drop_caches  # drop dentries/inodes
echo 3 > /proc/sys/vm/drop_caches  # drop both

# Swap usage
swapon -s
vmstat 1 | awk '{print $7, $8}'   # si=swap in, so=swap out
# If swap in/out > 0 constantly → system under memory pressure
```

### Disk I/O Troubleshooting

```bash
# I/O statistics
iostat -x 1 5                     # Extended I/O stats
# %util near 100% → disk saturated
# await high → high disk latency
# r/s, w/s → reads/writes per second

# Find process causing I/O
iotop -b -n 1 | head -20
pidstat -d 1                      # Per-process I/O

# Disk space
df -h                             # Space usage
du -sh /var/log/*                 # Directory sizes
du -sh /* 2>/dev/null | sort -h   # Find large dirs

# Find large files
find / -type f -size +100M 2>/dev/null | sort -k5 -n

# Inode usage
df -i

# Check disk errors
dmesg | grep -i "error\|I/O\|bad sector"
smartctl -a /dev/sda              # SMART disk health
```

### Network Troubleshooting

```bash
# Connection states
ss -s                             # Summary
ss -tulnp                         # Listening services
ss -tnp state established         # Established connections

# Network errors
ip -s link show eth0              # Interface errors/drops
netstat -s | grep -i error        # Protocol errors

# Bandwidth usage
iftop                             # Interactive bandwidth monitor
nethogs                           # Per-process bandwidth
nload eth0                        # Interface bandwidth graph

# DNS troubleshooting
dig +trace google.com             # Full resolution trace
systemd-resolve --status          # systemd DNS config
resolvectl query google.com       # Modern resolution

# tcpdump examples
tcpdump -i eth0 -nn port 80       # HTTP traffic (no DNS resolve)
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'  # SYN packets only
tcpdump -i any host 10.0.0.5 and port 5432      # DB connections
```

### Essential Log Files

```bash
# System logs
journalctl -f                     # Follow systemd logs
journalctl -u nginx -n 100        # Last 100 lines of nginx
journalctl --since "1 hour ago"   # Time-based filter
journalctl -p err                 # Errors only

# Traditional logs
tail -f /var/log/syslog           # System log
tail -f /var/log/auth.log         # Authentication
dmesg -T | tail -50               # Kernel messages with timestamps
cat /var/log/kern.log             # Kernel log

# Application logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
tail -f /var/log/postgresql/postgresql.log

# Analyze logs
grep "ERROR" /var/log/app.log | wc -l     # Count errors
grep "ERROR" /var/log/app.log | awk '{print $1}' | sort | uniq -c  # Errors by time
awk '/ERROR/{print $0}' /var/log/app.log | tail -100
```

---

## 11. Interview Questions — Phase 1

### Linux — Beginner

**Q: What is the difference between a process and a thread?**

Expected Answer:
- Process: independent execution unit with its own memory space, file descriptors, PID. Processes are isolated — one crashing doesn't affect others.
- Thread: lightweight unit of execution within a process. Threads share the same memory space, file descriptors, and address space of their parent process. Faster to create, lower overhead.
- In Linux, both are implemented via `clone()` syscall — a thread is a process that shares resources.

Production example: nginx uses multiple worker processes (isolation). Node.js uses a single thread with event loop (non-blocking I/O). Java apps use multiple threads within one JVM process.

**Q: What happens when you run `ls` in a shell?**

Answer (complete flow):
1. Shell calls `fork()` — creates a child process
2. Child calls `exec("ls", ...)` — replaces its memory with ls binary
3. ls reads `/bin/ls` binary, loads it
4. ls calls `opendir()`, `readdir()` syscalls
5. ls writes to stdout (file descriptor 1)
6. ls calls `exit(0)`
7. Kernel sends SIGCHLD to parent shell
8. Shell calls `wait()` — reaps zombie child
9. Shell displays prompt again

**Q: What is a zombie process?**

Answer: A zombie is a process that has finished execution (`exit()` called) but its parent hasn't yet called `wait()` to read its exit status. The kernel keeps the process table entry to store the exit code.

Production relevance: Too many zombies → process table fills up → can't create new processes. Usually indicates a bug in a parent process. Fix: fix the parent to call `wait()`. If parent is dead: re-parented to init (PID 1) which automatically reaps zombies.

### Linux — Advanced

**Q: Explain how Linux namespaces and cgroups together enable containers.**

Answer:
- **Namespaces** provide isolation (what a process can see):
  - `pid` namespace: process sees its own PID 1
  - `net` namespace: private network stack, interfaces
  - `mnt` namespace: private mount points / filesystem
  - `uts` namespace: own hostname
  - `user` namespace: own user/group IDs
  - `ipc` namespace: isolated IPC (semaphores, message queues)
  - `cgroup` namespace: isolated cgroup view

- **cgroups** (control groups) provide resource limits (what a process can use):
  - `cpu` cgroup: CPU quota/shares
  - `memory` cgroup: RAM/swap limits
  - `blkio` cgroup: disk I/O throttling
  - `net_cls` / `net_prio`: network priority

Together: A container is a process with its own namespaces (isolated environment) and cgroup limits (resource constraints). Docker's `docker run` calls `clone()` with namespace flags and configures cgroups via `/sys/fs/cgroup/`.

**Q: A server is running slowly but CPU and memory look fine. How do you diagnose?**

Systematic approach:
1. Check `iostat -x 1` → High `%util` or `await`? → Disk I/O bottleneck
2. Check `ss -s` → Too many connections in CLOSE_WAIT or TIME_WAIT?
3. Check `netstat -s | grep retransmit` → Network packet loss?
4. Check `vmstat 1` → `si`/`so` columns non-zero? → Swapping!
5. Check `dmesg | tail` → Kernel errors, disk errors?
6. Check `strace -p PID` → What syscalls is it doing?
7. Check application logs → application-level errors
8. Check `lsof -p PID` → Too many open files?
9. Check NFS/network mounts → `df -h` hang? → NFS timeout

### Networking — Beginner

**Q: What is the difference between TCP and UDP? When would you use each?**

TCP: Connection-oriented, reliable, ordered delivery. Overhead of handshake + ACKs. Use for: HTTP, HTTPS, SSH, databases, anything where data integrity matters.

UDP: Connectionless, fire-and-forget. No guarantee of delivery/order. Use for: DNS queries (fast, mostly succeed), video streaming (occasional loss acceptable), gaming (low latency > reliability), DHCP.

**Q: What happens when you type `google.com` in a browser?**

Complete flow:
1. Browser checks local DNS cache
2. OS checks `/etc/hosts`
3. OS sends DNS query to configured nameserver (UDP port 53)
4. Recursive resolver resolves via Root → TLD → Authoritative
5. IP returned: 142.250.x.x
6. Browser initiates TCP connection to port 443
7. TLS handshake: certificate verification, key exchange
8. HTTP/2 GET request sent over encrypted channel
9. Server returns 200 OK with HTML
10. Browser parses HTML, fetches additional resources (CSS, JS, images)
11. Each resource may require additional DNS lookups, TCP connections

### Networking — Advanced

**Q: Explain BGP and why it matters for DevOps.**

BGP (Border Gateway Protocol) is the routing protocol that runs the Internet. It's an external gateway protocol that exchanges routing information between autonomous systems (AS).

DevOps relevance:
- **Multi-cloud/multi-region**: BGP Anycast used by CDNs to route users to nearest node
- **AWS Direct Connect/VPN**: Uses BGP to exchange routes between on-prem and VPC
- **Kubernetes**: Some CNI plugins (Calico) use BGP to advertise pod routes
- **DDoS mitigation**: BGP blackholing routes malicious traffic to null
- **Failover**: BGP can withdraw routes to shift traffic (DNS failover is too slow)

Production scenario: "Why is traffic going to the wrong region?" → BGP route leak or misconfiguration. Use `traceroute` to see unexpected AS hops.

**Q: What is the difference between NAT, SNAT, DNAT, and MASQUERADE?**

- **NAT** (Network Address Translation): Generic term — translate IP addresses/ports
- **SNAT** (Source NAT): Change the source IP. Used: private network → Internet. Static source IP.
- **DNAT** (Destination NAT): Change the destination IP. Used: port forwarding, load balancers. "Redirect port 80 → internal server"
- **MASQUERADE**: Dynamic SNAT where source IP is the outgoing interface's IP. Used when IP is dynamic (DHCP). Docker uses this for container internet access.

Example:
```
Container (172.17.0.2) → MASQUERADE → host IP (eth0: 192.168.1.100) → Internet
```
Docker adds this rule: `iptables -t nat -A POSTROUTING -s 172.17.0.0/16 -j MASQUERADE`

---

*End of Phase 1 — Linux & Networking*
# Phase 2: Version Control — Git, GitHub, GitLab, Bitbucket

---

## Table of Contents
1. [What is Git?](#1-what-is-git)
2. [Git Internals — How it Really Works](#2-git-internals--how-it-really-works)
3. [Core Git Concepts](#3-core-git-concepts)
4. [Branching Strategies](#4-branching-strategies)
5. [Git Workflows for DevOps](#5-git-workflows-for-devops)
6. [GitHub vs GitLab vs Bitbucket](#6-github-vs-gitlab-vs-bitbucket)
7. [GitOps Principles](#7-gitops-principles)
8. [Troubleshooting Git](#8-troubleshooting-git)
9. [Interview Questions — Phase 2](#9-interview-questions--phase-2)

---

## 1. What is Git?

Git is a **distributed version control system (DVCS)** created by Linus Torvalds in 2005 (to manage Linux kernel development, after Bitkeeper revoked free access).

**Key properties:**
- **Distributed**: Every clone is a full repository with complete history
- **Snapshot-based**: Stores snapshots, not diffs (unlike SVN)
- **Content-addressable**: Everything stored by SHA-1 hash of content
- **Branching is cheap**: O(1) operation — just creates a pointer

**Why DevOps cares deeply about Git:**
- Foundation of all CI/CD pipelines (code push triggers pipeline)
- GitOps: Git as single source of truth for infrastructure state
- Code review via Pull Requests (change management process)
- Audit trail for all changes (who changed what, when, why)
- Rollback capability (revert to any point in history)
- Collaboration without conflicts (branching + merging)

---

## 2. Git Internals — How it Really Works

### Git Object Store

Git stores everything in a **content-addressable object store** at `.git/objects/`. There are 4 object types:

```
Git Object Types:
┌─────────┬──────────────────────────────────────────────────┐
│  Blob   │ File content (no name, no metadata)              │
├─────────┼──────────────────────────────────────────────────┤
│  Tree   │ Directory listing (name + mode → blob/tree SHA)  │
├─────────┼──────────────────────────────────────────────────┤
│  Commit │ Snapshot metadata (tree + parent + author + msg) │
├─────────┼──────────────────────────────────────────────────┤
│  Tag    │ Annotated tag (points to commit with message)    │
└─────────┴──────────────────────────────────────────────────┘
```

### How a Commit is Stored

```
Working Directory State:
  README.md → "Hello World"
  src/
    app.py → "print('hello')"

Git stores as objects:

blob: SHA = sha1("Hello World") = a1b2c3d4...
blob: SHA = sha1("print('hello')") = e5f6g7h8...

tree (src/):
  100644 blob e5f6g7h8... app.py

tree (root):
  100644 blob a1b2c3d4... README.md
  040000 tree xyz12345... src

commit:
  tree  rootTreeSHA
  parent prevCommitSHA
  author Jane Doe <jane@example.com> 1705000000 +0530
  committer Jane Doe <jane@example.com> 1705000000 +0530

  feat: add hello world app

Object names (SHA-1):
  git cat-file -t a1b2c3d4  → blob
  git cat-file -p a1b2c3d4  → Hello World
  git cat-file -p HEAD      → shows commit object
```

### The .git Directory Structure

```
.git/
├── HEAD                    # Points to current branch
│   └── "ref: refs/heads/main"
├── config                  # Repository configuration
├── description             # Repository description
├── index                   # Staging area (binary)
├── COMMIT_EDITMSG          # Last commit message
├── objects/                # All git objects
│   ├── a1/                 # First 2 chars of SHA
│   │   └── b2c3d4...       # Remaining 38 chars = filename
│   ├── pack/               # Packed objects (compressed)
│   └── info/
├── refs/
│   ├── heads/              # Local branches
│   │   ├── main            # File containing commit SHA
│   │   └── feature/auth
│   ├── remotes/            # Remote tracking branches
│   │   └── origin/
│   │       └── main
│   └── tags/               # Tags
├── logs/                   # Reflog (history of ref changes)
│   ├── HEAD
│   └── refs/heads/main
└── hooks/                  # Git hooks (scripts)
    ├── pre-commit.sample
    ├── pre-push.sample
    └── commit-msg.sample
```

### How git commit Works Internally

```bash
# When you run: git commit -m "feat: add login"
#
# Step 1: Index (staging area) is read
# Step 2: Git creates blob objects for any new/changed files
# Step 3: Git creates tree objects for all directories
# Step 4: Git creates a commit object:
#         - Points to root tree
#         - Points to parent commit (HEAD)
#         - Contains author, timestamp, message
# Step 5: HEAD (current branch) pointer updated to new commit SHA

# Visualize the object DAG:
git log --graph --oneline --all

# See exactly what's in any object
git cat-file -p HEAD
git cat-file -p HEAD^{tree}
git ls-tree -r HEAD
```

### How git clone Works

```
git clone https://github.com/org/repo.git
         │
         ▼
1. Create local .git directory
2. HTTPS: Negotiate capability (refs, objects available)
   SSH: Connect via SSH, same negotiation
         │
         ▼
3. Server sends "refs" (branches, tags + their SHAs)
         │
         ▼
4. Client determines what objects it needs
         │
         ▼
5. Server packs objects into packfile
   (delta compression: store differences between similar objects)
         │
         ▼
6. Client receives packfile, decompresses to .git/objects/pack/
         │
         ▼
7. Client checks out HEAD commit to working directory
         │
         ▼
8. Remote tracking branches created (origin/main etc.)

# Smart HTTP protocol:
# POST /repo.git/git-upload-pack HTTP/1.1
# Content-Type: application/x-git-upload-pack-request
```

---

## 3. Core Git Concepts

### The Three Trees (Most Important Concept)

```
                    git add           git commit
Working Directory ──────────► Index ──────────► HEAD (Repository)
   (what you see)            (staging area)      (committed history)

git diff           = Working Dir vs Index
git diff --staged  = Index vs HEAD (last commit)
git diff HEAD      = Working Dir vs HEAD

git reset HEAD~1   = Move HEAD back, update Index
git checkout file  = Restore file from Index to Working Dir
git restore file   = Modern equivalent
```

### Branching — How it Works

```
A branch is just a file containing a 40-char SHA:
cat .git/refs/heads/main
# a1b2c3d4e5f6g7h8i9j0...

HEAD is a pointer to the current branch:
cat .git/HEAD
# ref: refs/heads/feature/auth

Creating a branch (O(1) operation - just creates file):
git branch feature/auth
# Creates .git/refs/heads/feature/auth pointing to current commit

Timeline:
         A---B---C  main
              \
               D---E  feature/auth

After merge:
         A---B---C---F  main (F = merge commit)
              \       /
               D---E  feature/auth
```

### Git Reset vs Revert vs Checkout

```
git reset --soft HEAD~1
  - Move HEAD back 1 commit
  - Index unchanged (changes are staged)
  - Working directory unchanged
  - Use: "I committed too early, let me re-do"

git reset --mixed HEAD~1 (default)
  - Move HEAD back 1 commit
  - Index reset to HEAD (changes are unstaged)
  - Working directory unchanged
  - Use: "I committed the wrong files"

git reset --hard HEAD~1
  - Move HEAD back 1 commit
  - Index and working directory reset
  - Changes are LOST (unless reflog used)
  - ⚠️ Dangerous! Use with caution

git revert HEAD
  - Creates a NEW commit that undoes changes
  - History preserved (safe for shared branches)
  - Use: "I need to undo a commit that others have"

git checkout main
  - Switch branches
  - Update HEAD + working directory + index

git checkout HEAD~1 -- file.py
  - Restore specific file from commit
  - Working directory updated
```

### Merge vs Rebase

```
MERGE:
         A---B---C  main
              \
               D---E  feature

After: git checkout main && git merge feature
         A---B---C---M  main (M = merge commit)
              \     /
               D---E

Pros: Preserves history, safe
Cons: Merge commits make history messy

REBASE:
         A---B---C  main
              \
               D---E  feature

After: git checkout feature && git rebase main
         A---B---C  main
                  \
                   D'--E'  feature (new commits, new SHAs!)

Pros: Clean linear history
Cons: Rewrites history (never rebase shared branches!)

After rebase + fast-forward merge:
git checkout main && git merge feature (fast-forward)
         A---B---C---D'---E'  main
```

### Useful Git Commands

```bash
# Setup
git config --global user.name "Jane Doe"
git config --global user.email "jane@example.com"
git config --global core.editor "vim"
git config --global alias.lg "log --graph --oneline --all --decorate"

# Daily workflow
git status                          # Current state
git add -p                          # Interactive stage (review each hunk)
git add .                           # Stage all
git commit -m "type: description"   # Commit
git push origin feature/my-branch   # Push to remote
git pull --rebase                   # Pull with rebase (cleaner than merge)

# Log inspection
git log --oneline --graph --all     # Visual branch history
git log -p -n 5                     # Last 5 commits with diffs
git log --author="Jane" --since="1 week ago"
git log --grep="fix" --oneline      # Search commit messages
git log -- file.py                  # History of specific file

# Comparison
git diff main feature/auth          # Between branches
git diff HEAD~3 HEAD                # Last 3 commits
git diff --stat HEAD~1              # Changed files only

# Finding bugs
git bisect start
git bisect bad HEAD                 # Current commit is broken
git bisect good v1.0                # v1.0 was working
# Git checkouts middle commit, you test, then:
git bisect good   # or git bisect bad
# Repeat until culprit found!
git bisect reset

# Stash
git stash                           # Save dirty state
git stash pop                       # Restore + remove stash
git stash list                      # Show all stashes
git stash apply stash@{2}           # Apply specific stash

# Remote management
git remote -v                       # List remotes
git remote add upstream git@github.com:org/repo.git
git fetch upstream                  # Fetch without merging
git rebase upstream/main            # Rebase on upstream

# Cherry-pick
git cherry-pick a1b2c3d4            # Apply specific commit
git cherry-pick main~2..main        # Apply range of commits

# Reflog (time machine!)
git reflog                          # All HEAD movements
git reset --hard HEAD@{5}           # Go back to 5 moves ago
# Use this to recover from accidental reset --hard!

# Tagging
git tag v1.0.0                      # Lightweight tag
git tag -a v1.0.0 -m "Release v1.0" # Annotated tag
git push origin --tags              # Push tags
git tag -d v1.0.0                   # Delete local tag
git push origin :refs/tags/v1.0.0   # Delete remote tag
```

---

## 4. Branching Strategies

### Git Flow

```
Production-grade strategy for scheduled releases:

main ──────────────────────────────────────────────────► production
 │                                                        
hotfix/                                                   
 │ branch from main                                       
 │ fix → PR → merge to main AND develop
 │
develop ─────────────────────────────────────────────────
 │                                                        
 ├── feature/user-auth  (branches from develop, merges back)
 ├── feature/payments
 │
 └── release/1.0.0  (branches from develop when ready)
                    (only bugfixes)
                    (merge to main AND develop when done)

Flow:
1. Feature branches from develop
2. PRs merge back to develop
3. release/* branch cut from develop for QA
4. release/* merged to main → tag + deploy to prod
5. hotfix/* from main, merged to main AND develop

Best for: Products with versioned releases, multiple versions in production
```

### GitHub Flow (Simpler)

```
main ──────────────────────────────────────────────────► production
      \         /    \         /
       feature-A       feature-B

Flow:
1. branch from main
2. commit changes
3. Open PR
4. Code review + CI
5. Merge to main
6. Deploy to production immediately

Best for: Web apps, SaaS, continuous deployment
```

### Trunk-Based Development

```
main (trunk) ─────────────────────────────────────────►
                                                        
Developers commit directly to main (very small teams)  
OR use very short-lived feature branches (1-2 days max) 

Feature flags used to hide incomplete features in prod

Best for: High-performing teams, microservices, Google/Facebook style
```

### Branch Naming Conventions

```bash
# Feature work
feature/JIRA-123-user-authentication
feature/add-payment-gateway

# Bug fixes
fix/JIRA-456-login-crash
bugfix/null-pointer-dashboard

# Hotfixes (urgent production fixes)
hotfix/v1.2.1-security-patch

# Release branches
release/v1.2.0
release/2024-Q1

# Chores/infrastructure
chore/update-dependencies
infra/kubernetes-upgrade
ci/add-security-scanning
```

---

## 5. Git Workflows for DevOps

### CI/CD Trigger from Git Push

```
Developer
    │ git push origin feature/auth
    │
    ▼
GitHub/GitLab/Bitbucket
    │
    ├── Triggers webhook → CI system
    │   (Jenkins, GitHub Actions, GitLab CI)
    │
    ├── PR/MR created or updated
    │
    └── CI Pipeline starts:
         │
         ├── Lint + format check
         ├── Unit tests
         ├── Integration tests
         ├── Security scan (SAST)
         ├── Container build
         ├── Container scan
         └── Deploy to staging (on merge to main)
```

### Git Hooks for DevOps

```bash
# Hooks run automatically on git actions
# Located in .git/hooks/ or shared via pre-commit framework

# pre-commit: runs before commit is created
# .git/hooks/pre-commit
#!/bin/bash
# Run linter
python -m flake8 . --count --select=E9,F63,F7,F82
if [ $? -ne 0 ]; then
    echo "Lint failed! Fix errors before committing."
    exit 1
fi

# Run tests
python -m pytest tests/unit/ -q
if [ $? -ne 0 ]; then
    echo "Tests failed! Fix before committing."
    exit 1
fi

# commit-msg: validate commit message format
# .git/hooks/commit-msg
#!/bin/bash
COMMIT_MSG=$(cat "$1")
PATTERN="^(feat|fix|chore|docs|style|refactor|perf|test|ci)(\(.+\))?: .{1,72}$"
if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
    echo "Commit message must follow Conventional Commits format"
    echo "Example: feat(auth): add OAuth2 login"
    exit 1
fi

# pre-push: run checks before push
# Prevents pushing secrets
git diff HEAD origin/main -- | grep -r 'password\|secret\|api_key\|token'

# Share hooks with team using pre-commit framework:
# pip install pre-commit
# .pre-commit-config.yaml
# repos:
# - repo: https://github.com/pre-commit/pre-commit-hooks
#   hooks:
#   - id: detect-private-key
#   - id: check-merge-conflict
```

### Conventional Commits Standard

```
Format: <type>(<scope>): <description>

[optional body]

[optional footer(s)]

Types:
feat     - New feature (triggers MINOR version bump)
fix      - Bug fix (triggers PATCH version bump)
BREAKING CHANGE: in footer → triggers MAJOR bump
chore    - Build/tooling changes
docs     - Documentation only
style    - Formatting, no logic change
refactor - Refactoring, no new feature/fix
perf     - Performance improvement
test     - Adding/updating tests
ci       - CI/CD changes
revert   - Reverts a previous commit

Examples:
feat(auth): add JWT token refresh mechanism

fix(api): handle null response from payment gateway

Closes #234

feat!: remove deprecated /v1 API endpoints

BREAKING CHANGE: All v1 endpoints removed. Migrate to /v2.
```

---

## 6. GitHub vs GitLab vs Bitbucket

| Feature | GitHub | GitLab | Bitbucket |
|---------|--------|--------|-----------|
| Owner | Microsoft | GitLab Inc | Atlassian |
| CI/CD | GitHub Actions | GitLab CI (built-in) | Bitbucket Pipelines |
| Container Registry | GitHub Packages | GitLab Registry | - |
| Self-hosted | GitHub Enterprise | GitLab CE/EE | Bitbucket Data Center |
| Code Review | Pull Requests | Merge Requests | Pull Requests |
| Issue Tracking | GitHub Issues | GitLab Issues | Jira integration |
| DevSecOps | Actions + third-party | Built-in (SAST, DAST, fuzz) | Limited |
| Auto DevOps | No | Yes | No |
| Kubernetes integration | Yes | Deep (GitLab Agent) | Yes |
| Cost (free tier) | Unlimited public repos | Unlimited public+private | 5 users free |
| Best for | Open source, widely used | Full DevSecOps in one tool | Atlassian ecosystem |

### GitHub Actions — CI/CD Basics

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
      
      - name: Run tests
        run: |
          pytest tests/ -v --cov=app --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'CRITICAL,HIGH'

  docker:
    needs: [test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## 7. GitOps Principles

### What is GitOps?

GitOps is an operational framework where:
1. **Git is the single source of truth** for both application code AND infrastructure state
2. **Desired state is declared** in Git (Kubernetes manifests, Terraform, Helm charts)
3. **Automated agents** continuously reconcile actual state with desired state
4. **All changes** go through Git (pull requests, code review, audit trail)

```
GitOps Flow:

Developer
    │ Push to Git (infra manifest change)
    │
    ▼
Git Repository (desired state)
    │
    ▼
GitOps Operator (ArgoCD / Flux)
    │ continuously watches Git
    │ compares with cluster state
    ▼
Kubernetes Cluster (actual state)
    │
    ├── If drift detected → auto-reconcile to Git state
    └── If Git changes → apply changes to cluster

Benefits:
- Audit trail: all changes in Git history
- Rollback: git revert → cluster reverts
- Consistency: no manual kubectl apply in prod
- Disaster recovery: repo = full cluster definition
- Security: no direct cluster access needed
```

### GitOps Repository Structure

```
infrastructure-repo/
├── clusters/
│   ├── production/
│   │   ├── kustomization.yaml
│   │   └── namespace.yaml
│   └── staging/
│       └── kustomization.yaml
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml
│   └── database/
│       ├── statefulset.yaml
│       └── pvc.yaml
├── infrastructure/
│   ├── cert-manager/
│   ├── ingress-nginx/
│   └── monitoring/
└── README.md
```

---

## 8. Troubleshooting Git

### Common Issues and Fixes

```bash
# 1. Accidentally committed to main instead of feature branch
git log --oneline -3           # Find last commit SHA
git reset --soft HEAD~1        # Undo commit, keep changes staged
git checkout -b feature/my-work
git commit -m "Original message"

# 2. Merge conflict resolution
git merge feature/auth
# CONFLICT in auth.py
# Edit file to resolve markers:
<<<<<<< HEAD
   def login():
=======
   def authenticate():
>>>>>>> feature/auth
# Fix the code, then:
git add auth.py
git commit    # Merge commit created

# 3. Accidentally pushed sensitive data
# This is a serious incident!
# Step 1: Remove the file
git rm --cached secret.txt
echo "secret.txt" >> .gitignore
git add .gitignore
git commit -m "Remove sensitive data"
git push

# Step 2: Purge from history (rewrite history!)
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch secret.txt' \
  --prune-empty --tag-name-filter cat -- --all

# Modern approach with git-filter-repo:
pip install git-filter-repo
git filter-repo --path secret.txt --invert-paths

# Step 3: Force push ALL branches
git push origin --force --all
git push origin --force --tags

# Step 4: ROTATE ALL SECRETS IMMEDIATELY (always!)
# Purging from git doesn't help if secret was already cloned

# 4. Large repository (slow clone)
git clone --depth 1 URL               # Shallow clone (only latest)
git clone --filter=blob:none URL      # Partial clone (no blobs)
git fetch --unshallow                  # Convert to full clone later

# 5. Find which commit introduced a bug
git log --oneline --all | head -20    # Find range
git bisect start
git bisect bad HEAD
git bisect good v2.0
# Test and mark good/bad until culprit found
```

---

## 9. Interview Questions — Phase 2

**Q: What is the difference between `git merge` and `git rebase`? Which would you use and when?**

Merge creates a merge commit, preserving the full history of both branches. Rebase replays commits on top of another branch, creating a linear history with new commit SHAs.

Use merge when: merging feature branches to main (preserves context of when work was done), working with shared/public branches.

Use rebase when: keeping local feature branch up-to-date with main (cleaner than constant merge commits), cleaning up commits before PR (`git rebase -i`).

**Never rebase** shared/public branches — it rewrites history and causes problems for other developers who have those commits.

**Q: How do you handle a situation where a developer accidentally committed credentials to a public repository?**

1. **Immediately rotate/revoke** the exposed credentials — this is the most critical step. Assume the credentials are compromised.
2. Remove the file and add to `.gitignore`
3. Use `git filter-repo` to purge from history
4. Force push to overwrite remote history
5. Contact GitHub/GitLab to invalidate cached views
6. Implement pre-commit hooks to scan for secrets (git-secrets, detect-secrets)
7. Implement CI scanning (Gitleaks, TruffleHog)
8. Post-incident review and process improvement

**Q: Explain the Git objects model — what are blobs, trees, commits, and tags?**

Git is a content-addressable filesystem. Every object is stored by its SHA-1 hash:
- **Blob**: Raw file content (no filename, no metadata)
- **Tree**: A directory listing mapping names to blob/tree SHAs plus file permissions
- **Commit**: A snapshot pointer (tree SHA) + parent commit SHA + author/committer + message
- **Tag**: An annotated reference to a specific commit with a message and tagger

This means identical files across different commits share the same blob object. Git is highly efficient for code storage (most commits change few files).

**Q: What is `git reflog` and when would you use it?**

Reflog records every time HEAD or branch tips change — including resets, rebases, checkouts. It's your safety net for undoing "destructive" operations.

If you accidentally do `git reset --hard HEAD~5` and lose 5 commits, you can:
```bash
git reflog                    # See all HEAD movements
git reset --hard HEAD@{4}    # Restore to 4 moves ago (before the reset)
```

Reflog is local-only (not pushed to remote) and entries expire after 90 days by default.

**Q: How does your team's branching strategy work? What trade-offs did you consider?**

Example answer (GitHub Flow + Conventional Commits):

We use GitHub Flow because we deploy multiple times per day (continuous deployment). Every PR is:
1. Branch from main, named `type/description`
2. Code review required (2 approvals)
3. CI must pass (tests, lint, security scan, docker build)
4. Merged via squash merge to main
5. Immediately deployed to staging by ArgoCD
6. Manual promotion gate to production

Trade-offs considered: Git Flow's release branches are too heavy for our velocity. Trunk-based would require more mature feature flag infrastructure. GitHub Flow gives us speed with quality gates.

---

*End of Phase 2 — Version Control*
# Phase 3: CI/CD — Jenkins, GitHub Actions, GitLab CI, ArgoCD

---

## Table of Contents
1. [CI/CD Fundamentals](#1-cicd-fundamentals)
2. [Jenkins — Deep Dive](#2-jenkins--deep-dive)
3. [GitHub Actions — Deep Dive](#3-github-actions--deep-dive)
4. [GitLab CI — Deep Dive](#4-gitlab-ci--deep-dive)
5. [ArgoCD — GitOps CD](#5-argocd--gitops-cd)
6. [Pipeline Design Patterns](#6-pipeline-design-patterns)
7. [Security in CI/CD](#7-security-in-cicd)
8. [Troubleshooting CI/CD](#8-troubleshooting-cicd)
9. [Interview Questions — Phase 3](#9-interview-questions--phase-3)

---

## 1. CI/CD Fundamentals

### Continuous Integration vs Delivery vs Deployment

```
Continuous Integration (CI):
  - Developers merge frequently (multiple times/day)
  - Automated build + test on every push
  - Goal: catch integration bugs early
  - Output: verified build artifact

Continuous Delivery (CD):
  - CI + automated deployment to staging/pre-prod
  - Production deployment: manual approval gate
  - Goal: always have deployable software
  - Output: deployable release

Continuous Deployment (CD):
  - CI + automated deployment ALL the way to production
  - No manual gates
  - Goal: fastest feedback loop with users
  - Output: production deployment

┌─────────────────────────────────────────────────────────┐
│  CI:      Build ── Test ── Analysis                      │
│  CD:  CI ──────────────────────────── Staging ── [Gate]─►Production│
│  CD*: CI ──────────────────────────── Staging ──────────►Production│
└─────────────────────────────────────────────────────────┘
```

### The CI/CD Pipeline (Complete Flow)

```
Code Push
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Source Stage                                                │
│  - Checkout code                                            │
│  - Set up environment variables                             │
│  - Download dependencies (with cache)                       │
└─────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Quality Stage (runs in parallel)                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Lint &  │  │  Unit    │  │  SAST    │  │ License  │   │
│  │  Format  │  │  Tests   │  │  Scan    │  │  Check   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────┬───────────────────────────────────────┘
                       │ All must pass
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Build Stage                                                 │
│  - Build Docker image                                       │
│  - Tag with commit SHA + semantic version                   │
│  - Push to container registry                               │
└─────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Security Stage                                              │
│  - Container image vulnerability scan (Trivy, Grype)       │
│  - DAST if applicable                                       │
│  - Secrets scan                                             │
└─────────────────────┬───────────────────────────────────────┘
                       │ On main branch only
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Deploy to Staging                                           │
│  - Update Kubernetes manifests                              │
│  - ArgoCD detects change, syncs                             │
│  - Smoke tests                                              │
│  - Integration tests                                        │
└─────────────────────┬───────────────────────────────────────┘
                       │ Manual gate (or auto on success)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Deploy to Production                                        │
│  - Blue/green or rolling or canary                          │
│  - Health checks                                            │
│  - Automated rollback on failure                            │
│  - Notify team                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Jenkins — Deep Dive

### What is Jenkins?

Jenkins is an open-source automation server written in Java, used to build CI/CD pipelines. It's the most widely deployed CI/CD tool in enterprise environments.

**Why Jenkins exists:**
- Highly extensible (1800+ plugins)
- Self-hosted (full control, no SaaS pricing)
- Massive community and ecosystem
- Supports any language, any tool
- Fine-grained access control for enterprise

**Alternatives comparison:**

| Tool | Strength | Weakness |
|------|----------|----------|
| Jenkins | Plugins, flexibility, free | Complex to manage, Java overhead |
| GitHub Actions | Native GitHub, simple | GitHub vendor lock-in |
| GitLab CI | Built-in, no extra tool | GitLab required |
| CircleCI | SaaS simplicity | Cost, less control |
| TeamCity | JetBrains ecosystem | Licensing cost |
| Drone CI | Kubernetes-native | Smaller ecosystem |

### Jenkins Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Jenkins Controller (Master)               │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Job      │  │ Pipeline │  │ Plugin   │  │ REST API │   │
│  │ Scheduler│  │ Engine   │  │ Manager  │  │ Server   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                             │
│  Storage:                                                   │
│  - $JENKINS_HOME/jobs/ (job configs, build history)        │
│  - $JENKINS_HOME/plugins/ (plugin JARs)                    │
│  - $JENKINS_HOME/users/ (user accounts)                    │
│  - $JENKINS_HOME/secrets/ (credentials)                    │
└───────────────────────────┬─────────────────────────────────┘
                            │ JNLP (TCP 50000) or SSH
                            │ REST API (HTTP 8080)
          ┌─────────────────┼──────────────────┐
          ▼                 ▼                   ▼
    ┌──────────┐      ┌──────────┐       ┌──────────┐
    │  Agent 1 │      │  Agent 2 │       │  Agent 3 │
    │ (Linux)  │      │ (Docker) │       │ (K8s pod)│
    └──────────┘      └──────────┘       └──────────┘
    Build tasks       Build tasks         Build tasks
```

### Jenkins Communication Flow

```
Developer pushes code
        │
        ▼
GitHub/GitLab webhook → POST http://jenkins:8080/github-webhook/
        │
        ▼
Jenkins Controller receives webhook
  - Identifies which job(s) match the trigger
  - Adds build to queue
        │
        ▼
Scheduler assigns build to available agent
  - Agent selection based on labels (e.g., agent with 'docker' label)
        │
        ▼
Controller instructs Agent via JNLP (TCP 50000):
  - Send workspace files
  - Start build execution
        │
        ▼
Agent executes pipeline stages locally
  - Clones repository
  - Runs tests
  - Builds container
  - Agent → Docker daemon (localhost or remote socket)
        │
        ▼
Agent streams logs → Controller
Controller stores logs in $JENKINS_HOME
        │
        ▼
Agent reports success/failure → Controller
Controller sends notifications (Slack, email)
```

### Jenkins Pipeline (Declarative)

```groovy
// Jenkinsfile - declarative pipeline
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: python
                    image: python:3.11
                    command: [cat]
                    tty: true
                  - name: docker
                    image: docker:24
                    command: [cat]
                    tty: true
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock
                  volumes:
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
            '''
        }
    }
    
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE_NAME = 'myapp'
        DOCKER_CREDENTIALS = credentials('docker-registry-creds')
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('python') {
                            sh '''
                                pip install -r requirements.txt
                                pytest tests/unit/ -v --junitxml=test-results.xml
                            '''
                        }
                    }
                    post {
                        always {
                            junit 'test-results.xml'
                        }
                    }
                }
                
                stage('Lint') {
                    steps {
                        container('python') {
                            sh 'flake8 . --count --max-line-length=120'
                        }
                    }
                }
            }
        }
        
        stage('Build Image') {
            steps {
                container('docker') {
                    sh """
                        docker build \
                            --tag ${REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT} \
                            --tag ${REGISTRY}/${IMAGE_NAME}:latest \
                            --build-arg BUILD_DATE=\$(date -u +%Y-%m-%dT%H:%M:%SZ) \
                            --build-arg GIT_SHA=${GIT_COMMIT_SHORT} \
                            .
                    """
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                container('docker') {
                    sh """
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy:latest image \
                            --exit-code 1 \
                            --severity HIGH,CRITICAL \
                            ${REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                    """
                }
            }
        }
        
        stage('Push Image') {
            when {
                branch 'main'
            }
            steps {
                container('docker') {
                    sh """
                        echo \${DOCKER_CREDENTIALS_PSW} | \
                            docker login ${REGISTRY} -u \${DOCKER_CREDENTIALS_USR} --password-stdin
                        docker push ${REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                        docker push ${REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-staging', variable: 'KUBECONFIG')]) {
                    sh """
                        kubectl set image deployment/myapp \
                            myapp=${REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT} \
                            -n staging
                        kubectl rollout status deployment/myapp -n staging --timeout=300s
                    """
                }
            }
        }
        
        stage('Production Approval') {
            when {
                branch 'main'
            }
            steps {
                timeout(time: 4, unit: 'HOURS') {
                    input message: "Deploy ${GIT_COMMIT_SHORT} to production?",
                          ok: 'Deploy',
                          submitterParameter: 'APPROVED_BY'
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-prod', variable: 'KUBECONFIG')]) {
                    sh """
                        kubectl set image deployment/myapp \
                            myapp=${REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT} \
                            -n production
                        kubectl rollout status deployment/myapp -n production --timeout=300s
                    """
                }
            }
        }
    }
    
    post {
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ ${env.JOB_NAME} #${env.BUILD_NUMBER} deployed successfully! ${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ ${env.JOB_NAME} #${env.BUILD_NUMBER} FAILED! ${env.BUILD_URL}"
            )
        }
        always {
            cleanWs()
        }
    }
}
```

### Jenkins High Availability Setup

```
Production Jenkins Architecture:

┌──────────────────────────────────────────────────────────┐
│                    Load Balancer                          │
│              (HAProxy / AWS ALB)                          │
└───────────────────────┬──────────────────────────────────┘
                        │ HTTP 8080
                        │
          ┌─────────────┴────────────┐
          ▼                          ▼
  ┌───────────────┐         ┌───────────────┐
  │  Jenkins      │         │  Jenkins      │
  │  Active       │◄───────►│  Standby      │
  │  (Primary)    │ JNLP    │  (Secondary)  │
  └───────────────┘         └───────────────┘
          │                          │
          └───────────┬──────────────┘
                      ▼
             ┌────────────────┐
             │  Shared NFS /  │
             │  EFS Storage   │
             │  ($JENKINS_HOME)│
             └────────────────┘

Agent scaling with Kubernetes:
Jenkins → Kubernetes Plugin → Spin up Pod agents on demand
         → Agent runs build → Pod destroyed after build
         → No idle agents wasting resources
```

---

## 3. GitHub Actions — Deep Dive

### Architecture

```
GitHub Actions Architecture:

GitHub.com
┌──────────────────────────────────────────────────────────┐
│  Repository                                              │
│  ├── .github/workflows/*.yml                             │
│  │                                                        │
│  Events: push, pull_request, schedule, workflow_dispatch │
│           release, issue, repository_dispatch            │
└──────────────────────┬───────────────────────────────────┘
                       │ Trigger
                       ▼
┌──────────────────────────────────────────────────────────┐
│  GitHub Actions Service                                  │
│  - Parse workflow YAML                                   │
│  - Evaluate conditions (if:)                             │
│  - Queue jobs to runners                                 │
└──────────────────────┬───────────────────────────────────┘
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
  ┌───────────────┐        ┌────────────────┐
  │ GitHub-hosted │        │ Self-hosted    │
  │ Runners       │        │ Runners        │
  │ ubuntu-latest │        │ (your servers) │
  │ windows-latest│        │                │
  │ macos-latest  │        │                │
  └───────────────┘        └────────────────┘
```

### Complete GitHub Actions Workflow

```yaml
# .github/workflows/full-pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main, 'release/*']
    tags: ['v*']
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'    # Nightly at 2 AM UTC

# Cancel in-progress runs for same PR
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  PYTHON_VERSION: '3.11'

jobs:
  # ── Quality Gates ──────────────────────────────────────────
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install -r requirements-dev.txt
      
      - name: Lint with flake8
        run: flake8 . --max-line-length=120
      
      - name: Type check with mypy
        run: mypy app/ --strict
      
      - name: Format check with black
        run: black --check .

  test:
    name: Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt
      
      - name: Run tests with coverage
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
        run: |
          pytest tests/ \
            --cov=app \
            --cov-report=xml \
            --cov-report=html \
            --junitxml=test-results.xml \
            -v
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
          fail_ci_if_error: true
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results
          path: test-results.xml

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      
      - name: Run Gitleaks (secrets scan)
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'myapp'
          path: '.'
          format: 'HTML'

  # ── Build ──────────────────────────────────────────────────
  build:
    name: Build Container
    needs: [quality, test, security]
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=ref,event=branch
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}
      
      - name: Scan built image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  # ── Deploy Staging ────────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
        with:
          repository: org/infrastructure-repo
          token: ${{ secrets.INFRA_REPO_TOKEN }}
      
      - name: Update staging manifest
        run: |
          cd apps/myapp/staging
          sed -i "s|image:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}|" deployment.yaml
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git commit -am "chore: deploy sha-${{ github.sha }} to staging"
          git push
      
      # ArgoCD will detect the git change and sync automatically

  # ── Deploy Production ─────────────────────────────────────
  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production    # Requires manual approval in GitHub
    if: startsWith(github.ref, 'refs/tags/v')
    
    steps:
      - uses: actions/checkout@v4
        with:
          repository: org/infrastructure-repo
          token: ${{ secrets.INFRA_REPO_TOKEN }}
      
      - name: Update production manifest
        run: |
          cd apps/myapp/production
          sed -i "s|image:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}|" deployment.yaml
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git commit -am "chore: deploy ${{ github.ref_name }} to production"
          git push
```

---

## 4. GitLab CI — Deep Dive

### GitLab CI Architecture

```
GitLab Server
├── GitLab Rails (web UI, API)
├── Sidekiq (background jobs)
├── PostgreSQL (pipeline state)
└── Redis (job queues)
        │
        │ GitLab Runner registration (token-based)
        │ Runner polls: GET /api/v4/jobs/request
        │
        ▼
GitLab Runners (multiple executors):
├── Shell executor    (run directly on host)
├── Docker executor   (run in containers) ← most common
├── Kubernetes executor (run as K8s pods) ← enterprise
└── Docker Machine    (auto-scale EC2/GCP)
```

### Complete GitLab CI Pipeline

```yaml
# .gitlab-ci.yml

variables:
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  DOCKER_DRIVER: overlay2

stages:
  - validate
  - test
  - build
  - scan
  - deploy-staging
  - integration-test
  - deploy-production

# ── Templates ─────────────────────────────────────────────────
.docker-base: &docker-base
  image: docker:24
  services:
    - name: docker:24-dind
      command: ["--tls=false"]
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin

.k8s-base: &k8s-base
  image: bitnami/kubectl:latest
  before_script:
    - echo "$KUBE_CONFIG" | base64 -d > ~/.kube/config

# ── Validate ──────────────────────────────────────────────────
lint:
  stage: validate
  image: python:3.11
  script:
    - pip install flake8 black mypy
    - flake8 . --max-line-length=120
    - black --check .
    - mypy app/

validate-helm:
  stage: validate
  image: alpine/helm:3.12.0
  script:
    - helm lint charts/myapp/
    - helm template charts/myapp/ | kubectl apply --dry-run=client -f -

# ── Test ──────────────────────────────────────────────────────
unit-tests:
  stage: test
  image: python:3.11
  services:
    - name: postgres:15
      alias: postgres
    - name: redis:7
      alias: redis
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: testpass
    DATABASE_URL: "postgresql://postgres:testpass@postgres:5432/testdb"
    REDIS_URL: "redis://redis:6379"
  script:
    - pip install -r requirements.txt -r requirements-dev.txt
    - pytest tests/ --cov=app --cov-report=xml --junitxml=report.xml -v
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    expire_in: 1 week

sast:
  stage: test
  include:
    - template: Security/SAST.gitlab-ci.yml

dependency-scanning:
  stage: test
  include:
    - template: Security/Dependency-Scanning.gitlab-ci.yml

# ── Build ──────────────────────────────────────────────────────
build-image:
  <<: *docker-base
  stage: build
  script:
    - docker build
        --cache-from $CI_REGISTRY_IMAGE:latest
        --tag $IMAGE_TAG
        --tag $CI_REGISTRY_IMAGE:latest
        --label "org.opencontainers.image.revision=$CI_COMMIT_SHORT_SHA"
        .
    - docker push $IMAGE_TAG
    - docker push $CI_REGISTRY_IMAGE:latest
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# ── Scan ──────────────────────────────────────────────────────
container-scanning:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image
        --exit-code 1
        --severity HIGH,CRITICAL
        --no-progress
        $IMAGE_TAG
  allow_failure: false

# ── Deploy Staging ─────────────────────────────────────────────
deploy-staging:
  <<: *k8s-base
  stage: deploy-staging
  environment:
    name: staging
    url: https://staging.example.com
  variables:
    KUBE_CONFIG: $KUBE_CONFIG_STAGING
  script:
    - kubectl set image deployment/myapp
        myapp=$IMAGE_TAG
        -n staging
    - kubectl rollout status deployment/myapp -n staging --timeout=300s
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# ── Integration Tests ──────────────────────────────────────────
integration-tests:
  stage: integration-test
  image: python:3.11
  variables:
    TARGET_URL: https://staging.example.com
  script:
    - pip install pytest requests
    - pytest tests/integration/ -v --target=$TARGET_URL
  needs: ["deploy-staging"]
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# ── Deploy Production ──────────────────────────────────────────
deploy-production:
  <<: *k8s-base
  stage: deploy-production
  environment:
    name: production
    url: https://example.com
  variables:
    KUBE_CONFIG: $KUBE_CONFIG_PROD
  script:
    - kubectl set image deployment/myapp
        myapp=$IMAGE_TAG
        -n production
    - kubectl rollout status deployment/myapp -n production --timeout=600s
  when: manual          # Requires manual click in GitLab UI
  allow_failure: false
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  needs: ["integration-tests"]
```

---

## 5. ArgoCD — GitOps CD

### What is ArgoCD?

ArgoCD is a **declarative, GitOps-based continuous delivery tool for Kubernetes**. It watches a Git repository for changes to Kubernetes manifests and automatically (or manually) applies them to the cluster.

### ArgoCD Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    ArgoCD Server                         │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  API Server  │  │  Application │  │  Repository  │  │
│  │  (gRPC/HTTP) │  │  Controller  │  │  Server      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                           │                  │          │
│  ┌──────────────┐         │                  │          │
│  │  Dex (OIDC   │         │ Reconcile        │ Git Poll │
│  │  Auth)       │         │ Loop             │ /Webhook │
│  └──────────────┘         │                  │          │
└──────────────────────────┬──────────────────┬───────────┘
                           │                  │
                           ▼                  ▼
                    Kubernetes API          Git Repo
                    (apply manifests)    (desired state)

Components:
- argocd-server: WebUI + gRPC API
- argocd-repo-server: Clones repos, renders manifests (Helm, Kustomize, raw YAML)
- argocd-application-controller: Reconciliation loop (desired vs actual state)
- argocd-dex-server: SSO via OIDC
- Redis: Cache + pub/sub
```

### ArgoCD Application Definition

```yaml
# Application CRD - Tells ArgoCD what to deploy and where
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # Clean up on delete
spec:
  project: production  # ArgoCD Project (RBAC)
  
  source:
    repoURL: https://github.com/org/infrastructure-repo.git
    targetRevision: HEAD      # Branch, tag, or commit SHA
    path: apps/myapp/production
    
    # For Helm:
    # helm:
    #   valueFiles:
    #   - values-production.yaml
    #   parameters:
    #   - name: image.tag
    #     value: v1.2.3
    
    # For Kustomize:
    # kustomize:
    #   images:
    #   - myapp=registry.example.com/myapp:v1.2.3
  
  destination:
    server: https://kubernetes.default.svc  # Target cluster
    namespace: production
  
  syncPolicy:
    automated:
      prune: true          # Delete resources not in Git
      selfHeal: true       # Revert manual kubectl changes
      allowEmpty: false    # Never auto-sync empty state
    syncOptions:
    - CreateNamespace=true
    - ApplyOutOfSyncOnly=true
    - ServerSideApply=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas   # Ignore HPA-managed replicas
```

### ArgoCD App of Apps Pattern

```yaml
# Root application deploys all other applications
# "App of Apps" pattern for managing multiple apps

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-apps
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/org/infrastructure-repo.git
    targetRevision: HEAD
    path: argocd/apps           # Directory containing Application CRDs
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

# argocd/apps/ directory contains:
# - myapp-staging.yaml    (Application CRD)
# - myapp-production.yaml (Application CRD)
# - monitoring.yaml       (Application CRD)
# - ingress-nginx.yaml    (Application CRD)
```

### Deployment Strategies with ArgoCD

```yaml
# Blue-Green with Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    blueGreen:
      activeService: myapp-active    # Production traffic
      previewService: myapp-preview  # Preview traffic
      autoPromotionEnabled: false    # Manual promotion

# Canary with Argo Rollouts
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5         # 5% traffic to new version
      - pause: {duration: 10m}
      - setWeight: 20
      - pause: {duration: 10m}
      - analysis:            # Run metrics analysis
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: myapp
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100       # Full rollout
```

---

## 6. Pipeline Design Patterns

### Build Once, Deploy Many

```
Principle: Build artifact ONCE, use same artifact across all environments
Never rebuild for each environment (different builds = different code)

Bad pattern:
  Build for dev → deploy to dev
  Build for staging → deploy to staging  (different build!)
  Build for prod → deploy to prod        (different build!)

Good pattern:
  Build once → push to registry with SHA tag
  Deploy SHA tag to dev
  Same SHA tag to staging (promoted)
  Same SHA tag to prod (promoted)

Implementation:
  CI: docker build && push myapp:sha-abc123
  CD (dev): kubectl set image deployment myapp=myapp:sha-abc123
  CD (staging): kubectl set image deployment myapp=myapp:sha-abc123 (same!)
  CD (prod): kubectl set image deployment myapp=myapp:sha-abc123 (same!)
```

### Deployment Strategies Comparison

```
Rolling Update:
  Old: [A][A][A][A][A]
  →    [B][A][A][A][A]
  →    [B][B][A][A][A]
  →    [B][B][B][A][A]
  →    [B][B][B][B][B]
  
  Pros: Zero downtime, gradual
  Cons: Mixed versions running simultaneously

Blue-Green:
  Blue (current): [A][A][A][A][A]  ← all production traffic
  Green (new):    [B][B][B][B][B]  ← staging, running new version
  Switch load balancer: all traffic → green instantly
  
  Pros: Instant rollback (switch back to blue)
  Cons: 2x infrastructure cost

Canary:
  [A][A][A][A][A][A][A][A][A][B]
  90% traffic → A, 10% traffic → B
  Monitor metrics: errors, latency
  Gradually increase to 100% or rollback
  
  Pros: Real user testing, safe
  Cons: Complex, needs good metrics/monitoring
```

---

## 7. Security in CI/CD

### Secrets Management

```bash
# NEVER hardcode secrets in Jenkinsfile or .gitlab-ci.yml
# NEVER put secrets in environment variables in plain text

# Jenkins: Use Credentials Plugin
withCredentials([string(credentialsId: 'api-key', variable: 'API_KEY')]) {
    sh 'curl -H "Authorization: Bearer $API_KEY" ...'
}

# GitHub Actions: Use Secrets
# In workflow: ${{ secrets.API_KEY }}
# Never print: echo "${{ secrets.API_KEY }}"  ← GitHub masks this anyway

# GitLab CI: Protected Variables
# Settings → CI/CD → Variables → Protect variable (only for protected branches)

# Best: Vault + CI integration
# CI pipeline fetches secrets from HashiCorp Vault at runtime
# Short-lived tokens, audit trail, secret rotation
```

### Supply Chain Security (SLSA)

```
Software Supply Chain Attack surfaces:
1. Developer machine (compromised)
2. Source code (malicious PR)
3. Build system (compromised runner)
4. Dependencies (malicious package)
5. Artifacts (image tampering)
6. Deployment (malicious admission)

Defenses:
1. Signed commits (GPG) + branch protection
2. Required code reviews + CODEOWNERS
3. Ephemeral runners (no persistent state)
4. Dependency pinning + SBOMs
5. Image signing (cosign/Sigstore)
6. OPA/Kyverno admission control

# Sign container images with cosign (Sigstore)
cosign sign --key cosign.key myapp:sha-abc123
cosign verify --key cosign.pub myapp:sha-abc123

# Generate SBOM (Software Bill of Materials)
syft myapp:sha-abc123 -o cyclonedx-json > sbom.json
```

---

## 8. Troubleshooting CI/CD

### Common Issues

```bash
# Jenkins: Agent offline / stuck in queue
# Check: Manage Jenkins → Nodes → Agent status
# Fix: Restart agent, check connectivity, check labels

# Jenkins: Build stuck in "pending"
# Check: Are there agents available with required label?
# Fix: Scale up agents, check label mismatch

# GitHub Actions: Workflow not triggering
# Check: Branch name matches trigger exactly
# Check: YAML syntax valid (use https://rhysd.github.io/actionlint/)
# Check: repository settings → Actions → allowed

# ArgoCD: Application OutOfSync
argocd app get myapp                    # Show sync status
argocd app diff myapp                   # Show what's different
argocd app sync myapp --dry-run         # Preview sync
argocd app sync myapp                   # Force sync

# ArgoCD: Sync failed
argocd app logs myapp                   # Show sync logs
kubectl get events -n argocd           # Kubernetes events

# ArgoCD: Infinite sync loop
# Cause: Application modifies itself (timestamp, generated field)
# Fix: Add ignoreDifferences in Application spec

# Pipeline: Docker build cache miss (slow builds)
# Use: --cache-from previous image
# Use: GitHub Actions cache
# Use: BuildKit inline cache

# Pipeline: Flaky tests causing random failures
# Identify: grep "FAILED" build logs, count occurrences
# Fix: Fix the test or retry mechanism (pytest-retry)
```

---

## 9. Interview Questions — Phase 3

**Q: Explain the difference between CI and CD. What does a complete CI/CD pipeline look like?**

CI (Continuous Integration) is the practice of frequently merging code changes, with automated build and test validation on every merge. The goal is to catch integration bugs early.

CD can mean either Continuous Delivery (automated deployment to staging, manual gate to prod) or Continuous Deployment (automated all the way to production).

A complete pipeline includes: code checkout → dependency caching → linting → unit tests → security scanning (SAST, secrets, dependency vulnerabilities) → artifact build (Docker image) → artifact scanning (Trivy) → push to registry → deploy to staging → integration/smoke tests → manual approval (if Delivery) → production deployment → health checks → notification.

**Q: What is GitOps? How does ArgoCD implement it?**

GitOps is an operational model where Git is the single source of truth for desired system state. All changes go through Git (pull requests), providing version control, code review, and audit trails for infrastructure.

ArgoCD implements GitOps via a reconciliation loop: the Application Controller continuously compares the desired state in Git with the actual state in Kubernetes. If they diverge (someone did `kubectl apply` manually, or a Git commit changed a manifest), ArgoCD detects the drift and can automatically (selfHeal: true) or manually reconcile them.

**Q: How do you handle secrets in CI/CD pipelines?**

Never store secrets in: code, environment variables in plain text, CI configuration files.

Best practices:
1. Use CI-native secret stores (GitHub Secrets, GitLab Variables, Jenkins Credentials)
2. For production: use HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault
3. Inject secrets at runtime, not build time
4. Use short-lived credentials where possible (AWS IAM roles, Vault dynamic secrets)
5. Never echo/print secrets in logs
6. Rotate secrets regularly
7. Scan repos for leaked secrets (Gitleaks in pre-commit + CI)

**Q: What deployment strategy would you use for a high-traffic financial API?**

For a financial API where both reliability and correctness are critical, I'd use **canary deployment** with:
1. Canary starts at 1-5% traffic (real user validation)
2. Automated metrics gates: error rate, latency P99, business metrics (transaction success rate)
3. Minimum 30-minute soak time at each step
4. Automatic rollback if metrics degrade
5. Human approval gates at 50% and 100%
6. Full observability before proceeding (Prometheus, distributed tracing)

I'd avoid blue-green for this case because full cutover is risky for financial transactions in-flight. Canary gives real traffic validation with blast radius control.

---

*End of Phase 3 — CI/CD*
# Phase 4: Containers — Docker Deep Internals

---

## Table of Contents
1. [What is Docker?](#1-what-is-docker)
2. [Container Runtime Internals](#2-container-runtime-internals)
3. [Linux Namespaces — Complete Guide](#3-linux-namespaces--complete-guide)
4. [cgroups — Resource Control](#4-cgroups--resource-control)
5. [Docker Image Internals](#5-docker-image-internals)
6. [Docker Networking Deep Dive](#6-docker-networking-deep-dive)
7. [Dockerfile Best Practices](#7-dockerfile-best-practices)
8. [Docker Storage](#8-docker-storage)
9. [Docker Security](#9-docker-security)
10. [Production Troubleshooting](#10-production-troubleshooting)
11. [Interview Questions — Phase 4](#11-interview-questions--phase-4)

---

## 1. What is Docker?

Docker is a platform for **building, shipping, and running applications in containers**. A container is an isolated process running on a Linux host, using kernel features (namespaces, cgroups) for isolation.

**Key distinction: Container vs VM**

```
Virtual Machine:
┌──────────────────────────────────────────────────┐
│ Hardware                                          │
├──────────────────────────────────────────────────┤
│ Host OS + Hypervisor (KVM, VMware, Hyper-V)      │
├────────────────┬─────────────────────────────────┤
│  VM 1          │  VM 2          │  VM 3           │
│  Guest OS      │  Guest OS      │  Guest OS       │
│  (Linux 2GB)   │  (Windows 2GB) │  (Linux 2GB)    │
│  App           │  App           │  App            │
└────────────────┴─────────────────────────────────┘
VMs: Full OS per instance, slow boot, heavy, strong isolation

Container:
┌──────────────────────────────────────────────────┐
│ Hardware                                          │
├──────────────────────────────────────────────────┤
│ Host OS (Linux kernel)                           │
├────────────────┬─────────────────────────────────┤
│  Container 1   │  Container 2   │  Container 3    │
│  (namespaced)  │  (namespaced)  │  (namespaced)   │
│  App + libs    │  App + libs    │  App + libs     │
│  (no kernel!)  │  (no kernel!)  │  (no kernel!)   │
└────────────────┴─────────────────────────────────┘
Containers: Shared kernel, instant start, lightweight, process-level isolation
```

**Why Docker matters for DevOps:**
- Eliminates "works on my machine" → same image everywhere
- Faster deployment (seconds vs minutes for VMs)
- Higher density (more apps per server)
- Immutable infrastructure (rebuild, don't patch)
- Microservices enablement (one container per service)
- Foundation for Kubernetes

### Docker Architecture

```
docker CLI (client)
     │ REST API
     ▼
Docker Daemon (dockerd)
     │ gRPC via /run/containerd/containerd.sock
     ▼
containerd (higher-level runtime)
     │ calls OCI runtime
     ▼
runc (low-level OCI runtime)
     │ creates namespaces, cgroups
     ▼
Container (process on host)

Network:
docker CLI → HTTP REST → dockerd (unix:///var/run/docker.sock or tcp)

Port flow for remote Docker API:
Client → TCP 2375 (insecure) or TCP 2376 (TLS) → dockerd
```

---

## 2. Container Runtime Internals

### What Happens When You Run `docker run nginx`

```
$ docker run -d -p 80:80 --name web nginx

Step 1: docker CLI parses command
        Sends POST /containers/create to dockerd via Docker socket

Step 2: dockerd checks if image exists locally
        If not: pull from registry
          - Parse "nginx" → "docker.io/library/nginx:latest"
          - GET /v2/library/nginx/manifests/latest from registry
          - Download layers (gzipped tarballs)
          - Verify layer digests (SHA256)
          - Store in /var/lib/docker/overlay2/

Step 3: dockerd instructs containerd to create container
        containerd pulls image spec, prepares rootfs
        Creates container bundle:
          - config.json (OCI runtime spec)
          - rootfs/ (merged overlay filesystem)

Step 4: containerd calls runc
        runc reads config.json:
          - Creates namespaces: pid, net, mnt, uts, ipc, user
          - Sets up cgroups: memory limit, cpu quota
          - Mounts overlayfs as rootfs
          - Sets up network (calls CNI plugin for networking)
          - pivot_root or chroot to container's rootfs
          - Drops capabilities (e.g., removes CAP_SYS_ADMIN)
          - Executes container entrypoint: /docker-entrypoint.sh

Step 5: Container process is running
        PID 1 in container's PID namespace
        (may be different PID number on host, e.g., PID 12345)

Step 6: dockerd sets up port forwarding
        Adds iptables DNAT rule:
        PREROUTING: tcp dport 80 → DNAT to 172.17.0.2:80
        
Step 7: Container is listed:
        docker ps → shows running container
```

### OCI (Open Container Initiative) Standards

```
OCI defines two standards:

1. Image Spec: How container images are stored and distributed
   - Manifest: JSON listing layers + config
   - Config: env vars, entrypoint, working dir, etc.
   - Layers: gzipped tar archives

2. Runtime Spec: How to run containers
   - config.json: namespaces, cgroups, mounts, process
   - Runtime interface: create, start, kill, delete
   - runc is the reference implementation

Why this matters: Any OCI-compliant runtime (runc, kata-containers, gVisor)
can run OCI images. Kubernetes uses CRI (Container Runtime Interface)
to talk to containerd/CRI-O, which use runc underneath.

Container Runtime Landscape:
  containerd ← Docker, Kubernetes
  CRI-O      ← Kubernetes (Red Hat)
  kata-containers ← hardware virtualization (VMs as containers)
  gVisor     ← Google's kernel in userspace (security)
  runc       ← low-level OCI runtime (used by above)
  crun       ← faster C reimplementation of runc
```

---

## 3. Linux Namespaces — Complete Guide

### All 8 Namespace Types

**1. PID Namespace — Process Isolation**
```
Host PID space:           Container PID space:
PID 1  (systemd)          PID 1  (nginx) ← maps to host PID 12345
PID 2  (kthreadd)         PID 2  (nginx worker) ← maps to host PID 12346
...
PID 12345 (nginx)
PID 12346 (nginx worker)

Container cannot see host PIDs (PID 12345 → appears as PID 1 inside)
Container cannot kill host processes
```

**2. Network Namespace — Network Isolation**
```
Container gets its own:
- Network interfaces (eth0 in container, different from host eth0)
- IP address (e.g., 172.17.0.2)
- Routing table
- iptables rules
- Port space (container can use port 80, even if host also uses it)

Implementation:
Host: veth0 ←──────────────── Docker bridge (docker0)
       │                               │
       │ veth pair (virtual cable)     │
       │                               │
Container: eth0                  Other containers

# See container's network namespace
ip netns list
nsenter -t <pid> -n ip addr show   # Enter container's netns
```

**3. Mount Namespace — Filesystem Isolation**
```
Container sees:
/           ← from image overlayfs
/proc       ← container's proc namespace
/sys        ← partially from host
/dev        ← controlled device access
/tmp        ← ephemeral

Host mounts are not visible in container
Container mounts are not visible on host
(unless explicitly shared via bind mount / volume)
```

**4. UTS Namespace — Hostname Isolation**
```
Container can have its own hostname:
hostname inside container: "web-server"
hostname on host: "prod-node-01"

docker run --hostname myapp nginx
```

**5. IPC Namespace — Inter-Process Communication Isolation**
```
Shared memory, semaphores, message queues are isolated per namespace
Containers cannot access each other's IPC mechanisms
(Unless: docker run --ipc=host or --ipc=container:name)
```

**6. User Namespace — User Isolation**
```
Map container root (UID 0) to non-privileged host UID
Container root = host UID 100000 (rootless containers)

/etc/subuid:
  user1:100000:65536   # user1 can map UIDs 100000-165536

Benefits: Container root can't damage host filesystem even if it escapes
docker run --user 1000:1000 nginx   # Run as non-root
```

**7. Cgroup Namespace**
```
Container sees its own cgroup hierarchy
Can't see or modify host cgroup settings
```

**8. Time Namespace (Linux 5.6+)**
```
Container can have different system time offset
Useful for testing time-sensitive applications
```

### Demonstrating Namespaces

```bash
# Create a new network namespace
ip netns add testns
ip netns exec testns ip link show
ip netns exec testns ip addr

# See container's namespaces
docker run -d --name test nginx
PID=$(docker inspect test --format '{{.State.Pid}}')
ls -la /proc/$PID/ns/
# lrwxrwxrwx 1 root root 0 Jan 15 net -> net:[4026532345]
# lrwxrwxrwx 1 root root 0 Jan 15 pid -> pid:[4026532346]

# Enter container's namespaces (same as docker exec)
nsenter --target $PID --mount --uts --ipc --net --pid -- /bin/bash

# Two containers sharing same network namespace (sidecar pattern!)
docker run -d --name app nginx
docker run -d --name sidecar --network container:app busybox
# sidecar shares app's network namespace
# sidecar can reach app via localhost
```

---

## 4. cgroups — Resource Control

### What are cgroups?

**Control Groups (cgroups)** limit, account for, and isolate resource usage (CPU, memory, disk I/O, network) of process groups.

```
cgroup hierarchy (v2 unified):
/sys/fs/cgroup/
├── system.slice/
│   ├── docker-abc123.scope/    ← Container 1
│   │   ├── memory.max          "536870912" (512MB)
│   │   ├── cpu.max             "50000 100000" (50% of 1 CPU)
│   │   └── blkio.weight        "100"
│   └── docker-def456.scope/    ← Container 2
│       ├── memory.max          "1073741824" (1GB)
│       └── cpu.max             "200000 100000" (2 CPUs)
└── user.slice/
```

### cgroup Subsystems

**Memory cgroup:**
```bash
# Docker uses cgroups to enforce memory limits
docker run --memory=512m --memory-swap=512m nginx
# → writes 512MB to container's memory.max
# → memory-swap=memory means no swap (harder limit)

# What happens when container exceeds memory limit:
# 1. Kernel tries to reclaim memory (drop caches)
# 2. If still over limit: OOM killer kills process in cgroup
# 3. Container exits with exit code 137 (128 + SIGKILL)

# Find which container was OOM killed:
dmesg | grep -i "oom kill"
docker inspect container_name | jq '.State.OOMKilled'

# Kubernetes memory limits use same mechanism:
# resources.limits.memory → sets memory.max in cgroup
```

**CPU cgroup:**
```bash
# CPU quota: container gets 50% of one CPU
docker run --cpus=0.5 nginx
# → cpu.max = "50000 100000" (50ms per 100ms period)

# CPU shares (soft limit - only matters under contention)
docker run --cpu-shares=512 nginx    # default is 1024

# Pin container to specific CPUs
docker run --cpuset-cpus="0,1" nginx  # Only on CPUs 0 and 1

# Monitor container CPU usage:
docker stats container_name
# Or: cat /sys/fs/cgroup/.../cpu.stat
```

**I/O cgroup:**
```bash
# Limit disk I/O
docker run --device-read-bps /dev/sda:10mb \
           --device-write-bps /dev/sda:10mb nginx

# Monitor: cat /sys/fs/cgroup/.../blkio.io_stat
```

---

## 5. Docker Image Internals

### Image Layers

```
Docker Image = Stack of read-only layers + metadata

Dockerfile:
FROM ubuntu:22.04          # Layer 1: base ubuntu image
RUN apt-get update && \    # Layer 2: installed packages
    apt-get install -y python3
COPY requirements.txt /    # Layer 3: requirements file
RUN pip install -r /req... # Layer 4: installed Python packages
COPY . /app                # Layer 5: application code
CMD ["python", "/app/main.py"]  # Metadata only (no new layer)

Each RUN/COPY/ADD creates a new layer:
Layer 5 [app code]
  ↑ overrides
Layer 4 [pip packages]
  ↑ overrides
Layer 3 [requirements.txt]
  ↑ overrides
Layer 2 [apt packages]
  ↑ overrides
Layer 1 [ubuntu base]

Layer caching: If a layer hasn't changed, Docker reuses cached layer
(extremely important for build speed!)
```

### OverlayFS — How Layers Are Mounted

```
OverlayFS merges multiple directories into unified view:

lowerdir (read-only layers, bottom to top):
  /var/lib/docker/overlay2/layer1/
  /var/lib/docker/overlay2/layer2/
  /var/lib/docker/overlay2/layer3/

upperdir (read-write, container-specific):
  /var/lib/docker/overlay2/container-abc/diff/

workdir (OverlayFS internal):
  /var/lib/docker/overlay2/container-abc/work/

merged (what container sees):
  /var/lib/docker/overlay2/container-abc/merged/

Write flow:
  Container writes to /etc/nginx/nginx.conf
  → Copy-on-Write: file copied from lowerdir to upperdir
  → Modified file now in upperdir (lowerdir unchanged)
  → Container sees modified version via merged view

# Inspect overlay mounts:
docker inspect container_name | jq '.[0].GraphDriver'
mount | grep overlay
```

### Multi-Stage Builds

```dockerfile
# Multi-stage: Build in large image, copy binary to small image

# Stage 1: Builder (large, has all tools)
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download                    # Separate layer for cache
COPY . .
RUN CGO_ENABLED=0 GOOS=linux \
    go build -ldflags="-w -s" \
    -o /app/server ./cmd/server

# Stage 2: Final image (tiny, no build tools)
FROM gcr.io/distroless/static:nonroot
# distroless has no shell, no package manager → minimal attack surface

COPY --from=builder /app/server /server
COPY --from=builder /app/configs /configs

USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]

# Result: ~10MB image vs ~1GB builder image!

# Python example:
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
USER 1000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Image Naming and Registry

```
Full image reference:
registry.example.com:5000/org/repo:tag@sha256:abc123...
│                   │    │   │    │   └── Content digest (immutable!)
│                   │    │   │    └── Tag (mutable - "latest" changes!)
│                   │    │   └── Repository name
│                   │    └── Organization/namespace
│                   └── Port (optional)
└── Registry hostname (default: docker.io)

Canonical form examples:
nginx                           → docker.io/library/nginx:latest
ubuntu:22.04                    → docker.io/library/ubuntu:22.04
ghcr.io/org/app:v1.2.3         → GitHub Container Registry
gcr.io/project/app:latest       → Google Container Registry
123.dkr.ecr.us-east-1.amazonaws.com/app:prod → AWS ECR

Best practices:
1. Always pin digest in production: myapp@sha256:abc123...
2. Never use :latest in production
3. Use semantic versioning: :v1.2.3
4. Also tag with SHA: :sha-abc1234

# Inspect image layers
docker history myapp:v1.2.3
docker inspect myapp:v1.2.3 | jq '.[0].RootFS.Layers'

# Dive tool for detailed layer inspection
dive myapp:v1.2.3
```

---

## 6. Docker Networking Deep Dive

### Docker Network Types

```
1. bridge (default) - Container-to-container on same host
2. host  - Container shares host network namespace
3. none  - No network (maximum isolation)
4. overlay - Multi-host networking (Docker Swarm, K8s)
5. macvlan - Container gets its own MAC/IP on host network
6. ipvlan  - Similar to macvlan, layer 3

Default bridge network (docker0):
Host: docker0 interface (172.17.0.1)
Containers: 172.17.0.2, 172.17.0.3, etc.
Container-to-container: via docker0 bridge
Container-to-internet: via NAT (iptables MASQUERADE)

User-defined bridge (recommended!):
docker network create myapp-net

Benefits over default bridge:
- DNS resolution by container name (ping app by name!)
- Better isolation
- Can be created with custom subnet
```

### Complete Network Flow: Container → Internet

```
Container (nginx) makes HTTP request to api.example.com
                │
                ▼
eth0 in container (172.17.0.2)
                │
                ▼ via veth pair
vethXXX on host (connected to docker0)
                │
                ▼
docker0 bridge (172.17.0.1)
                │
                ▼
Kernel routing: 172.17.0.2 → 0.0.0.0/0 (default route) → eth0
                │
                ▼
iptables POSTROUTING MASQUERADE:
Source: 172.17.0.2 → replaced with host IP (10.0.0.5)
                │
                ▼
eth0 (host interface, 10.0.0.5)
                │
                ▼
Router / Internet → api.example.com

# See the iptables rules Docker creates:
iptables -t nat -L -n -v
# MASQUERADE all -- 172.17.0.0/16 !docker0

# Verify with:
docker run --rm busybox traceroute 8.8.8.8
```

### Container Port Mapping Flow

```
External request: curl http://host-ip:8080
                │
                ▼
Host network interface (eth0: 10.0.0.5)
                │
                ▼
iptables PREROUTING (nat table):
DNAT: tcp dport 8080 → 172.17.0.2:80
                │
                ▼
Kernel routes to container IP (172.17.0.2)
                │
                ▼ via docker0 bridge
Container eth0 (172.17.0.2:80)
                │
                ▼
nginx process inside container

# Docker creates these rules automatically:
# iptables -t nat -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80

# See mapped ports:
docker port container_name
docker inspect container_name | jq '.[0].NetworkSettings.Ports'
```

### Docker DNS Resolution

```
Container queries: curl http://api-service:8080

DNS resolution flow:
1. Container's /etc/resolv.conf:
   nameserver 127.0.0.11  ← Docker's embedded DNS server
   options ndots:0

2. Docker DNS (127.0.0.11) intercepts query
   If same docker network: returns container IP
   If external: forwards to host's /etc/resolv.conf DNS

3. Container gets IP, makes connection

# Container DNS config
docker run --dns 8.8.8.8 nginx      # Custom DNS
docker run --add-host api:10.0.0.5 nginx  # Add /etc/hosts entry
```

---

## 7. Dockerfile Best Practices

### Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.5
# Use BuildKit features

ARG PYTHON_VERSION=3.11
ARG APP_VERSION=unknown

# ─── Base ────────────────────────────────────────────────────
FROM python:${PYTHON_VERSION}-slim AS base

# Security: create non-root user early
RUN groupadd --gid 1000 appuser && \
    useradd --uid 1000 --gid appuser --shell /bin/bash appuser

# System dependencies (rarely change → cache early)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates && \
    rm -rf /var/lib/apt/lists/*    # Clean up to reduce layer size

WORKDIR /app

# ─── Dependencies ─────────────────────────────────────────────
FROM base AS deps

# Copy ONLY requirements first (changes rarely → better cache)
COPY requirements.txt .

# Install with cache mount (BuildKit feature)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

# ─── Production ───────────────────────────────────────────────
FROM base AS production

# Copy installed packages from deps stage
COPY --from=deps /usr/local/lib/python3.11 /usr/local/lib/python3.11
COPY --from=deps /usr/local/bin /usr/local/bin

# Copy application code last (changes most frequently)
COPY --chown=appuser:appuser . .

# Labels for metadata (OCI spec)
LABEL org.opencontainers.image.version="${APP_VERSION}" \
      org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.vendor="ACME Corp"

# Security: run as non-root
USER appuser

# Healthcheck
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

# Use exec form (not shell form) for proper signal handling
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", \
     "--workers", "4", "--log-config", "logging.yaml"]
```

### Image Size Optimization

```bash
# Common mistakes and fixes:

# BAD: apt cache not cleaned
RUN apt-get install -y python3
# GOOD:
RUN apt-get update && apt-get install -y --no-install-recommends python3 \
    && rm -rf /var/lib/apt/lists/*

# BAD: Multiple RUN commands (each creates a layer)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
# GOOD: Chain commands
RUN apt-get update && apt-get install -y --no-install-recommends curl git \
    && rm -rf /var/lib/apt/lists/*

# BAD: Copying everything including .git, node_modules, etc.
COPY . .
# GOOD: Use .dockerignore
# .dockerignore:
# .git/
# node_modules/
# __pycache__/
# *.pyc
# .env
# tests/
# docs/
# *.md
# Dockerfile
# .dockerignore

# BAD: Dev dependencies in production image
# GOOD: Multi-stage builds (shown above)

# Check image size breakdown:
docker history myapp:latest
dive myapp:latest    # Interactive exploration
```

### Signal Handling and Graceful Shutdown

```dockerfile
# WRONG: Shell form (PID 1 is /bin/sh, not your app)
CMD python app.py
# → app gets SIGTERM via /bin/sh, may not handle it correctly

# RIGHT: Exec form (your app is PID 1)
CMD ["python", "app.py"]
# → app receives SIGTERM directly

# Or use tini for proper signal handling
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["python", "app.py"]
# tini reaps zombies and forwards signals correctly
```

---

## 8. Docker Storage

### Volume Types

```
1. Volumes (managed by Docker - PREFERRED for production)
   Location: /var/lib/docker/volumes/
   Created by: docker volume create mydata
   Usage: -v mydata:/app/data
   Benefits: managed, backup-friendly, share between containers

2. Bind Mounts (host path mounted in container)
   Usage: -v /host/path:/container/path
   Benefits: development (hot reload), access host files
   Risks: container can write anywhere on host

3. tmpfs mounts (in-memory, not persisted)
   Usage: --tmpfs /tmp
   Benefits: sensitive data (secrets), fast temp storage
   Limitation: lost when container stops

# Volume commands
docker volume create mydata
docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune      # Remove all unused volumes ⚠️

# Backup a volume
docker run --rm \
    -v mydata:/data \
    -v $(pwd):/backup \
    alpine tar czf /backup/mydata-backup.tar.gz /data

# Restore
docker run --rm \
    -v mydata:/data \
    -v $(pwd):/backup \
    alpine tar xzf /backup/mydata-backup.tar.gz -C /
```

---

## 9. Docker Security

### Security Best Practices

```bash
# 1. Run as non-root
docker run --user 1000:1000 nginx
# Or in Dockerfile: USER 1000

# 2. Read-only filesystem
docker run --read-only nginx
# Mount writable dirs where needed:
docker run --read-only \
    --tmpfs /tmp \
    --tmpfs /var/cache/nginx \
    nginx

# 3. Drop capabilities (least privilege)
docker run \
    --cap-drop=ALL \
    --cap-add=NET_BIND_SERVICE \
    nginx
# Only add what's needed

# 4. No new privileges
docker run --security-opt=no-new-privileges nginx

# 5. Limit resources (prevent DoS)
docker run \
    --memory=256m \
    --cpus=0.5 \
    --pids-limit=100 \
    nginx

# 6. Disable inter-container communication on default bridge
# In /etc/docker/daemon.json:
# {"icc": false}

# 7. Use specific image tags + digest
docker pull nginx:1.25.3@sha256:abc123...

# 8. Scan images for vulnerabilities
trivy image nginx:1.25.3
grype nginx:1.25.3
docker scout cves nginx:1.25.3

# 9. Use Docker Content Trust (sign + verify images)
export DOCKER_CONTENT_TRUST=1
docker push myregistry.com/myapp:v1.0    # Requires signing key

# 10. Secrets (never in ENV or image layers)
# Use Docker Secrets (Swarm) or runtime injection:
docker run -e DB_PASSWORD=$(vault kv get -field=password secret/db) myapp
# Or: mount Vault agent sidecar
```

### Docker Daemon Security

```json
// /etc/docker/daemon.json
{
    "icc": false,                        // No inter-container comms
    "no-new-privileges": true,           // Default no-new-privileges
    "userns-remap": "default",           // User namespace remapping
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    },
    "live-restore": true,               // Keep containers running on daemon restart
    "storage-driver": "overlay2",
    "default-ulimits": {
        "nofile": {
            "Hard": 64000,
            "Soft": 64000
        }
    }
}
```

---

## 10. Production Troubleshooting

### Container Debugging Commands

```bash
# ── Running container ─────────────────────────────────────────
# Get shell in running container
docker exec -it container_name bash
docker exec -it container_name sh  # if no bash

# Run as root (override USER)
docker exec -u 0 -it container_name bash

# View logs
docker logs container_name
docker logs -f container_name      # Follow (like tail -f)
docker logs --tail 100 container_name
docker logs --since 30m container_name

# Resource usage
docker stats                       # All containers
docker stats container_name       # Specific container
docker top container_name         # Processes in container

# Inspect
docker inspect container_name
docker inspect container_name | jq '.[0].State'
docker inspect container_name | jq '.[0].NetworkSettings'
docker inspect container_name | jq '.[0].HostConfig.Memory'

# ── Crashed container ──────────────────────────────────────────
# View logs of crashed/stopped container
docker logs dead_container_name

# Inspect exit code
docker inspect dead_container --format '{{.State.ExitCode}}'
# 0 = clean exit
# 1 = application error
# 137 = OOM killed (128 + 9)
# 143 = SIGTERM (128 + 15)

# Copy files from stopped container
docker cp dead_container:/var/log/app.log .

# ── Image debugging ───────────────────────────────────────────
# Run shell in image without starting normal entrypoint
docker run -it --entrypoint sh nginx
docker run -it --entrypoint bash python:3.11

# Debug a failing build
docker build --target=deps .       # Stop at specific stage
docker run -it <intermediate_sha> bash  # Debug that stage

# ── Network debugging ──────────────────────────────────────────
# From inside container (minimal image - add tools)
docker run --network container:myapp \
    --pid container:myapp \
    nicholasjackson/network-tools

# From host
docker run --rm --net=host nicolaka/netshoot tcpdump -i any port 80

# ── Performance issues ──────────────────────────────────────────
# Container using too much memory?
docker stats --no-stream container_name
# Check: actual RSS vs limit
cat /sys/fs/cgroup/memory/docker/<id>/memory.usage_in_bytes

# Container CPU throttled?
cat /sys/fs/cgroup/cpu/docker/<id>/cpu.stat | grep throttled
# If throttled_time > 0: container hitting CPU limit
# Fix: increase --cpus or optimize application

# Container slow I/O?
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    google/cadvisor   # Full metrics for all containers
```

### Common Production Issues

```
Issue: Container keeps restarting (CrashLoopBackOff in K8s)
Cause: 
  - OOM kill (exit 137)
  - Crash on startup (missing env var, config error)
  - Health check failing
Diagnosis:
  docker logs container_name 2>&1 | tail -50
  docker inspect container_name | jq '.[0].State'
Fix:
  - Increase memory if OOM
  - Fix config/env vars
  - Fix health check

Issue: Can't connect to container port
Cause:
  - Port not exposed (no -p flag)
  - App binding to 127.0.0.1 instead of 0.0.0.0
  - Firewall blocking
Diagnosis:
  docker port container_name
  docker exec container_name ss -tlnp
  docker inspect container_name | jq '.[0].NetworkSettings.Ports'
Fix:
  - Add -p 8080:8080 to docker run
  - Change app to bind 0.0.0.0

Issue: Container disk full
Cause:
  - Logs filling container's writable layer
  - Large files written to container filesystem
  - Docker's /var/lib/docker full
Diagnosis:
  docker system df
  docker exec container_name df -h
Fix:
  - Mount volumes for log directories
  - Configure log rotation in daemon.json
  - docker system prune (carefully!)
  - Add data: tmpfs for temp files
```

---

## 11. Interview Questions — Phase 4

**Q: What is the difference between a container and a virtual machine?**

Both provide isolation, but fundamentally differently. A VM runs a complete operating system on virtualized hardware, including a kernel. Containers share the host kernel and use Linux namespaces for isolation. 

VMs: Full OS per instance (gigabytes), boot in minutes, strong hardware-level isolation, can run different OS kernels.

Containers: Share kernel (megabytes), start in milliseconds, process-level isolation via namespaces + cgroups, must be same kernel (Linux containers on Linux kernel).

In practice: Containers for application isolation and density; VMs for multi-tenant security boundaries or running different OS families.

**Q: Explain what happens when you run `docker run -p 8080:80 nginx`.**

1. Docker checks for nginx image locally → pulls if not found
2. Docker daemon tells containerd to create a container
3. containerd calls runc with OCI config including namespaces and cgroups
4. runc creates PID, net, mnt, uts, ipc namespaces
5. Sets up overlayfs with nginx image layers as lowerdir, new writable layer as upperdir
6. Creates veth pair: one end in container's network namespace (eth0), other on host bridge (docker0)
7. Assigns container IP (e.g., 172.17.0.2) from bridge's subnet
8. Adds iptables DNAT rule: `tcp dport 8080 → DNAT to 172.17.0.2:80`
9. Adds MASQUERADE rule for container → Internet traffic
10. Starts nginx process as PID 1 in container's PID namespace
11. nginx listens on 0.0.0.0:80 inside container

**Q: How do Docker image layers work? Why is layer ordering important in Dockerfile?**

Docker images are a stack of read-only layers. Each `RUN`, `COPY`, `ADD` instruction creates a new layer. OverlayFS merges them into a unified filesystem view.

Layer caching: Docker caches each layer by instruction hash + parent layer hash. If a layer hasn't changed, it's reused from cache, dramatically speeding up builds.

Ordering matters because: a cache miss invalidates all subsequent layers. Therefore: put frequently changing instructions LAST.

Best ordering:
1. Base image (changes never)
2. System packages (changes rarely)
3. Application dependencies/requirements (changes occasionally)
4. Application code (changes constantly)

If you copy all code first and then install dependencies, every code change rebuilds the dependency layer (slow). Copy requirements.txt first, install dependencies, THEN copy code.

**Q: How would you diagnose a container that's using excessive memory?**

```bash
# 1. Check current usage vs limit
docker stats container_name --no-stream
# or: cat /sys/fs/cgroup/memory/docker/<id>/memory.usage_in_bytes

# 2. Check if OOM killing is happening
dmesg | grep -i oom
docker inspect container_name | jq '.[0].State.OOMKilled'

# 3. Look at memory breakdown inside container
docker exec container_name cat /proc/meminfo
docker exec container_name smem -r | head -20

# 4. For JVM: check heap vs off-heap
docker exec java_container jcmd 1 VM.native_memory summary

# 5. Profile memory with specific tools
docker run --pid=container:myapp -it \
    my-profiling-image \
    jmap -heap $(cat /proc/1/status | grep Tgid | awk '{print $2}')
```

Common causes: Memory leak in application code, JVM heap not sized properly, caching too much data, connection pool not bounded.

---

*End of Phase 4 — Docker Internals*
# Phase 5: Kubernetes — Complete Internal Deep Dive

---

## Table of Contents
1. [What is Kubernetes?](#1-what-is-kubernetes)
2. [Control Plane Components](#2-control-plane-components)
3. [Worker Node Components](#3-worker-node-components)
4. [Kubernetes Networking](#4-kubernetes-networking)
5. [Workload Resources](#5-workload-resources)
6. [Services and Ingress](#6-services-and-ingress)
7. [Storage](#7-storage)
8. [Configuration and Secrets](#8-configuration-and-secrets)
9. [RBAC and Security](#9-rbac-and-security)
10. [Scaling and Scheduling](#10-scaling-and-scheduling)
11. [Service Mesh — Istio](#11-service-mesh--istio)
12. [Production Troubleshooting](#12-production-troubleshooting)
13. [Interview Questions — Phase 5](#13-interview-questions--phase-5)

---

## 1. What is Kubernetes?

Kubernetes (K8s) is a **container orchestration platform** that automates deployment, scaling, and management of containerized applications. Originally built by Google (based on Borg), now CNCF-maintained.

**Problems Kubernetes solves:**
- Scheduling containers across multiple hosts
- Automatic restart of failed containers
- Horizontal scaling (scale out/in based on load)
- Service discovery and load balancing
- Rolling updates and rollbacks
- Secret and configuration management
- Storage orchestration

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                             │
│  ┌──────────────┐  ┌──────────┐  ┌───────────────────────┐  │
│  │ kube-apiserver│  │  etcd    │  │  kube-scheduler       │  │
│  │ (REST API)   │  │(key-value│  │  (pod placement)      │  │
│  │              │  │  store)  │  │                       │  │
│  └──────────────┘  └──────────┘  └───────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  kube-controller-manager                             │    │
│  │  (ReplicaSet, Node, Service, Endpoint controllers)   │    │
│  └──────────────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  cloud-controller-manager (LoadBalancer, PV, Node)   │    │
│  └──────────────────────────────────────────────────────┘    │
└─────────────────────────────┬────────────────────────────────┘
                              │ API calls (HTTPS 6443)
              ┌───────────────┼──────────────────┐
              ▼               ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   Worker Node 1  │ │   Worker Node 2  │ │   Worker Node 3  │
│  ┌────────────┐  │ │  ┌────────────┐  │ │  ┌────────────┐  │
│  │  kubelet   │  │ │  │  kubelet   │  │ │  │  kubelet   │  │
│  ├────────────┤  │ │  ├────────────┤  │ │  ├────────────┤  │
│  │ kube-proxy │  │ │  │ kube-proxy │  │ │  │ kube-proxy │  │
│  ├────────────┤  │ │  ├────────────┤  │ │  ├────────────┤  │
│  │ containerd │  │ │  │ containerd │  │ │  │ containerd │  │
│  ├────────────┤  │ │  ├────────────┤  │ │  ├────────────┤  │
│  │ Pod 1 Pod 2│  │ │  │ Pod 3 Pod 4│  │ │  │ Pod 5 Pod 6│  │
│  └────────────┘  │ │  └────────────┘  │ │  └────────────┘  │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## 2. Control Plane Components

### kube-apiserver — The Heart of Kubernetes

The API server is the **only component that talks to etcd** and the **central point of communication** for all other components.

```
ALL Kubernetes operations go through kube-apiserver:

kubectl apply -f deployment.yaml
    │ HTTPS/REST
    ▼
kube-apiserver
    │
    ├── Authentication: Who are you?
    │   (X.509 cert, Bearer token, OIDC, ServiceAccount)
    │
    ├── Authorization: Can you do this?
    │   (RBAC, ABAC, Node, Webhook)
    │
    ├── Admission Control: Should this be allowed?
    │   (ValidatingWebhook, MutatingWebhook, built-in plugins)
    │   MutatingAdmission: can modify request (inject sidecars!)
    │   ValidatingAdmission: can reject request (enforce policy)
    │
    ├── Validate request against API schema
    │
    └── Write to etcd
        │
        └── Notify watchers (controllers, scheduler, kubelet)
            via Watch mechanism (long-poll + event streaming)
```

**API Server Request Flow:**
```
1. kubectl sends: PATCH /apis/apps/v1/namespaces/default/deployments/myapp
2. TLS termination
3. Authentication
4. Authorization (RBAC check)
5. Admission Webhooks (OPA Gatekeeper, etc.)
6. Schema validation
7. Write to etcd
8. Return response to kubectl
9. Broadcast change to watchers (controller manager, scheduler, kubelets)
```

### etcd — The Database

etcd is a **distributed key-value store** using the Raft consensus algorithm. It's the sole persistent store for all cluster state.

```
etcd Architecture:

  etcd Node 1 (Leader)
      │ Raft protocol
      ├── etcd Node 2 (Follower)
      └── etcd Node 3 (Follower)

Write path:
  API server → Leader → Raft log (majority must ack) → Apply

Read path:
  API server → Any node (linearizable reads go to leader)

Data stored in etcd:
  /registry/namespaces/default
  /registry/deployments/default/myapp
  /registry/pods/default/myapp-abc123
  /registry/services/default/myapp
  /registry/configmaps/default/app-config
  /registry/secrets/default/app-secret

# etcdctl commands (for debugging)
ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    get /registry/pods --prefix --keys-only | head -20

# Check cluster health
etcdctl endpoint health
etcdctl endpoint status

# Backup etcd (CRITICAL for disaster recovery)
ETCDCTL_API=3 etcdctl snapshot save backup.db
ETCDCTL_API=3 etcdctl snapshot status backup.db
```

**etcd HA Requirements:**
- Always run odd number: 3, 5, or 7 nodes
- Quorum = (n+1)/2 must be up
- 3 nodes: can lose 1 (2/3 quorum)
- 5 nodes: can lose 2 (3/5 quorum)
- Performance: etcd needs low-latency disk (SSD) — high disk latency causes leader elections!

### kube-scheduler

The scheduler assigns Pods to Nodes by:

```
New Pod created (no NodeName set)
         │
         ▼
Scheduler's watch loop detects unscheduled Pod
         │
         ▼
FILTERING (Predicates): Which nodes CAN run this pod?
  ├── NodeSelector / NodeAffinity match?
  ├── TaintToleration: Does pod tolerate node taints?
  ├── Resources: Does node have enough CPU/memory?
  ├── PodAffinity/Anti-Affinity satisfied?
  ├── HostPort conflicts?
  └── NodeReady? Disk pressure? Memory pressure?
         │
         ▼
SCORING (Priorities): Which node BEST fits?
  ├── Least requested resources (spread load)
  ├── Node affinity preferred weight
  ├── Pod affinity/anti-affinity preferred
  ├── Image locality (image already cached on node)
  └── Inter-pod spreading (spread across zones/nodes)
         │
         ▼
Highest score node selected
         │
         ▼
Scheduler writes NodeName to Pod spec in etcd
         │
         ▼
kubelet on selected node detects Pod via Watch
Starts containers
```

### kube-controller-manager

Contains ~50 controllers, each watching specific resources and reconciling state:

```
Controller Manager = Multiple controllers in one process

Key Controllers:
┌─────────────────────────────────────────────────────────┐
│ ReplicaSet Controller                                    │
│  Watch: ReplicaSets + Pods                              │
│  If current replicas < desired: create Pod              │
│  If current replicas > desired: delete Pod              │
├─────────────────────────────────────────────────────────┤
│ Deployment Controller                                   │
│  Watch: Deployments                                     │
│  Creates/manages ReplicaSets for rolling updates        │
│  Old RS: scale down, New RS: scale up                   │
├─────────────────────────────────────────────────────────┤
│ Node Controller                                         │
│  Watch: Nodes                                           │
│  Marks nodes NotReady if no heartbeat for 40s           │
│  Evicts pods from failed nodes after 5min               │
├─────────────────────────────────────────────────────────┤
│ Endpoint Controller                                     │
│  Watch: Services + Pods                                 │
│  Updates Endpoints object with ready Pod IPs            │
├─────────────────────────────────────────────────────────┤
│ ServiceAccount Controller                               │
│  Creates default ServiceAccount per namespace           │
│  Creates token Secret for ServiceAccount                │
├─────────────────────────────────────────────────────────┤
│ Job Controller, CronJob Controller                      │
│ PersistentVolume Controller, Namespace Controller       │
└─────────────────────────────────────────────────────────┘

Reconciliation Loop Pattern (all controllers):
for {
    desired := getDesiredState()       // from etcd
    actual := getActualState()         // from API server / cloud
    diff := desired - actual
    if diff != 0:
        applyChanges(diff)             // Create/delete/update resources
    sleep(resyncPeriod)
}
```

---

## 3. Worker Node Components

### kubelet

The kubelet is the **primary node agent** that runs on each worker node. It manages Pod lifecycle.

```
kubelet responsibilities:
  1. Registers node with API server
  2. Watches for Pod specs assigned to its node
  3. Starts/stops containers via CRI (Container Runtime Interface)
  4. Reports Pod and Node status back to API server
  5. Runs health checks (liveness, readiness, startup probes)
  6. Manages volumes (mounts, unmounts)
  7. Pulls container images

Pod lifecycle via kubelet:
  API server: "Run nginx pod on this node"
         │
         ▼
  kubelet receives pod spec via Watch
         │
         ▼
  kubelet calls CRI (containerd/CRI-O):
    - RunPodSandbox: create network namespace, infra/pause container
    - PullImage: if not cached
    - CreateContainer: create container from spec
    - StartContainer: start the container
         │
         ▼
  kubelet runs probes:
    - startupProbe: wait for app to start
    - livenessProbe: restart if unhealthy
    - readinessProbe: add/remove from Service endpoints
         │
         ▼
  kubelet reports status to API server every ~10s

# kubelet configuration
cat /var/lib/kubelet/config.yaml
systemctl status kubelet
journalctl -u kubelet -f
```

### The Pause Container (Infra Container)

```
Every Pod has a "pause" container:
  - First container started in Pod
  - Holds the shared network namespace for all Pod containers
  - PID 1 in Pod's PID namespace (reaps zombie processes)
  - Very small image (~300KB)

Pod with 2 containers:
  pause container (holds namespaces)
    ├── Container 1 (app) → joins pause's network namespace
    └── Container 2 (sidecar) → joins pause's network namespace

Both containers share:
  - Same network namespace (same IP, same localhost)
  - Same IPC namespace (can share memory)
  - Optional: same PID namespace

This is why sidecar can reach main app via localhost!
```

### kube-proxy

kube-proxy implements **Service virtual IPs** by maintaining iptables/ipvs rules:

```
Service ClusterIP (10.96.0.1) is a virtual IP
No actual process listens on it!

kube-proxy maintains iptables rules to implement it:

iptables (default mode):
  -A KUBE-SERVICES -d 10.96.0.1/32 -p tcp --dport 80 \
    -j KUBE-SVC-ABCDEF12345

  # Load balancing across 3 pods (statistic/probability chains):
  -A KUBE-SVC-ABCDEF12345 -m statistic --mode random \
    --probability 0.33 -j KUBE-SEP-POD1
  -A KUBE-SVC-ABCDEF12345 -m statistic --mode random \
    --probability 0.50 -j KUBE-SEP-POD2
  -A KUBE-SVC-ABCDEF12345 -j KUBE-SEP-POD3

  -A KUBE-SEP-POD1 -j DNAT --to-destination 10.244.1.5:80
  -A KUBE-SEP-POD2 -j DNAT --to-destination 10.244.2.3:80
  -A KUBE-SEP-POD3 -j DNAT --to-destination 10.244.3.7:80

IPVS mode (better for large clusters, 1000+ services):
  Uses kernel IPVS (IP Virtual Server) for O(1) lookup
  vs iptables O(n) sequential rule traversal
  
  ipvsadm -L -n
  # TCP  10.96.0.1:80 rr (round-robin)
  #   → 10.244.1.5:80    1
  #   → 10.244.2.3:80    1
  #   → 10.244.3.7:80    1
```

### Container Runtime Interface (CRI)

```
kubelet → CRI (gRPC) → containerd → runc → container

CRI Protocol (gRPC):
  RuntimeService:
    - RunPodSandbox / StopPodSandbox / RemovePodSandbox
    - CreateContainer / StartContainer / StopContainer
    - ExecSync / Exec / Attach / PortForward
    - ContainerStats / ListContainerStats
  ImageService:
    - PullImage / RemoveImage / ListImages

# Verify CRI
crictl ps                         # List running containers (bypasses Docker)
crictl logs container_id          # Container logs
crictl exec -it container_id sh   # Execute in container
crictl images                     # List images
crictl stats                      # Resource stats
```

---

## 4. Kubernetes Networking

### Pod Networking (CNI)

**Every Pod gets a unique, routable IP address** across the entire cluster. This is the fundamental K8s networking requirement.

```
CNI (Container Network Interface) handles Pod networking:

When Pod is created:
  kubelet → calls CNI plugin → sets up Pod networking

CNI plugins:
  - Calico: BGP-based routing, NetworkPolicy, eBPF
  - Flannel: Simple overlay (VXLAN), easy to understand
  - Cilium: eBPF-based, best performance, L7 policy, service mesh
  - WeaveNet: Mesh overlay
  - AWS VPC CNI: Native VPC IPs for pods (AWS)
  - Azure CNI: Native VNet IPs (Azure)
  - Antrea: VMware, uses OVS

Flannel VXLAN mode:
Pod on Node 1 (10.244.1.5) → Pod on Node 2 (10.244.2.3)
    │
    ▼
Container eth0 (10.244.1.5)
    │ veth pair
    ▼
flannel.1 (VXLAN device)
    │ encapsulate in UDP packet (destination: Node2 IP)
    ▼
eth0 on Node 1 (10.0.0.1)
    │ UDP packet to 10.0.0.2:8472
    ▼
eth0 on Node 2 (10.0.0.2)
    │ decapsulate VXLAN
    ▼
flannel.1 on Node 2
    │ route to 10.244.2.3
    ▼
Container eth0 (10.244.2.3)
```

### Service Networking

```
Services: Stable virtual IPs for dynamic Pods

Types:
1. ClusterIP (default) - Internal only, virtual IP
2. NodePort - External access via each node's IP:port
3. LoadBalancer - External LB (cloud provider)
4. ExternalName - DNS CNAME mapping
5. Headless (clusterIP: None) - Direct Pod IPs via DNS

Service Discovery:
  Every Pod gets env vars:
    MYAPP_SERVICE_HOST=10.96.0.100
    MYAPP_SERVICE_PORT=80
  
  And DNS (via CoreDNS):
    myapp.default.svc.cluster.local → 10.96.0.100
    myapp.default → 10.96.0.100
    myapp → 10.96.0.100 (within same namespace)

# Full DNS FQDN format:
# <service>.<namespace>.svc.<cluster-domain>
# nginx.production.svc.cluster.local
```

### Complete Service Traffic Flow

```
External User → http://api.example.com/users

Step 1: DNS resolves to LoadBalancer IP (e.g., 52.1.2.3)
Step 2: Request hits cloud Load Balancer (AWS ALB/ELB)
Step 3: LB routes to one of the healthy nodes (NodePort: 30080)
Step 4: Node receives packet on port 30080
Step 5: iptables/IPVS DNAT: 30080 → Service ClusterIP:80
Step 6: iptables/IPVS selects a Pod IP (load balances)
Step 7: DNAT: → 10.244.x.x:8080 (actual Pod IP)
Step 8: Packet routed to Pod (same or different node via CNI)
Step 9: Application processes request
Step 10: Response flows back through reverse NAT
```

### Ingress Controller

```
Ingress: HTTP/HTTPS routing rules at L7

Without Ingress:
  api.example.com → LoadBalancer LB 1 → Service A
  web.example.com → LoadBalancer LB 2 → Service B
  (cost: 2 cloud LBs = expensive!)

With Ingress:
  api.example.com → LoadBalancer LB 1 (single LB!)
  web.example.com ↗     │
                         ▼
                  Ingress Controller (nginx/traefik/haproxy)
                  reads Ingress rules
                         │
                  ├── /api/* → Service A
                  └── /web/* → Service B

Ingress Controller flow:
  1. Ingress Controller Pod runs in cluster
  2. Watches Ingress objects via K8s API Watch
  3. Updates nginx.conf / traefik config when Ingress changes
  4. Routes L7 traffic to backend Services

# Example Ingress resource
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1/
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 8080
      - path: /v2/
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 8080
```

### NetworkPolicy

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}    # Selects ALL pods in namespace
  policyTypes:
  - Ingress

---
# Allow only frontend to reach backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:
        matchLabels:
          name: monitoring   # Allow Prometheus from monitoring namespace
    ports:
    - protocol: TCP
      port: 8080
```

---

## 5. Workload Resources

### Pod Spec Deep Dive

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: v1.2.3
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
spec:
  # Scheduling
  nodeSelector:
    kubernetes.io/arch: amd64
  
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values: [compute]
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname  # Spread across nodes
  
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  
  # Init containers (run to completion before main containers)
  initContainers:
  - name: db-migration
    image: myapp:v1.2.3
    command: ["python", "manage.py", "migrate"]
    envFrom:
    - secretRef:
        name: db-secret
  
  containers:
  - name: myapp
    image: myregistry.io/myapp:v1.2.3@sha256:abc123...
    imagePullPolicy: IfNotPresent
    
    ports:
    - name: http
      containerPort: 8080
      protocol: TCP
    - name: metrics
      containerPort: 9090
    
    env:
    - name: APP_ENV
      value: production
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName   # Downward API
    
    envFrom:
    - configMapRef:
        name: app-config
    
    resources:
      requests:
        cpu: "100m"         # 0.1 CPU - scheduling guarantee
        memory: "256Mi"     # 256MB RAM - scheduling guarantee
      limits:
        cpu: "500m"         # 0.5 CPU - hard cap
        memory: "512Mi"     # 512MB RAM - OOM if exceeded
    
    # Probes
    startupProbe:
      httpGet:
        path: /health/startup
        port: 8080
      failureThreshold: 30    # Wait up to 5 minutes (30 * 10s)
      periodSeconds: 10
    
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 0   # startupProbe guards this
      periodSeconds: 10
      failureThreshold: 3      # Restart after 3 failures
      successThreshold: 1
    
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 3      # Remove from Service if fails
    
    volumeMounts:
    - name: config
      mountPath: /etc/app
      readOnly: true
    - name: data
      mountPath: /data
    - name: tmp
      mountPath: /tmp
    
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 10"]  # Drain connections
    
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      runAsGroup: 1000
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
  
  # Sidecar container
  - name: log-shipper
    image: fluent/fluent-bit:2.1
    resources:
      requests:
        cpu: "10m"
        memory: "32Mi"
      limits:
        cpu: "50m"
        memory: "64Mi"
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  
  volumes:
  - name: config
    configMap:
      name: app-config
  - name: data
    persistentVolumeClaim:
      claimName: myapp-data
  - name: tmp
    emptyDir:
      medium: Memory   # tmpfs
      sizeLimit: 100Mi
  - name: logs
    emptyDir: {}
  
  # Pod-level security
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
    fsGroup: 1000          # Volume files owned by this GID
  
  terminationGracePeriodSeconds: 60   # Give app 60s to shutdown
  restartPolicy: Always
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false  # Explicit: don't mount if not needed
  
  # Image pull from private registry
  imagePullSecrets:
  - name: registry-credentials
  
  # DNS config
  dnsConfig:
    options:
    - name: ndots
      value: "1"   # Reduce DNS lookups for external domains
```

### Deployment — Rolling Update Internals

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Can have 12 pods temporarily
      maxUnavailable: 1  # At most 1 pod unavailable at a time
      # So: scale up 2 new, then scale down 1 old, repeat
  
  selector:
    matchLabels:
      app: myapp
  
  minReadySeconds: 30   # Wait 30s after pod ready before proceeding
  
  revisionHistoryLimit: 5   # Keep 5 old ReplicaSets for rollback
  
  progressDeadlineSeconds: 600  # Fail if not done in 10min
  
  template:
    metadata:
      labels:
        app: myapp
    spec:
      # ... pod spec ...

# Rolling update flow:
# Current: RS-v1 with 10 pods
# New: RS-v2 with 0 pods
#
# Step 1: RS-v2 scale to 2 (maxSurge)  → 10+2 = 12 pods
# Step 2: RS-v1 scale down by 1        → 11 pods (wait for RS-v2 pods ready)
# Step 3: RS-v2 scale to 3             → still 12 max
# Step 4: RS-v1 scale down              → repeat until done

# Rollback:
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=3
kubectl rollout history deployment/myapp
kubectl rollout status deployment/myapp
kubectl rollout pause deployment/myapp    # Pause update
kubectl rollout resume deployment/myapp  # Resume
```

### StatefulSet

```yaml
# For stateful apps: databases, Kafka, ZooKeeper
# Guarantees: stable network identity, ordered deployment, stable storage

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless   # Required: headless service for DNS
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  
  # Ordered deployment: postgres-0, postgres-1, postgres-2
  # Ordered deletion: postgres-2, postgres-1, postgres-0
  podManagementPolicy: OrderedReady  # (or Parallel)
  
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0   # Only update pods with ordinal >= partition
                     # Use partition: 2 to only update postgres-2 (canary!)
  
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        # postgres-0.postgres-headless.default.svc.cluster.local
        # → Always resolves to same pod's IP regardless of restarts!
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  
  # Each pod gets its own PVC (not shared)
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
# Result: postgres-0-data, postgres-1-data, postgres-2-data PVCs
# PVCs are NOT deleted when StatefulSet deleted! (protect data)
```

### DaemonSet

```yaml
# Runs one Pod per node (or subset of nodes)
# Use: log collectors, monitoring agents, CNI plugins, security agents

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # Update 1 node at a time
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true        # Access host network metrics
      hostPID: true            # Access host processes
      tolerations:
      - operator: Exists       # Run on ALL nodes including masters
        effect: NoSchedule
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        securityContext:
          privileged: true   # Need host access
        volumeMounts:
        - mountPath: /host/sys
          mountPropagation: HostToContainer
          name: sys
          readOnly: true
        - mountPath: /host/root
          mountPropagation: HostToContainer
          name: root
          readOnly: true
      volumes:
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

---

## 6. Services and Ingress

### Service Types

```yaml
# 1. ClusterIP (default)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80           # Port the Service listens on
    targetPort: 8080   # Port the Pod listens on
    protocol: TCP
  type: ClusterIP      # Virtual IP, internal only

---
# 2. NodePort
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080   # External port on EVERY node (30000-32767)
  # Access: http://<any-node-ip>:30080

---
# 3. LoadBalancer (cloud)
spec:
  type: LoadBalancer
  # Cloud controller creates external LB
  # AWS: creates ELB/ALB/NLB
  # GCP: creates Cloud Load Balancer
  # Azure: creates Azure LB
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
  ports:
  - port: 80
    targetPort: 8080

---
# 4. Headless Service (for StatefulSets, direct Pod DNS)
spec:
  clusterIP: None     # No virtual IP
  selector:
    app: postgres
  ports:
  - port: 5432
# DNS: postgres-0.postgres.default.svc.cluster.local → Pod IP directly
```

---

## 7. Storage

### Persistent Volumes

```
PV lifecycle:
  Provision (static/dynamic)
       │
  Bind to PVC
       │
  Mount to Pod
       │
  Release (PVC deleted)
       │
  Reclaim (Delete/Retain/Recycle)

Storage Classes enable dynamic provisioning:
kubectl get sc
# NAME                 PROVISIONER           RECLAIMPOLICY
# gp2 (default)        ebs.csi.aws.com       Delete
# gp3                  ebs.csi.aws.com       Delete
# efs                  efs.csi.aws.com       Retain
```

```yaml
# StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # Provision in same AZ as pod

---
# PVC (request storage)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
spec:
  accessModes:
    - ReadWriteOnce    # Only one node at a time
    # ReadWriteMany    # Multiple nodes simultaneously (NFS, EFS)
    # ReadOnlyMany     # Multiple readers
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 50Gi
```

---

## 8. Configuration and Secrets

### ConfigMap and Secret

```yaml
# ConfigMap - non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      pool_size: 10

---
# Secret - sensitive data (base64 encoded, NOT encrypted by default!)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # echo -n "mysecretpassword" | base64
  password: bXlzZWNyZXRwYXNzd29yZA==
  
stringData:  # Plain text, auto-encoded
  username: dbuser

# IMPORTANT: Secrets in etcd are base64 encoded, NOT encrypted
# For real encryption: enable EncryptionConfiguration in API server
# Or use external secret management: External Secrets Operator + Vault/AWS SM
```

### External Secrets Operator

```yaml
# Sync secrets from AWS Secrets Manager to Kubernetes
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: prod/myapp/db
      property: password
```

---

## 9. RBAC and Security

### RBAC Model

```
RBAC building blocks:

Role/ClusterRole: WHAT can be done
  - Resources (pods, deployments, secrets, *)
  - Verbs (get, list, watch, create, update, patch, delete, *)

RoleBinding/ClusterRoleBinding: WHO can do it
  - Subjects: User, Group, ServiceAccount
  - References a Role/ClusterRole

ServiceAccount: Identity for Pods

Role = namespace-scoped
ClusterRole = cluster-wide

# Example: Allow myapp to read its own secrets
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["myapp-secret"]  # Only this specific secret
  verbs: ["get"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: myapp-role

# Debug RBAC
kubectl auth can-i create pods --as=system:serviceaccount:production:myapp-sa
kubectl auth can-i get secrets --as=system:serviceaccount:production:myapp-sa
kubectl auth can-i "*" "*" --as=cluster-admin
```

---

## 10. Scaling and Scheduling

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # Scale when avg CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 400Mi       # Scale when avg memory > 400Mi
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"      # Custom metric from Prometheus
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # React quickly to traffic
      policies:
      - type: Percent
        value: 100                # Can double replicas per 60s
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Be conservative scaling down
      policies:
      - type: Pods
        value: 2                  # Max 2 pods removed per 60s
        periodSeconds: 60
```

### Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"   # Auto restarts pods with new recommendations
    # "Off" = only show recommendations (no changes)
    # "Initial" = only set on pod creation
  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      minAllowed:
        cpu: 50m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 4Gi
```

### Pod Disruption Budget (PDB)

```yaml
# Ensure availability during voluntary disruptions (drains, upgrades)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2         # Always keep at least 2 pods
  # OR:
  # maxUnavailable: 1     # At most 1 pod can be unavailable
  selector:
    matchLabels:
      app: myapp

# When kubectl drain node:
# Kubernetes checks PDB before evicting pods
# Eviction blocked if it would violate PDB
```

---

## 11. Service Mesh — Istio

### What is a Service Mesh?

A service mesh adds a **sidecar proxy** (Envoy) to every Pod, providing:
- **mTLS**: Automatic mutual TLS between all services
- **Traffic management**: Circuit breaking, retries, timeouts, canary
- **Observability**: Distributed tracing, metrics per service pair
- **Policy**: Authorization policies, rate limiting

```
Without service mesh:
  Service A → TCP → Service B
  No encryption, no observability at connection level

With Istio:
  Service A → [Envoy sidecar] → [Envoy sidecar] → Service B
              mTLS automatically     decrypts here
              traces request         applies policy

Istio Components:
  - istiod (control plane): Pilot, Citadel, Galley merged
    - Pilot: configures Envoy sidecars from Kubernetes resources
    - Citadel: certificate management for mTLS
  - istio-ingressgateway: Ingress using Envoy
  - Envoy sidecars: data plane (injected automatically)

# Enable automatic sidecar injection
kubectl label namespace production istio-injection=enabled
```

```yaml
# VirtualService: Traffic routing rules
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - match:
    - headers:
        x-user-group:
          exact: beta-testers
    route:
    - destination:
        host: myapp
        subset: v2     # Beta users → new version
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90       # 90% → stable version
    - destination:
        host: myapp
        subset: v2
      weight: 10       # 10% canary
    timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 3s
      retryOn: "5xx,reset,connect-failure"
```

---

## 12. Production Troubleshooting

### Debugging Commands

```bash
# ── Pod issues ───────────────────────────────────────────────
# Pod status overview
kubectl get pods -A -o wide              # All pods with node
kubectl get pods --field-selector=status.phase!=Running -A  # Non-running

# Describe pod (events are KEY)
kubectl describe pod myapp-abc123 -n production

# Common Status → Cause:
# Pending:
#   - Insufficient resources (CPU/memory)
#   - No nodes matching NodeSelector
#   - PVC not bound
#   - Image pull backoff
# CrashLoopBackOff:
#   - App crashing (check logs!)
#   - OOM killed (exit 137)
#   - Readiness probe failing too fast
# OOMKilled:
#   - Memory limit too low
# ImagePullBackOff:
#   - Image doesn't exist
#   - Registry auth failing (check imagePullSecrets)

# Logs
kubectl logs myapp-abc123 -n production
kubectl logs myapp-abc123 -n production -c sidecar     # Specific container
kubectl logs myapp-abc123 -n production --previous     # Crashed container logs
kubectl logs -l app=myapp -n production --tail=100     # All pods with label
kubectl logs myapp-abc123 -n production -f --since=30m # Follow + time filter

# Execute commands in pod
kubectl exec -it myapp-abc123 -n production -- bash
kubectl exec -it myapp-abc123 -n production -c sidecar -- sh

# Copy files
kubectl cp myapp-abc123:/var/log/app.log ./app.log -n production
kubectl cp ./config.yaml myapp-abc123:/etc/app/config.yaml

# Port forwarding (bypass Service, connect direct to pod)
kubectl port-forward pod/myapp-abc123 8080:8080 -n production
kubectl port-forward service/myapp 8080:80 -n production

# ── Node issues ──────────────────────────────────────────────
kubectl get nodes
kubectl describe node node-01
kubectl top nodes                        # CPU/memory usage
kubectl get events --sort-by='.lastTimestamp' -A

# Drain a node (for maintenance)
kubectl drain node-01 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon node-01   # Mark schedulable again

# ── Deployment issues ────────────────────────────────────────
kubectl rollout status deployment/myapp -n production
kubectl rollout history deployment/myapp -n production
kubectl rollout undo deployment/myapp -n production

# ── Service issues ───────────────────────────────────────────
# Check Service endpoints (pods must be Ready)
kubectl get endpoints myapp -n production
# If endpoints empty: check pod labels match service selector

kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
    curl http://myapp.production.svc.cluster.local:80

# DNS debugging
kubectl run -it --rm debug --image=busybox --restart=Never -- \
    nslookup myapp.production.svc.cluster.local

# ── Resource issues ──────────────────────────────────────────
kubectl top pods -n production --sort-by=memory
kubectl describe nodes | grep -A 5 "Allocated resources"

# ── etcd issues ──────────────────────────────────────────────
etcdctl endpoint health --cluster
etcdctl endpoint status --cluster -w table
# High DB size → compact + defrag
etcdctl compact $(etcdctl endpoint status --write-out="json" | jq '.[] | .Status.header.revision')
etcdctl defrag --cluster
```

### Common Kubernetes Issues

```
Issue: Pod stuck in Pending
Diagnosis:
  kubectl describe pod <name> → Events section
  "0/3 nodes are available: 3 Insufficient memory"
  → Resource requests too high
  kubectl describe nodes → Allocated resources

Fix options:
  - Reduce resource requests
  - Add more nodes (cluster autoscaler)
  - Use VPA to right-size

Issue: Deployment stuck (no progress)
Diagnosis:
  kubectl rollout status deployment/myapp
  # Waiting for rollout to finish: 3 out of 5 new replicas updated
  kubectl get pods → some pods in CrashLoopBackOff
  kubectl logs <crashing pod>

Fix:
  kubectl rollout undo deployment/myapp
  # Fix the issue
  # Redeploy

Issue: Service not routing traffic
Diagnosis:
  kubectl get endpoints myapp
  # If empty: pods might not be Ready
  kubectl get pods -l app=myapp
  # If pods Ready but still empty: check label selector
  kubectl describe service myapp
  # Check: Selector matches pod labels?
  
Issue: Certificate expired (TLS)
Diagnosis:
  kubectl get certificate -n cert-manager
  kubectl describe certificate -n production
  openssl x509 -in tls.crt -noout -dates

Fix:
  kubectl delete secret tls-secret   # Force re-issue
  # cert-manager will auto-renew if within renewal window (30 days before expiry)
```

---

## 13. Interview Questions — Phase 5

**Q: Explain how Kubernetes handles a Pod failure and ensures availability.**

When a Pod fails (crashes, OOM killed, node failure):

1. **kubelet** detects the container has exited → checks restartPolicy
2. For `restartPolicy: Always` → kubelet restarts container with exponential backoff (10s, 20s, 40s... up to 5min = CrashLoopBackOff)
3. If the Node itself fails → Node Controller marks it NotReady after 40s
4. After 5 minutes of NotReady, Node Controller evicts Pods from that node
5. **ReplicaSet Controller** detects desired replicas ≠ actual running pods
6. Creates replacement Pods
7. **Scheduler** assigns new Pods to healthy nodes
8. **Pod Disruption Budgets** ensure minimum availability is maintained during voluntary disruptions

**Q: What is the difference between liveness, readiness, and startup probes?**

- **Startup probe**: Gates liveness/readiness probes. Used for slow-starting apps. Until startup probe succeeds, no other probes run. Prevents CrashLoopBackOff during initialization. Use for Java apps that take 2+ minutes to start.

- **Liveness probe**: "Is the container alive?" If fails → restart container. Use for detecting deadlocks or hung processes that haven't crashed but are non-functional.

- **Readiness probe**: "Is the container ready to serve traffic?" If fails → remove from Service Endpoints (stop sending traffic) but don't restart. Use for apps that need to load data before serving, or are temporarily overloaded.

Common mistake: Making liveness probe too aggressive (causes restart storms) or making readiness probe too lenient (sends traffic to unhealthy pods).

**Q: How does Kubernetes Service load balancing work? What's the difference between kube-proxy modes?**

Services implement virtual IPs. No process actually listens on the ClusterIP — it's a forwarding construct.

**iptables mode** (default): kube-proxy writes iptables rules for each Service. Traffic is redirected via DNAT. Problem: Rules are processed linearly (O(n) for n endpoints). With 10,000 services, each packet traverses thousands of rules. Updates require rewriting the entire ruleset.

**IPVS mode**: Uses kernel IPVS (IP Virtual Server). O(1) hash-based lookup regardless of number of services. Supports multiple load balancing algorithms (round-robin, least connection, source hash). Preferred for large clusters (100+ services).

**eBPF (Cilium)**: Bypasses iptables entirely. Processes packets at kernel level using eBPF programs. Lowest overhead, fastest, no conntrack overhead.

**Q: How would you design a Kubernetes cluster for a production financial application?**

Control plane:
- 3 or 5 etcd nodes on dedicated SSD (low-latency disk critical)
- 3 API server nodes behind internal LB
- etcd backups every 30 minutes to S3
- etcd encryption at rest (EncryptionConfiguration)

Worker nodes:
- Separate node pools: compute, memory-optimized, spot
- PodDisruptionBudgets for all critical workloads
- Multi-AZ deployment (pod anti-affinity + topology spread)
- Cluster Autoscaler for dynamic scaling

Security:
- RBAC with least-privilege ServiceAccounts
- NetworkPolicies: default-deny, explicit allow
- OPA Gatekeeper for admission control policies
- Secrets via External Secrets + Vault (not plain K8s secrets)
- Pod Security Standards: Restricted profile
- Container image signing (cosign)
- Regular vulnerability scanning

Observability:
- Prometheus + Grafana for metrics
- Jaeger/Tempo for distributed tracing
- Centralized logging (Loki/ELK)
- Alerting on SLOs (not just saturation metrics)

---

*End of Phase 5 — Kubernetes*
# Phase 6: Infrastructure as Code (IaC)

---

## 6.1 What is Infrastructure as Code?

**Infrastructure as Code (IaC)** is the practice of managing and provisioning computing infrastructure through machine-readable configuration files rather than through manual processes or interactive configuration tools.

### Why IaC Exists

**Before IaC — The Dark Ages:**
- Servers configured manually via SSH → "snowflake servers" (unique, irreproducible)
- No version history for infrastructure changes
- "It works on my machine" extended to "it works in staging but not prod"
- Disaster recovery = rebuild from memory + tribal knowledge
- Scaling = ops team manually spinning up servers for hours/days

**Problems IaC Solves:**

| Problem | IaC Solution |
|---------|-------------|
| Manual configuration drift | Declarative state enforced on every run |
| No audit trail | Git history of every infrastructure change |
| Slow provisioning | Automated, repeatable in minutes |
| Environment inconsistency | Same code deploys identical dev/staging/prod |
| Knowledge silos | Code is documentation |
| Disaster recovery | Rebuild entire infra from code in minutes |

### IaC Approaches

```
IMPERATIVE (HOW)                    DECLARATIVE (WHAT)
─────────────────                   ──────────────────
"Create a VM"                       "I want a VM to exist"
"Install nginx on it"               "I want nginx installed"
"Open port 80"                      "I want port 80 open"

Ansible (procedural mode)           Terraform, CloudFormation
Scripts (bash, Python)              Pulumi (can be declarative)

You define the steps                You define the end state
Order matters                       Engine figures out order
Idempotency is YOUR problem         Engine handles idempotency
```

---

## 6.2 Terraform — Deep Internals

### What is Terraform?

Terraform is an open-source IaC tool by HashiCorp that lets you define cloud and on-premises infrastructure in human-readable HCL (HashiCorp Configuration Language) and manages the full lifecycle of that infrastructure.

**Key Differentiator:** Terraform is **cloud-agnostic** — one tool for AWS, Azure, GCP, Kubernetes, Datadog, GitHub, etc.

### Terraform Internal Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        TERRAFORM CLI                             │
│                                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │  Parser   │  │  Loader  │  │ Evaluator│  │ Plan/Apply    │  │
│  │  (HCL)   │  │ (modules)│  │ (exprs)  │  │ Engine        │  │
│  └──────────┘  └──────────┘  └──────────┘  └───────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    CORE ENGINE                            │   │
│  │  Resource Graph Builder → Dependency Resolver            │   │
│  │  Diff Engine (desired state vs current state)            │   │
│  │  Walk Algorithm (parallel execution of independent nodes) │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                  PROVIDER PLUGIN SYSTEM                    │  │
│  │  Provider = gRPC server implementing ResourceCRUD          │  │
│  │  AWS Provider │ Azure Provider │ GCP Provider │ K8s ...    │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                    gRPC (tfplugin protocol)
                              │
              ┌───────────────┴────────────────┐
              │        PROVIDER BINARY          │
              │   (separate process, auto-DL)   │
              │  Translates HCL → API calls     │
              │  aws_instance → EC2 Create API  │
              └───────────────┬────────────────┘
                              │
                    HTTPS/REST/SDK calls
                              │
              ┌───────────────┴────────────────┐
              │         CLOUD PROVIDER API      │
              │  AWS API / Azure ARM / GCP API  │
              └────────────────────────────────┘
```

### HCL Language Deep Dive

```hcl
# terraform/main.tf

# BLOCK TYPES: terraform, provider, resource, data, variable, 
#              output, locals, module

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"    # registry.terraform.io/hashicorp/aws
      version = "~> 5.0"          # >= 5.0.0, < 6.0.0
    }
  }

  # Remote State Backend
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # Distributed locking
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
    }
  }
}

# VARIABLES — inputs to the module
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
  
  validation {
    condition     = contains(["us-east-1", "us-west-2", "eu-west-1"], var.aws_region)
    error_message = "Must be a valid production region."
  }
}

variable "environment" {
  type = string
}

variable "vpc_config" {
  type = object({
    cidr_block           = string
    enable_dns_hostnames = bool
    availability_zones   = list(string)
  })
}

# LOCALS — computed values within module
locals {
  name_prefix = "${var.environment}-${var.aws_region}"
  common_tags = {
    Environment = var.environment
    Region      = var.aws_region
    CreatedAt   = timestamp()
  }
  
  # Subnet CIDRs computed from VPC CIDR
  public_subnet_cidrs  = [for i, az in var.vpc_config.availability_zones : 
                           cidrsubnet(var.vpc_config.cidr_block, 8, i)]
  private_subnet_cidrs = [for i, az in var.vpc_config.availability_zones : 
                           cidrsubnet(var.vpc_config.cidr_block, 8, i + 10)]
}

# RESOURCE — actual infrastructure
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_config.cidr_block
  enable_dns_hostnames = var.vpc_config.enable_dns_hostnames
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

# DATA SOURCE — read existing infrastructure
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# DYNAMIC BLOCKS — generate repeated nested blocks
resource "aws_security_group" "web" {
  name   = "${local.name_prefix}-web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = [80, 443]
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}

# COUNT vs FOR_EACH
# Count — simple repetition (index-based, fragile on delete)
resource "aws_subnet" "public_count" {
  count             = length(local.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnet_cidrs[count.index]
  availability_zone = var.vpc_config.availability_zones[count.index]
}

# For_each — map-based (stable on delete, preferred)
resource "aws_subnet" "public" {
  for_each = tomap({
    for i, az in var.vpc_config.availability_zones :
    az => local.public_subnet_cidrs[i]
  })
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value
  availability_zone = each.key
  
  tags = { Name = "${local.name_prefix}-public-${each.key}" }
}

# OUTPUTS — expose values for other modules/humans
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = [for subnet in aws_subnet.public : subnet.id]
}
```

### Terraform State — The Brain

```
STATE FILE (terraform.tfstate) — JSON format

{
  "version": 4,
  "terraform_version": "1.5.7",
  "serial": 42,              ← increments on every change
  "lineage": "uuid-...",     ← unique ID for this state
  "outputs": { ... },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "vpc-0abc123",
            "cidr_block": "10.0.0.0/16",
            "arn": "arn:aws:ec2:us-east-1:123456789:vpc/vpc-0abc123",
            ...ALL RESOURCE ATTRIBUTES...
          },
          "sensitive_attributes": [],
          "dependencies": ["aws_internet_gateway.main"]
        }
      ]
    }
  ]
}
```

**State Locking Flow (S3 + DynamoDB):**

```
terraform apply
       │
       ▼
┌─────────────────┐
│ Read State from  │ ──── GET s3://bucket/terraform.tfstate
│ S3 Backend       │
└─────────────────┘
       │
       ▼
┌─────────────────┐
│ Acquire Lock     │ ──── DynamoDB PutItem (conditional)
│ DynamoDB         │      Item: {LockID: "bucket/key", ...}
└─────────────────┘      If item exists → "Error acquiring lock"
       │
       ▼
┌─────────────────┐
│ Plan + Apply     │
│ Execute Changes  │
└─────────────────┘
       │
       ▼
┌─────────────────┐
│ Write New State  │ ──── PUT s3://bucket/terraform.tfstate
│ to S3            │
└─────────────────┘
       │
       ▼
┌─────────────────┐
│ Release Lock     │ ──── DynamoDB DeleteItem
│ DynamoDB         │
└─────────────────┘
```

### Terraform Workflow Deep Dive

```
terraform init
├── Download provider plugins (~/.terraform/providers/)
├── Initialize backend (connect to S3/remote)
├── Download modules
└── Create .terraform.lock.hcl (provider version pinning)

terraform plan
├── Read current state (from backend)
├── Refresh state (API calls to get real current state)
├── Load configuration (.tf files)
├── Build dependency graph (DAG)
├── Walk the graph → compute diff
│   ├── In state, not in config → DESTROY
│   ├── In config, not in state → CREATE
│   └── In both, attributes differ → UPDATE or REPLACE
└── Output human-readable plan

terraform apply
├── Show plan (or re-run plan)
├── User confirms (or -auto-approve)
├── Lock state
├── Walk dependency graph in parallel
│   ├── Independent resources execute concurrently
│   └── Dependent resources wait for dependencies
├── Call provider gRPC: Create/Read/Update/Delete
├── Update state after each resource success/fail
├── Write final state
└── Release lock
```

### Terraform Modules — Production Pattern

```
infrastructure/
├── main.tf                     # Root module
├── variables.tf
├── outputs.tf
├── terraform.tfvars            # Variable values (gitignore secrets!)
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
└── modules/
    ├── vpc/                    # Reusable VPC module
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── eks/                    # EKS cluster module
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── rds/                    # RDS module
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

**Module usage:**

```hcl
# Root main.tf — composing modules
module "vpc" {
  source = "./modules/vpc"      # Local module
  # source = "git::https://github.com/org/tf-modules.git//vpc?ref=v1.2.0"
  # source = "registry.terraform.io/hashicorp/vpc/aws"  # Public registry
  
  environment        = var.environment
  cidr_block         = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

module "eks" {
  source = "./modules/eks"
  
  vpc_id          = module.vpc.vpc_id           # Module output reference
  subnet_ids      = module.vpc.private_subnet_ids
  cluster_version = "1.28"
}

module "rds" {
  source = "./modules/rds"
  
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnet_ids
  allowed_cidr_blocks = [module.vpc.vpc_cidr_block]
}
```

### Terraform Workspaces vs. Separate State

```
WORKSPACES (for small teams/simple envs)
├── terraform workspace new dev
├── terraform workspace new staging  
├── terraform workspace new prod
├── State stored separately per workspace
│   s3://bucket/env:/dev/terraform.tfstate
│   s3://bucket/env:/prod/terraform.tfstate
└── Access: terraform.workspace variable

SEPARATE STATE FILES (recommended for prod)
├── environments/
│   ├── dev/
│   │   └── terraform.tfstate  (in S3: prod-state/dev/...)
│   ├── staging/
│   │   └── terraform.tfstate
│   └── prod/
│       └── terraform.tfstate  (strict access control!)
└── Completely isolated — prod changes can't affect dev
```

### Terraform Production Best Practices

```hcl
# 1. ALWAYS pin provider versions
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "= 5.31.0" }
  }
}

# 2. Use remote state with locking
backend "s3" {
  bucket         = "company-terraform-state"
  key            = "${var.environment}/terraform.tfstate"
  region         = "us-east-1"
  encrypt        = true
  kms_key_id     = "arn:aws:kms:..."
  dynamodb_table = "terraform-locks"
}

# 3. Sensitive outputs
output "db_password" {
  value     = random_password.db.result
  sensitive = true    # Hidden in CLI output, but still in state!
}

# 4. Prevent accidental deletion
resource "aws_rds_cluster" "main" {
  # ...
  deletion_protection = true
  
  lifecycle {
    prevent_destroy = true    # terraform destroy will error
    ignore_changes  = [engine_version]  # Ignore drift on this attr
    create_before_destroy = true        # Blue-green for replacements
  }
}

# 5. Import existing resources
# terraform import aws_vpc.main vpc-0abc123
```

### Terraform Troubleshooting

```bash
# Enable debug logging
export TF_LOG=DEBUG         # DEBUG, INFO, WARN, ERROR, TRACE
export TF_LOG_PATH=/tmp/terraform.log
terraform apply

# State manipulation (use with extreme caution!)
terraform state list                          # List all resources
terraform state show aws_vpc.main             # Show resource details
terraform state mv aws_vpc.old aws_vpc.new    # Rename resource
terraform state rm aws_instance.web           # Remove from state (doesn't destroy)
terraform import aws_vpc.main vpc-0abc123    # Import existing resource

# Refresh state (sync state with real world)
terraform refresh

# Target specific resources
terraform plan -target=module.eks
terraform apply -target=aws_vpc.main

# Force unlock (if stuck lock)
terraform force-unlock LOCK_ID

# Validate syntax
terraform validate
terraform fmt -recursive .  # Format all .tf files

# Common errors and fixes:
# "Error: Provider produced inconsistent result after apply"
# → Provider bug or race condition, retry usually works

# "Error acquiring the state lock"  
# → Check DynamoDB for stale lock, terraform force-unlock

# "Error: Cycle: resource A → resource B → resource A"
# → Circular dependency in configuration

# "│ Error: creating EC2 VPC: VpcLimitExceeded"
# → AWS account VPC limit hit, request limit increase
```

---

## 6.3 Ansible — Agentless Automation

### What is Ansible?

Ansible is an open-source automation tool for configuration management, application deployment, and task automation. It is **agentless** — no software needs to be installed on managed nodes.

### Ansible Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                      CONTROL NODE                               │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              ansible / ansible-playbook CLI               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐              │
│  │  Inventory  │  │  Playbooks │  │  Roles     │              │
│  │  (hosts)   │  │  (YAML)    │  │  (reusable)│              │
│  └────────────┘  └────────────┘  └────────────┘              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    MODULE LIBRARY                         │  │
│  │  apt, yum, copy, template, service, file, user,          │  │
│  │  shell, command, docker_container, k8s, aws_ec2 ...      │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
                         │
               SSH (port 22) or WinRM
               Pushes Python module
               Executes remotely
               Returns JSON result
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Managed     │ │  Managed     │ │  Managed     │
│  Node 1      │ │  Node 2      │ │  Node 3      │
│  (no agent!) │ │  (no agent!) │ │  (no agent!) │
│  Python req  │ │  Python req  │ │  Python req  │
└──────────────┘ └──────────────┘ └──────────────┘
```

**How Ansible Executes a Task (Under the Hood):**

```
1. ansible-playbook site.yml runs on control node
2. Ansible reads inventory → gets list of hosts
3. For each task, Ansible:
   a. Connects via SSH (uses paramiko or OpenSSH)
   b. Creates temp directory on remote: /tmp/.ansible/tmp/xxx/
   c. Copies the module Python file to remote temp dir
   d. Executes: /usr/bin/python /tmp/.ansible/tmp/xxx/module.py
   e. Module reads args from a JSON args file
   f. Module executes the actual work (install package, write file, etc.)
   g. Module prints JSON result to stdout
   h. Ansible reads JSON: {"changed": true/false, "rc": 0, ...}
   i. Removes temp files
   j. Reports result (OK / CHANGED / FAILED / SKIPPED)
```

### Inventory

```ini
# hosts.ini — Static inventory

[webservers]
web1.prod.example.com
web2.prod.example.com ansible_user=ec2-user ansible_port=22

[databases]
db1.prod.example.com
db2.prod.example.com

[prod:children]  # Group of groups
webservers
databases

[prod:vars]  # Variables for all prod hosts
ansible_python_interpreter=/usr/bin/python3
environment=production

[webservers:vars]
http_port=80
nginx_worker_processes=auto
```

```yaml
# inventory/hosts.yml — YAML inventory (preferred)
all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/prod-key.pem
  
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 10.0.1.10
          nginx_port: 80
        web2:
          ansible_host: 10.0.1.11
          nginx_port: 80
    
    databases:
      hosts:
        db1:
          ansible_host: 10.0.2.10
          postgresql_version: "15"
    
    monitoring:
      hosts:
        prometheus:
          ansible_host: 10.0.3.10
```

```bash
# Dynamic Inventory — auto-discover from cloud
# aws_ec2 plugin
# inventory/aws_ec2.yml
plugin: aws_ec2
regions:
  - us-east-1
filters:
  instance-state-name: running
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
hostnames:
  - private-ip-address

# Use: ansible-playbook -i inventory/aws_ec2.yml site.yml
```

### Playbooks — Complete Reference

```yaml
# site.yml — Master playbook
---
- name: Configure Web Servers
  hosts: webservers
  become: true          # sudo
  become_user: root
  gather_facts: true    # Collect system info (OS, IP, memory, etc.)
  serial: 2             # Run on 2 hosts at a time (rolling update)
  max_fail_percentage: 20  # Abort if >20% of hosts fail
  
  vars:
    nginx_version: "1.24"
    app_port: 8080
  
  vars_files:
    - vars/common.yml
    - "vars/{{ ansible_os_family }}.yml"  # Dynamic var file
  
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
  
  roles:
    - common          # Applied to all servers
    - nginx           # Web server config
    - app             # Application deployment
  
  post_tasks:
    - name: Verify nginx is serving
      uri:
        url: "http://{{ ansible_default_ipv4.address }}/health"
        return_content: true
      register: health_check
      failed_when: health_check.status != 200
  
  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted

# Handlers are triggered by 'notify', run at end of play
# If 5 tasks all notify 'restart nginx', it runs ONCE
```

### Roles — Production Structure

```
roles/
└── nginx/
    ├── tasks/
    │   ├── main.yml          # Entry point
    │   ├── install.yml
    │   └── configure.yml
    ├── handlers/
    │   └── main.yml          # restart nginx handler
    ├── templates/
    │   ├── nginx.conf.j2     # Jinja2 templates
    │   └── vhost.conf.j2
    ├── files/
    │   └── ssl/              # Static files (not templated)
    ├── vars/
    │   └── main.yml          # Role variables (high priority)
    ├── defaults/
    │   └── main.yml          # Default variables (low priority)
    ├── meta/
    │   └── main.yml          # Role dependencies
    └── README.md
```

```yaml
# roles/nginx/tasks/main.yml
---
- name: Install nginx
  package:
    name: "nginx={{ nginx_version }}*"
    state: present
  notify: restart nginx

- name: Deploy nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    validate: '/usr/sbin/nginx -t -c %s'  # Validate before deploying!
  notify: restart nginx

- name: Deploy virtual host configs
  template:
    src: vhost.conf.j2
    dest: "/etc/nginx/sites-available/{{ item.name }}.conf"
  loop: "{{ vhosts }}"  # Iterate over list of vhosts
  
- name: Enable virtual hosts
  file:
    src: "/etc/nginx/sites-available/{{ item.name }}.conf"
    dest: "/etc/nginx/sites-enabled/{{ item.name }}.conf"
    state: link
  loop: "{{ vhosts }}"
  notify: restart nginx

- name: Ensure nginx is started and enabled
  service:
    name: nginx
    state: started
    enabled: true

- name: Open firewall port
  ufw:
    rule: allow
    port: "{{ nginx_port }}"
    proto: tcp
```

```jinja2
{# roles/nginx/templates/nginx.conf.j2 #}
user {{ nginx_user | default('www-data') }};
worker_processes {{ nginx_worker_processes | default('auto') }};
error_log /var/log/nginx/error.log {{ nginx_error_log_level | default('warn') }};

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
    use epoll;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout {{ nginx_keepalive_timeout | default(65) }};
    
    {% if nginx_gzip_enabled | default(true) %}
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    {% endif %}
    
    {% for vhost in vhosts %}
    include /etc/nginx/sites-enabled/{{ vhost.name }}.conf;
    {% endfor %}
}
```

### Ansible Vault — Secrets Management

```bash
# Encrypt a file
ansible-vault encrypt vars/secrets.yml

# Create encrypted file directly
ansible-vault create vars/secrets.yml

# Edit encrypted file
ansible-vault edit vars/secrets.yml

# Encrypt a single string (inline)
ansible-vault encrypt_string 'myS3cr3tP@ss' --name 'db_password'
# Output:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   38316537353230366666...

# Run playbook with vault password
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

### Ansible Troubleshooting

```bash
# Check connectivity
ansible all -i hosts.ini -m ping

# Run ad-hoc commands
ansible webservers -i hosts.ini -m shell -a "df -h"
ansible webservers -i hosts.ini -m service -a "name=nginx state=restarted" --become

# Dry run (check mode)
ansible-playbook site.yml --check --diff

# Step through tasks interactively
ansible-playbook site.yml --step

# Verbose output (increase -v for more)
ansible-playbook site.yml -v    # Task output
ansible-playbook site.yml -vv   # + Connection info
ansible-playbook site.yml -vvv  # + SSH commands
ansible-playbook site.yml -vvvv # + Connection debug

# Limit to specific hosts/groups
ansible-playbook site.yml --limit web1.example.com
ansible-playbook site.yml --limit "webservers:!web1"  # All except web1

# Start at specific task
ansible-playbook site.yml --start-at-task="Deploy nginx configuration"

# Tag-based execution
ansible-playbook site.yml --tags install,configure
ansible-playbook site.yml --skip-tags deploy

# List tasks without running
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --list-hosts
```

---

## 6.4 Helm — The Kubernetes Package Manager

### What is Helm?

Helm is the package manager for Kubernetes. A **Helm chart** is a collection of Kubernetes YAML manifests templated with Go templating, packaged into a distributable artifact.

### Why Helm?

```
WITHOUT HELM:
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml
... 20 more files ...

# To change environment: edit each file manually
# To upgrade: reapply all files
# To rollback: git revert and re-apply
# To share: copy all files

WITH HELM:
helm install my-app ./my-chart --values prod.values.yaml
helm upgrade my-app ./my-chart --values prod.values.yaml
helm rollback my-app 1
helm package ./my-chart → my-chart-1.0.0.tgz  (shareable artifact)
```

### Helm Chart Structure

```
my-app/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default values
├── values-prod.yaml        # Environment override (not part of chart spec)
├── templates/
│   ├── _helpers.tpl        # Named templates (reusable snippets)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   └── NOTES.txt           # Post-install instructions
├── charts/                 # Dependency charts (subcharts)
│   └── postgresql/
└── crds/                   # Custom Resource Definitions
```

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
description: Production web application
type: application   # or "library" for shared templates
version: 1.2.3      # Chart version (SemVer)
appVersion: "2.1.0" # App version being packaged

dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled  # Only if .Values.postgresql.enabled=true
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

```yaml
# values.yaml — defaults, overridable
replicaCount: 2

image:
  repository: mycompany/my-app
  tag: ""         # Defaults to .Chart.AppVersion
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled: true
  auth:
    postgresPassword: ""  # Set via --set or secrets

env: []
  # - name: DATABASE_URL
  #   valueFrom:
  #     secretKeyRef:
  #       name: app-secrets
  #       key: database-url
```

```yaml
# templates/deployment.yaml — with Go templating
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}   {{- /* Named template */}}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}   {{- /* Indent 4 spaces */}}
  annotations:
    helm.sh/chart: {{ include "my-app.chart" . }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- /* ↑ Force pod restart when configmap changes */}}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          {{- with .Values.env }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /health
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 5
            periodSeconds: 5
```

```
# templates/_helpers.tpl — reusable named templates
{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

### Helm Hooks

```yaml
# templates/db-migrate-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-app.fullname" . }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install       # When to run
    "helm.sh/hook-weight": "-5"                   # Order (lower = earlier)
    "helm.sh/hook-delete-policy": hook-succeeded  # Cleanup after success
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["python", "manage.py", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
```

### Helm Commands — Production Reference

```bash
# Repo management
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm search repo nginx --versions

# Install
helm install my-release ./my-chart \
  --namespace production \
  --create-namespace \
  --values values-prod.yaml \
  --set image.tag=v2.1.0 \
  --set postgresql.auth.postgresPassword=secret \
  --wait \              # Wait for all pods to be ready
  --timeout 10m

# Upgrade
helm upgrade my-release ./my-chart \
  --namespace production \
  --values values-prod.yaml \
  --set image.tag=v2.2.0 \
  --atomic \            # Rollback automatically if upgrade fails
  --cleanup-on-fail

# Rollback
helm history my-release -n production    # See revision history
helm rollback my-release 3 -n production # Roll back to revision 3

# Debug/dry-run
helm template my-release ./my-chart --values values-prod.yaml
helm install my-release ./my-chart --dry-run --debug

# Package and push to OCI registry
helm package ./my-chart
helm push my-chart-1.2.3.tgz oci://registry.example.com/charts

# Uninstall
helm uninstall my-release -n production
```

---

## 6.5 Pulumi — IaC with Real Programming Languages

### What is Pulumi?

Pulumi allows you to define infrastructure using **real programming languages** — TypeScript, Python, Go, C#, Java — instead of DSLs like HCL.

```typescript
// index.ts — Pulumi TypeScript example
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

// Create VPC with subnets automatically
const vpc = new awsx.ec2.Vpc("main-vpc", {
    cidrBlock: "10.0.0.0/16",
    numberOfAvailabilityZones: 3,
    subnetStrategy: awsx.ec2.SubnetAllocationStrategy.Auto,
});

// EKS Cluster
const cluster = new awsx.eks.Cluster("prod-cluster", {
    vpcId: vpc.vpcId,
    privateSubnetIds: vpc.privateSubnetIds,
    instanceType: "t3.medium",
    desiredCapacity: 3,
    minSize: 1,
    maxSize: 10,
});

// Use real language features: loops, conditionals, functions
const buckets = ["assets", "logs", "backups"].map(name => 
    new aws.s3.Bucket(`${name}-bucket`, {
        versioning: { enabled: name !== "logs" },
        serverSideEncryptionConfiguration: {
            rule: {
                applyServerSideEncryptionByDefault: {
                    sseAlgorithm: "AES256"
                }
            }
        }
    })
);

// Export outputs
export const kubeconfig = cluster.kubeconfig;
export const vpcId = vpc.vpcId;
```

### Terraform vs Pulumi vs Ansible vs Helm

| Feature | Terraform | Pulumi | Ansible | Helm |
|---------|-----------|--------|---------|------|
| Language | HCL | TS/Python/Go/C# | YAML | YAML + Go templates |
| Approach | Declarative | Imperative+Declarative | Procedural | Declarative |
| State | tfstate file | Pulumi Service/S3 | None (idempotent) | Kubernetes Secrets |
| Agentless? | Yes | Yes | Yes (SSH) | Yes (kubectl) |
| Scope | Cloud infra | Cloud infra | Config mgmt | K8s packaging |
| Learning curve | Medium | Low (if you know TS/Python) | Low | Medium |
| Best for | Multi-cloud provisioning | Devs who prefer code | Server config | K8s app packaging |

---

## 6.6 IaC Interview Questions

### Beginner

**Q: What is the difference between Terraform plan and apply?**

> **Expected Answer:** `terraform plan` shows what changes WOULD be made without making them — it computes a diff between desired state (config) and current state (state file + real infrastructure). `terraform apply` executes those changes. Always run plan first in production. Plan output should be reviewed and ideally stored as an artifact for audit trails.
>
> **Why asked:** Tests understanding of Terraform's core workflow and safety practices.

**Q: What is Ansible's agentless architecture and how does it work?**

> **Expected Answer:** Ansible connects to managed nodes via SSH (Linux) or WinRM (Windows). No agent/daemon runs on managed nodes. Ansible copies Python module files to the remote node, executes them, reads the JSON result back, then deletes the temp files. Only requirement on managed nodes is Python (usually pre-installed) and SSH access.
>
> **Common mistake:** Saying "Ansible uses an agent like Chef/Puppet."

### Advanced

**Q: Explain Terraform state drift and how to handle it.**

> **Expected Answer:** State drift occurs when real infrastructure differs from the Terraform state file — caused by manual changes in the cloud console, changes by other tools, or failed applies. Detection: `terraform plan` shows unexpected diffs; `terraform refresh` updates state from real infra. Handling: import manually created resources (`terraform import`), use `lifecycle.ignore_changes` for attributes that drift intentionally, implement drift detection in CI/CD pipelines. Prevention: enforce IaC-only changes via IAM policies that deny console modifications.

**Q: How do you handle secrets in Terraform?**

> **Expected Answer:** Never store secrets in .tf files or commit tfvars to git. Options: (1) Environment variables (`TF_VAR_db_password`), (2) AWS Secrets Manager/Parameter Store data sources — fetch at apply time, (3) HashiCorp Vault provider, (4) SOPS for encrypted variable files. Critical: State file contains secrets in plaintext — encrypt state backend with KMS, restrict S3 bucket access, enable versioning for state recovery.

**Q: Design a Terraform architecture for a company with 50 engineers, 3 environments, and multiple cloud accounts.**

> **Expected Answer:**
> - Separate AWS accounts per environment (dev/staging/prod) — blast radius isolation
> - One state file per environment per service — avoid monolithic state
> - Terraform modules in a separate repository, versioned with git tags
> - Terragrunt for DRY configurations across environments  
> - CI/CD: plan on PR, apply on merge (with approval gates for prod)
> - State buckets encrypted + versioned, IAM-restricted
> - `atlantis` or Terraform Cloud for collaborative plan/apply workflow
> - Sentinel policies (Terraform Enterprise) for policy-as-code
```
# Phase 7: Monitoring & Observability

---

## 7.1 The Three Pillars of Observability

```
OBSERVABILITY = METRICS + LOGS + TRACES
                    │          │        │
                    ▼          ▼        ▼
             "Is it slow?"  "Why?"  "Where?"
             Prometheus     ELK     Jaeger
             Grafana        Loki    Tempo
```

**Monitoring vs Observability:**

| Monitoring | Observability |
|-----------|--------------|
| Known unknowns | Unknown unknowns |
| "Is CPU > 80%?" | "Why is p99 latency high for user X?" |
| Pre-defined dashboards | Ad-hoc exploration |
| Alerts on thresholds | Understand system state from outputs |
| "Is the system up?" | "What is the system doing?" |

**The Four Golden Signals (Google SRE Book):**
1. **Latency** — time to serve a request (separate success vs error latency)
2. **Traffic** — demand on the system (requests/sec, queries/sec)
3. **Errors** — rate of requests that fail (explicit: 500s; implicit: wrong data)
4. **Saturation** — how "full" the service is (CPU, memory, queue depth)

---

## 7.2 Prometheus — Metrics at Scale

### What is Prometheus?

Prometheus is an open-source systems monitoring and alerting toolkit. It collects **time-series metrics** by **scraping** HTTP endpoints, stores them in its custom TSDB, and provides a powerful query language (PromQL) for analysis.

### Prometheus Internal Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PROMETHEUS SERVER                            │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    RETRIEVAL (Scraper)                       │    │
│  │  Job: kubernetes-nodes                                       │    │
│  │  Targets: [10.0.0.1:9100, 10.0.0.2:9100, ...]              │    │
│  │  Scrape interval: 15s                                        │    │
│  │  HTTP GET /metrics → text exposition format                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                         │                                             │
│                         ▼                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              SERVICE DISCOVERY                               │    │
│  │  Kubernetes SD: watches API → discovers pods/nodes          │    │
│  │  File SD: reads targets from JSON/YAML files                │    │
│  │  EC2 SD, Consul SD, DNS SD, static_configs                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                         │                                             │
│                         ▼                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              TSDB (Time Series Database)                     │    │
│  │                                                               │    │
│  │  head block (in memory, last 2h)                             │    │
│  │  ├── WAL (Write-Ahead Log) for durability                    │    │
│  │  └── chunks in memory                                        │    │
│  │                                                               │    │
│  │  persistent blocks (on disk, 2h - retention period)         │    │
│  │  ├── chunks/ (compressed time series data)                   │    │
│  │  ├── index (series → chunk mapping)                          │    │
│  │  └── meta.json                                               │    │
│  │                                                               │    │
│  │  Compaction: small blocks → larger blocks (efficiency)       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              RULES ENGINE                                    │    │
│  │  Recording Rules: pre-compute expensive queries              │    │
│  │  Alerting Rules: evaluate → fire to Alertmanager            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────┐                      ┌──────────────────────────┐  │
│  │ HTTP API     │                      │  ALERTMANAGER            │  │
│  │ /api/v1/     │                      │  Dedup → Group → Route   │  │
│  │ query        │                      │  → PagerDuty/Slack/Email  │  │
│  │ query_range  │                      └──────────────────────────┘  │
│  │ series       │                                                      │
│  └─────────────┘                                                      │
└─────────────────────────────────────────────────────────────────────┘
          ▲                ▲              ▲
   scrape /metrics    scrape /metrics   Pushgateway
          │                │              │
   ┌──────────┐     ┌──────────┐    ┌──────────┐
   │ Node      │     │  App     │    │ Batch    │
   │ Exporter  │     │ /metrics │    │ Jobs     │
   │ (port     │     │ (port    │    │(push,    │
   │  9100)    │     │  8080)   │    │ not pull)│
   └──────────┘     └──────────┘    └──────────┘
```

### Metric Types

```
COUNTER — monotonically increasing (never decreases, resets to 0 on restart)
  http_requests_total{method="GET", status="200"} 1234567
  Use for: request counts, error counts, bytes transferred
  Always use _total suffix

GAUGE — can go up and down
  memory_usage_bytes{container="api"} 536870912
  Use for: current memory, active connections, queue size, temperature

HISTOGRAM — samples observations, counts them in configurable buckets
  http_request_duration_seconds_bucket{le="0.1"} 24054  ← ≤100ms
  http_request_duration_seconds_bucket{le="0.5"} 33444  ← ≤500ms
  http_request_duration_seconds_bucket{le="1.0"} 33478
  http_request_duration_seconds_bucket{le="+Inf"} 33480  ← all requests
  http_request_duration_seconds_sum 144758.0          ← total seconds
  http_request_duration_seconds_count 33480           ← total requests
  Use for: latency, request sizes (allows percentile calculation)

SUMMARY — similar to histogram but calculates quantiles client-side
  rpc_duration_seconds{quantile="0.5"}  0.012
  rpc_duration_seconds{quantile="0.99"} 0.085
  Use for: latency when you need precise quantiles
  ⚠️ Cannot aggregate across instances (use histogram instead in k8s)
```

### Prometheus Text Exposition Format

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",handler="/api/users",status="200"} 1234
http_requests_total{method="GET",handler="/api/users",status="404"} 56
http_requests_total{method="POST",handler="/api/users",status="201"} 789

# HELP http_request_duration_seconds HTTP request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{handler="/api/users",le="0.005"} 100
http_request_duration_seconds_bucket{handler="/api/users",le="0.01"}  200
http_request_duration_seconds_bucket{handler="/api/users",le="0.025"} 400
http_request_duration_seconds_bucket{handler="/api/users",le="+Inf"} 1000
http_request_duration_seconds_sum{handler="/api/users"} 125.0
http_request_duration_seconds_count{handler="/api/users"} 1000
```

### PromQL — Query Language Deep Dive

```promql
# ===== INSTANT VECTORS =====
# Current value of a metric
http_requests_total                                # All time series
http_requests_total{status="200"}                  # Filter by label
http_requests_total{status=~"2.."}                 # Regex: 200, 201, etc
http_requests_total{status!="500"}                 # Not equal

# ===== RANGE VECTORS =====
# Values over a time range [5m = last 5 minutes]
http_requests_total[5m]                            # Used with rate/irate

# ===== FUNCTIONS =====

# rate() — per-second rate (smoothed over range, handles counter resets)
rate(http_requests_total[5m])                      # Req/sec over 5min window

# irate() — instantaneous rate (last 2 samples, more responsive)
irate(http_requests_total[5m])                     # Use for spiky metrics

# increase() — total increase over range
increase(http_requests_total[1h])                  # Total requests in 1h

# sum() — aggregate across all instances
sum(rate(http_requests_total[5m]))
sum by (status) (rate(http_requests_total[5m]))    # Group by status code
sum without (pod) (rate(http_requests_total[5m]))  # Remove pod label

# histogram_quantile() — calculate percentiles from histograms
histogram_quantile(0.99,                           # p99 latency
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)

histogram_quantile(0.50,                           # Median latency
  sum by (le, handler) (                           # Per handler
    rate(http_request_duration_seconds_bucket[5m])
  )
)

# ===== PRACTICAL EXAMPLES =====

# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# CPU usage by pod
sum by (pod) (
  rate(container_cpu_usage_seconds_total{namespace="production"}[5m])
) * 100

# Memory usage (bytes)
sum by (pod) (
  container_memory_working_set_bytes{namespace="production"}
)

# Request rate per endpoint
sum by (handler) (
  rate(http_requests_total[5m])
)

# Apdex score (Application Performance Index)
# Satisfied: < 0.3s, Tolerating: < 1.2s, Frustrated: >= 1.2s
(
  sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
  +
  sum(rate(http_request_duration_seconds_bucket{le="1.2"}[5m]))
) / 2
/
sum(rate(http_request_duration_seconds_count[5m]))
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s          # Default: scrape every 15s
  evaluation_interval: 15s      # Evaluate rules every 15s
  scrape_timeout: 10s
  
  external_labels:              # Labels added to all time series
    cluster: production
    region: us-east-1

rule_files:
  - "rules/*.yml"               # Alert and recording rules

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  # Kubernetes Pods auto-discovery
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      # Use custom port if specified
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (\d+)
        replacement: $1
      # Keep only useful labels
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
  
  # Node exporters
  - job_name: 'node-exporter'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    
  # Static targets
  - job_name: 'postgres-exporter'
    static_configs:
      - targets: ['postgres-exporter:9187']
    metrics_path: /metrics
    scheme: https
    tls_config:
      ca_file: /etc/ssl/certs/ca-cert.pem
```

### Alerting Rules

```yaml
# rules/alerts.yml
groups:
  - name: application
    interval: 30s      # Override global evaluation_interval
    rules:
      # Recording rule: pre-compute expensive query
      - record: job:http_requests_total:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
      
      # Alerting rule
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          * 100 > 5
        for: 5m          # Must be true for 5min before firing
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High HTTP error rate on {{ $labels.job }}"
          description: "Error rate is {{ $value | printf \"%.2f\" }}% (threshold: 5%)"
          runbook: "https://wiki.company.com/runbooks/high-error-rate"
      
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency {{ $value | printf \"%.3f\" }}s on {{ $labels.job }}"
      
      - alert: PodNotReady
        expr: kube_pod_status_ready{condition="true"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not ready"
      
      - alert: DiskSpaceRunningLow
        expr: |
          (node_filesystem_avail_bytes{mountpoint="/"} 
           / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}: {{ $value | printf \"%.1f\" }}% free"
```

### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/...'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  group_by: ['alertname', 'cluster', 'job']
  group_wait: 30s          # Wait to batch alerts
  group_interval: 5m       # How long before resending
  repeat_interval: 4h      # Resend if not resolved
  receiver: 'slack-general'
  
  routes:
    - match:
        severity: critical
      receiver: pagerduty
      continue: true   # Also send to next matching route
    
    - match:
        severity: critical
        team: backend
      receiver: slack-backend-critical
      group_wait: 0s   # Page immediately for critical
    
    - match_re:
        alertname: ^(Watchdog|InfoInhibitor)$
      receiver: 'null'  # Silence informational alerts

receivers:
  - name: 'null'
  
  - name: slack-general
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
  
  - name: pagerduty
    pagerduty_configs:
      - service_key: '{{ .ExternalURL }}'
        severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'

inhibit_rules:
  # If cluster is down, don't alert for individual components
  - source_match:
      alertname: 'ClusterDown'
    target_match_re:
      alertname: '.+'
    equal: ['cluster']
```

---

## 7.3 Grafana — Visualization

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                    GRAFANA                           │
│                                                       │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │  Web UI      │  │  Backend API  │  │  Database  │ │
│  │  (React)     │  │  (Go)         │  │  (SQLite/  │ │
│  │              │  │               │  │  MySQL/PG) │ │
│  └─────────────┘  └──────────────┘  └────────────┘ │
│                         │                             │
│              ┌──────────┼──────────┐                 │
│              ▼          ▼          ▼                 │
│        Dashboards   Alerting   Users/Orgs            │
│        Panels       Rules      Permissions           │
│        Variables    Contacts   API Keys              │
│                                                       │
│  ┌─────────────────────────────────────────────┐    │
│  │              DATA SOURCE PLUGINS             │    │
│  │  Prometheus │ Loki │ Tempo │ ES │ InfluxDB  │    │
│  │  CloudWatch │ MySQL │ PostgreSQL │ ...       │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
         │           │           │
   Prometheus      Loki        Tempo
   (metrics)      (logs)      (traces)
```

### Dashboard as Code (Grafonnet/JSON)

```json
// dashboard.json — programmatic dashboard
{
  "title": "Application Overview",
  "uid": "app-overview",
  "refresh": "30s",
  "time": {"from": "now-1h", "to": "now"},
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_pod_info, namespace)",
        "multi": false,
        "includeAll": false
      }
    ]
  },
  "panels": [
    {
      "title": "Request Rate",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
      "datasource": "Prometheus",
      "targets": [
        {
          "expr": "sum by (status) (rate(http_requests_total{namespace=\"$namespace\"}[5m]))",
          "legendFormat": "{{status}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "custom": {"lineWidth": 2}
        }
      }
    }
  ]
}
```

---

## 7.4 ELK Stack — Log Management

### Architecture Overview

```
APPLICATION LOGS
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│                     LOG SHIPPERS (Beat agents)                   │
│                                                                   │
│  Filebeat (log files)    Metricbeat (system metrics)            │
│  Packetbeat (network)    Heartbeat (uptime)                      │
│                                                                   │
│  Lightweight Go agents — installed on every server              │
│  Read from file/syslog/stdin → ship to Logstash or ES directly  │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼ (port 5044 / Beats protocol or port 5000 / syslog)
┌─────────────────────────────────────────────────────────────────┐
│                         LOGSTASH                                 │
│                                                                   │
│  INPUT → FILTER → OUTPUT pipeline                               │
│                                                                   │
│  Input:   beats, syslog, kafka, http, stdin                     │
│  Filter:  grok (parse unstructured), mutate (rename/remove),    │
│           geoip (add location), date (parse timestamps)         │
│  Output:  elasticsearch, kafka, s3, stdout                      │
│                                                                   │
│  Horizontally scalable (multiple Logstash nodes)                │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼ (port 9200 / HTTP or port 9300 / TCP cluster comms)
┌─────────────────────────────────────────────────────────────────┐
│                      ELASTICSEARCH                               │
│                                                                   │
│  Node 1 (Master)  ←→  Node 2 (Data)  ←→  Node 3 (Data)        │
│                                                                   │
│  Index: logs-2024.01.15  (daily rolling index)                  │
│  ├── Primary shard 0 (on Node 2)                                │
│  ├── Primary shard 1 (on Node 3)                                │
│  ├── Replica shard 0 (on Node 3)  ← replica of primary 0       │
│  └── Replica shard 1 (on Node 2)  ← replica of primary 1       │
│                                                                   │
│  Inverted index: word → [document IDs] (fast full-text search)  │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼ (port 5601 / HTTP)
┌─────────────────────────────────────────────────────────────────┐
│                         KIBANA                                   │
│  Discover: browse/search logs                                    │
│  Dashboard: visualizations                                       │
│  APM: application performance                                    │
│  Alerting: log-based alerts                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Logstash Pipeline

```ruby
# /etc/logstash/conf.d/application.conf

input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
  }
}

filter {
  # Parse JSON logs (for structured logging apps)
  if [fields][log_type] == "application" {
    json {
      source => "message"
      target => "app"
    }
    
    # Parse timestamp
    date {
      match => ["[app][timestamp]", "ISO8601"]
      target => "@timestamp"
    }
    
    # Rename fields
    mutate {
      rename => { "[app][level]" => "log_level" }
      rename => { "[app][trace_id]" => "trace_id" }
      add_field => { "environment" => "%{[fields][environment]}" }
    }
  }
  
  # Parse nginx access logs (unstructured)
  if [fields][log_type] == "nginx" {
    grok {
      match => {
        "message" => '%{IPORHOST:client_ip} - %{USER:auth} \[%{HTTPDATE:timestamp}\] "%{WORD:http_method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}" %{NUMBER:status_code:int} %{NUMBER:body_bytes:int} "%{GREEDYDATA:referrer}" "%{GREEDYDATA:user_agent}" %{NUMBER:request_time:float}'
      }
    }
    
    geoip {
      source => "client_ip"
      target => "geoip"
    }
    
    useragent {
      source => "user_agent"
      target => "ua"
    }
  }
  
  # Drop health check noise
  if [request] =~ "/health" {
    drop { }
  }
}

output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    user => "logstash_writer"
    password => "${LOGSTASH_PASSWORD}"
    ssl => true
    cacert => "/etc/logstash/certs/ca.crt"
    
    # Daily rolling indices
    index => "logs-%{[fields][log_type]}-%{+YYYY.MM.dd}"
    
    # Use ILM for lifecycle management
    ilm_enabled => true
    ilm_rollover_alias => "logs-app"
    ilm_policy => "logs-30d-policy"
  }
}
```

### Elasticsearch Index Lifecycle Management (ILM)

```json
// PUT _ilm/policy/logs-30d-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",        // Rollover after 1 day
            "max_size": "50gb"      // or when 50GB
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "3d",           // Move to warm after 3 days
        "actions": {
          "shrink": { "number_of_shards": 1 },  // Reduce shards
          "forcemerge": { "max_num_segments": 1 }, // Optimize
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "7d",           // Move to cold after 7 days
        "actions": {
          "freeze": {},            // Read-only, minimal resources
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "30d",          // Delete after 30 days
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## 7.5 Loki — Logs for Kubernetes

### Why Loki Instead of ELK?

```
ELK:                                    LOKI:
Full-text indexes EVERYTHING            Indexes ONLY labels (like Prometheus)
$$$$ storage (indexes ~10x log size)    $ storage (compressed raw logs)
Complex setup                           Simple (runs in K8s easily)
Powerful search                         LogQL (same syntax as PromQL)
Best for: structured log analysis       Best for: K8s operational logs
```

### Loki Architecture

```
┌────────────┐  ┌────────────┐  ┌────────────┐
│  Promtail   │  │  Promtail  │  │  Promtail  │
│  (DaemonSet)│  │  (DaemonSet│  │  (DaemonSet│
│  K8s Node 1 │  │  Node 2    │  │  Node 3    │
│  Tails pod  │  │  Tails pod │  │  Tails pod │
│  log files  │  │  log files │  │  log files │
└────────────┘  └────────────┘  └────────────┘
       │                │               │
       └────────────────┼───────────────┘
                        │
                        ▼ (port 3100 / HTTP)
                 ┌─────────────┐
                 │    LOKI     │
                 │             │
                 │  Distributor│ ← Receives writes, hashes stream
                 │      │      │
                 │  Ingester   │ ← In-memory recent logs + WAL
                 │      │      │
                 │  Querier    │ ← Executes LogQL queries
                 │      │      │
                 │  Compactor  │ ← Manages long-term storage
                 └─────────────┘
                        │
                 ┌──────┴──────┐
                 │  S3/GCS/    │  ← Chunks (compressed log data)
                 │  Azure Blob │  ← Index (label → chunk mapping)
                 └─────────────┘
```

### LogQL — Query Language

```logql
# ===== LOG QUERIES =====

# Simple label filter
{namespace="production", app="api-server"}

# Multiple labels
{namespace="production"} |= "ERROR"    # Contains "ERROR"
{namespace="production"} != "DEBUG"    # Exclude "DEBUG"
{namespace="production"} |~ "error|Error|ERROR"  # Regex

# Parse JSON logs
{app="api"} | json | status_code >= 500

# Parse with pattern
{app="nginx"} 
  | pattern `<ip> - - [<ts>] "<method> <path> HTTP/<_>" <status> <size>`
  | status >= 500

# Pipeline filters
{namespace="production"}
  | json
  | line_format "{{.level}} {{.message}}"    # Reformat output
  | label_format severity=level              # Rename labels

# ===== METRIC QUERIES =====

# Log rate
rate({app="api"} |= "error" [5m])    # Errors per second

# Count by status code
sum by (status) (
  rate({app="nginx"} | json | __error__="" [5m])
)

# Error rate percentage  
sum(rate({app="api"} |= "ERROR" [5m]))
/
sum(rate({app="api"} [5m]))
* 100

# p99 latency from logs
quantile_over_time(0.99,
  {app="api"} | json | unwrap response_time [5m]
) by (endpoint)
```

---

## 7.6 Distributed Tracing — Jaeger & OpenTelemetry

### Why Distributed Tracing?

```
User request → API Gateway → Auth Service → User Service → DB
                                         → Order Service → Inventory Service → DB
                                                         → Payment Service → External API

PROBLEM: Request is slow. Which service is slow?
SOLUTION: Trace the request through all services with timing data.

TRACE: A complete request from start to finish
  └── SPAN: A unit of work (one service, one DB query)
        ├── trace_id: abc123 (same across all services)
        ├── span_id: xyz789 (unique per span)
        ├── parent_span_id: parent (links spans in tree)
        ├── operation: "HTTP GET /api/orders"
        ├── service: "order-service"
        ├── start_time: 1704067200000
        ├── duration: 45ms
        └── tags: {http.status=200, db.type=postgresql}
```

### OpenTelemetry — The Standard

```
OpenTelemetry (OTel) is the CNCF standard for:
  - Metrics (replaces Prometheus client libraries)
  - Logs (structured logging)  
  - Traces (replaces Jaeger/Zipkin client libraries)

Architecture:
  App → OTel SDK → OTel Collector → Jaeger/Tempo/DataDog/etc.
  
Benefits:
  - Vendor-neutral: switch backends without changing app code
  - Auto-instrumentation: zero-code instrumentation for common frameworks
  - W3C TraceContext: standard trace propagation headers
```

```python
# Python app with OpenTelemetry
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

# Setup
provider = TracerProvider(
    resource=Resource.create({
        "service.name": "order-service",
        "service.version": "1.2.0",
        "deployment.environment": "production"
    })
)
provider.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(endpoint="http://otel-collector:4317")
    )
)
trace.set_tracer_provider(provider)

# Auto-instrument FastAPI and SQLAlchemy
FastAPIInstrumentor.instrument_app(app)
SQLAlchemyInstrumentor().instrument(engine=engine)

# Manual instrumentation
tracer = trace.get_tracer(__name__)

@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    with tracer.start_as_current_span("get_order") as span:
        span.set_attribute("order.id", order_id)
        
        # This creates a child span automatically
        order = await db.query(Order).filter_by(id=order_id).first()
        
        if not order:
            span.set_attribute("error", True)
            span.set_status(trace.StatusCode.ERROR, "Order not found")
            raise HTTPException(404)
        
        # Propagate trace to downstream service
        inventory = await inventory_client.get_inventory(order.items)
        
        span.add_event("order_retrieved", {"item_count": len(order.items)})
        return order
```

### OpenTelemetry Collector

```yaml
# otel-collector-config.yml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"
  
  # Collect Prometheus metrics too
  prometheus:
    config:
      scrape_configs:
        - job_name: 'otel-collector'
          static_configs:
            - targets: ['localhost:8888']

processors:
  batch:                          # Batch for efficiency
    send_batch_size: 10000
    timeout: 10s
  
  memory_limiter:                 # Prevent OOM
    check_interval: 1s
    limit_mib: 512
  
  resource:                       # Add resource attributes
    attributes:
      - key: environment
        value: production
        action: upsert

exporters:
  jaeger:
    endpoint: "jaeger:14250"
    tls:
      insecure: true
  
  prometheus:
    endpoint: "0.0.0.0:8889"     # Prometheus scrapes this
  
  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"
  
  logging:                        # Debug: print to stdout
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger]
    
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
    
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

---

## 7.7 Production Observability Stack Setup

### Complete Kubernetes Monitoring Stack

```yaml
# Using kube-prometheus-stack Helm chart
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values monitoring-values.yaml
```

```yaml
# monitoring-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: 50GB
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          resources:
            requests:
              storage: 100Gi
    
    # Scrape all ServiceMonitors in all namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    
    # Remote write to Thanos/Cortex for long-term storage
    remoteWrite:
      - url: https://thanos-receive:10908/api/v1/receive

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 10Gi
  config:
    global:
      slack_api_url: 'https://hooks.slack.com/...'
    route:
      receiver: slack

grafana:
  adminPassword: "changeme"
  persistence:
    enabled: true
    size: 10Gi
  
  # Auto-provision dashboards from ConfigMaps
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
  
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://kube-prometheus-stack-prometheus:9090
          isDefault: true
        - name: Loki
          type: loki
          url: http://loki:3100
        - name: Tempo
          type: tempo
          url: http://tempo:3100
```

### Service Monitor — Auto-Discover App Metrics

```yaml
# servicemonitor.yaml — tells Prometheus to scrape your app
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app          # Selects Services with this label
  endpoints:
    - port: metrics        # Port name in the Service
      path: /metrics
      interval: 15s
      scrapeTimeout: 10s
  namespaceSelector:
    matchNames:
      - production
```

---

## 7.8 Observability Interview Questions

### Beginner

**Q: What is the difference between monitoring and observability?**

> Monitoring tells you WHEN something is wrong by alerting on pre-defined thresholds. Observability tells you WHY something is wrong by providing the ability to ask arbitrary questions about system state using metrics, logs, and traces. A highly observable system lets you debug novel failure modes you've never seen before without deploying new instrumentation.

**Q: What are the four golden signals?**

> Latency, Traffic, Errors, and Saturation. These four signals, from Google's SRE Book, give you enough information to detect virtually any user-visible problem. Latency: how long requests take (separately track error latency). Traffic: demand on the system (req/sec). Errors: rate of failed requests. Saturation: how "full" the service is (CPU %, queue depth, disk usage).

### Advanced

**Q: Explain how Prometheus handles counter resets.**

> When a process restarts, counters reset to 0. The `rate()` function detects resets: if a sample is smaller than the previous sample, it assumes a reset occurred and adjusts the calculation accordingly. This is why you should always use `rate()` or `increase()` on counters, never raw counter values in graphs.

**Q: How would you debug high p99 latency that only affects 1% of users?**

> 1. **Identify the pattern**: Use histogram_quantile with breakdowns by endpoint, region, user_id to isolate the scope. 2. **Correlate with traces**: Find slow traces in Jaeger/Tempo that match the time window — they'll show which span is slow. 3. **Check logs**: Correlate trace_id from slow traces with logs to find errors or slow queries. 4. **Hypotheses**: Hot partition in DB (specific user's data on slow shard), GC pauses, connection pool exhaustion for specific backends, CDN miss patterns, specific payload sizes triggering slow code paths. 5. **Resolution**: Depends on root cause — add DB indexes, tune JVM GC, increase connection pool, add caching layer.

**Q: Design an observability stack for a 100-microservice system.**

> - **Metrics**: Prometheus per cluster + Thanos for global view and long-term storage (2 years). Alertmanager with PagerDuty integration, tiered severity routing.
> - **Logs**: Loki (or ELK for compliance/search requirements). Structured JSON logging standard enforced. Log correlation by trace_id.
> - **Traces**: OTel collector as sidecar/DaemonSet, Tempo or Jaeger for storage. 1% sampling for normal traffic, 100% for errors.
> - **Dashboards**: Grafana with USE method (Utilization, Saturation, Errors) per service, RED method (Rate, Errors, Duration) for APIs, auto-provisioned via GitOps.
> - **SLOs**: Pyrra or Sloth for SLO tracking. Error budget burn rate alerts.
> - **On-call**: Runbooks linked from every alert. Incident.io or PagerDuty for rotation.
```
# Phase 8: Cloud & Security

---

## 8.1 Cloud Computing Fundamentals

### Service Models

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLOUD SERVICE MODELS                         │
│                                                                   │
│  IaaS                  PaaS                    SaaS             │
│  (Infrastructure)      (Platform)              (Software)        │
│                                                                   │
│  YOU manage:           YOU manage:             YOU manage:       │
│  ├── Applications      ├── Applications        └── (nothing)    │
│  ├── Data              ├── Data                                  │
│  ├── Runtime           └── (rest is cloud)     Examples:         │
│  ├── Middleware                                 Gmail, Salesforce │
│  └── OS                Examples:               Slack, Zoom       │
│                         Heroku, GCP App Engine                   │
│  CLOUD manages:         AWS Elastic Beanstalk                   │
│  ├── Virtualization                                              │
│  ├── Servers           CLOUD manages:                            │
│  ├── Storage           ├── Runtime                               │
│  └── Networking        ├── Middleware                            │
│                        ├── OS                                    │
│  Examples:             ├── Virtualization                        │
│  AWS EC2, Azure VM     ├── Servers                               │
│  GCP Compute Engine    ├── Storage                               │
│                        └── Networking                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.2 AWS — Deep Dive

### AWS Global Infrastructure

```
REGION (us-east-1, eu-west-1, ap-southeast-1...)
  └── AVAILABILITY ZONE (us-east-1a, 1b, 1c)
        └── DATA CENTERS (multiple physical buildings)
              └── SERVERS

EDGE LOCATIONS: 400+ POPs for CloudFront CDN
LOCAL ZONES: AWS infrastructure closer to specific cities
WAVELENGTH ZONES: AWS infra embedded in 5G networks
OUTPOSTS: AWS rack in your on-premises data center
```

### VPC — Virtual Private Cloud (Most Critical Service)

```
REGION: us-east-1
┌──────────────────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                                                │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  AVAILABILITY ZONE: us-east-1a                              │  │
│  │                                                              │  │
│  │  ┌─────────────────┐    ┌─────────────────┐               │  │
│  │  │ Public Subnet    │    │ Private Subnet   │               │  │
│  │  │ 10.0.1.0/24     │    │ 10.0.11.0/24    │               │  │
│  │  │                  │    │                  │               │  │
│  │  │ [NAT Gateway]    │    │ [EC2 App]        │               │  │
│  │  │ [Load Balancer]  │    │ [RDS Primary]    │               │  │
│  │  │ [Bastion Host]   │    │ [ElastiCache]    │               │  │
│  │  └─────────────────┘    └─────────────────┘               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  AVAILABILITY ZONE: us-east-1b                              │  │
│  │  ┌─────────────────┐    ┌─────────────────┐               │  │
│  │  │ Public Subnet    │    │ Private Subnet   │               │  │
│  │  │ 10.0.2.0/24     │    │ 10.0.12.0/24    │               │  │
│  │  │ [NAT Gateway]    │    │ [EC2 App]        │               │  │
│  │  └─────────────────┘    │ [RDS Standby]    │               │  │
│  │                          └─────────────────┘               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  [Internet Gateway] — connects VPC to internet                    │
│  [Route Tables] — controls traffic routing                        │
│  [Security Groups] — stateful firewall per resource              │
│  [NACLs] — stateless firewall per subnet                         │
│  [VPC Endpoints] — private access to AWS services (no internet)  │
└──────────────────────────────────────────────────────────────────┘
```

**Network Flow — User to App:**

```
User (Internet)
      │
      ▼ HTTPS:443
[Route 53 DNS] → resolves to ALB IP
      │
      ▼
[Internet Gateway] → entry point to VPC
      │
      ▼
[Application Load Balancer] (in Public Subnets, across AZs)
      │  Security Group: allow 80,443 from 0.0.0.0/0
      │
      ▼ HTTP:8080
[EC2 / ECS / EKS] (in Private Subnets)
      │  Security Group: allow 8080 from ALB Security Group only
      │
      ▼ TCP:5432
[RDS PostgreSQL] (in Private Subnets)
      │  Security Group: allow 5432 from App Security Group only
      │
      ▼ (outbound only, no direct internet access)
[NAT Gateway] (in Public Subnet) → Internet for outbound calls
```

### Core AWS Services Reference

```
COMPUTE:
  EC2          Virtual machines
  ECS          Container orchestration (uses EC2 or Fargate)
  EKS          Managed Kubernetes
  Fargate      Serverless containers (no EC2 management)
  Lambda       Serverless functions (event-driven, 15min max)
  Batch        Batch computing jobs
  Lightsail    Simple VPS (beginner-friendly)

STORAGE:
  S3           Object storage (unlimited, any file)
  EBS          Block storage (attached to EC2, like a hard drive)
  EFS          NFS shared file system (multi-EC2)
  FSx          Windows File Server / Lustre (HPC)
  S3 Glacier   Archival storage (cheap, slow retrieval)
  AWS Backup   Centralized backup service

DATABASE:
  RDS          Managed relational DB (MySQL, PG, MariaDB, Oracle, SQL Server)
  Aurora       AWS-optimized MySQL/PostgreSQL (5x faster, auto-scaling)
  DynamoDB     Managed NoSQL (key-value + document, millisecond latency)
  ElastiCache  Managed Redis/Memcached
  Redshift     Data warehouse (columnar, petabyte scale)
  DocumentDB   MongoDB-compatible
  Neptune      Graph database
  Timestream   Time-series database

NETWORKING:
  VPC          Virtual private cloud
  Route 53     DNS + health checks + routing policies
  CloudFront   CDN (edge caching)
  ELB          Load balancing (ALB=L7, NLB=L4, GWLB=L3)
  Direct Connect  Dedicated private link from on-prem to AWS
  VPN          Site-to-site or client VPN
  Transit Gateway  Hub-and-spoke VPC interconnect
  PrivateLink  Private connectivity to SaaS services

SECURITY:
  IAM          Identity & Access Management
  KMS          Key Management Service (encryption keys)
  Secrets Manager  Store and rotate secrets
  Certificate Manager  Free SSL/TLS certs
  WAF          Web Application Firewall
  Shield       DDoS protection
  GuardDuty    Threat detection (ML-based)
  Security Hub  Security posture management
  Macie        S3 data classification / PII detection
  Inspector    Vulnerability scanning

MONITORING:
  CloudWatch   Metrics, logs, alarms, dashboards
  CloudTrail   API audit log (who did what, when, from where)
  Config       Resource configuration history + compliance
  X-Ray        Distributed tracing

DEVELOPER TOOLS:
  CodeCommit   Git repositories
  CodeBuild    CI build service
  CodeDeploy   Deployment automation
  CodePipeline CI/CD pipeline
  ECR          Container registry
  SAM          Serverless Application Model

MESSAGING:
  SQS          Message queue (decoupling)
  SNS          Pub/sub notifications
  EventBridge  Event bus (formerly CloudWatch Events)
  Kinesis      Real-time streaming data
  MSK          Managed Kafka
```

### EKS — Elastic Kubernetes Service

```
EKS Architecture:

AWS MANAGED CONTROL PLANE (you pay per cluster/hour)
┌─────────────────────────────────────────────────────┐
│  kube-apiserver (multi-AZ, highly available)         │
│  etcd (multi-AZ, encrypted)                          │
│  kube-scheduler, controller-manager (AWS managed)    │
└─────────────────────────────────────────────────────┘
                        │
                   (AWS VPC endpoint)
                        │
YOUR VPC (worker nodes)
┌─────────────────────────────────────────────────────┐
│  Node Group 1 (EC2 + Auto Scaling Group)             │
│  ├── Node (EC2 t3.large) — kubelet + kube-proxy     │
│  ├── Node (EC2 t3.large)                             │
│  └── Node (EC2 t3.large)                             │
│                                                       │
│  Node Group 2 (GPU nodes for ML workloads)           │
│  ├── Node (EC2 p3.2xlarge)                           │
│  └── Node (EC2 p3.2xlarge)                           │
│                                                       │
│  Fargate Profile (serverless, no EC2)                │
│  └── namespace: serverless-apps → runs on Fargate    │
└─────────────────────────────────────────────────────┘
```

```bash
# EKS Setup with eksctl
cat > cluster.yaml << 'EOF'
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: production
  region: us-east-1
  version: "1.28"
  tags:
    Environment: production

vpc:
  subnets:
    private:
      us-east-1a: { id: subnet-xxx }
      us-east-1b: { id: subnet-yyy }
    public:
      us-east-1a: { id: subnet-aaa }
      us-east-1b: { id: subnet-bbb }

managedNodeGroups:
  - name: general
    instanceType: t3.large
    minSize: 3
    maxSize: 20
    desiredCapacity: 5
    privateNetworking: true      # Nodes in private subnets
    volumeSize: 100
    volumeType: gp3
    iam:
      withAddonPolicies:
        autoScaler: true
        awsLoadBalancerController: true
        certManager: true
        ebs: true
        efs: true
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/production: "owned"
  
  - name: spot-workers
    instanceTypes: ["t3.large", "t3a.large", "t3.xlarge"]
    spot: true                   # 70% cost savings
    minSize: 0
    maxSize: 50
    privateNetworking: true

addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest

iam:
  withOIDC: true               # Enable IRSA
EOF

eksctl create cluster -f cluster.yaml
```

### IAM — Identity and Access Management (Critical)

**IAM Policy Structure:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-bucket",
        "arn:aws:s3:::my-company-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        },
        "Bool": {
          "aws:SecureTransport": "true"
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-company-bucket/*"
    }
  ]
}
```

**IAM Policy Evaluation Logic:**

```
Is there an explicit DENY? ──── YES ──→ DENY (final)
         │
         NO
         │
         ▼
Is there a matching ALLOW? ──── NO ───→ DENY (implicit)
         │
         YES
         │
         ▼
Is there an SCP (Service Control Policy)? ──── YES, and it DENIES → DENY
         │
         NO (or allows)
         │
         ▼
      ALLOW ✓
```

**IRSA — IAM Roles for Service Accounts (EKS):**

```
Without IRSA:                   With IRSA:
EC2 Instance Role               Pod-level IAM role
ALL pods on node get            ONLY specific pods get
SAME permissions                specific permissions
                                Least privilege!

How IRSA works:
1. EKS cluster has OIDC provider
2. ServiceAccount annotated with IAM role ARN
3. Pod mounts projected service account token
4. AWS SDK calls STS AssumeRoleWithWebIdentity
5. Gets temporary credentials for that IAM role
```

```yaml
# Pod with IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/s3-reader-role
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: s3-reader  # Uses the role above
      containers:
        - name: app
          image: my-app:latest
          # AWS SDK automatically picks up IRSA credentials
          # via AWS_WEB_IDENTITY_TOKEN_FILE + AWS_ROLE_ARN env vars
```

---

## 8.3 HashiCorp Vault — Secrets Management

### What is Vault?

Vault is a secrets management tool that provides secure, dynamic, short-lived credentials rather than static long-lived secrets.

### Vault Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                       VAULT SERVER                              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                     API (HTTPS :8200)                     │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐  │
│  │ Auth       │  │ Secret     │  │ Audit                  │  │
│  │ Methods    │  │ Engines    │  │ Backend                │  │
│  │            │  │            │  │                        │  │
│  │ JWT/OIDC   │  │ KV v2      │  │ File/Syslog           │  │
│  │ Kubernetes │  │ AWS        │  │ Every API call logged  │  │
│  │ AWS IAM    │  │ Database   │  └────────────────────────┘  │
│  │ GitHub     │  │ PKI        │                               │
│  │ LDAP       │  │ Transit    │                               │
│  └────────────┘  └────────────┘                               │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    STORAGE BACKEND                        │ │
│  │  Raft (built-in HA) | Consul | DynamoDB | S3             │ │
│  │  ALL data encrypted at rest with master key              │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘

VAULT IS SEALED ON START — must be unsealed with key shares
Shamir's Secret Sharing: 5 key shares, 3 needed to unseal
(or auto-unseal using AWS KMS / GCP KMS)
```

### Dynamic Secrets — The Key Feature

```
TRADITIONAL (static):               VAULT (dynamic):
app gets DB password                app requests credentials
password never expires              Vault creates temp DB user
password is shared                  user has 1-hour TTL
if leaked → problem forever         if leaked → expires in 1 hour
rotation is painful                 Vault auto-rotates
                                    app never knows the password
                                    
Flow:
App → Vault API: "I need DB creds"
Vault: authenticates app (K8s SA token)
Vault → PostgreSQL: CREATE USER vault_app_xyz WITH PASSWORD '...'
Vault → App: {username: "vault_app_xyz", password: "abc123", lease_ttl: "1h"}
[1 hour later]
Vault → PostgreSQL: DROP USER vault_app_xyz
App requests new creds (or Vault SDK auto-renews)
```

### Vault Kubernetes Integration

```bash
# Enable Kubernetes auth method
vault auth enable kubernetes

vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create policy
vault policy write app-policy - <<EOF
path "secret/data/production/app/*" {
  capabilities = ["read"]
}
path "database/creds/app-role" {
  capabilities = ["read"]
}
EOF

# Bind policy to Kubernetes service account
vault write auth/kubernetes/role/app-role \
    bound_service_account_names=my-app \
    bound_service_account_namespaces=production \
    policies=app-policy \
    ttl=1h
```

```yaml
# Using Vault Agent Injector (sidecar pattern)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "app-role"
        
        # Inject a secret as a file
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/production/app/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/production/app/config" -}}
          DATABASE_URL=postgresql://{{ .Data.data.db_user }}:{{ .Data.data.db_pass }}@db:5432/mydb
          API_KEY={{ .Data.data.api_key }}
          {{- end }}
    spec:
      serviceAccountName: my-app
      containers:
        - name: app
          image: my-app:latest
          # Vault Agent sidecar writes secrets to /vault/secrets/config
          # App reads from that file
          env:
            - name: CONFIG_FILE
              value: /vault/secrets/config
```

---

## 8.4 Zero Trust Security

### Principles

```
OLD MODEL (Perimeter Security):
Internet → Firewall → TRUSTED ZONE (everything inside trusted)
Problem: Once attacker is inside, free reign

ZERO TRUST MODEL:
"Never trust, always verify"
Every request authenticated + authorized, regardless of network location
Even internal traffic is untrusted

ZERO TRUST PILLARS:
1. Identity Verification    — strong auth for every user/device/service
2. Least Privilege Access  — minimum permissions needed, nothing more
3. Microsegmentation       — network isolated per workload
4. Continuous Validation   — re-verify, don't assume
5. Assume Breach           — monitor, detect, respond
```

### Zero Trust in Kubernetes

```
mTLS (mutual TLS) via Service Mesh (Istio/Linkerd):
  Every service has a certificate (SPIFFE/SPIRE)
  Connections require certificate validation BOTH ways
  
  Service A → [presents cert] → Service B
  Service B → [validates cert] → Service A  
  Service B → [presents cert] → Service A
  Service A → [validates cert] → Service B
  Connection established (fully authenticated)
  
RBAC: Every action requires explicit permission
  User/ServiceAccount → Role → Permission
  
Network Policies: Default deny, explicit allow
  All pods isolated by default
  Allow only required paths
  
Pod Security Standards: Restrict container capabilities
  Non-root user, no privileged mode, read-only filesystem
```

---

## 8.5 Multi-Cloud and Cost Optimization

### Multi-Cloud Strategy

```
WHY MULTI-CLOUD:
  ├── Avoid vendor lock-in
  ├── Best-of-breed services (GCP BigQuery, AWS Lambda)
  ├── Regulatory compliance (data residency)
  ├── Disaster recovery (entire cloud region failure)
  └── Negotiation leverage

CHALLENGES:
  ├── Data egress costs ($$$ to transfer between clouds)
  ├── Different APIs, CLIs, IAM models
  ├── Networking complexity
  ├── Operational overhead
  └── Security policy consistency

TOOLS FOR MULTI-CLOUD:
  Kubernetes     — same workload definition across clouds
  Terraform      — provision infrastructure on any cloud
  Crossplane     — K8s-native cloud resource management
  Anthos (GCP)   — multi-cloud K8s management
  Istio          — service mesh across clusters/clouds
```

### Cost Optimization

```
COMPUTE:
  ├── Right-sizing: Use CloudWatch/metrics to find overprovisioned instances
  ├── Reserved Instances: 1-3yr commit → 30-60% savings
  ├── Savings Plans: Compute/EC2, flexible commitment
  ├── Spot Instances: 70-90% off, can be interrupted (use for batch/K8s)
  └── Graviton (ARM): 20% cheaper, 40% better perf than x86

STORAGE:
  ├── S3 Intelligent-Tiering: auto-move to cheaper tier if not accessed
  ├── EBS gp3 vs gp2: gp3 is cheaper, provision IOPS separately
  ├── Delete unattached EBS volumes and old snapshots
  └── S3 Lifecycle rules: move old data to Glacier

NETWORKING:
  ├── VPC Endpoints: avoid NAT Gateway costs for AWS API calls
  ├── CloudFront: cache at edge, reduce origin load
  ├── Same-AZ traffic: free; cross-AZ: $0.01/GB (place DB in same AZ as app)
  └── Data Transfer: same-region free between most services

TAGGING STRATEGY:
  Every resource tagged:
  ├── Environment: production/staging/dev
  ├── Team: backend/frontend/data
  ├── Project: order-service/payment-service
  ├── ManagedBy: terraform/manual
  └── CostCenter: engineering/marketing
  
  → Use Cost Explorer + tag-based cost allocation reports
  → Set budget alerts per team/project
```

---

## 8.6 AWS Interview Questions

### Beginner

**Q: What is the difference between Security Groups and NACLs?**

> Security Groups are **stateful** firewalls at the resource level (EC2, RDS). If you allow inbound on port 443, the return traffic is automatically allowed. NACLs are **stateless** subnet-level firewalls — you must explicitly allow both inbound AND outbound rules. Security Groups are the primary tool; NACLs add an extra layer. Security Groups default to "deny all" with explicit allows; NACLs default to "allow all."

**Q: What is the difference between S3 and EBS?**

> EBS is block storage — like a hard drive, attached to exactly ONE EC2 instance at a time, accessible as a filesystem, used for OS and databases. S3 is object storage — files stored as objects, accessible via HTTP API, virtually unlimited scale, accessed by any number of clients, no filesystem (key-value model). EBS is for "I need a disk for my EC2." S3 is for "I need to store files accessible over the internet or by many services."

### Intermediate

**Q: How does an Application Load Balancer route traffic?**

> ALB (L7) receives HTTPS traffic, terminates TLS, then routes based on: host headers (`api.example.com` vs `app.example.com`), path prefixes (`/api/*` vs `/static/*`), HTTP method, headers, or query parameters. Each rule forwards to a Target Group (EC2 instances, ECS tasks, Lambda, IP addresses, or another ALB). Target groups perform health checks and only route to healthy targets. ALB supports sticky sessions, weighted routing (blue-green canary), and WebSocket.

**Q: Explain VPC peering vs Transit Gateway.**

> VPC Peering connects two VPCs directly (one-to-one), non-transitive (A↔B, B↔C doesn't mean A↔C). Cheap, simple, good for 2-3 VPCs. Transit Gateway is a hub-and-spoke hub — all VPCs/VPNs/Direct Connect attach to one TGW, and TGW routes between them. Transitive by design. For 5+ VPCs, TGW is simpler to manage despite higher cost. TGW also supports multi-account and inter-region peering.

### Advanced

**Q: Design a highly available, disaster-recoverable database architecture on AWS.**

> **Architecture:**
> - Aurora PostgreSQL with Multi-AZ (synchronous replication to standby in different AZ — automatic failover in ~30s)
> - Aurora Global Database for cross-region replication (~1s replication lag)
> - Read replicas in same region for read scaling
> - Automated backups to S3 with 35-day retention, plus manual snapshots before major changes
> - Parameter groups and option groups version-controlled in Terraform
>
> **RTO/RPO:**
> - Same-region AZ failure: RTO ~30s, RPO ~0 (synchronous)
> - Region failure: RTO ~1min (promote global DB reader), RPO ~1s
>
> **Security:**
> - Private subnets only, no public access
> - Security group: only app servers allowed
> - KMS encryption at rest
> - SSL/TLS enforced (parameter `rds.force_ssl=1`)
> - IAM database authentication for application access
> - Audit logging enabled (pgaudit)

---

## 8.7 Security Deep Dive

### Secrets Management Best Practices

```
NEVER:
  ├── Hard-code secrets in source code
  ├── Put secrets in environment variable files committed to git
  ├── Log secrets (check logging framework for accidental logging)
  ├── Pass secrets as command-line arguments (visible in ps aux)
  └── Store secrets in Docker images or Kubernetes ConfigMaps

ALWAYS:
  ├── Use a secrets manager (Vault, AWS Secrets Manager, GCP Secret Manager)
  ├── Rotate secrets regularly (automate it!)
  ├── Use dynamic secrets with TTL when possible
  ├── Audit who accesses secrets (CloudTrail, Vault audit log)
  ├── Use least-privilege IAM to access secrets
  └── Encrypt secrets at rest AND in transit

GIT SECRET DETECTION:
  ├── pre-commit hooks: detect-secrets, gitleaks
  ├── CI/CD scanning: truffleHog, GitGuardian
  ├── If secret committed: immediately rotate it, rewrite git history
  └── git filter-branch or BFG Repo-Cleaner to remove from history
```

### Container Security

```bash
# Dockerfile security best practices

# Use specific digest, not just tag (immutable)
FROM python:3.11.7-slim-bookworm@sha256:abc123...

# Run as non-root
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Don't cache secrets in build layers
# BAD:
RUN pip install -r requirements.txt --index-url https://user:password@pypi.internal/
# GOOD:
RUN --mount=type=secret,id=pip_config,target=/etc/pip.conf \
    pip install -r requirements.txt

# Minimal image = minimal attack surface
# Multi-stage: build stage vs runtime stage
FROM python:3.11 AS builder
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.11-slim AS runtime  # Much smaller!
COPY --from=builder /usr/local/lib/python3.11 /usr/local/lib/python3.11
COPY --from=builder /usr/local/bin /usr/local/bin
COPY app/ /app/
USER appuser
EXPOSE 8080
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0"]

# Kubernetes Pod Security
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 3000
  readOnlyRootFilesystem: true    # Can't write to container FS
  allowPrivilegeEscalation: false  # Can't sudo
  capabilities:
    drop: ["ALL"]                  # Drop all Linux capabilities
    add: ["NET_BIND_SERVICE"]      # Only add what's needed (bind <1024 ports)
  seccompProfile:
    type: RuntimeDefault           # Linux system call filtering
```

### Supply Chain Security (SLSA / Sigstore)

```
SOFTWARE SUPPLY CHAIN ATTACKS:
  SolarWinds (2020): Malicious code injected into build process
  log4shell (2021): Vulnerability in widely-used library
  
PROTECTION:
  
  1. SBOM (Software Bill of Materials)
     List of all components + versions in your software
     Tools: Syft, Trivy, SPDX/CycloneDX formats
     
  2. Image Signing (Cosign / Sigstore)
     # Sign image after build
     cosign sign --key cosign.key myregistry/myapp:v1.0.0
     
     # Verify before deploy (in admission controller)
     cosign verify --key cosign.pub myregistry/myapp:v1.0.0
     
  3. Vulnerability Scanning
     # Scan at build time
     trivy image myapp:v1.0.0 --exit-code 1 --severity HIGH,CRITICAL
     
     # Scan running containers continuously
     # Tools: Falco (runtime), Aqua, Sysdig, Snyk
     
  4. SLSA Framework (Supply chain Levels for Software Artifacts)
     Level 1: Build scripted, provenance generated
     Level 2: Build service used, signed provenance
     Level 3: Build service hardened, provenance authenticated
     Level 4: Two-party review, hermetic builds
```

---

## 8.8 DevSecOps — Security in CI/CD

```
PIPELINE WITH SECURITY GATES:

Code Push
    │
    ▼
[Pre-commit hooks]
├── detect-secrets (no secrets committed)
├── gitleaks (git history scan)
└── SAST: semgrep, bandit (quick static analysis)
    │
    ▼
[CI: Pull Request]
├── SAST: SonarQube, Checkmarx, Semgrep (full scan)
├── Dependency scanning: Snyk, OWASP Dependency Check
├── License compliance: FOSSA
└── Code review (human, 2-person rule for prod)
    │
    ▼
[CI: Build]
├── Docker build
├── Trivy image scan (HIGH/CRITICAL = FAIL)
├── Cosign image signing
└── Generate SBOM (Syft)
    │
    ▼
[CD: Staging Deploy]
├── DAST: OWASP ZAP (dynamic scan against running app)
├── Integration tests
└── Penetration testing (periodic, not every deploy)
    │
    ▼
[CD: Production Deploy] (with approval gate)
├── Cosign verify (ensure signed image)
├── OPA/Gatekeeper policies enforced
├── Blue-green or canary deployment
└── Runtime security: Falco monitoring
    │
    ▼
[Runtime Monitoring]
├── Falco: detect unexpected syscalls/behaviors
├── GuardDuty: AWS threat detection
├── CloudTrail: audit every API call
└── SIEM: centralized security event correlation
```

---

## 8.9 Final System Design: Production-Grade Platform

```
COMPLETE PRODUCTION ARCHITECTURE:

                          ┌────────────────┐
                          │   Route 53     │
                          │   (DNS + HC)   │
                          └───────┬────────┘
                                  │
                          ┌───────▼────────┐
                          │   CloudFront   │
                          │   (CDN + WAF)  │
                          └───────┬────────┘
                                  │
              ┌───────────────────▼──────────────────┐
              │          Application Load Balancer    │
              │          (ACM SSL, multi-AZ)          │
              └──────┬────────────────────────┬───────┘
                     │                        │
          ┌──────────▼──────┐      ┌──────────▼──────┐
          │   EKS Cluster   │      │   EKS Cluster   │
          │   us-east-1     │      │   us-west-2     │
          │                 │      │   (DR/standby)  │
          │  [Istio mesh]   │      │                 │
          │  [ArgoCD]       │      │  [ArgoCD]       │
          │  [Prometheus]   │      │  [Prometheus]   │
          │  [app pods]     │      │  [app pods]     │
          └────────┬────────┘      └─────────────────┘
                   │
         ┌─────────┼──────────┐
         │         │          │
    ┌────▼───┐ ┌───▼───┐ ┌───▼─────────┐
    │Aurora  │ │ElastiC│ │  Kafka MSK  │
    │Global  │ │ache   │ │  (events)   │
    │(PG)    │ │(Redis)│ │             │
    └────────┘ └───────┘ └─────────────┘

GitOps Flow:
Developer → Git PR → Review → Merge
→ GitHub Actions CI (test+build+scan+sign)
→ Push image to ECR
→ Update Helm values in GitOps repo
→ ArgoCD detects diff → auto-deploys to staging
→ Manual approval for production
→ ArgoCD deploys to production
→ Prometheus + Grafana monitoring
→ PagerDuty if SLO breach
→ Runbook → Resolution → Post-mortem
```

---

## 8.10 Career Interview Preparation

### System Design Questions

**Q: Design a CI/CD pipeline for a microservices application with 50 services.**

> **Answer Framework:**
> 1. **Source Control**: Mono-repo (Nx/Turborepo for affected-only builds) or poly-repo (per-service pipelines)
> 2. **CI per service**: On PR, run tests → SAST → build Docker image → scan → push to ECR with SHA tag
> 3. **Artifact versioning**: Immutable image tags (git SHA), Helm chart versioning
> 4. **GitOps CD**: ArgoCD app-of-apps pattern, environment promotions via PR to GitOps repo
> 5. **Deployment strategy**: Blue-green for stateless, canary for high-risk changes (Argo Rollouts)
> 6. **Testing gates**: Integration tests in staging, smoke tests post-deploy to prod
> 7. **Rollback**: ArgoCD one-click rollback (reverts Helm release), plus feature flags for instant disable
> 8. **Secrets**: External Secrets Operator pulling from Vault/AWS Secrets Manager
> 9. **Observability**: Deploy updates trigger Grafana annotations, watch error rate for 15min post-deploy

**Q: How would you handle a production outage?**

> **Incident Response Process:**
> 1. **Detect**: Alert fires via PagerDuty from Prometheus/CloudWatch
> 2. **Acknowledge**: On-call engineer ACKs within SLA (5min for P1)
> 3. **Communicate**: Post in #incidents Slack, update status page (Atlassian Statuspage)
> 4. **Triage**: What's broken? What's the blast radius? How many users affected?
> 5. **Mitigate first**: Rollback deployment, increase capacity, failover to DR — stop the bleeding BEFORE root cause analysis
> 6. **Root cause**: Logs → traces → metrics → code → infra
> 7. **Fix**: Deploy fix, verify metrics return to normal
> 8. **Post-mortem**: Blameless, within 48h, 5-whys, action items with owners and due dates
> 9. **Prevention**: Implement action items (alerting improvements, runbooks, code fixes, architecture changes)

### DevOps Behavioral Questions

**Q: Tell me about a time you improved system reliability.**

> Framework: Situation → Task → Action → Result
> Example: "Our checkout service had 99.5% availability (4.4h downtime/month). I implemented health checks + HPA + PodDisruptionBudgets to ensure rolling updates never brought us below 2 replicas. Added circuit breakers to payment provider calls. Set up synthetic monitoring. Result: Improved to 99.95% availability (4.4h → 26min/month downtime). Reduced MTTR from 45min to 8min with runbook automation."

### Key Technical Depth Areas for Senior Roles

```
DEPTH QUESTIONS SENIOR DEVOPS/SRE MUST ANSWER:

1. "How does iptables implement Kubernetes Services?"
   → iptables DNAT rules, kube-proxy modes (iptables vs IPVS)

2. "Explain etcd Raft consensus and why split-brain matters"
   → Leader election, quorum (n/2+1), 3 vs 5 nodes tradeoffs

3. "How does Kubernetes scheduler choose a node?"
   → Filtering (resource fit, taints, affinity), Scoring (spread, pack),
     Binding, watch mechanism

4. "How do you debug a pod that's OOMKilled?"
   → kubectl describe pod, check limits vs requests, heapster/metrics-server,
     analyze memory leak in app (pprof for Go, jmap for Java)

5. "What causes etcd compaction and backup?"
   → MVCC keeps all historical revisions, compact removes old revisions,
     backup: etcdctl snapshot save, restore with etcdctl snapshot restore

6. "How does Prometheus TSDB work under the hood?"
   → Chunks, head block, WAL, compaction, index structure

7. "Explain VPC routing when an EC2 in a private subnet calls S3"
   → Private subnet → route table → NAT Gateway (public subnet) →
     IGW → S3 public endpoint
     OR: VPC Endpoint for S3 (stays in AWS network, cheaper, faster)
```
# Phase 9: Advanced Kubernetes — Operators, CRDs, Multi-Cluster

---

## 9.1 The Kubernetes Extension Model

Kubernetes was designed to be extended. Every cloud-native tool you use (ArgoCD, Prometheus Operator, cert-manager, Istio) is built on these extension points.

```
KUBERNETES EXTENSION POINTS:
                                                           
  API Server Extensions:          Webhook Extensions:
  ├── CRDs (Custom Resources)     ├── MutatingAdmissionWebhook
  ├── Aggregated APIs             └── ValidatingAdmissionWebhook
  └── APIServices                  
                                   Scheduler Extensions:
  Controller Extensions:           ├── Scheduler plugins
  └── Custom Controllers           └── Multiple schedulers
      └── Operators (CRD + CC)
```

---

## 9.2 Custom Resource Definitions (CRDs)

### What is a CRD?

A CRD extends the Kubernetes API to add your own resource types. After installing a CRD, your custom resource behaves like any native Kubernetes resource — you can `kubectl get`, `kubectl apply`, RBAC it, and watch it.

```yaml
# database.example.com_postgresclusters.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.database.example.com
spec:
  group: database.example.com
  versions:
    - name: v1alpha1
      served: true
      storage: true    # Only ONE version is the storage version
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["instances", "postgresVersion"]
              properties:
                instances:
                  type: integer
                  minimum: 1
                  maximum: 10
                  description: "Number of PostgreSQL instances"
                postgresVersion:
                  type: string
                  pattern: '^[0-9]+\.[0-9]+$'
                  description: "PostgreSQL major.minor version"
                storageSize:
                  type: string
                  default: "10Gi"
                  description: "Storage size per instance"
                backupSchedule:
                  type: string
                  description: "Cron schedule for backups"
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: [Pending, Initializing, Ready, Degraded, Failed]
                readyInstances:
                  type: integer
                primaryInstance:
                  type: string
                message:
                  type: string
      additionalPrinterColumns:     # kubectl get output columns
        - name: Instances
          type: integer
          jsonPath: .spec.instances
        - name: Ready
          type: integer
          jsonPath: .status.readyInstances
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
      subresources:
        status: {}    # Separate status subresource (controller updates status)
  scope: Namespaced
  names:
    plural: postgresclusters
    singular: postgrescluster
    kind: PostgresCluster
    shortNames: ["pg", "pgc"]    # kubectl get pg
```

**Using the custom resource:**

```yaml
# my-postgres.yaml
apiVersion: database.example.com/v1alpha1
kind: PostgresCluster
metadata:
  name: production-db
  namespace: production
spec:
  instances: 3
  postgresVersion: "15.4"
  storageSize: "100Gi"
  backupSchedule: "0 2 * * *"
```

```bash
kubectl apply -f my-postgres.yaml
kubectl get postgresclusters -n production
kubectl get pg -n production   # Short name works!
kubectl describe pg production-db -n production
```

---

## 9.3 Building a Kubernetes Operator

### What is an Operator?

An Operator = **CRD** (what to manage) + **Custom Controller** (how to manage it).

The operator encodes the operational knowledge of a human expert into code. It watches custom resources and reconciles the actual state with the desired state.

```
OPERATOR PATTERN:

User applies CR:
apiVersion: database.example.com/v1alpha1
kind: PostgresCluster
spec:
  instances: 3
         │
         ▼
  Kubernetes API Server stores CR in etcd
         │
         ▼ (watch event)
  OPERATOR (Custom Controller)
  ├── Reads desired state: instances=3
  ├── Checks actual state: running instances=1 (just created)
  ├── Delta: need 2 more instances
  ├── Creates StatefulSets, Services, ConfigMaps, PVCs
  ├── Creates secrets (postgres passwords)
  ├── Configures replication between instances
  ├── Updates CR status: phase=Initializing, readyInstances=1
  │
  └── [Later, instances=3 running]
      Updates CR status: phase=Ready, readyInstances=3, primaryInstance=production-db-0
```

### Operator Implementation — controller-runtime (Go)

```go
// controllers/postgrescluster_controller.go
package controllers

import (
    "context"
    "fmt"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"
    
    databasev1alpha1 "example.com/postgres-operator/api/v1alpha1"
)

type PostgresClusterReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=database.example.com,resources=postgresclusters,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.example.com,resources=postgresclusters/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services;configmaps;persistentvolumeclaims,verbs=get;list;watch;create;update;patch;delete

func (r *PostgresClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)
    
    // 1. Fetch the PostgresCluster resource
    cluster := &databasev1alpha1.PostgresCluster{}
    if err := r.Get(ctx, req.NamespacedName, cluster); err != nil {
        if errors.IsNotFound(err) {
            // Resource deleted — cleanup handled by ownerReferences/finalizers
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    
    // 2. Handle deletion (finalizer pattern)
    if !cluster.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, cluster)
    }
    
    // 3. Add finalizer if not present
    if !containsString(cluster.Finalizers, "database.example.com/finalizer") {
        cluster.Finalizers = append(cluster.Finalizers, "database.example.com/finalizer")
        if err := r.Update(ctx, cluster); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 4. Reconcile StatefulSet
    sts := r.buildStatefulSet(cluster)
    existing := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKeyFromObject(sts), existing)
    if errors.IsNotFound(err) {
        log.Info("Creating StatefulSet", "name", sts.Name)
        if err := ctrl.SetControllerReference(cluster, sts, r.Scheme); err != nil {
            return ctrl.Result{}, err
        }
        if err := r.Create(ctx, sts); err != nil {
            return ctrl.Result{}, err
        }
    } else if err == nil {
        // Update if spec changed
        if existing.Spec.Replicas != sts.Spec.Replicas {
            existing.Spec.Replicas = sts.Spec.Replicas
            if err := r.Update(ctx, existing); err != nil {
                return ctrl.Result{}, err
            }
        }
    } else {
        return ctrl.Result{}, err
    }
    
    // 5. Reconcile Services (primary + replica)
    if err := r.reconcileServices(ctx, cluster); err != nil {
        return ctrl.Result{}, err
    }
    
    // 6. Update status
    cluster.Status.Phase = "Ready"
    cluster.Status.ReadyInstances = int(*sts.Spec.Replicas)
    cluster.Status.PrimaryInstance = fmt.Sprintf("%s-0", cluster.Name)
    if err := r.Status().Update(ctx, cluster); err != nil {
        return ctrl.Result{}, err
    }
    
    log.Info("Reconciled PostgresCluster", "name", cluster.Name, "phase", cluster.Status.Phase)
    
    // Requeue after 30s to detect drift
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *PostgresClusterReconciler) buildStatefulSet(cluster *databasev1alpha1.PostgresCluster) *appsv1.StatefulSet {
    replicas := int32(cluster.Spec.Instances)
    return &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      cluster.Name,
            Namespace: cluster.Namespace,
        },
        Spec: appsv1.StatefulSetSpec{
            Replicas:    &replicas,
            ServiceName: cluster.Name + "-headless",
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{"app": cluster.Name},
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{"app": cluster.Name},
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "postgres",
                            Image: fmt.Sprintf("postgres:%s", cluster.Spec.PostgresVersion),
                            Ports: []corev1.ContainerPort{{ContainerPort: 5432}},
                            Env: []corev1.EnvVar{
                                {Name: "POSTGRES_DB", Value: "mydb"},
                                {Name: "POSTGRES_PASSWORD", ValueFrom: &corev1.EnvVarSource{
                                    SecretKeyRef: &corev1.SecretKeySelector{
                                        LocalObjectReference: corev1.LocalObjectReference{
                                            Name: cluster.Name + "-secret",
                                        },
                                        Key: "password",
                                    },
                                }},
                            },
                            VolumeMounts: []corev1.VolumeMount{
                                {Name: "data", MountPath: "/var/lib/postgresql/data"},
                            },
                        },
                    },
                },
            },
            VolumeClaimTemplates: []corev1.PersistentVolumeClaim{
                {
                    ObjectMeta: metav1.ObjectMeta{Name: "data"},
                    Spec: corev1.PersistentVolumeClaimSpec{
                        AccessModes: []corev1.PersistentVolumeAccessMode{corev1.ReadWriteOnce},
                        Resources: corev1.ResourceRequirements{
                            Requests: corev1.ResourceList{
                                corev1.ResourceStorage: resource.MustParse(cluster.Spec.StorageSize),
                            },
                        },
                    },
                },
            },
        },
    }
}

func (r *PostgresClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1alpha1.PostgresCluster{}).  // Watch PostgresCluster CRs
        Owns(&appsv1.StatefulSet{}).               // Watch StatefulSets we own
        Owns(&corev1.Service{}).                    // Watch Services we own
        Complete(r)
}
```

### Operator SDK — Scaffolding

```bash
# Initialize operator project
operator-sdk init \
    --domain example.com \
    --repo github.com/example/postgres-operator \
    --plugins go/v4

# Create API + controller skeleton
operator-sdk create api \
    --group database \
    --version v1alpha1 \
    --kind PostgresCluster \
    --resource \
    --controller

# Generate deepcopy methods, CRD YAML, RBAC
make generate
make manifests

# Run locally (connects to current kubectl context)
make run

# Build and push operator image
make docker-build docker-push IMG=myregistry/postgres-operator:v0.1.0

# Deploy to cluster
make deploy IMG=myregistry/postgres-operator:v0.1.0
```

---

## 9.4 Admission Webhooks

Admission webhooks intercept API server requests and can **mutate** (modify) or **validate** (accept/reject) them before resources are stored.

```
kubectl apply → kube-apiserver
                     │
           ┌─────────▼──────────┐
           │  Authentication     │
           │  Authorization      │
           │  Admission Control  │
           │  ┌───────────────┐  │
           │  │ MutatingWebhook│  │ ← Called first, can modify request
           │  │ (your webhook) │  │   e.g., inject sidecar, add labels
           │  └───────────────┘  │
           │  ┌───────────────┐  │
           │  │ValidatingWebhook│ │ ← Called second, can reject
           │  │ (your webhook) │  │   e.g., block non-signed images
           │  └───────────────┘  │
           └─────────┬───────────┘
                     │
                  etcd storage
```

```go
// Example: Mutating webhook — inject default resource limits
func (h *PodMutationHandler) Handle(ctx context.Context, req admission.Request) admission.Response {
    pod := &corev1.Pod{}
    if err := h.decoder.Decode(req, pod); err != nil {
        return admission.Errored(http.StatusBadRequest, err)
    }
    
    // Inject default resources if not set
    for i := range pod.Spec.Containers {
        c := &pod.Spec.Containers[i]
        if c.Resources.Requests == nil {
            c.Resources.Requests = corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("100m"),
                corev1.ResourceMemory: resource.MustParse("128Mi"),
            }
        }
        if c.Resources.Limits == nil {
            c.Resources.Limits = corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("500m"),
                corev1.ResourceMemory: resource.MustParse("512Mi"),
            }
        }
    }
    
    // Add company-required labels
    if pod.Labels == nil {
        pod.Labels = make(map[string]string)
    }
    pod.Labels["managed-by"] = "platform-team"
    
    marshaledPod, err := json.Marshal(pod)
    if err != nil {
        return admission.Errored(http.StatusInternalServerError, err)
    }
    return admission.PatchResponseFromRaw(req.Object.Raw, marshaledPod)
}

// Example: Validating webhook — require signed images
func (h *ImageValidationHandler) Handle(ctx context.Context, req admission.Request) admission.Response {
    pod := &corev1.Pod{}
    if err := h.decoder.Decode(req, pod); err != nil {
        return admission.Errored(http.StatusBadRequest, err)
    }
    
    for _, c := range pod.Spec.Containers {
        if !h.isImageSigned(c.Image) {
            return admission.Denied(fmt.Sprintf(
                "container %q uses unsigned image %q; all images must be signed with cosign",
                c.Name, c.Image,
            ))
        }
    }
    return admission.Allowed("all images are signed")
}
```

---

## 9.5 Multi-Cluster Kubernetes

### Why Multi-Cluster?

```
REASONS FOR MULTI-CLUSTER:
├── Blast radius isolation (production failure doesn't affect staging)
├── Geographic distribution (low latency for global users)
├── Compliance (data residency requirements)
├── Scale (single cluster limit: ~5000 nodes, 150k pods)
├── Team autonomy (each team owns their cluster)
└── Disaster recovery (active-passive or active-active)

CHALLENGES:
├── Service discovery (how does service in cluster A find service in cluster B?)
├── Networking (pods in different clusters can't talk by default)
├── Consistent policy enforcement (RBAC, NetworkPolicy, OPA)
├── Unified observability (metrics/logs from all clusters)
├── Configuration management (GitOps across clusters)
└── Secret management (shared secrets across clusters)
```

### ArgoCD Multi-Cluster Setup

```yaml
# Register external cluster with ArgoCD
argocd cluster add prod-cluster-1 \
    --kubeconfig ~/.kube/prod1.yaml \
    --name prod-us-east-1

argocd cluster add prod-cluster-2 \
    --kubeconfig ~/.kube/prod2.yaml \
    --name prod-us-west-2

# ApplicationSet — deploy to ALL clusters matching label
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-all-clusters
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production    # All clusters labeled "production"
  template:
    metadata:
      name: "{{name}}-my-app"         # {{name}} = cluster name
    spec:
      project: default
      source:
        repoURL: https://github.com/company/gitops
        targetRevision: HEAD
        path: "apps/my-app/{{metadata.labels.region}}"  # Region-specific values
      destination:
        server: "{{server}}"           # {{server}} = cluster API URL
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Cluster API (CAPI) — Cluster Lifecycle Management

```yaml
# Cluster API lets you manage cluster creation/deletion as Kubernetes resources
# Think of it as "Kubernetes to manage Kubernetes"

apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: prod-cluster-3
  namespace: clusters
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: prod-cluster-3-cp
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: prod-cluster-3
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSCluster
metadata:
  name: prod-cluster-3
spec:
  region: eu-central-1
  sshKeyName: prod-key
  network:
    vpc:
      cidrBlock: "10.0.0.0/16"
```

### Cilium Cluster Mesh — Multi-Cluster Networking

```bash
# Enable cluster mesh on each cluster
cilium clustermesh enable --service-type LoadBalancer

# Connect clusters
cilium clustermesh connect \
    --context cluster1 \
    --destination-context cluster2

# Now pods in cluster1 can reach Services in cluster2
# Service discovery via DNS:
# service-name.namespace.svc.cluster.local (same cluster)
# service-name.namespace.svc.clustername.local (cross-cluster)

# Global Services — same service across clusters, automatic failover
apiVersion: v1
kind: Service
metadata:
  name: api-server
  annotations:
    service.cilium.io/global: "true"       # Reachable from other clusters
    service.cilium.io/shared: "true"        # Load balance across clusters
spec:
  selector:
    app: api-server
  ports:
    - port: 80
```

---

## 9.6 OPA/Gatekeeper — Policy as Code

### What is OPA?

Open Policy Agent (OPA) is a general-purpose policy engine. In Kubernetes, **Gatekeeper** uses OPA as a validating admission webhook to enforce policies written in **Rego**.

```
kubectl apply (any resource)
        │
        ▼
   kube-apiserver
        │
        ▼ (ValidatingAdmissionWebhook)
   Gatekeeper
   ├── Loads ConstraintTemplates (Rego policies)
   ├── Evaluates request against all active Constraints
   └── ALLOW or DENY with message
```

```rego
# ConstraintTemplate — defines the policy logic in Rego
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requireresourcelimits
spec:
  crd:
    spec:
      names:
        kind: RequireResourceLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            exemptImages:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requireresourcelimits

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          not exemptImage(container.image)
          msg := sprintf("Container '%v' must have CPU limits", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          not exemptImage(container.image)
          msg := sprintf("Container '%v' must have memory limits", [container.name])
        }

        exemptImage(image) {
          exempt := input.parameters.exemptImages[_]
          startswith(image, exempt)
        }
---
# Constraint — applies the policy to specific resources
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireResourceLimits
metadata:
  name: must-have-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["production", "staging"]    # Only enforce in prod/staging
    excludedNamespaces: ["kube-system"]
  parameters:
    exemptImages:
      - "gcr.io/google-containers/"
      - "k8s.gcr.io/"
```

```rego
# More Gatekeeper policies:

# 1. Require specific labels
violation[{"msg": msg}] {
  required_labels := {"app", "team", "environment"}
  provided_labels := {label | input.review.object.metadata.labels[label]}
  missing := required_labels - provided_labels
  count(missing) > 0
  msg := sprintf("Missing required labels: %v", [missing])
}

# 2. Block latest tag
violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  endswith(container.image, ":latest")
  msg := sprintf("Container '%v' uses ':latest' tag which is not allowed", [container.name])
}

# 3. Require non-root
violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  not container.securityContext.runAsNonRoot
  msg := sprintf("Container '%v' must set runAsNonRoot: true", [container.name])
}
```

---

## 9.7 Kubernetes Performance Tuning

### etcd Performance

```bash
# etcd is the bottleneck for large clusters
# Key metrics to watch:
# etcd_disk_wal_fsync_duration_seconds (p99 < 10ms on SSD)
# etcd_disk_backend_commit_duration_seconds (p99 < 25ms)
# etcd_server_proposals_failed_total

# Defrag etcd (reduces disk usage, improves performance)
# ⚠️ Takes cluster offline briefly — do during maintenance
ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    defrag

# Compact etcd (free historical revisions)
# Get current revision
rev=$(ETCDCTL_API=3 etcdctl endpoint status --write-out="json" | \
    jq '.[] | .Status.header.revision')
# Compact to current revision
ETCDCTL_API=3 etcdctl compact $rev

# etcd backup (run regularly!)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db
```

### API Server Tuning

```yaml
# kube-apiserver flags for high-traffic clusters
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    # Rate limiting
    - --max-requests-inflight=400          # Default 400 — increase for large clusters
    - --max-mutating-requests-inflight=200 # Default 200
    # Watch cache
    - --watch-cache-sizes=node#1000,pod#5000  # Cache sizes per resource
    # Audit logging (selective — full audit is expensive)
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxsize=100
    - --audit-log-maxbackup=5
```

### Node Performance

```bash
# kubelet configuration for high-density nodes
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
maxPods: 250                          # Default 110 — increase with care
podsPerCore: 10                       # Limit pods per CPU core
imageGCHighThresholdPercent: 85       # GC images when disk > 85%
imageGCLowThresholdPercent: 80        # GC until disk < 80%
evictionHard:
  memory.available: "200Mi"           # Evict pods when < 200Mi free
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
systemReserved:
  cpu: "500m"
  memory: "1Gi"
  ephemeral-storage: "5Gi"
kubeReserved:
  cpu: "500m"
  memory: "1Gi"
cpuManagerPolicy: static              # Guarantee CPUs for Guaranteed QoS pods
topologyManagerPolicy: single-numa-node  # NUMA-aware scheduling for latency-sensitive
```

### Resource QoS Classes

```yaml
# GUARANTEED — requests == limits
# Highest priority, never OOMKilled unless node is full
# Scheduled to dedicated CPU (with cpuManagerPolicy: static)
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "1"
    memory: "2Gi"

# BURSTABLE — requests < limits, OR only limits set
# Can use more resources when available
# OOMKilled when node is under pressure
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# BESTEFFORT — no requests or limits set
# Lowest priority, first to be evicted
# ⚠️ Never use in production
resources: {}
```

---

## 9.8 Advanced Interview Questions

**Q: Explain the Operator pattern and when you'd build one vs use an existing tool.**

> An Operator packages human operational knowledge into code. It uses the controller pattern: watch a Custom Resource, compute the difference between desired and actual state (reconcile loop), and take actions to converge them. Build an operator when: you're managing stateful applications with complex lifecycle operations (failover, backup, upgrade), you need to encode domain expertise (how to safely upgrade PostgreSQL without data loss), or you want self-healing behavior. Use existing operators (CloudNativePG, Strimzi, Prometheus Operator) for common software — don't reinvent the wheel.

**Q: How does Gatekeeper differ from a LimitRange/ResourceQuota?**

> LimitRange sets default/max requests and limits for containers in a namespace. ResourceQuota sets total resource consumption limits for a namespace. Both are built-in and simple. Gatekeeper (OPA) is a general-purpose policy engine that can enforce ANY policy expressible in Rego: require labels, block untrusted registries, enforce naming conventions, require specific annotations, validate custom fields. The power is arbitrary policy logic, not just resource limits.

**Q: Design a multi-cluster deployment strategy for a global application.**

> Architecture: Hub-spoke with ArgoCD as the GitOps controller in a management cluster, pushing to regional workload clusters (us-east-1, eu-west-1, ap-southeast-1). Each region runs active-active for the same application. DNS handled by Route 53 latency-based routing or Cloudflare load balancing.
>
> Consistency: ApplicationSet in ArgoCD deploys same Helm chart to all clusters with region-specific values files. Single GitOps repo is the source of truth.
>
> Networking: Cilium Cluster Mesh for service-to-service communication. Or Istio multi-cluster with east-west gateway.
>
> Data: Read replicas in each region from Aurora Global Database. Writes route to primary region, eventual consistency for reads.
>
> Failover: If a region fails, DNS TTL 60s → Route 53 health check removes the region → traffic redistributes. RTO ~60-120s.
```
# Phase 10: Platform Engineering, SRE & Production Excellence

---

## 10.1 Platform Engineering

### What is Platform Engineering?

Platform Engineering builds **Internal Developer Platforms (IDPs)** — self-service infrastructure and tooling that lets application developers deploy, observe, and manage their services without deep infrastructure knowledge.

```
TRADITIONAL DEVOPS MODEL:
Dev Team → "We need a database" → Tickets to Ops → 2-week wait → Database
Dev Team → "We need to deploy" → Tickets to Ops → Wait → Deploy
Dev Team → "We need monitoring" → Tickets to Ops → Wait → Dashboard

PLATFORM ENGINEERING MODEL:
Dev Team → Internal Developer Portal → Self-service in minutes
         → Scaffolding: "Create microservice" → repo + CI + deploy + monitoring
         → "I need a database" → click → Crossplane provisions RDS in 5 minutes
         → "Deploy to prod" → PR merge → ArgoCD auto-deploys
         → "Show metrics" → pre-built Grafana dashboard auto-created
```

### The Golden Path

```
GOLDEN PATH = Opinionated, paved road for shipping software

Components:
├── Service Catalog (Backstage)
│   ├── Template: "Create a Python microservice"
│   │   → Generates repo, Dockerfile, Helm chart, CI pipeline, monitoring
│   ├── Service registry: all services, their owners, runbooks, dashboards
│   └── API catalog: all internal APIs with documentation
│
├── Self-Service Infrastructure (Crossplane / Terraform Cloud)
│   ├── "I need PostgreSQL 15 with 100GB" → YAML → RDS provisioned
│   ├── "I need Redis cache" → YAML → ElastiCache provisioned
│   └── Infrastructure as API resources in Kubernetes
│
├── Deployment Platform (ArgoCD / Flux)
│   └── Every service deploys same way: GitOps
│
├── Observability (pre-configured)
│   ├── Auto-scraped Prometheus metrics (ServiceMonitor via annotations)
│   ├── Pre-built Grafana dashboard per service template
│   └── Pre-configured alerts (error rate, latency, saturation)
│
└── Developer Portal (Backstage)
    ├── Single pane of glass for all services
    ├── CI/CD status, deployments, on-call, runbooks
    └── Cost by service/team
```

### Backstage — Developer Portal

```yaml
# catalog-info.yaml (in each service repo)
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  description: Handles all payment processing
  tags:
    - python
    - fastapi
    - payments
  annotations:
    github.com/project-slug: company/payment-service
    backstage.io/techdocs-ref: dir:.
    prometheus.io/alert-rules: "alerts/payment-service.yml"
    grafana/dashboard-url: https://grafana.company.com/d/payment-service
    pagerduty.com/service-id: P123456
spec:
  type: service
  lifecycle: production
  owner: payments-team
  system: payment-platform
  providesApis:
    - payment-api
  consumesApis:
    - fraud-detection-api
    - notification-api
  dependsOn:
    - resource:default/payment-db
    - resource:default/payment-cache
```

### Crossplane — Infrastructure as Kubernetes Resources

```yaml
# Crossplane lets you define cloud infrastructure as Kubernetes CRDs
# Platform team defines Compositions (abstract the cloud details)
# Dev teams use simple Claims (request infrastructure)

# Platform team creates a Composition (once)
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xpostgresqlinstances.aws.platform.company.com
spec:
  compositeTypeRef:
    apiVersion: platform.company.com/v1alpha1
    kind: XPostgreSQLInstance
  resources:
    - name: rds-instance
      base:
        apiVersion: rds.aws.crossplane.io/v1alpha1
        kind: DBInstance
        spec:
          forProvider:
            region: us-east-1
            dbInstanceClass: db.t3.medium
            engine: postgres
            engineVersion: "15.4"
            multiAZ: false
            storageEncrypted: true
      patches:
        - fromFieldPath: spec.parameters.storageGB
          toFieldPath: spec.forProvider.allocatedStorage
        - fromFieldPath: spec.parameters.size
          toFieldPath: spec.forProvider.dbInstanceClass
          transforms:
            - type: map
              map:
                small: db.t3.medium
                medium: db.r6g.large
                large: db.r6g.xlarge

---
# Dev team creates a Claim (simple, no cloud knowledge needed)
apiVersion: platform.company.com/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: payment-db
  namespace: production
spec:
  parameters:
    storageGB: 100
    size: medium
  writeConnectionSecretToRef:
    name: payment-db-credentials  # Created automatically
```

---

## 10.2 Site Reliability Engineering (SRE)

### SRE vs DevOps

```
SRE = "Software Engineering applied to Operations problems"
      (Google's implementation of DevOps principles)

DevOps: Cultural movement, shared responsibility between Dev and Ops
SRE:    Specific engineering role with quantitative targets (SLOs)
        "If you give me enough DevOps culture, I'll figure out SRE"

SRE Responsibilities:
├── Service Level Objectives (define what "reliable" means)
├── Error budgets (manage the cost of reliability)
├── Toil reduction (automate everything repetitive)
├── Capacity planning
├── Incident response and post-mortems
└── Production software engineering (reliability features)
```

### SLI / SLO / SLA

```
SLI (Service Level Indicator): A metric that measures service behavior
  → "The proportion of requests served successfully in under 200ms"
  → Measured value: 99.7%

SLO (Service Level Objective): The target for the SLI
  → "99.9% of requests served successfully in under 200ms over 30 days"
  → Internal agreement — what the team commits to maintaining
  → Should be LESS than what you can achieve (leave room for error budget)

SLA (Service Level Agreement): External contract with consequences
  → "We guarantee 99.9% availability, or we credit you 10%"
  → SLA should be LESS STRICT than internal SLO
  → e.g., SLO = 99.9%, SLA = 99.5%
  
ERROR BUDGET = 1 - SLO
  → 99.9% SLO = 0.1% error budget = 43.8 minutes/month of allowed downtime
  → 99.95% SLO = 0.05% = 21.9 minutes/month
  → 99.99% SLO = 0.01% = 4.4 minutes/month
  
  Error budget governs:
  ├── Feature velocity: if error budget is healthy → ship features fast
  ├── Reliability work: if error budget is burning → freeze features, fix reliability
  └── Risk decisions: "Can we do this risky migration?" → check error budget
```

### SLO Implementation

```yaml
# Using Pyrra (SLO tool) or Sloth for Prometheus-based SLOs

# pyrra-slo.yaml
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: api-server-availability
  namespace: monitoring
spec:
  target: "99.9"           # 99.9% SLO
  window: 30d              # 30-day rolling window
  
  # SLI definition
  indicator:
    ratio:
      errors:
        metric: http_requests_total{job="api-server",code=~"5.."}
      total:
        metric: http_requests_total{job="api-server"}
```

```promql
# Error budget burn rate alerts
# Alert when burning through budget too fast

# Fast burn: 1-hour window burning at 14x rate
# (would exhaust monthly budget in 2 days)
- alert: ErrorBudgetFastBurn
  expr: |
    (
      rate(http_requests_total{status=~"5.."}[1h])
      /
      rate(http_requests_total[1h])
    ) > 14 * (1 - 0.999)
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "High error budget burn rate — page immediately"

# Slow burn: 6-hour window burning at 6x rate  
# (would exhaust budget in ~5 days)
- alert: ErrorBudgetSlowBurn
  expr: |
    (
      rate(http_requests_total{status=~"5.."}[6h])
      /
      rate(http_requests_total[6h])
    ) > 6 * (1 - 0.999)
  for: 15m
  labels:
    severity: warning
```

### Toil — What to Eliminate

```
TOIL = Manual, repetitive, operational work that scales with service growth
       and provides no enduring value

Examples of TOIL:
├── Manually restarting pods when they crash (automate with liveness probes)
├── Manually scaling services for traffic spikes (automate with HPA)
├── Manually reviewing and approving every deployment (automate with CI gates)
├── Manually creating monitoring dashboards for each new service (template it)
├── Manually rotating secrets every 90 days (automate with Vault dynamic secrets)
└── Manually provisioning databases when devs ask (automate with Crossplane)

SRE RULE: Toil should be < 50% of work time
          Rest = engineering work that reduces future toil
          
HOW TO REDUCE TOIL:
1. Identify: track time spent on repetitive tasks
2. Quantify: how much time per week?
3. Automate: scripts, operators, self-healing
4. Eliminate: fix root cause so it never recurs
5. Delegate: if unavoidable, move to on-call rotation
```

---

## 10.3 Incident Management

### Incident Severity Levels

```
SEVERITY 1 (P1) — CRITICAL
├── Complete service outage
├── Data loss or corruption
├── Security breach
├── Revenue impact: high
├── Response: immediate (< 5 min)
├── All-hands — every engineer available
└── External comms: status page update in 15min

SEVERITY 2 (P2) — HIGH
├── Major feature broken affecting many users
├── Significant performance degradation
├── Revenue impact: moderate
├── Response: < 30 min
└── External comms: status page if > 30min

SEVERITY 3 (P3) — MEDIUM
├── Non-critical feature broken
├── Single-region or subset of users affected
├── Response: < 4 hours
└── External comms: only if customer-reported

SEVERITY 4 (P4) — LOW
├── Minor issues, workarounds available
├── Response: next business day
└── External comms: none
```

### Incident Response Process

```
DETECTION (T+0)
├── PagerDuty fires alert
├── On-call acknowledges within SLA
└── Incident declared (Slack channel: #inc-YYYYMMDD-service)

INITIAL TRIAGE (T+5min)
├── Incident Commander (IC) assigned
├── Comms Lead assigned (handles status page, stakeholder updates)
├── Severity assessed
└── Initial Slack post: what's broken, who's on it, next update in X min

MITIGATION — STOP THE BLEEDING (T+5 to T+30min)
├── Rollback deployment if recent release is suspect
├── Increase capacity if resource saturation
├── Failover to DR if primary is down
├── Enable circuit breakers / feature flags to isolate damage
└── Status page updated every 15-30 min during incident

ROOT CAUSE ANALYSIS (parallel to mitigation)
├── Check recent deployments (what changed?)
├── Check metrics around incident start time
├── Correlate traces for failing requests
├── Check logs for errors
└── Timeline construction

RESOLUTION (T+?)
├── Fix deployed and verified
├── Metrics return to normal
├── All users confirmed working
└── Incident declared resolved in status page

POST-MORTEM (T+24h to T+48h)
└── See next section
```

### Blameless Post-Mortem Template

```markdown
# Post-Mortem: [Service] [Brief Description]
Date: YYYY-MM-DD
Status: In Review / Approved
Severity: P1
Duration: 47 minutes (14:23 UTC — 15:10 UTC)
Impact: ~30% of users unable to checkout; est. $45,000 revenue impact

## Summary
[2-3 sentences: what happened, root cause, resolution]

The payment service experienced elevated error rates due to a database connection 
pool exhaustion caused by a slow query introduced in release v2.3.1. 47 minutes of 
degraded checkout availability. Fixed by rolling back to v2.3.0 and adding query timeout.

## Timeline
| Time (UTC) | Event |
|-----------|-------|
| 13:55 | v2.3.1 deployed to production |
| 14:23 | PagerDuty fires: error rate > 5% |
| 14:28 | On-call acknowledges, incident channel created |
| 14:35 | Identified recent deployment as suspect |
| 14:42 | Rollback initiated |
| 15:10 | Rollback complete, error rate normal |

## Root Cause
A new product search query in v2.3.1 lacked a database index, 
causing full table scans that held DB connections for 30+ seconds. 
This exhausted the 50-connection pool, causing all subsequent 
checkout requests to fail.

## What Went Well
- PagerDuty alert fired within 30 seconds of threshold breach
- Team had clear rollback procedure, executed in < 10 minutes
- Status page updated within 15 minutes of incident start

## What Went Poorly
- No performance testing caught the slow query pre-deploy
- No query timeout configured — connection held indefinitely
- Connection pool exhaustion not monitored with an alert

## Action Items
| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| Add pg_stat_statements monitoring + alert on p99 query time | @alice | 2024-02-01 | P1 |
| Configure 5s statement_timeout for application queries | @bob | 2024-01-25 | P1 |
| Add connection pool utilization alert (> 80% = warning) | @alice | 2024-01-25 | P1 |
| Require EXPLAIN ANALYZE for PRs touching DB queries | @team | 2024-02-15 | P2 |
| Load test staging environment before production deploys | @carol | 2024-03-01 | P2 |

## Lessons Learned
Connection pool exhaustion is a common failure mode that is easily preventable 
with query timeouts and monitoring. We need query performance gates in CI/CD.
```

---

## 10.4 Chaos Engineering

### What is Chaos Engineering?

Chaos Engineering is the practice of deliberately injecting failures into a system to test its resilience and discover weaknesses before they cause real outages.

```
CHAOS ENGINEERING PRINCIPLES:
1. Hypothesize about steady state: "System will serve < 0.1% errors under normal load"
2. Vary real-world events: inject failures (network partition, pod kill, disk full)
3. Run in production (or staging that mirrors prod traffic)
4. Minimize blast radius: start small, controlled experiments
5. Automate experiments to run continuously

FAILURE MODES TO TEST:
├── Pod termination (are your replicas actually helping?)
├── Node failure (does the scheduler reschedule pods quickly?)
├── Network latency injection (are your timeouts correct?)
├── DNS failure (does DNS caching protect you?)
├── Disk full (do you alert before it causes issues?)
├── Memory pressure (do priority classes protect critical pods?)
├── Dependency failure (payment service down — does checkout degrade gracefully?)
└── Cascading failures (one failure causing others?)
```

### Chaos Mesh — Kubernetes Chaos Engineering

```yaml
# Kill random pods in production namespace
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-experiment
spec:
  action: pod-kill
  mode: random-max-percent    # Kill random percentage
  value: "20"                 # Up to 20% of selected pods
  selector:
    namespaces: [production]
    labelSelectors:
      app: api-server
  scheduler:
    cron: "@every 168h"       # Weekly during business hours

---
# Inject network latency
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: payment-service-latency
spec:
  action: delay
  mode: all
  selector:
    namespaces: [production]
    labelSelectors:
      app: payment-service
  delay:
    latency: "200ms"
    jitter: "50ms"
    correlation: "25"
  duration: "30m"             # 30-minute experiment
  direction: to               # Incoming traffic to payment-service

---
# Simulate AZ failure (drain all nodes in one AZ)
apiVersion: chaos-mesh.org/v1alpha1
kind: NodeChaos
metadata:
  name: az-failure-simulation
spec:
  action: node-stop
  mode: fixed
  value: "3"
  selector:
    nodeSelectors:
      topology.kubernetes.io/zone: us-east-1a
```

### GameDays

```
GAMEDAY = Scheduled, facilitated chaos experiment with the whole team

Format:
1. Define hypothesis: "If payment service goes down, checkout shows graceful error"
2. Define success criteria: < 1% of users see generic error, 0 data loss
3. Schedule (business hours, staff available)
4. Brief team: what we're testing, what to watch
5. Execute experiment (Chaos Mesh / manual kill)
6. Observe: metrics, logs, user reports
7. Remediate if needed
8. Debrief: what we learned, action items

Example GameDay scenarios:
├── "Kill the primary database" — does failover work in < 30s?
├── "Remove all prod nodes except 2" — do pods reschedule and serve traffic?
├── "Cut off external payment provider" — does circuit breaker activate?
├── "Inject 2s latency to auth service" — do dependent services time out gracefully?
└── "Fill disk on logging host" — does app continue working, does alert fire?
```

---

## 10.5 FinOps — Cloud Cost Management

### Cloud Cost Visibility

```
COST ALLOCATION PYRAMID:

Organization Level: Total cloud spend
        │
Business Unit Level: Engineering / Marketing / Data
        │
Team Level: Backend / Frontend / Platform
        │
Service Level: order-service / payment-service
        │
Environment Level: production / staging / dev
        │
Resource Level: specific EC2 instance, RDS, EKS node

TOOLS:
├── AWS Cost Explorer + Cost Allocation Tags
├── Kubecost — Kubernetes-native cost visibility
│   kubectl cost pod --namespace production
│   kubectl cost namespace --historical
├── OpenCost — CNCF standard for K8s cost attribution
└── Custom dashboards in Grafana pulling billing APIs
```

### Kubecost Setup

```bash
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer \
    --namespace kubecost \
    --create-namespace \
    --set kubecostToken="your-token" \
    --set global.prometheus.enabled=false \
    --set global.prometheus.fqdn="http://kube-prometheus-stack-prometheus.monitoring:9090"

# Query costs via CLI
kubectl cost namespace \
    --show-all-resources \
    --window 30d

# Output:
# Namespace         CPU     Memory  Network  PV      GPU    Total
# production        $312    $89     $12      $45     $0     $458
# staging           $45     $23     $3       $10     $0     $81
# monitoring        $23     $67     $0       $20     $0     $110
```

### Cost Optimization Automation

```python
#!/usr/bin/env python3
# rightsize-recommendations.py
# Find overprovisioned pods and generate rightsizing recommendations

import subprocess
import json
from collections import defaultdict

def get_pod_metrics():
    """Get CPU/memory usage vs requests for all pods"""
    result = subprocess.run([
        "kubectl", "top", "pods", "-A", "--no-headers"
    ], capture_output=True, text=True)
    
    metrics = {}
    for line in result.stdout.strip().split('\n'):
        parts = line.split()
        if len(parts) >= 4:
            namespace, pod, cpu, memory = parts[0], parts[1], parts[2], parts[3]
            metrics[f"{namespace}/{pod}"] = {
                "cpu_usage": int(cpu.rstrip('m')),      # millicores
                "memory_usage": int(memory.rstrip('Mi'))  # MiB
            }
    return metrics

def get_pod_requests():
    """Get CPU/memory requests for all pods"""
    result = subprocess.run([
        "kubectl", "get", "pods", "-A", "-o", "json"
    ], capture_output=True, text=True)
    
    pods = json.loads(result.stdout)
    requests = {}
    
    for pod in pods['items']:
        namespace = pod['metadata']['namespace']
        name = pod['metadata']['name']
        key = f"{namespace}/{name}"
        
        cpu_req = 0
        mem_req = 0
        for container in pod['spec']['containers']:
            res = container.get('resources', {}).get('requests', {})
            cpu = res.get('cpu', '0m')
            mem = res.get('memory', '0Mi')
            cpu_req += int(cpu.rstrip('m')) if 'm' in cpu else int(cpu) * 1000
            mem_req += int(mem.rstrip('Mi'))
        
        requests[key] = {"cpu_request": cpu_req, "memory_request": mem_req}
    
    return requests

def find_overprovisioned(metrics, requests, cpu_threshold=0.3, mem_threshold=0.4):
    """Find pods using < threshold% of their requested resources"""
    recommendations = []
    
    for pod_key in metrics:
        if pod_key not in requests:
            continue
        
        usage = metrics[pod_key]
        req = requests[pod_key]
        
        if req['cpu_request'] == 0 or req['memory_request'] == 0:
            continue
        
        cpu_ratio = usage['cpu_usage'] / req['cpu_request']
        mem_ratio = usage['memory_usage'] / req['memory_request']
        
        if cpu_ratio < cpu_threshold or mem_ratio < mem_threshold:
            recommendations.append({
                "pod": pod_key,
                "cpu_usage_pct": f"{cpu_ratio*100:.1f}%",
                "mem_usage_pct": f"{mem_ratio*100:.1f}%",
                "recommended_cpu": f"{int(usage['cpu_usage'] * 1.3)}m",  # +30% headroom
                "recommended_memory": f"{int(usage['memory_usage'] * 1.3)}Mi"
            })
    
    return sorted(recommendations, key=lambda x: x['cpu_usage_pct'])

if __name__ == "__main__":
    print("Analyzing pod resource utilization...")
    metrics = get_pod_metrics()
    requests = get_pod_requests()
    recommendations = find_overprovisioned(metrics, requests)
    
    print(f"\nFound {len(recommendations)} overprovisioned pods:\n")
    for r in recommendations[:20]:
        print(f"  {r['pod']}")
        print(f"    CPU: {r['cpu_usage_pct']} of request → suggest {r['recommended_cpu']}")
        print(f"    MEM: {r['mem_usage_pct']} of request → suggest {r['recommended_memory']}\n")
```

---

## 10.6 Developer Experience (DX) Metrics

### DORA Metrics — Measuring DevOps Performance

```
DORA (DevOps Research and Assessment) Metrics:

1. DEPLOYMENT FREQUENCY
   How often does your team deploy to production?
   Elite: Multiple times per day
   High:  Once per day to once per week
   Med:   Once per week to once per month
   Low:   Less than once per month

2. LEAD TIME FOR CHANGES
   Time from code commit to running in production
   Elite: < 1 hour
   High:  1 day to 1 week
   Med:   1 week to 1 month
   Low:   > 6 months

3. CHANGE FAILURE RATE
   % of changes causing incidents/rollbacks
   Elite: 0-15%
   High:  16-30%
   Med/Low: 16-30% (same in original study)

4. TIME TO RESTORE SERVICE (MTTR)
   How long to recover from a failure
   Elite: < 1 hour
   High:  < 1 day
   Med:   1 day to 1 week
   Low:   > 6 months

MEASURING IN YOUR PLATFORM:
├── Deployment Frequency: count deploys per day from ArgoCD/CD system
├── Lead Time: git commit timestamp → production deploy timestamp
├── Change Failure Rate: % deployments requiring rollback (tag in CD system)
└── MTTR: incident open time → resolved time (from PagerDuty)
```

---

## 10.7 Runbook Automation

### Runbook as Code

```python
#!/usr/bin/env python3
# runbooks/high_memory_usage.py
# Automated runbook: triggered by PagerDuty webhook

import subprocess
import json
import requests
from datetime import datetime

SLACK_WEBHOOK = "https://hooks.slack.com/services/..."
PAGERDUTY_TOKEN = "..."

def get_top_pods_by_memory(namespace="production", top_n=5):
    """Get top N pods by memory usage"""
    result = subprocess.run([
        "kubectl", "top", "pods", 
        "-n", namespace,
        "--sort-by=memory",
        "--no-headers"
    ], capture_output=True, text=True)
    
    pods = []
    for line in result.stdout.strip().split('\n')[:top_n]:
        parts = line.split()
        if parts:
            pods.append({"name": parts[0], "cpu": parts[1], "memory": parts[2]})
    return pods

def check_oom_events(namespace="production", minutes=30):
    """Check for OOMKilled events in last N minutes"""
    result = subprocess.run([
        "kubectl", "get", "events",
        "-n", namespace,
        "--field-selector", "reason=OOMKilling",
        "-o", "json"
    ], capture_output=True, text=True)
    
    events = json.loads(result.stdout)
    return events.get("items", [])

def get_node_memory():
    """Get node memory utilization"""
    result = subprocess.run([
        "kubectl", "top", "nodes", "--no-headers"
    ], capture_output=True, text=True)
    
    nodes = []
    for line in result.stdout.strip().split('\n'):
        parts = line.split()
        if parts:
            nodes.append({
                "name": parts[0], 
                "cpu": parts[1], "cpu_pct": parts[2],
                "memory": parts[3], "mem_pct": parts[4]
            })
    return nodes

def post_slack_diagnostic(data):
    """Post diagnostic report to Slack"""
    blocks = [
        {
            "type": "header",
            "text": {"type": "plain_text", "text": "🔴 High Memory Alert — Automated Diagnosis"}
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"*Time:* {datetime.utcnow().strftime('%Y-%m-%d %H:%M UTC')}\n*Triggered by:* Automated runbook"
            }
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn", 
                "text": f"*Top Memory Pods:*\n" + "\n".join([
                    f"• `{p['name']}`: CPU={p['cpu']} MEM={p['memory']}"
                    for p in data["top_pods"]
                ])
            }
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"*Node Memory:*\n" + "\n".join([
                    f"• `{n['name']}`: {n['mem_pct']} used"
                    for n in data["nodes"]
                ])
            }
        },
        {
            "type": "actions",
            "elements": [
                {
                    "type": "button",
                    "text": {"type": "plain_text", "text": "View Grafana"},
                    "url": "https://grafana.company.com/d/memory-dashboard"
                },
                {
                    "type": "button", 
                    "text": {"type": "plain_text", "text": "View Runbook"},
                    "url": "https://wiki.company.com/runbooks/high-memory"
                }
            ]
        }
    ]
    
    requests.post(SLACK_WEBHOOK, json={"blocks": blocks})

def run_automated_diagnosis():
    print("Running automated memory diagnosis...")
    
    top_pods = get_top_pods_by_memory()
    oom_events = check_oom_events()
    nodes = get_node_memory()
    
    data = {
        "top_pods": top_pods,
        "oom_events": oom_events,
        "nodes": nodes
    }
    
    # Post to Slack immediately — humans see diagnostic within seconds
    post_slack_diagnostic(data)
    
    # Auto-remediation: if single pod is clearly the culprit and OOMKilled
    if len(oom_events) > 0 and len(top_pods) > 0:
        worst_pod = top_pods[0]
        print(f"OOMKilled detected. Top memory pod: {worst_pod['name']}")
        print("Auto-remediation: Restarting problematic pod...")
        
        # Safe auto-remediation: delete pod (Deployment will recreate it)
        subprocess.run([
            "kubectl", "delete", "pod", worst_pod["name"],
            "-n", "production"
        ])
        print(f"Pod {worst_pod['name']} deleted. Deployment will recreate.")

if __name__ == "__main__":
    run_automated_diagnosis()
```

---

## 10.8 SRE Interview Questions

**Q: What is an error budget and how does it influence engineering decisions?**

> An error budget is the maximum allowed unreliability derived from the SLO. If your SLO is 99.9% availability, your monthly error budget is 0.1% × 43,800 minutes = 43.8 minutes of downtime. When the budget is healthy (not exhausted), teams can take risks — deploy frequently, experiment. When the budget is burning fast, teams freeze feature work and focus on reliability. The key insight: reliability has a cost (slower development) and unreliability has a cost (user trust, revenue). Error budgets make that tradeoff explicit and quantitative rather than political.

**Q: How do you handle a situation where your team's error budget is frequently exhausted?**

> 1. **Analyze burn patterns**: is it a single recurring failure mode or many small ones? 2. **Prioritize reliability work**: freeze feature development, make SRE work the priority 3. **Tighten SLO**: if the SLO is too strict for the current architecture, negotiate a temporary relaxation while investing in reliability 4. **Fix root causes**: not symptoms — if deployments cause most errors, invest in better testing and canary deployments 5. **Adjust toil**: if on-call is burning the team out, hire or reduce scope 6. **Consider SLO revision**: if the current SLO is consistently unachievable, it may be set too high relative to user expectations

**Q: Design an on-call rotation for a team of 8 engineers.**

> Structure: weekly rotation, one primary on-call + one secondary (backup) at all times. Business hours (9am-6pm): primary on-call. Nights and weekends: primary on-call with page first, secondary as backup if primary doesn't respond in 10 minutes. Each person is primary ~once every 4 weeks, secondary ~once every 4 weeks (staggered). Compensate: time-in-lieu for weekend/night pages (company policy). Runbooks: every alert must have a linked runbook before it can page. Post-mortems: all P1/P2 incidents get post-mortems, action items tracked. Health checks: monthly review of on-call burden — if anyone averages > 10 pages/week, that's unsustainable and needs engineering attention.
```
# Phase 11: Performance Engineering, Databases & Networking

---

## 11.1 Performance Engineering Fundamentals

### The Performance Testing Pyramid

```
         ┌─────────────────────┐
         │   CHAOS / GAMEDAY   │  ← Real failure injection in prod
         ├─────────────────────┤
         │  SOAK / ENDURANCE   │  ← 24-72h sustained load (memory leaks)
         ├─────────────────────┤
         │     STRESS TEST     │  ← Push beyond limits, find breaking point
         ├─────────────────────┤
         │     SPIKE TEST      │  ← Sudden 10x traffic surge
         ├─────────────────────┤
         │     LOAD TEST       │  ← Sustained expected peak load
         ├─────────────────────┤
         │   BENCHMARK / UNIT  │  ← Individual function/endpoint performance
         └─────────────────────┘
```

### Load Testing with k6

```javascript
// k6 load test — production-grade example
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const checkoutDuration = new Trend('checkout_duration', true);  // true = milliseconds
const successfulOrders = new Counter('successful_orders');

export const options = {
  // Load profile: ramp up, sustain, ramp down
  stages: [
    { duration: '2m', target: 100 },    // Ramp up to 100 users
    { duration: '5m', target: 100 },    // Hold at 100 users
    { duration: '2m', target: 500 },    // Ramp up to 500 users (spike)
    { duration: '5m', target: 500 },    // Hold at 500 users
    { duration: '2m', target: 0 },      // Ramp down
  ],
  
  thresholds: {
    // Test FAILS if these thresholds are breached
    'http_req_duration': ['p(95)<500', 'p(99)<1000'],  // p95<500ms, p99<1s
    'errors': ['rate<0.01'],                             // Error rate < 1%
    'http_req_failed': ['rate<0.01'],
    'checkout_duration': ['p(95)<2000'],                 // Checkout p95 < 2s
  },
};

const BASE_URL = __ENV.BASE_URL || 'https://staging.example.com';

export default function () {
  // Test flow: browse → add to cart → checkout
  
  // 1. Browse products
  const productsRes = http.get(`${BASE_URL}/api/products`, {
    tags: { name: 'browse_products' },
  });
  
  check(productsRes, {
    'products status 200': (r) => r.status === 200,
    'products has items': (r) => JSON.parse(r.body).length > 0,
  }) || errorRate.add(1);
  
  sleep(1);
  
  // 2. Add to cart
  const cartRes = http.post(`${BASE_URL}/api/cart`, 
    JSON.stringify({
      product_id: "prod-123",
      quantity: 2
    }),
    {
      headers: { 'Content-Type': 'application/json' },
      tags: { name: 'add_to_cart' },
    }
  );
  
  check(cartRes, {
    'cart status 200': (r) => r.status === 200,
  }) || errorRate.add(1);
  
  sleep(2);
  
  // 3. Checkout (most important flow)
  const start = Date.now();
  const checkoutRes = http.post(`${BASE_URL}/api/checkout`,
    JSON.stringify({
      cart_id: JSON.parse(cartRes.body).cart_id,
      payment_method: "card",
      card_token: "tok_test_visa"
    }),
    {
      headers: { 'Content-Type': 'application/json' },
      tags: { name: 'checkout' },
      timeout: '10s',
    }
  );
  checkoutDuration.add(Date.now() - start);
  
  const checkoutOk = check(checkoutRes, {
    'checkout status 200': (r) => r.status === 200,
    'checkout has order_id': (r) => JSON.parse(r.body).order_id !== undefined,
  });
  
  if (checkoutOk) {
    successfulOrders.add(1);
  } else {
    errorRate.add(1);
  }
  
  sleep(3);
}

// Teardown: summary reporting
export function handleSummary(data) {
  return {
    'summary.json': JSON.stringify(data),
    stdout: textSummary(data, { indent: ' ', enableColors: true }),
  };
}
```

```bash
# Run k6 load test
k6 run --out influxdb=http://influxdb:8086/k6 loadtest.js

# With environment variables
k6 run \
    --env BASE_URL=https://staging.example.com \
    --vus 100 \
    --duration 5m \
    loadtest.js

# Cloud execution (distributed load from multiple regions)
k6 cloud loadtest.js
```

---

## 11.2 Database Performance Deep Dive

### PostgreSQL Internals

```
POSTGRES PROCESS ARCHITECTURE:
┌─────────────────────────────────────────────────────────────────┐
│  postmaster (main process)                                       │
│  ├── Background Writer   — writes dirty pages to disk           │
│  ├── WAL Writer          — flushes WAL to disk                  │
│  ├── Checkpointer        — periodic sync of dirty pages         │
│  ├── Autovacuum Launcher — spawns autovacuum workers            │
│  ├── Stats Collector     — query statistics (pg_stat_*)         │
│  └── Per-connection backends (one process per client connection) │
└─────────────────────────────────────────────────────────────────┘

QUERY LIFECYCLE:
Client → Parser → Rewriter → Planner/Optimizer → Executor
                                    │
                                    ▼
                          QUERY PLAN (EXPLAIN output)
                          Seq Scan vs Index Scan vs Bitmap Scan
                          Hash Join vs Nested Loop vs Merge Join
                          Cost estimates based on table statistics
```

### PostgreSQL Performance Tuning

```sql
-- ===== IDENTIFY SLOW QUERIES =====

-- Enable slow query logging (postgresql.conf)
-- log_min_duration_statement = 1000  (log queries > 1s)

-- Find slowest queries via pg_stat_statements
SELECT 
    left(query, 80) AS query,
    calls,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- ===== EXPLAIN ANALYZE =====
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, u.email 
FROM orders o 
JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days';

-- Key things to look for in EXPLAIN output:
-- Seq Scan (no index) → add index
-- High "actual rows" vs "estimated rows" → run ANALYZE
-- Nested Loop with large outer set → might need Hash Join hint
-- "Buffers: shared hit=X read=Y" — high "read" = not cached

-- ===== INDEXES =====

-- B-tree (default): equality, range, ORDER BY, LIKE 'foo%'
CREATE INDEX CONCURRENTLY idx_orders_status_created 
    ON orders(status, created_at DESC);

-- Partial index: index only a subset of rows
CREATE INDEX CONCURRENTLY idx_orders_pending 
    ON orders(created_at DESC) 
    WHERE status = 'pending';  -- Only index pending orders

-- Expression index: index on a function result
CREATE INDEX CONCURRENTLY idx_users_lower_email 
    ON users(lower(email));    -- For case-insensitive lookups

-- GIN index: full-text search, JSONB, arrays
CREATE INDEX CONCURRENTLY idx_products_description_fts 
    ON products USING GIN(to_tsvector('english', description));

CREATE INDEX CONCURRENTLY idx_orders_metadata 
    ON orders USING GIN(metadata jsonb_path_ops);

-- Check index usage
SELECT 
    schemaname, tablename, indexname,
    idx_scan, idx_tup_read, idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;  -- Low scan count = potentially unused

-- ===== VACUUM & BLOAT =====

-- Check table bloat
SELECT 
    schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_dead_tup AS dead_tuples,
    n_live_tup AS live_tuples,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Manual vacuum (when autovacuum is lagging)
VACUUM ANALYZE orders;           -- Reclaim dead tuples + update stats
VACUUM FULL orders;              -- ⚠️ Lock entire table, rewrites to new file

-- ===== CONNECTION POOLING =====
-- PgBouncer between app and PostgreSQL
-- PostgreSQL: max_connections = 200 (each connection = ~10MB memory)
-- PgBouncer: app → 5000 connections → PgBouncer → 50 real PG connections
-- 
-- pooler modes:
-- session: connection kept for full client session (most compatible)
-- transaction: connection released after each transaction (most efficient)
-- statement: released after each statement (breaks multi-statement transactions)

-- ===== PARTITIONING =====
-- For very large tables (100M+ rows)
CREATE TABLE orders (
    id BIGSERIAL,
    created_at TIMESTAMPTZ NOT NULL,
    user_id BIGINT,
    status TEXT,
    total DECIMAL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Query planner will only scan relevant partitions
-- Old partitions can be dropped instantly (DROP TABLE orders_2022_01)
```

### Redis Patterns

```
REDIS DATA STRUCTURES AND USE CASES:

STRING: simple key-value, counters, sessions
  SET session:abc123 '{"user_id":1,"email":"a@b.com"}' EX 3600
  INCR rate_limit:user:123:requests   # Atomic increment
  GET session:abc123

LIST: queues, message streams, recent items
  LPUSH recent_orders:user:123 "order-456"  # Add to front
  LTRIM recent_orders:user:123 0 9          # Keep only last 10
  LRANGE recent_orders:user:123 0 -1        # Get all

HASH: objects, user profiles, cache with fields
  HSET product:789 name "Widget" price 29.99 stock 150
  HGET product:789 price
  HGETALL product:789
  HINCRBY product:789 stock -1  # Decrement stock atomically

SET: unique items, tags, relationships
  SADD product:789:tags electronics gadgets
  SMEMBERS product:789:tags
  SINTER user:123:products user:456:products  # Common products

SORTED SET: leaderboards, time-series, priority queues
  ZADD leaderboard 1500 "user:123"
  ZADD leaderboard 2200 "user:456"
  ZREVRANGE leaderboard 0 9 WITHSCORES  # Top 10 with scores
  ZRANK leaderboard "user:123"           # Rank of user

STREAM: append-only log, event sourcing
  XADD events * event_type order_placed user_id 123 order_id 456
  XREAD COUNT 10 STREAMS events 0  # Read 10 events from start

PATTERNS:

1. CACHE-ASIDE (most common):
   app → check Redis → miss → query DB → store in Redis → return

2. WRITE-THROUGH:
   app → write to Redis AND DB simultaneously → consistent

3. DISTRIBUTED LOCK (prevent race conditions):
   SET lock:order:123 "process-id" NX EX 30  # NX=only if not exists
   # If SET returns OK → acquired lock
   # DELETE lock:order:123 when done
   # Use Redlock for multi-node reliability

4. RATE LIMITING:
   MULTI
   INCR rate:user:123
   EXPIRE rate:user:123 60
   EXEC
   # If result[0] > 100: rate limited

5. PUB/SUB:
   SUBSCRIBE order-events
   PUBLISH order-events '{"type":"order_placed","id":"456"}'
```

---

## 11.3 Networking Deep Dive

### TCP Deep Dive

```
TCP 3-WAY HANDSHAKE:
Client          Server
  │                │
  ├──SYN──────────►│  Client: "I want to connect, my seq=1000"
  │                │
  │◄──SYN-ACK─────┤  Server: "OK, my seq=2000, ack=1001"
  │                │
  ├──ACK──────────►│  Client: "Got it, ack=2001"
  │                │
  └── ESTABLISHED ─┘  Data can flow both directions

TCP 4-WAY CLOSE:
Client          Server
  │                │
  ├──FIN──────────►│  Client: "I'm done sending"
  │◄──ACK──────────┤  Server: "OK"
  │◄──FIN──────────┤  Server: "I'm also done sending"
  ├──ACK──────────►│  Client: "OK"
  │                │
  └── CLOSED ──────┘
  
  TIME_WAIT: Client waits 2*MSL (60-120s) before fully closing
  Reason: ensure server receives the final ACK
  Problem: Can exhaust ports on high-traffic systems (65535 ports)
  Fix: SO_REUSEADDR, shorter TIME_WAIT via net.ipv4.tcp_tw_reuse
```

### HTTP/2 vs HTTP/3

```
HTTP/1.1:
  ├── One request per connection (or keep-alive with pipelining, but buggy)
  ├── Head-of-line blocking: slow request blocks all others
  └── Many connections needed → expensive

HTTP/2 (over TLS):
  ├── Multiplexing: many requests over ONE connection
  ├── Header compression (HPACK)
  ├── Server Push: server proactively sends resources
  ├── Prioritization: important resources first
  └── Still TCP HOL blocking (lost packet blocks all streams)

HTTP/3 (over QUIC/UDP):
  ├── QUIC = HTTP/2 semantics over UDP
  ├── No TCP head-of-line blocking (each stream independent)
  ├── 0-RTT connection establishment (vs 1-RTT for HTTP/2)
  ├── Connection migration: IP change doesn't break connection
  └── Built-in TLS 1.3
  
PERFORMANCE COMPARISON:
  HTTP/1.1: 6 parallel connections, sequential within each
  HTTP/2:   1 connection, true multiplexing, 2x faster on most sites
  HTTP/3:   1 connection, no TCP, 10-20% faster on lossy networks
```

### Service Mesh — Envoy Proxy Internals

```
ENVOY is the data plane used by Istio, Linkerd, Consul Connect

ENVOY ARCHITECTURE:

┌────────────────────────────────────────────────────────────────────┐
│                         ENVOY PROXY                                │
│                                                                     │
│  LISTENERS        FILTERS           CLUSTERS       ENDPOINTS       │
│  ──────────       ──────────────    ──────────     ─────────────   │
│                                                                     │
│  :80 (HTTP)  →   HTTP conn mgr  →  backend-svc → 10.0.1.5:8080   │
│                  ├── Router                     → 10.0.1.6:8080   │
│                  ├── RateLimit                  → 10.0.1.7:8080   │
│                  ├── JWT Authn                                      │
│                  └── Metrics                                        │
│                                                                     │
│  :443(HTTPS) →   TLS termination → payment-svc → 10.0.2.5:8080   │
│                  + HTTP conn mgr                                    │
│                                                                     │
│  FEATURES:                                                          │
│  ├── Load balancing: Round Robin, Least Requests, Ring Hash        │
│  ├── Circuit breaking: max connections, pending requests           │
│  ├── Retry policies: retry on 5xx, with backoff                    │
│  ├── Timeout policies: per-route, per-cluster                      │
│  ├── Observability: metrics, access logs, distributed traces       │
│  ├── mTLS: terminate and originate TLS with certs from Istio CA   │
│  └── Rate limiting: local or global (via ext rate limit service)   │
└────────────────────────────────────────────────────────────────────┘
                                 │ xDS API (gRPC)
                                 ▼
                        CONTROL PLANE (Istiod)
                        Pushes config to all Envoy sidecars
```

```yaml
# Istio Traffic Management — VirtualService
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    # Retry configuration
    - route:
        - destination:
            host: payment-service
            port:
              number: 80
      timeout: 3s
      retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: 5xx,reset,connect-failure,retriable-4xx

---
# Circuit breaker via DestinationRule
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:     # Circuit breaker
      consecutive5xxErrors: 5           # Trip after 5 consecutive 5xx
      interval: 30s                     # Evaluation interval
      baseEjectionTime: 30s             # Eject for 30s initially
      maxEjectionPercent: 50            # Never eject more than 50% of endpoints
      minHealthPercent: 0
```

---

## 11.4 Linux Performance Analysis Tools

### The USE Method (Brendan Gregg)

```
USE METHOD: For every resource, check:
  U — Utilization (% of time resource is busy)
  S — Saturation  (queue depth, extra work waiting)
  E — Errors      (count of error operations)

RESOURCES TO CHECK:
CPU     → utilization: mpstat, top | saturation: vmstat r column | errors: mcelog
Memory  → util: free -m | saturation: vmstat si,so (swap in/out) | errors: dmesg
Disk I/O → util: iostat %util | saturation: iostat await | errors: dmesg
Network → util: nicstat %util | saturation: netstat drop,overrun | errors: ip -s l
```

### Performance Analysis Toolkit

```bash
# ===== CPU =====
top                     # Real-time processes
htop                    # Better top
mpstat -P ALL 1         # Per-CPU stats every 1s
perf top                # CPU flame graph in terminal
perf record -g app      # Profile app (then: perf report)
flamegraph.pl           # Generate flame graphs from perf data

# CPU flame graph (Brendan Gregg's tool)
perf record -F 99 -a -g -- sleep 30
perf script | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl > flame.svg

# ===== MEMORY =====
free -h                 # Available/used memory
vmstat 1               # Virtual memory stats (si=swap in, so=swap out → bad)
cat /proc/meminfo      # Detailed memory breakdown
smem -t -k             # Memory by process (accounts for shared memory correctly)
/proc/<pid>/status     # VmRSS=resident, VmSwap=swapped

# ===== DISK I/O =====
iostat -xz 1           # Disk I/O stats (watch: await, %util)
iotop                  # I/O by process (like top for disk)
ioping /dev/sda        # Disk latency test
dd if=/dev/zero of=/tmp/test bs=1G count=1 oflag=dsync  # Write speed test
blktrace + blkparse    # Block-level I/O tracing

# iostat key metrics:
# await: average I/O wait time (< 1ms for SSD, < 10ms for HDD)
# %util: disk utilization (>80% = saturated)
# r/s, w/s: reads/writes per second

# ===== NETWORK =====
ss -tuanp              # Socket statistics (replaces netstat)
ss -s                  # Summary
ip -s link             # Network interface stats
nicstat 1              # Network utilization by interface
sar -n DEV 1           # Network stats via sysstat
tcpdump -i eth0 port 443 -w capture.pcap  # Capture traffic
wireshark capture.pcap # Analyze capture

# ===== SYSTEM-WIDE =====
dstat -cdngy            # CPU/disk/net/paging combined view
sar -u -r -d 1 10       # CPU, memory, disk every 1s for 10s
strace -p PID -c        # System calls summary for process
lsof -p PID             # Open files for process
/proc/PID/fd/           # File descriptors
pmap -x PID             # Memory map of process

# ===== LINUX TRACING =====
# BPF-based (modern, low overhead, production-safe)
bpftrace -e 'tracepoint:syscalls:sys_enter_open { printf("%s\n", comm); }'
execsnoop               # Trace new process executions
opensnoop               # Trace file opens
tcpretrans              # Trace TCP retransmissions
biolatency              # Block I/O latency histogram
profile                 # CPU profiling (BPF-based flame graph)
```

### Kernel Tuning for Production

```bash
# /etc/sysctl.d/99-production.conf

# ===== NETWORK =====
# Increase network buffer sizes (for high-throughput servers)
net.core.rmem_max = 134217728      # Max TCP receive buffer (128MB)
net.core.wmem_max = 134217728      # Max TCP send buffer
net.ipv4.tcp_rmem = 4096 87380 67108864   # Min/default/max receive
net.ipv4.tcp_wmem = 4096 65536 67108864   # Min/default/max send

# Backlog (queue for incoming connections)
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# TIME_WAIT tuning (for high connection-rate servers)
net.ipv4.tcp_tw_reuse = 1          # Reuse TIME_WAIT sockets
net.ipv4.tcp_fin_timeout = 15      # Reduce FIN_WAIT2 timeout

# Ephemeral port range (default: 32768-60999, ~28k ports)
net.ipv4.ip_local_port_range = 1024 65535  # 64k ports

# Keep-alive (detect dead connections faster)
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6

# ===== MEMORY =====
# Virtual memory
vm.swappiness = 10                 # Prefer RAM over swap (default 60)
vm.dirty_ratio = 15                # Write back dirty pages when > 15% of RAM
vm.dirty_background_ratio = 5     # Background writeback at 5%
vm.overcommit_memory = 1          # Allow overcommit (for Redis: set to 1)

# ===== FILE SYSTEM =====
fs.file-max = 2097152              # System-wide open files limit
fs.inotify.max_user_watches = 1048576  # For tools watching many files

# Per-process limits (/etc/security/limits.conf):
# * soft nofile 1048576
# * hard nofile 1048576
```

---

## 11.5 Interview Questions

**Q: How do you find the root cause of a performance degradation?**

> **Systematic approach:**
> 1. **Characterize**: Is it CPU, memory, network, or I/O? Use `top`, `vmstat`, `iostat`, `ss` to identify the bottleneck resource.
> 2. **Scope**: All users or specific ones? All endpoints or specific ones? Started after deployment or gradually?
> 3. **USE method**: For each resource — Utilization, Saturation, Errors.
> 4. **Profile**: `perf top`, flame graphs for CPU; `pmap`, `valgrind` for memory; `iostat`, `blktrace` for disk.
> 5. **Trace**: `strace` for system calls; `tcpdump` for network; distributed traces in Jaeger for microservices.
> 6. **Correlate**: Match timeline with deployments, config changes, traffic patterns.
> 7. **Hypothesize and test**: Change one thing at a time, measure before and after.

**Q: Explain TCP's role in service-to-service latency and how to optimize it.**

> TCP adds latency at several points: 3-way handshake (1 RTT for new connections), slow-start (throughput builds gradually on new connections), Nagle's algorithm (buffers small packets, adds ~200ms). Optimizations: connection pooling and keep-alive (avoid handshake per request), TCP_NODELAY (disable Nagle for interactive traffic), SO_REUSEPORT (multiple sockets on same port for CPU parallelism), larger buffer sizes for high-bandwidth connections, tune `tcp_keepalive` to detect dead connections faster. In Kubernetes, HTTP/2 with multiplexing (gRPC) dramatically reduces connection overhead for microservices.

**Q: A PostgreSQL query that ran in 5ms now takes 5 seconds. What happened?**

> Common causes and diagnostics:
> 1. **Missing statistics**: `ANALYZE table_name` to refresh — planner chooses wrong plan with stale stats.
> 2. **Index bloat/corruption**: `REINDEX CONCURRENTLY idx_name`.
> 3. **Table bloat**: too many dead tuples — `VACUUM ANALYZE table_name`.
> 4. **Parameter change**: different input → different query plan → Seq Scan instead of Index Scan. Check with `EXPLAIN (ANALYZE, BUFFERS)`.
> 5. **Lock contention**: `pg_stat_activity` — check for waiting queries blocked by locks.
> 6. **New data volume**: table grew past a threshold where Seq Scan was optimal.
> 7. **Missing index after schema change**: someone dropped and didn't recreate an index.
> Fix sequence: `EXPLAIN ANALYZE` → identify operation → `ANALYZE` → `VACUUM` → `REINDEX` → add index if needed.
```
# Phase 12: Career Roadmap, Real-World Scenarios & Interview Mastery

---

## 12.1 DevOps Career Levels

### Junior DevOps Engineer (0–2 years)

```
WHAT YOU KNOW:
├── Can run basic Linux commands, write simple bash scripts
├── Understands Git workflow, can create PRs
├── Can deploy a Dockerized app following a guide
├── Familiar with one cloud provider (AWS or GCP or Azure)
└── Can modify existing CI/CD pipelines

WHAT YOU'RE LEARNING:
├── Writing Terraform for basic infrastructure
├── Setting up monitoring alerts
├── Kubernetes fundamentals (pods, services, deployments)
└── Understanding networking basics

COMMON INTERVIEW QUESTIONS:
├── "What is a Docker container vs a VM?"
├── "Explain git branching strategies"
├── "What is CI/CD?"
├── "How does DNS resolution work?"
└── "What is a load balancer?"

TYPICAL SALARY (India/US):
├── India: ₹6–12 LPA
└── US: $80,000–$100,000
```

### Mid-Level DevOps Engineer (2–5 years)

```
WHAT YOU KNOW:
├── Writes production Terraform, manages state, modules
├── Builds Jenkins/GitHub Actions pipelines from scratch
├── Kubernetes: troubleshoots, configures, operates clusters
├── Implements monitoring (Prometheus + Grafana + Alertmanager)
├── Familiar with multiple cloud services and their trade-offs
└── Handles on-call, writes post-mortems

WHAT YOU'RE LEARNING:
├── Building operators/custom controllers
├── Platform engineering concepts (Backstage, Crossplane)
├── SRE practices (SLOs, error budgets)
├── Security engineering (SAST/DAST, RBAC, secrets management)
└── Advanced networking (service mesh, BGP, eBPF)

TYPICAL SALARY:
├── India: ₹12–25 LPA
└── US: $100,000–$140,000
```

### Senior DevOps/SRE Engineer (5–10 years)

```
WHAT YOU KNOW:
├── Designs entire platform architecture from scratch
├── Makes build-vs-buy decisions for infrastructure
├── Sets reliability targets (SLOs) and enforces them
├── Runs blameless post-mortems and drives systemic improvements
├── Mentors junior engineers, contributes to hiring
├── Contributes to open-source CNCF projects
└── Deep expertise in at least one area (security, networking, databases)

TECHNICAL DEPTH EXPECTED:
├── Can explain etcd Raft consensus
├── Understands kernel networking (iptables, eBPF, cgroups)
├── Can design a multi-region, multi-cloud architecture
├── Knows when NOT to use Kubernetes
└── Has production experience with real incidents and their lessons

TYPICAL SALARY:
├── India: ₹25–50 LPA (top companies: 60–80 LPA)
└── US: $150,000–$200,000
```

### Staff/Principal Engineer (10+ years)

```
SCOPE: Organization-wide technical direction

├── Defines technical strategy for the platform (3–5 year horizon)
├── Drives cross-team alignment on infrastructure standards
├── Works with VPs/CTOs on build vs buy vs partner decisions
├── Represents company at industry conferences (KubeCon, AWS re:Invent)
├── Rarely writes code — writes design documents, RFCs
└── Accountable for reliability of the entire platform

TYPICAL SALARY:
├── India: ₹50–100 LPA + equity
└── US: $200,000–$300,000+ total compensation
```

---

## 12.2 Learning Roadmap by Timeline

### 3-Month Plan (Zero → Junior)

```
MONTH 1: Linux & Cloud Fundamentals
Week 1-2: Linux CLI (file system, processes, networking commands)
Week 3-4: AWS fundamentals (EC2, S3, VPC, IAM) — get AWS Solutions Architect Associate

MONTH 2: Containers & CI/CD
Week 5-6: Docker (build images, compose, networking)
Week 7-8: Git + GitHub Actions (build a real CI pipeline for a sample app)

MONTH 3: Kubernetes & IaC
Week 9-10: Kubernetes basics (minikube/kind, deploy apps, services)
Week 11-12: Terraform (deploy AWS infra, remote state, modules)

PROJECT: Deploy a web app (frontend + backend + database) on Kubernetes 
         on AWS EKS, provisioned with Terraform, deployed via GitHub Actions
```

### 6-Month Plan (Junior → Mid-Level)

```
MONTH 4: Kubernetes Deep Dive
├── Networking internals (CNI, Services, Ingress)
├── Storage (PV/PVC, StatefulSets)
├── Security (RBAC, NetworkPolicy, PodSecurityStandards)
└── Helm (write a chart for your app)

MONTH 5: Observability
├── Set up kube-prometheus-stack
├── Write PromQL queries for your app
├── Build Grafana dashboards
└── Configure Alertmanager with Slack/PagerDuty

MONTH 6: GitOps & Production Patterns
├── ArgoCD (GitOps flow for your app)
├── Secrets management (External Secrets + AWS Secrets Manager)
├── Incident response (practice post-mortems)
└── Cost optimization (tag resources, analyze spend)

PROJECT: Production-grade platform:
         Multi-environment (dev/staging/prod via ArgoCD)
         Full observability stack
         Automated secrets rotation
         Runbooks for top 5 incidents
```

### 12-Month Plan (Mid → Senior)

```
MONTH 7-8: Platform Engineering
├── Backstage setup + service catalog
├── Crossplane for self-service infrastructure
├── Developer experience metrics (DORA)
└── Golden path implementation

MONTH 9-10: Advanced Security
├── Falco runtime security
├── OPA/Gatekeeper policies
├── Supply chain security (Cosign, SBOM, Trivy)
└── Zero trust implementation with Istio mTLS

MONTH 11-12: Specialization
Choose one deep area:
├── SRE: SLOs, error budgets, chaos engineering, GameDays
├── Platform: Operator development, multi-cluster
├── Security: DevSecOps pipeline, threat modeling
└── Performance: Load testing, profiling, optimization

PROJECT: Open-source contribution to a CNCF project
         Write a blog post about something you built
         Speak at a local meetup
```

---

## 12.3 Real-World Scenarios — War Stories

### Scenario 1: The Database That Destroyed Production

```
SITUATION:
Friday 3 PM. You pushed a Terraform change to add a new security group rule
to the RDS instance. The plan looked fine. Apply runs. Production is down.
Users can't log in. Revenue stops.

WHAT HAPPENED:
You forgot that the security group change required replacing the RDS instance
because of a provider bug. Terraform replaced (destroyed + recreated) your
3TB database. The new instance has no data.

Timeline:
T+0: terraform apply
T+2m: realize nothing works
T+5m: check RDS console — it's CREATING
T+7m: realization hits: it was REPLACED
T+10m: check backups — last backup was 8 hours ago
T+???: restore from backup, 8 hours of data loss

LESSONS LEARNED:
1. ALWAYS use 'lifecycle { prevent_destroy = true }' on databases
2. Read plan output carefully — look for "must be replaced" 
3. Never run terraform apply on Fridays
4. Automated backups are not enough — test restoration regularly
5. Use deletion_protection = true on RDS
6. Review plan in PR, not just before applying
7. Point-in-time recovery (PITR) would have reduced data loss to minutes

INTERVIEW USAGE:
This is a perfect "tell me about a failure" story because you:
- Take ownership (no blame-shifting)
- Explain technical root cause clearly
- Show what you learned
- Describe preventative measures you implemented
```

### Scenario 2: The Kubernetes Cascade

```
SITUATION:
Monday morning. CPU alert fires for one pod. You investigate.
By the time you look, half the production cluster is unavailable.

WHAT HAPPENED:
Step 1: High-CPU pod triggers HPA scale-out
Step 2: New pods scheduled on nodes that were near capacity
Step 3: Nodes hit memory limit, start OOMKilling pods
Step 4: OOMKilled pods restart, go back to high-CPU state
Step 5: More HPA scale-out, more nodes full
Step 6: kube-scheduler can't find nodes with enough resources
Step 7: Pods stuck Pending, services degraded
Step 8: Liveness probes fail on surviving pods (resource-starved), they restart
Step 9: 🔥🔥🔥

ROOT CAUSE:
A memory leak in a dependency (crypto library update) caused memory growth.
Combined with no PodDisruptionBudgets, no resource limits on some pods,
and PriorityClasses not configured — everything cascaded.

MITIGATION:
1. kubectl delete pod <high-CPU pods> — rolling restart broke the cycle
2. kubectl cordon <most stressed nodes>
3. Deployed with reduced replica count to give cluster room to breathe

FIXES:
1. Add memory limits to ALL pods
2. Configure PriorityClasses (kube-system > critical apps > normal)
3. PodDisruptionBudgets for all stateful services
4. Memory leak fix in application (upgraded crypto library)
5. Cluster autoscaler properly configured to add nodes before full
6. Alerts on pending pods (not just on CPU/memory)
```

### Scenario 3: The Secret That Wasn't

```
SITUATION:
Security audit reveals that AWS access keys have been committed to a 
public GitHub repository 6 months ago. The keys are still active.
You have no idea if they've been used maliciously.

INCIDENT RESPONSE:
T+0: Rotate keys immediately (disable old, create new)
T+5m: Check CloudTrail for usage of the old keys in past 6 months
T+20m: Analyze all API calls made with the old keys
T+1h: Determine blast radius — what resources were accessible?
T+2h: Check for any resources created/modified by external actor
T+4h: Security team briefed, legal notified if customer data accessed
T+1d: Full audit complete, no evidence of malicious use (lucky)

REMEDIATION:
1. Implement pre-commit hooks (detect-secrets) in all repos
2. Enable GitHub secret scanning (alerts on new commits with secrets)
3. Rotate ALL secrets org-wide as precautionary measure
4. Switch to IAM roles + IRSA (no static keys in code ever)
5. Implement AWS Config rule: alert if IAM access key > 90 days old
6. Git history rewrite (BFG Repo Cleaner) to remove secret from history
7. Add to onboarding: "Never put secrets in code"
```

---

## 12.4 The Complete Interview Playbook

### Technical Phone Screen (45 min)

```
TYPICALLY COVERS:
├── Background walk-through (10 min)
│   "Walk me through your last project and the tech stack"
│   "What was the biggest technical challenge?"
│
├── Core knowledge questions (25 min)
│   2-3 questions from these areas:
│   ├── Linux/Networking (DNS, TCP, Linux internals)
│   ├── Docker/Kubernetes (how it works, not just how to use)
│   ├── CI/CD (pipeline design, GitOps)
│   └── Cloud (AWS architecture, IAM)
│
└── Questions for them (10 min)
    "How do you handle on-call? Rotation, compensation?"
    "What's the current biggest reliability challenge?"
    "How do teams deploy? GitOps or push-button?"
    "What does the monitoring stack look like?"

TIPS:
├── Think out loud — show your reasoning, not just the answer
├── Ask clarifying questions before diving in
├── If you don't know: "I haven't used X specifically, but I know Y
│   which works similarly, and here's how I'd approach learning X"
└── Have 2-3 real incidents/projects ready to discuss in detail
```

### System Design Round (60 min)

```
COMMON PROMPTS:
├── "Design the CI/CD pipeline for our 100-service platform"
├── "Design the monitoring and alerting for our e-commerce system"
├── "How would you migrate our monolith to Kubernetes?"
├── "Design a multi-region deployment strategy"
└── "How would you implement zero-downtime deployments?"

FRAMEWORK FOR ANSWERING:
1. CLARIFY (5 min)
   "Before I dive in, a few questions:
   - What scale? Users, RPS, data size?
   - What are the team's current pain points?
   - Any existing constraints (budget, existing tools, compliance)?"

2. HIGH-LEVEL DESIGN (10 min)
   Draw on whiteboard/Excalidraw — don't start with details
   Show main components and how they connect

3. DEEP DIVE (30 min)
   Pick 2-3 areas to go deep on
   Show trade-offs: "I could do X or Y. X is simpler but Y scales better.
   Given your scale (1000 deployments/day), I'd choose Y because..."

4. RELIABILITY & SECURITY (10 min)
   What happens when X fails?
   How do you secure the pipeline?
   How do you scale this?

5. SUMMARY (5 min)
   "To summarize: main components are X, Y, Z. The key trade-off I made was
   [trade-off]. The main risk is [risk] and I'd mitigate it by [mitigation]."
```

### Behavioral Round (STAR Format)

```
COMMON QUESTIONS AND FRAMEWORKS:

"Tell me about a production incident you handled"
STAR:
S: "We had a P1 outage on checkout. Revenue impact was ~$X/minute"
T: "I was the IC. My task was to restore service within our RTO"
A: "I [specific actions]: identified the issue via [metrics/logs],
    coordinated the team, implemented the fix"
R: "Restored in X minutes. Then [prevention: post-mortem, alert, etc]"

"Tell me about a time you disagreed with a technical decision"
S: "Team wanted to use X approach for database migrations"
T: "I believed Y approach was safer (less risky in production)"
A: "I documented both approaches with pros/cons, created a small POC,
    presented to the team with data"
R: "Team adopted Y approach. Zero migration incidents since then"
(Key: You pushed back WITH DATA, you were collaborative not combative)

"Describe the most complex system you've built"
Structure:
├── Start with the problem (why did it need to exist?)
├── Architecture overview (components and how they connect)
├── What made it complex (trade-offs, edge cases, scale)
├── What you'd do differently now
└── What it taught you

"How do you handle on-call burnout on your team?"
Good answer shows:
├── You measure: track pages/week per engineer
├── You automate: runbooks → reduce manual resolution
├── You improve: every alert has an action item to reduce it
├── You protect: time-in-lieu for night/weekend pages
└── You escalate: if no improvement, reduce scope or hire
```

---

## 12.5 Technologies to Know in 2024–2026

### Must-Know (Core)

```
CONTAINER & ORCHESTRATION:
├── Docker + containerd
├── Kubernetes (EKS/GKE/AKS or self-managed)
├── Helm + Helmfile
└── ArgoCD or Flux (GitOps)

IaC:
├── Terraform (primary)
├── Ansible (configuration management)
└── Helm (Kubernetes packaging)

CLOUD:
├── AWS (dominant, must know deeply)
├── GCP or Azure (secondary)
└── Multi-cloud networking concepts

CI/CD:
├── GitHub Actions (current industry standard)
├── GitLab CI (popular in Europe/enterprises)
└── Jenkins (legacy, still common)

OBSERVABILITY:
├── Prometheus + Grafana (metrics)
├── ELK or Loki (logs)
└── Jaeger or Tempo (traces)

PROGRAMMING:
├── Python (scripting, tools, automation)
├── Go (operators, tools, K8s ecosystem)
├── Bash (essential)
└── YAML + Jinja2 (Ansible, Helm)
```

### Emerging (Get Ahead of the Curve)

```
PLATFORM ENGINEERING:
├── Backstage (developer portal)
├── Crossplane (infrastructure as Kubernetes resources)
└── Port, Cortex (Backstage alternatives)

SECURITY:
├── Falco (runtime security)
├── OPA/Gatekeeper (policy as code)
├── Cosign/Sigstore (supply chain security)
└── Trivy (vulnerability scanning)

NETWORKING:
├── eBPF (observability, networking, security without kernel modules)
├── Cilium (eBPF-based CNI + network policy)
└── Gateway API (replaces Kubernetes Ingress)

AI/ML OPS:
├── Ray (distributed ML training)
├── KServe (model serving on Kubernetes)
├── MLflow (experiment tracking)
└── Kubeflow Pipelines (ML workflows)

WASM:
└── WebAssembly on server-side (Fermyon Spin, WasmEdge)
    Lighter than containers, faster cold start
```

---

## 12.6 Certifications Roadmap

```
BEGINNER:
├── AWS Solutions Architect Associate (SAA-C03) ← Start here
│   Study time: 1-2 months | Cost: $150
│   Value: Opens many doors, foundational AWS knowledge
│
└── CKA (Certified Kubernetes Administrator)
    Study time: 2-3 months of hands-on practice
    Cost: $395 (includes 2 attempts)
    Value: Hands-on exam, respected in industry

INTERMEDIATE:
├── AWS DevOps Engineer Professional (DOP-C02)
│   Prereq: Recommended to have SAA first
│   Study time: 2-3 months
│
├── CKAD (Certified Kubernetes Application Developer)
│   Developer-focused K8s (faster to get than CKA if you develop apps)
│
└── CKS (Certified Kubernetes Security Specialist)
    Prereq: CKA required
    Advanced K8s security

ADVANCED:
├── GCP Professional Cloud DevOps Engineer
├── HashiCorp Terraform Associate
└── AWS Advanced Networking Specialty

MOST VALUABLE ORDER:
AWS SAA → CKA → AWS DevOps Pro → CKS

NOTE: Real experience > certifications.
      A person with 3 years of EKS production experience + CKA
      beats a person with 5 certifications and no production exposure.
      Certs get you past HR filters. Experience wins the technical round.
```

---

## 12.7 Quick Reference — Emergency Cheat Sheet

### Kubernetes Debugging — 5 Minutes to Root Cause

```bash
# Pod not starting?
kubectl get pod <pod> -n <ns> -o wide          # Basic status
kubectl describe pod <pod> -n <ns>              # Events section at bottom!
kubectl logs <pod> -n <ns> --previous           # Logs from previous (crashed) container
kubectl logs <pod> -n <ns> -c <container>       # Specific container

# Pending pod?
kubectl describe pod <pod> -n <ns> | grep Events -A 20
# Common: "Insufficient memory", "No matching node", "Taints"

# Node issues?
kubectl get nodes                               # Check Ready status
kubectl describe node <node>                    # Events + resource pressure
kubectl top nodes                               # CPU/memory usage

# Service not reachable?
kubectl get endpoints <service> -n <ns>         # Are endpoints populated?
kubectl exec -it <pod> -- nslookup <service>    # DNS resolution works?
kubectl exec -it <pod> -- curl http://<service>:<port>/health  # Direct test

# Network policy blocking?
kubectl get networkpolicy -n <ns>
kubectl exec -it <pod> -- curl http://<service> -v  # Verbose for errors

# Resource constrained?
kubectl top pods -n <ns> --sort-by=memory
kubectl describe node | grep -A 10 "Allocated resources"
```

### AWS Quick Debugging

```bash
# EC2 can't reach internet?
# 1. Check security group outbound rules (allow 443, 80)
# 2. Check NACL outbound + inbound (stateless!)  
# 3. Check route table: 0.0.0.0/0 → IGW (public) or NAT (private)
# 4. Check IGW attached to VPC
# 5. Check EC2 has public IP or EIP (for public subnet)

# EKS pods can't reach AWS services?
# Option A: NAT Gateway in public subnet (outbound)
# Option B: VPC Endpoint for that service (cheaper, faster)
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id>

# CloudWatch Logs not appearing?
# Check IAM role has logs:CreateLogGroup, logs:PutLogEvents
aws iam get-role-policy --role-name <role> --policy-name <policy>

# RDS connection refused?
# 1. Security group allows your IP/CIDR on port 5432/3306
# 2. RDS is in "Available" state
# 3. Correct endpoint (not ARN, not ID — the endpoint hostname)
# 4. If multi-AZ: connect to cluster endpoint, not instance endpoint
```

### The Most Important Commands

```bash
# KUBERNETES
kubectl get all -n <namespace>                          # Everything
kubectl get events -n <namespace> --sort-by=lastTimestamp  # Recent events
kubectl exec -it <pod> -n <ns> -- sh                   # Shell into pod
kubectl port-forward svc/<service> 8080:80 -n <ns>     # Local access
kubectl rollout status deployment/<name> -n <ns>        # Rollout progress
kubectl rollout undo deployment/<name> -n <ns>          # Emergency rollback
kubectl top pods -n <ns>                               # Resource usage
kubectl get pod -o wide                                # Node assignment
kubectl cordon <node>                                  # Stop scheduling
kubectl drain <node> --ignore-daemonsets               # Evacuate node

# TERRAFORM
terraform plan -out=tfplan                             # Save plan
terraform apply tfplan                                 # Apply saved plan
terraform state list                                   # What's in state
terraform state show <resource>                        # Resource details
terraform import <resource_type.name> <id>             # Import existing
terraform force-unlock <lock-id>                       # Stuck lock

# DOCKER
docker ps -a                                           # All containers
docker logs <container> --tail=100 -f                  # Follow logs
docker exec -it <container> sh                         # Shell
docker stats                                          # Resource usage
docker system prune -a                                 # Cleanup

# GIT
git log --oneline --graph --all                        # Visual history
git bisect start; git bisect bad; git bisect good <hash>  # Find regression
git stash list; git stash apply stash@{0}              # Stash management
git reflog                                             # Recovery lifeline
```

---

## 12.8 Final Words

```
THE MINDSET OF A GREAT DEVOPS ENGINEER:

1. EVERYTHING FAILS — design for it
   Not "will this fail?" but "when this fails, what happens?"
   Test your failure modes. Run chaos experiments.
   
2. MEASURE EVERYTHING — no gut feelings in production
   If you can't measure it, you can't improve it.
   Instrument your systems obsessively.

3. AUTOMATE THE TOIL — protect your time
   If you do it twice manually, automate it.
   Manual processes don't scale and introduce errors.

4. SECURITY IS NOT AN AFTERTHOUGHT
   Build security into the pipeline, not bolted on afterward.
   The breach you prevent is the one you never hear about.

5. DOCUMENTATION IS CODE
   If it's not written down, it doesn't exist.
   Runbooks save you at 3 AM when you're barely awake.

6. EMPATHY FOR DEVELOPERS
   Your job is to make developers more productive.
   Every friction point in the developer experience is your problem.

7. SYSTEMS THINKING
   Optimize the whole system, not individual components.
   A fast CI pipeline that deploys broken code is worthless.

8. CONTINUOUS LEARNING
   The cloud-native ecosystem evolves faster than any other field.
   Read: SRE Book (Google), DORA reports, CNCF blog, Brendan Gregg.
   Do: KubeCon, AWS re:Invent, HashiConf talks (all free on YouTube).
   Build: personal lab (k3s on Raspberry Pi, free tier AWS, kind clusters).

9. GIVE BACK
   Write blog posts. Contribute to open source. Speak at meetups.
   The knowledge you share comes back to you tenfold.

10. RELIABILITY IS A FEATURE
    Every minute of downtime has a cost.
    Every unreliable alert erodes trust.
    Build things that work, and work reliably.
    That's the whole job.
```

---

*This completes the DevOps Learning System — Phases 1–12.*
*From "what is Linux" to Principal Engineer. The rest is practice.*
*Build things. Break things. Learn from both.*
```
