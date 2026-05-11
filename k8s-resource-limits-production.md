# Kubernetes Resource Limits Are Lying to You
## What We Discovered After 6 Months in Production

> *Written by a Senior Platform Engineer — 6 months of production incidents*

---

We spent six months convinced our cluster was well-configured. Our dashboards were green, our pods were running, and our YAML looked exactly like the docs said it should. Then the first major traffic event hit — and everything we thought we knew about resource limits turned out to be incomplete, misleading, or just flat wrong.

This post is not a tutorial. It's a field report. Everything here comes from real incidents: 3 AM pages, postmortems with angry SLO burn-down charts, and a slow accumulation of hard-won intuition about how Kubernetes *actually* manages CPU and memory under load. If you deploy to production clusters, you'll recognize most of this.

---

## The Starting Assumptions (That Bite You)

Before we burned ourselves, here's the mental model most engineers carry into Kubernetes resource management — including us:

> **"Setting a CPU limit to `500m` means my container gets exactly 500 millicores. It can't use more. It's capped."**
>
> **"Memory limit of `512Mi` is a ceiling. If I'm under it, I'm safe."**
>
> **"Requests are for scheduling, limits are for runtime enforcement. They're separate concerns."**

All three of these are partially true, dangerously incomplete, or actively misleading at production scale. Let's take them apart.

---

## Chapter 1: CPU Limits and the Throttling Trap

### How CPU limits actually work under cgroups

Kubernetes enforces CPU limits using the Linux `cgroup v1` CFS (Completely Fair Scheduler) bandwidth control. The implementation is elegant in theory and brutal in practice. When you set `limits.cpu: 500m`, the kernel translates this to:

```
cpu.cfs_quota_us  = 50000   # 50ms of CPU per period
cpu.cfs_period_us = 100000  # 100ms scheduling period
```

This means: within every 100ms window, your container can consume at most 50ms of CPU time. Once it hits the quota, it's **throttled** — forcibly paused — until the next period begins. This is not graceful degradation. The process is suspended mid-execution.

```
CPU Throttling Timeline — 100ms CFS period
──────────────────────────────────────────────────────────────────
  0ms                 50ms                              100ms
  │                   │                                 │
  ├──── Container ────┤──────── THROTTLED (dead wait) ──┤─── New period
       runs / bursts        Quota exhausted. Suspended.      Quota refills.
                            Even if node has spare CPU.
──────────────────────────────────────────────────────────────────
```

> **Key insight:** Even if the node has completely idle CPUs, your pod sits frozen waiting for the next CFS period to begin. This throttling is invisible to most monitoring setups.

### The incident that taught us this

We had a Go service — a critical data pipeline — with a 99th-percentile latency SLO of 200ms. Monitoring showed average CPU at ~35% of its limit. P99 latency was gradually creeping toward 180ms and we couldn't explain it. Memory was fine. No OOMKills. Pods looked healthy.

The smoking gun was this Prometheus query:

```promql
rate(container_cpu_cfs_throttled_periods_total[5m])
  / rate(container_cpu_cfs_periods_total[5m])
```

Our service was being throttled **67% of the time**. The CPU usage metric showed a comfortable average, but it concealed violent bursts every few milliseconds — spikes the Go GC was producing during concurrent marking cycles. Average utilization means nothing when your workload is bursty.

> **💀 Incident Pattern #1 — Silent GC Throttling**
>
> The Go GC, JVM garbage collector, and any runtime that pauses or spikes work in short bursts will hit CFS quota far harder than the average CPU metric suggests. A service "using 40% of its CPU limit" can still be throttled 60% of the time.

### The cgroups v2 nuance

On newer kernels with `cgroup v2` (default on Kubernetes 1.25+ nodes using Ubuntu 22.04 or RHEL 9), some of this behavior improves. The scheduler is better at burst credit carry-over. But it's not uniformly deployed, and your managed cloud provider's kernel may still run v1. Always verify:

```bash
stat -fc %T /sys/fs/cgroup/
# outputs "cgroup2fs" (v2) or "tmpfs" (v1)
```

---

## Chapter 2: Memory Limits and the OOMKill Lottery

### The difference between limits and requests is not what you think

Memory requests affect scheduling — the Kubernetes scheduler places your pod on a node with enough allocatable memory. Memory limits affect what happens at runtime. But here's the critical point: **memory limits are enforced by the kernel OOM killer, not by Kubernetes.**

When your container exceeds its memory limit, the kernel sends a `SIGKILL` to the container's main process. No warning. No graceful shutdown. The kernel picks the highest-`oom_score_adj` process in the cgroup. In a multi-threaded application, this might not even be your main process — it could be a worker thread, leaving your application in a partially torn-down zombie state that confuses your readiness probes.

