# The ADR that saved us 8 weeks: how we chose ArgoCD over Flux for a 200-service GitOps migration

> **Platform Engineering · ADR Retrospective** · 18 min read

---

## 01 · The fire that started it

### Three broken deployments in one sprint. That was the last straw.

It was week eleven of the quarter and we'd hit the wall. Our platform team — four engineers supporting roughly 200 microservices across two Kubernetes clusters — had just watched three separate teams deploy broken configs to staging because someone had manually edited a YAML file and never committed the change. Not the first time. The fourteenth.

We'd been drifting toward a deployment model that looked like this: half the services were using a CI pipeline to `kubectl apply` directly, a quarter were managed by Helm releases with no unified state tracking, and the remaining quarter were sitting in a grey zone where nobody was quite sure how they got there. Ticket queues for deployment help were running a two-day SLA. Engineers were spending more time debugging drift than writing features.

> The question wasn't whether to adopt GitOps. The question was how to adopt it without grinding three months of product velocity into dust in the process.

GitOps was the obvious answer. Declarative source-of-truth, automated sync, drift detection. The hard part was choosing a tool and shipping a migration plan that 12 product teams would actually adopt — fast.

---

## 02 · Context & constraints

### Who we are and what we couldn't compromise

Four platform engineers. Two production clusters (one AWS EKS, one GKE for a regional partner). A mix of 200 services spanning Go, Python, and Node.js. Helm releases ranging from chart version 2.x to 3.x. Six years of accumulated Kubernetes YAML with a cultural tendency to treat it as write-only configuration.

### Non-negotiables

We had three hard constraints coming out of an Engineering Leadership conversation:

**Constraint 1 — No big-bang migration.**
Migration couldn't be "big bang." We'd been burned by a big-bang Istio rollout two years prior. Any GitOps tool had to support incremental adoption — existing services could stay on the old model temporarily while new services onboarded to GitOps from day one.

**Constraint 2 — Self-service onboarding.**
Developer experience had to be self-service. Our team couldn't be the bottleneck for every new app onboarding. Engineers needed to be able to register a new service without filing a ticket.

**Constraint 3 — Audit trail and RBAC.**
We were heading into a SOC 2 Type II audit in Q3. Any deployment tooling needed to satisfy the auditor's requirements for who-deployed-what-when.

### Existing pain points

The hardest part to admit in the ADR: we'd built a lot of bespoke deployment scaffolding over the years. Python scripts wrapping `helm upgrade` calls, a homegrown "deployment controller" that lived on a single engineer's laptop, and a Slack bot named `shipbot` that — I am not making this up — had no test coverage. Any migration had to be compatible with a gradual decommissioning of these tools.

---

## 03 · What we evaluated

### ArgoCD vs Flux: an honest breakdown

We ran a structured four-week evaluation. Two engineers each took ownership of one tool and stood up a proof-of-concept using 10 representative services — a mix of high-traffic, stateful, and multi-cluster-dependent workloads. We used a shared scoring rubric and presented our findings in a working session with the broader engineering team.

Here's where each tool landed across our key criteria:

| Criterion | ArgoCD | Flux v2 | Winner |
|---|---|---|---|
| **UI / Dev experience** | Rich web UI, sync status, resource graph, diff view built in | CLI-first; `flux get kustomizations` is not intuitive for app teams | **ArgoCD** |
| **Multi-cluster support** | First-class via ApplicationSets; hub-spoke model works out of the box | Supported but requires more manual Flux instance management per cluster | **ArgoCD** |
| **Sync & drift detection** | Reconciles every 3 min by default; webhook-triggered sync also available | Continuous reconciliation via controller; often faster in practice | **Flux** |
| **CRD complexity** | Application, AppProject, ApplicationSet — CRDs proliferate quickly in large orgs | GitRepository + Kustomization model is more composable and K8s-native | **Flux** |
| **Helm support** | Native Helm rendering with override values; works with existing charts | HelmRelease CRD works well but adds cognitive overhead for migrated services | **ArgoCD** |
| **Debuggability** | Sync errors surfaced in UI with diffs; logs accessible without cluster access | Requires `kubectl` access or Flux CLI; harder for app teams to self-diagnose | **ArgoCD** |
| **RBAC model** | AppProject-based RBAC; powerful but verbose to configure | Leverages native Kubernetes RBAC; less bespoke policy to maintain | **Flux** |
| **Onboarding speed** | App teams can self-onboard via Application manifest; UI gives immediate feedback | Steeper learning curve; onboarding requires understanding of Kustomize overlay model | **ArgoCD** |
| **Ecosystem maturity** | CNCF graduated; large community; wide operator familiarity | CNCF graduated; strong community but less operational tooling (e.g., notifications) | **Tie** |
| **Resource footprint** | Higher baseline memory (~500MB controller + Redis); HA setup adds cost | Lighter resource usage; controller set is modular | **Flux** |

