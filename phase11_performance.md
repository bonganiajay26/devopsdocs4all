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