```
OOMKill Cascade — What Actually Happens
────────────────────────────────────────────────────────────────────
  Container memory > limit
         │
         ▼
  Kernel OOM scorer runs (oom_score_adj per process)
         │
         ▼
  SIGKILL sent to highest scorer — may not be main process
         │
         ▼
  Pod restarts → CrashLoopBackOff begins
         │
         ▼
  Readiness probe fails during init window
         │
         ▼
  ⚠ Traffic may still route to the pod (race condition!)
────────────────────────────────────────────────────────────────────
```

### The Java heap sizing trap

This incident cost us a weekend. A Spring Boot service kept getting OOMKilled with a heap setting of `-Xmx400m` inside a container with `limits.memory: 512Mi`. The math seemed fine: 400MB heap + 112MB headroom. Shouldn't be a problem.

Except the JVM doesn't live in just the heap. The actual resident set size of a JVM includes:

```
# JVM memory regions outside the heap:
Metaspace:          ~80-150Mi  (class metadata, grows with classes loaded)
Code Cache:         ~50-100Mi  (JIT-compiled code)
Direct Buffers:     variable   (Netty, NIO — often not tracked by devs)
JVM internals:      ~20-40Mi
Thread stacks:      ~1Mi per thread × N threads
GC overhead:        variable

# Real resident memory = Heap + ALL of the above
```

Our service was loading 400+ classes via Spring's autoconfiguration and using Netty for async HTTP. Actual RSS was routinely hitting 580–620Mi. The fix was not to increase the limit blindly — it was to measure RSS with `kubectl exec` and `cat /proc/self/status | grep VmRSS`, then set limits based on actual observation rather than heap size alone.

> **⚠ JVM Users — Read This Carefully**
>
> For Java containers, use `-XX:+UseContainerSupport` (Java 10+) and `-XX:MaxRAMPercentage=75.0` rather than a static `-Xmx`. This instructs the JVM to respect cgroup memory limits. Also monitor `container_memory_working_set_bytes` in Prometheus — it tracks RSS + dirty cache and is what Kubernetes uses for eviction decisions, **not** `container_memory_usage_bytes`.

---

## Chapter 3: Noisy Neighbors and Node-Level Chaos

### Why node-level constraints override your pod specs

This is the one almost nobody talks about in onboarding docs. Kubernetes resource requests and limits operate within a pod's cgroup. But multiple pods share a node, and the node itself has system-level processes — the kubelet, container runtime (containerd/CRI-O), kube-proxy, CNI plugins, and your logging daemonsets — all consuming resources outside of any pod's cgroup.

Kubernetes reserves a portion of node resources via `--kube-reserved` and `--system-reserved`, but these are often set to optimistic minimums. When a noisy neighbor pod on the same node starts thrashing memory and triggering swap activity (if swap is enabled), or pinning CPU on hot cache lines, your perfectly-configured pod pays the price in ways your metrics won't show clearly.

| Metric | Value | Context |
|--------|-------|---------|
| Node CPU stolen by noisy neighbor pods | **38%** | During production incident |
| P99 latency spike (same window) | **4.1×** vs. baseline P99 | Direct consequence |
| Pod CPU utilization (appeared) | **41%** | Looked completely normal |

### Diagnosing the noisy neighbor

The symptom: a critical API service had random latency spikes on certain nodes but not others. Same pod spec, same deployment, same region. We initially blamed network partitions.

The tool that exposed it was `perf` at the node level combined with Prometheus `node_cpu_seconds_total` broken down by mode:

```bash
# Find which node is suffering
kubectl get pods -o wide | grep api-service

# Check CPU steal (virtualized environments — or use Prometheus):
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Check per-namespace CPU usage on the node
kubectl top pods --all-namespaces --sort-by=cpu | head -30

# Identify which pods share the node with your service
kubectl get pods -o wide | grep <node-name>
```

The culprit was a batch job pod with no CPU limit set at all. It would saturate all 8 vCPUs during its run window, and because it had no limit, the CFS scheduler allowed it. Our API service, with a limit, was throttled further because CFS quota accounting is relative — unlimited pods can consume proportionally more time slices during contention.

> **💀 Incident Pattern #2 — Limitless Batch Jobs**
>
> Pods without CPU limits are not just undefined — they're dangerous neighbors. Under CFS, they can starve limited pods even if your limited pods have headroom within their quota. Enforce `LimitRanges` in every namespace. Treat a missing limit as a misconfiguration, not a default.