*Evaluation conducted Q2 on EKS 1.28. ArgoCD version 2.9, Flux version 2.2.*

### The uncomfortable discovery

Flux won on architectural elegance. Its Kustomization-first model is genuinely more composable, its CRDs are more K8s-native, and it uses significantly fewer resources. If we were starting from a clean slate with a team of senior platform engineers, Flux would have been a serious contender — possibly the winner.

But we weren't starting from a clean slate. We had 200 services to migrate, a team of 12 product squads with varying Kubernetes fluency, and a hard deadline tied to the SOC 2 audit. That changed the calculus entirely.

---

## 04 · Decision rationale

### The ADR: why ArgoCD won in this context

```
ADR-047 · GitOps tooling selection for platform migration
Status: Accepted

Decision:
  Adopt ArgoCD as the primary GitOps controller for all Kubernetes
  workloads across EKS and GKE clusters.

Context:
  Migration of ~200 services from ad-hoc Helm/kubectl deployments to
  a unified GitOps model. Team of 4 platform engineers, 12 product squads,
  Q3 SOC 2 audit deadline. Two clusters with multi-cluster orchestration
  requirements.

Drivers:
  - Self-service onboarding
  - Audit trail
  - Developer debuggability without requiring cluster credentials
  - Multi-cluster support via single control plane
  - Compatibility with existing Helm charts

Decision:
  ArgoCD's UI-first operational model, ApplicationSet-driven multi-cluster
  support, and lower onboarding overhead for non-platform engineers outweigh
  Flux's architectural advantages given our specific constraints.

Tradeoffs accepted:
  - Higher resource overhead (~500MB baseline vs Flux ~120MB)
  - Increased CRD surface area
  - AppProject RBAC is more verbose than native K8s RBAC
  - Flux's reconciliation model is architecturally superior for
    greenfield environments

Consequences:
  - Platform team owns ArgoCD HA deployment and AppProject templates
  - Product teams own their Application manifests
  - All drift events emit to PagerDuty via Argo notifications
  - Review in 12 months
```

One line in the ADR draft that we argued about for two hours: *"ArgoCD's UI is a feature, not a crutch."* Some of us felt that rewarding engineers for not learning the CLI was wrong. We kept it in. The pragmatic argument won: if a service team can self-diagnose a failed sync from the ArgoCD UI without paging the platform team, that's three fewer incidents per sprint. We validated this assumption in the PoC — four out of five sync failures in our pilot were resolved by the app team themselves using the diff view.

---

## 05 · Migration approach

### Rolling out GitOps to 200 services without setting anything on fire

We rejected the idea of a dedicated "migration sprint." Experience told us that big-bang migrations almost always leave a tail of stragglers that never get cleaned up. Instead, we designed a four-phase rollout that ran in parallel with normal product work.

**Phase 0 — Foundations (weeks 1–2)**
Stood up ArgoCD HA on a dedicated `platform` namespace. Defined AppProject templates for three trust levels: `platform`, `product`, and `experiment`. Built a self-service onboarding script that generated an Application manifest from a team's existing Helm values. Wrote the runbook. Ran it past the security team for the SOC 2 requirements.

**Phase 1 — Canary services (weeks 3–5)**
Migrated 12 services owned by two teams who'd volunteered to be early adopters. These were non-critical, low-traffic services with simple Helm charts. We deliberately included one stateful service (a Postgres-backed cache) to force ourselves to think through sync ordering and resource hooks early.

**Phase 2 — Team-by-team rollout (weeks 6–14)**
A platform engineer was embedded in each team's sprint planning for one cycle. We ran office hours twice a week. Each team owned their migration — we held the platform steady, they drove the Application manifests. The rule: any new service deployed after week 6 had to use ArgoCD. No exceptions. This stopped the sprawl from growing while we worked backward.

**Phase 3 — Legacy cleanup & shipbot retirement (weeks 15–18)**
By this phase, ~170 of 200 services were managed by Argo. The remaining 30 were complex stateful workloads or services with bespoke CI integrations. We migrated these with explicit runbooks for each. `shipbot` was decommissioned in week 16. Nobody filed a ticket asking for it back.

**Phase 4 — Multi-cluster ApplicationSets (weeks 18–20)**
Graduated to ApplicationSets for services running on both clusters. The generator pattern meant we could express a multi-cluster deployment in about 40 lines of YAML instead of maintaining two identical Application manifests.

> **Guardrail we almost skipped:**
> We nearly skipped adding `syncPolicy.automated.selfHeal: false` as the default for production namespaces. A team member pushed back — "shouldn't the whole point be auto-sync?" The answer is: yes, eventually, but not on week one of a migration. We enabled auto-sync in staging, manual sync in production for the first 60 days. This meant humans were approving production syncs explicitly. Three teams caught breaking changes this way before they hit production.

