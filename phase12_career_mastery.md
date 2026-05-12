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