---

## Chapter 4: The Requests vs. Limits Ratio Problem

### The scheduling illusion

Resource requests determine scheduling placement. If a node has 4 CPU allocatable and your 10 pods each request `100m`, Kubernetes happily schedules all 10 on the same node — 10 × 100m = 1000m, within the 4000m available. But if those same pods each have a CPU limit of `2000m`, you've set up a scenario where actual peak consumption can exceed physical capacity by 20×. This is called **overcommitment**, and it's intentional for certain workloads — but it will burn you if applied indiscriminately.

```
Overcommitment — The Invisible Risk
────────────────────────────────────────────────────────────────────────
SCHEDULED VIEW (looks fine)          RUNTIME REALITY (can explode)
─────────────────────────────        ─────────────────────────────
Node: 4000m allocatable              10 pods × 2000m limit = 20,000m
10 pods × 100m request = 1000m       Node capacity: 4000m
✓ Scheduler: placement OK            ✗ 5× physical overcommit possible
────────────────────────────────────────────────────────────────────────
```

### Memory overcommitment is in a different category

CPU overcommitment causes throttling — it's painful but recoverable. Memory overcommitment causes **node-level OOM events**. When a node runs out of memory, the kernel's OOM killer starts terminating processes — not just the offending pod's process, but potentially anything on the node with a high enough OOM score. We saw a node go into a death spiral: an OOMKill triggered a CrashLoopBackOff, which triggered a restart, which re-allocated memory, which triggered another OOMKill on a different pod, and so on.

The kubelet has an eviction mechanism designed to prevent this, but the default eviction thresholds are conservative:

```bash
# Default kubelet eviction thresholds (too low for production)
--eviction-hard=memory.available<100Mi
--eviction-hard=nodefs.available<10%
--eviction-hard=imagefs.available<15%

# Recommended production settings (adjust per node size):
--eviction-hard=memory.available<500Mi
--eviction-soft=memory.available<1Gi
--eviction-soft-grace-period=memory.available=2m
```

With the default 100Mi threshold on an 8Gi node, by the time eviction triggers, you're already in kernel OOM territory.

---

## Chapter 5: The Debugging Stack We Actually Use

### Prometheus queries for resource limit pathologies

**1. CPU throttle ratio** — values above 0.25 are worth investigating immediately:

```promql
sum by (namespace, pod, container) (
  rate(container_cpu_cfs_throttled_periods_total[5m])
) / sum by (namespace, pod, container) (
  rate(container_cpu_cfs_periods_total[5m])
)
```

**2. OOMKill events** — page on any non-zero value:

```promql
sum by (namespace, pod) (
  kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}
)
```

**3. Memory working set vs. limit** — alert above 80%:

```promql
(
  sum by (namespace, pod, container) (container_memory_working_set_bytes)
  /
  sum by (namespace, pod, container) (kube_pod_container_resource_limits{resource="memory"})
) > 0.8
```

**4. CPU request vs. actual usage ratio** — spots over-allocated pods:

```promql
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total[5m]))
/
sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu"})
```

### kubectl diagnostic toolkit

```bash
# Check pod resource consumption live
kubectl top pods -n <namespace> --containers

# Inspect pod events for OOMKills and throttling context
kubectl describe pod <pod-name> | grep -A10 "Events:"

# Look at container restart reasons
kubectl get pod <pod-name> -o jsonpath=\
  '{.status.containerStatuses[*].lastState.terminated}'

# Directly inspect cgroup limits inside a running container
kubectl exec -it <pod> -- cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# Check actual RSS vs. limit from inside the container
kubectl exec -it <pod> -- cat /proc/meminfo | grep -E "MemTotal|MemAvail"
```

### Node-level inspection

```bash
# Find all pods on a specific node
kubectl get pods --all-namespaces \
  --field-selector spec.nodeName=<node-name> -o wide

# Check node allocatable vs. requested
kubectl describe node <node-name> | \
  grep -A20 "Allocated resources"

# Look for system-level memory pressure signals
kubectl get events --all-namespaces | grep -i "evict\|oom\|memor"
```

---

## Chapter 6: Six Months of Lessons Distilled

### Lesson 01 — CPU average utilization is a lie

Your CPU limit headroom means nothing if your workload spikes in bursts shorter than the CFS period. Always instrument `container_cpu_cfs_throttled_periods_total`. Add it to your team's SLO dashboards before you ever touch CPU limit values.

### Lesson 02 — Memory limits must account for the full process footprint

Every runtime has overhead beyond your configured heap. Measure `container_memory_working_set_bytes` under load for at least a week before locking in limits. Add a minimum 30% buffer above observed peak. Then document *why* that number was chosen — you will forget.