---

## 06 · Impact

### Where the 8 weeks came from

| Metric | Result |
|---|---|
| Weeks saved vs original estimate | **8 weeks** |
| Platform team deployment tickets | **↓ 73%** |
| New service onboarding speed | **~4× faster** |
| Drift incidents post-migration | **0** (vs 14 in prior quarter) |

The 8-week estimate deserves some unpacking, because "we saved X weeks" is the kind of claim that sounds like marketing copy if you don't explain the math.

Our original migration plan — written before we chose a tool — assumed a hybrid model where platform engineers would onboard services in batches. Based on the complexity of our legacy deployment scripts, we estimated 4–5 hours per service for a full migration, which put the tail of the project at roughly 22 weeks for all 200 services.

ArgoCD changed the unit economics. The self-service onboarding script we built — which took the output of a `helm get values <release-name>` and generated a valid Application manifest — reduced the per-service migration effort to roughly 45 minutes when handled by the app team themselves. More importantly, it removed the platform team from the critical path entirely for the 70% of services that had a clean Helm release with no custom hooks.

The remaining 30% still needed platform involvement, but with ArgoCD's resource graph and sync status UI, the diagnostic conversations were shorter. An engineer showing up to office hours with a screenshot of a failed sync and a rendered diff is a very different conversation from one showing up with a vague "my deployment is broken."

> **The hidden saving:** Cognitive load reduction is hard to quantify but impossible to ignore. After the migration, our on-call rotation saw a meaningful drop in "what state is X service in?" questions. The ArgoCD UI became the authoritative answer to that question. Platform engineers stopped context-switching mid-task to check whether a service was drifted.

---

## 07 · Lessons learned

### What we'd do differently — and when Flux is actually the right answer

- **Start with ApplicationSets, not Applications.** We retrofitted ApplicationSets in phase 4 for multi-cluster services. If we'd started with the ApplicationSet model from day one, we'd have avoided maintaining duplicate manifests for six weeks.

- **Don't let auto-sync creep into production on week one.** Teams wanted it immediately. We held the line. It was the right call — manual sync in production for the first 60 days gave us a safety net while we built confidence in the pipeline.

- **The onboarding script was the migration.** The single highest-leverage investment we made was that 200-line Python script that generated Application manifests from existing Helm releases. If we'd spent a week on documentation instead, migration would still be running.

- **Resource hooks for stateful workloads are a first-class concern.** We learned this the hard way when a schema migration job ran twice because we hadn't configured `argocd.argoproj.io/hook: PreSync` correctly. Document this before Phase 1, not during Phase 3.

- **The RBAC conversation needs to happen before any service is onboarded.** We had to retroactively tighten AppProject policies after a team accidentally published secrets into a public sync repo. A 30-minute workshop on secret management — external secrets operator, not plaintext YAML — should be mandatory before onboarding.

- **Audit your CRD count at 3 months.** We had 847 Application objects by month three. Not all of them were still needed. CRD sprawl is real with ArgoCD — build a pruning process early.

### When Flux is the better choice

I want to be specific here, because "it depends" is not useful guidance.

Choose Flux if: your team is composed entirely of experienced Kubernetes operators who are comfortable with Kustomize; you're building from scratch with no legacy Helm releases; you have tight resource constraints where ArgoCD's footprint is a meaningful cost; or your organizational security model strongly prefers native Kubernetes RBAC over a bespoke policy engine. Flux's architectural cleanliness is a real advantage in those contexts — we just weren't in them.

Also: if you're building a platform that will be operated by a small number of engineers all working from the CLI, Flux's operator-first design is genuinely better. ArgoCD's UI is most valuable when you're trying to extend visibility to engineers who aren't living in `kubectl` all day.

---

## 08 · Takeaway

### The real lesson isn't about ArgoCD

The most important thing we did wasn't choosing ArgoCD. It was writing the ADR at all — being forced to articulate our constraints explicitly, document the tradeoffs we were accepting, and commit to a review date.

Platform engineering decisions made in a hurry tend to optimize for the tool's best properties in the abstract rather than the team's actual constraints. Flux has better architectural properties for a Kubernetes-native, greenfield deployment. We didn't have a greenfield deployment. We had 200 services, four engineers, a legacy scaffolding problem, and twelve product teams who needed to stay productive during the migration.

> The right tool for a migration is the one your team will actually use — and that the teams depending on you will adopt without a support ticket.

ArgoCD was the right tool for our context. It might not be for yours. Write the ADR. Document the tradeoffs. Set a review date and honor it. That discipline will serve you better than any particular technology choice.

---

*We're twelve months in now. The review is on the calendar. We're watching Flux v2's multi-cluster story with interest. The ADR is still open.*
