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