### Lesson 03 — Never set limits without setting requests

A pod with `requests.cpu: 10m` and `limits.cpu: 2000m` is a scheduling time bomb. Never set requests significantly below what the application actually uses. Use VPA (Vertical Pod Autoscaler) in recommendation mode to get empirical baselines before committing to values.

### Lesson 04 — Consider removing CPU limits for latency-sensitive services

This is controversial, but increasingly defensible. If your node is properly sized and your autoscaling is working, CPU limits can cause more harm than good for services where latency matters. Use CPU requests to guarantee scheduling fairness, but let the pod burst freely. The Shopify and Zalando engineering teams have documented this approach in production at scale.

### Lesson 05 — Enforce LimitRanges in every namespace, without exception

Pods without limits are a cluster-level risk. One unconfigured batch job was responsible for two of our most severe production incidents. Make missing limits a CI/CD policy violation. Automate it. Trust nothing manually applied.

### Lesson 06 — Treat memory and CPU limits differently from each other

CPU throttling degrades performance gracefully. Memory OOMKills are hard failures. Be conservative with memory limits. Be more willing to experiment with CPU limits — the failure mode is increased latency, not application crashes.

---

## Practical Recommendations

**01. Use VPA in recommendation mode before setting any limits**

Deploy the Vertical Pod Autoscaler with `updateMode: "Off"` and run it for 2–4 weeks. Read `kubectl describe vpa <name>` to get observed memory and CPU percentile recommendations. These are grounded in real behavior, not guess-work YAML.

**02. Set requests = p50 of observed usage; limits = p95 + 30%**

For memory, this gives the scheduler an accurate picture and still protects against outlier spikes. For CPU, consider whether a hard limit is even the right choice for your workload type (see Lesson 04).

**03. Tune kubelet eviction thresholds on every new node type**

The defaults are wrong for production. Set soft eviction with grace periods to give the cluster time to reschedule gracefully before kernel OOM events occur. Document these settings in your node provisioning runbooks.

**04. Separate latency-critical and batch workloads onto different node pools**

Use node taints and pod tolerations to enforce this. Noisy neighbors are a structural problem — you cannot debug your way out of co-location. Use `PriorityClasses` to ensure critical services get eviction protection above batch jobs.

**05. Build the throttle ratio alert before anything else**

This single Prometheus alert would have caught three of our six production incidents in minutes instead of hours. If `throttled_periods / total_periods > 0.25` for more than 5 minutes, page someone. No other single metric gives you more signal about CPU limit misconfiguration.

---

## The Configuration Template We Settled On

```yaml
resources:
  requests:
    # Set to p50 of observed usage. This is what the scheduler sees.
    cpu: "250m"
    memory: "256Mi"
  limits:
    # CPU: Set to p95 + 20% for batch/background;
    #      Consider omitting entirely for latency-sensitive APIs.
    cpu: "1000m"
    # Memory: Set to p95 + 30% minimum.
    #         Never less than 1.5x requests for JVM services.
    memory: "512Mi"
```

For JVM services, always add container-aware JVM flags:

```yaml
env:
  - name: JAVA_TOOL_OPTIONS
    value: >-
      -XX:+UseContainerSupport
      -XX:MaxRAMPercentage=75.0
      -XX:InitialRAMPercentage=50.0
      -XX:+ExitOnOutOfMemoryError
```

---

## Quick Reference — The Metrics That Matter

| What to monitor | Prometheus metric / query |
|----------------|--------------------------|
| CPU throttle ratio | `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])` |
| Memory health (% of limit) | `container_memory_working_set_bytes / kube_pod_container_resource_limits{resource="memory"}` |
| OOMKill events | `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` |
| Node memory pressure | `kube_node_status_condition{condition="MemoryPressure",status="true"}` |
| cgroups version on node | `stat -fc %T /sys/fs/cgroup/` (in node shell) |

---

## Closing Thoughts

After six months and more incidents than we'd like to admit, the core insight is this: Kubernetes resource management is not a configuration problem. It's an **observability problem**.

The defaults and documentation nudge you toward a false confidence. The platform *looks* like it's enforcing what you configured, until the day it doesn't. The engineers who operate stable clusters aren't the ones who got the YAML right the first time — they're the ones who built the dashboards to see what the scheduler actually sees, and treated their resource settings as living configuration rather than set-it-and-forget-it policy.

Start with measurement. Build the throttle ratio alert tonight. Everything else follows.

---

*Production Post-Mortem Series — Platform Engineering*
