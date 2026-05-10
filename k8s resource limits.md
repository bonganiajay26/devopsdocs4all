## **Kubernetes Resource Limits Are Lying to You — What We Discovered After 6 Months in Production**

If you’ve ever set a CPU limit on a container and wondered why your application's tail latency spiked despite the dashboard showing "30% utilization," you’ve been lied to.

After six months of managing high-traffic clusters in production, we learned that Kubernetes resource management isn't a simple bucket system—it's a complex interaction of Linux kernel primitives that often behave counter-intuitively. Here is what we discovered when the theory met the reality of the metal.

### ---

**The Grand Illusion: Understanding the Mental Model vs. Reality**

Most engineers approach Kubernetes resources with a "physical box" mental model. They assume:

1. **Requests** are the minimum guaranteed resources.  
2. **Limits** are a hard ceiling that, once hit, simply prevents the app from taking more.

While mostly true for **Memory**, it is fundamentally false for **CPU**.

In reality, K8s doesn't "give" a container CPU; it uses **Completely Fair Scheduler (CFS) Quotas**. A CPU limit is actually a time budget. If you set a limit of 200m (20% of a core) on a 100ms period, your application gets 20ms of execution time. Once those 20ms are used, the kernel **throttles** the process—effectively pausing it—until the next period starts.

### ---

**Production Failure Modes: When the Lies Unravel**

During our six-month tenure, we witnessed three primary failure modes that cost us significant uptime before we identified the root causes.

#### **1\. The "Ghost" Latency (CPU Throttling)**

We had a Java microservice with a CPU limit of 2 cores. Prometheus showed average usage at 0.8 cores. Yet, our P99 latency was triple what it was on a VM.

* **The Discovery:** The application was highly multi-threaded. It would burst, consuming its entire 100ms quota in the first 10ms of a cycle, and then get throttled for the remaining 90ms.  
* **The Lesson:** Low average utilization does not mean you aren't being throttled.

#### **2\. The Silent Killer (OOMKill and Page Cache)**

Memory limits are enforced by cgroups. When a container hits its memory limit, the kernel doesn't just stop it; it invokes the **Out Of Memory (OOM) Killer**.

* **The Discovery:** We saw pods crashing with OOMKilled even though the application heap was tuned correctly. The culprit? The kernel's page cache. The app was reading large files, filling the page cache, and the kernel opted to kill the process rather than efficiently reclaiming cache under certain pressure conditions.

#### **3\. The Noisy Neighbor (Node-Level Exhaustion)**

If you set your requests too low but don't set limits, your pod can "burst" to use all available CPU on the node.

* **The Discovery:** A non-critical batch job scaled up, saw "free" CPU on the node, and consumed it all. This starved the Kubelet (the node's agent), causing the entire node to go NotReady and triggering a massive, unnecessary pod migration storm.

### ---

**Debugging the Kernel: Tools of the Trade**

Standard kubectl top pods is insufficient for deep reliability work. It provides a snapshot, not the "why." To get the truth, we had to look deeper.

#### **Inspecting cgroups Directly**

To see if you are being throttled, don't look at CPU usage; look at the **throttled time**.

Bash

\# Inside the node/container  
cat /sys/fs/cgroup/cpu/cpu.stat

* nr\_periods: Total periods elapsed.  
* nr\_throttled: How many times the app was paused.  
* throttled\_time: The total nanoseconds the app sat waiting for its next quota.

#### **Prometheus/Grafana Metrics**

We moved our monitoring focus to container\_cpu\_cfs\_throttled\_periods\_total. If this number is trending up, your performance is degraded, regardless of what the "CPU Usage" percentage says.

### ---

**Lessons Learned: The Production Playbook**

After 6 months, we scrapped our initial "best practices" and adopted these hard-earned rules:

1. **CPU Limits are Optional (but Requests are Mandatory):** In high-performance, low-latency environments, we often **remove CPU limits entirely**. By setting a high request and no limit, we allow the app to use idle cycles without the risk of aggressive CFS throttling, while still ensuring it has its guaranteed slice of the pie.  
2. **The "Buffer Rule" for Memory:** Always set Memory limits equal to requests. This creates "Guaranteed" Quality of Service (QoS), making the pod the last to be evicted when the node is under pressure.  
3. **Observability over Guesswork:** Never set a limit without looking at the throttled\_seconds metric. If you see throttling at 40% utilization, your limit is too low for the "burstiness" of your threads.

### ---

**Practical Recommendations**

| Resource | Strategy | Why? |
| :---- | :---- | :---- |
| **CPU Requests** | Set based on P50/P90 usage | Ensures the pod is scheduled where it can actually run. |
| **CPU Limits** | 2x-5x Requests or **None** | Prevents artificial latency spikes in multi-threaded apps. |
| **Memory Requests** | Match the Limit | Prevents memory overcommitment and unpredictable OOMKills. |
| **Memory Limits** | Heap \+ 20% | Accounts for non-heap memory, overhead, and the stack. |

### **Final Verdict**

Kubernetes resource limits are a blunt instrument for a sharp problem. If you treat them as "set and forget" values, you are leaving your application's reliability to the whims of the Linux scheduler. Start monitoring your **throttling** and **eviction** metrics today—because the "utilization" graphs are lying to you.