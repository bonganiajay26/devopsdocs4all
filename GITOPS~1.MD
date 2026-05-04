# GitOps Multi-Cloud Mastery

**A progressive, production-grade learning guide — Beginner to Expert**
**Author Persona:** Staff DevOps / Platform Engineer | GitOps Architect | Multi-cloud K8s (EKS, AKS, GKE)

---

## How To Use This Guide

Four levels, two parts each:
- **Part A — Concepts:** the *why* — principles, trade-offs, mental models.
- **Part B — Hands-on:** the *how* — real repos, real manifests, real commands.

You'll learn **both ArgoCD and FluxCD** so you can pick intelligently rather than tribally.

Prereqs:
- A Kubernetes cluster (any cloud or `kind`/`minikube`).
- A Git repo you control (GitHub/GitLab/Bitbucket).
- `kubectl`, `git`, plus `argocd` and/or `flux` CLIs.

---
---

# LEVEL 1 — BEGINNER

## PART A — CONCEPTS

### What GitOps Actually Is

**GitOps = operating infrastructure and applications by reconciling cluster state against a Git repository.** Git is the source of truth; a controller running *inside* the cluster pulls the desired state and continuously makes reality match.

That's it. Everything else is consequence.

### The Four Principles (per OpenGitOps)
1. **Declarative.** The system is described declaratively (YAML), not as a sequence of imperative steps.
2. **Versioned and immutable.** Desired state is stored in Git. Every change is a commit.
3. **Pulled automatically.** A software agent in the cluster pulls the desired state.
4. **Continuously reconciled.** The agent observes actual state and converges it to desired state, forever.

### GitOps vs Traditional CI/CD

| | Traditional CI/CD ("CIOps") | GitOps |
|---|---|---|
| **Push or pull?** | CI runner pushes to cluster | Agent in cluster pulls from Git |
| **Where credentials live** | In CI runner (often long-lived) | In cluster only |
| **Source of truth** | Last `kubectl apply` someone ran | Git repository |
| **Drift behavior** | Silent | Detected and reverted |
| **Audit** | CI logs (often ephemeral) | `git log` |
| **Rollback** | Re-run an older pipeline | `git revert` |

CIOps is fine; GitOps is better at scale because *the cluster never trusts the CI system*. You revoke a CI runner without revoking access to prod.

### The Reconciliation Loop
```
   Git repo (desired state)
         |
         | (poll / webhook)
         v
   GitOps controller in cluster
         |
         | compute diff (desired vs actual)
         v
   Apply diff via Kubernetes API
         |
         v
   Cluster state converges
         |
         | observe drift
         +-----> back to top
```

### Drift Detection
Drift = actual cluster state ≠ desired state in Git. Sources:
- Someone ran `kubectl edit` in an outage.
- Another operator (HPA, VPA) changed a field.
- A controller wrote a default.

GitOps tools either **alert on drift** (Flux default for some resources) or **auto-revert** (`selfHeal: true` in ArgoCD).

### Real-world Use Cases
- Multi-environment promotion (dev → staging → prod) by merging PRs.
- Multi-cluster fleet management from one repo.
- Audited compliance (HIPAA, SOC2) — every cluster change is in `git log`.
- Disaster recovery — rebuild a cluster by pointing the new one at the same repo.

### Best Practices (Day 1)
- One repo for cluster config (separate from app source code).
- One commit = one reviewable change.
- Tag releases; pin Helm chart versions and image digests.
- Branch protection on `main`; require PR review.

### Common Mistakes
- Putting cleartext secrets in the GitOps repo.
- Mixing application code and deployment manifests in one repo (couples build + deploy lifecycles).
- Letting humans `kubectl apply` while a reconciler is running. The reconciler will undo their change. Confusion ensues.

### Pro Tips
- Always make the GitOps controller the **only writer** to managed namespaces. Document this.
- Treat the cluster as **derived data**. The repo is canonical; the cluster can always be recreated.
- Start with one cluster + one app + one repo. Don't design for 50 clusters on day one.

---

## PART B — HANDS-ON: Your First GitOps Sync (ArgoCD)

### Objective
Install ArgoCD on a cluster, point it at a Git repo containing a single nginx Deployment, and watch it reconcile.

### Architecture
```
+------------------+        +----------------------+
| GitHub repo      |        |  Cluster             |
|  apps/web/       | <----- |  ArgoCD controller   |
|    deploy.yaml   | poll   |                      |
|    svc.yaml      |        |  --> applies to API  |
+------------------+        +----------------------+
        ^                              |
        | git push                     v
   developer                  +------------------+
                              |  web Deployment  |
                              |  web Service     |
                              +------------------+
```

### Repo Layout
```
gitops-demo/
├── apps/
│   └── web/
│       ├── deploy.yaml
│       └── svc.yaml
└── argocd/
    └── web-app.yaml
```

### Files

#### `apps/web/deploy.yaml`
**What:** A standard Kubernetes Deployment.
**Why:** This is the actual workload — GitOps doesn't change YAML, it changes *who applies it*.
**Where:** The `apps/` folder is the directory ArgoCD will watch.
**When:** Always — every workload starts as plain manifests in Git.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: web
spec:
  replicas: 2
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27.2
          ports: [{ containerPort: 80, name: http }]
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits:   { cpu: 250m, memory: 128Mi }
          readinessProbe: { httpGet: { path: /, port: http }, periodSeconds: 5 }
```
**Key lines:** `image: nginx:1.27.2` is **pinned** — the only safe default in GitOps. Floating tags cause silent drift.

#### `apps/web/svc.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: web
spec:
  selector: { app: web }
  ports: [{ port: 80, targetPort: http }]
```

#### `argocd/web-app.yaml` — The ArgoCD `Application`
**What:** Tells ArgoCD *what to sync from where to where*.
**Why:** ArgoCD's primary CRD. One per app per cluster.
**Where:** Lives in the `argocd` namespace.
**When:** Every app you manage with ArgoCD.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/gitops-demo.git
    targetRevision: main
    path: apps/web
  destination:
    server: https://kubernetes.default.svc   # this same cluster
    namespace: web
  syncPolicy:
    automated:
      prune: true        # delete resources removed from Git
      selfHeal: true     # revert out-of-band changes
    syncOptions:
      - CreateNamespace=true
```
**Key lines:**
- `source.path` — directory in the repo to sync (everything under it).
- `destination.server` — `https://kubernetes.default.svc` means "the cluster ArgoCD itself runs in"; for multi-cluster, this becomes a remote API URL.
- `syncPolicy.automated` — without this, ArgoCD only *notices* drift; it doesn't fix it.
- `prune: true` — git is the source of truth, including for deletions.
- `CreateNamespace=true` — saves a manifest. Common convenience.

### Commands

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Wait for it to be ready
kubectl -n argocd rollout status deploy/argocd-server

# 3. Get the initial admin password (auto-generated)
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# 4. Port-forward and log in
kubectl -n argocd port-forward svc/argocd-server 8080:443 &
argocd login localhost:8080 --username admin --insecure

# 5. Apply the Application manifest (the only kubectl apply you'll ever do for this app)
kubectl apply -f argocd/web-app.yaml

# 6. Watch it sync
argocd app get web
argocd app sync web         # manual sync (auto-sync also works)
argocd app wait web --health

# 7. Inspect on the cluster
kubectl -n web get pods,svc

# 8. Try drift recovery — change something on the cluster and watch it revert
kubectl -n web scale deploy/web --replicas=10
# wait ~30s ...
kubectl -n web get deploy/web    # back to 2 replicas
```

### Git Workflow
```bash
# 1. Make a change
vi apps/web/deploy.yaml          # bump replicas: 2 -> 4

# 2. Commit and PR
git checkout -b bump-web-replicas
git add apps/web/deploy.yaml
git commit -m "web: bump replicas to 4"
git push origin bump-web-replicas
# Open PR, get review, merge

# 3. ArgoCD picks up the change within ~3 minutes (or instantly via webhook)
argocd app get web
```

### Expected Output
- ArgoCD UI shows `web` Application as **Synced + Healthy**.
- Cluster has 2 pods of nginx running in the `web` namespace.
- Editing the Deployment via `kubectl` triggers a revert within ~30 seconds (`selfHeal`).

### Advanced Thinking Layer
- **Why is the GitOps controller in the cluster, not in CI?** Because the cluster *only trusts itself*. CI systems get rotated, hacked, or revoked; the controller keeps reconciling regardless.
- **Trade-off:** With auto-sync + selfHeal, you can no longer hot-fix via `kubectl`. That feels restrictive — until your prod stops mysteriously diverging from your repo.
- **Anti-pattern:** Two reconcilers (ArgoCD AND a CI pipeline that runs `kubectl apply`). Pick one.

---
---

# LEVEL 2 — INTERMEDIATE

## PART A — CONCEPTS

### Helm + GitOps
Helm gives you **templating + values + release lifecycle**. GitOps tools integrate with Helm in two flavors:
1. **Render at sync time** (ArgoCD's Helm support, Flux `HelmRelease`) — controller calls `helm template` and applies the result.
2. **Render at PR time** (your CI runs `helm template` and commits the rendered YAML to git) — clearer diffs, no Helm at runtime.

Pick **render-at-sync** for speed and ergonomics; **render-at-PR** for paranoid environments where you want every byte that hits the cluster reviewed.

### Kustomize + GitOps
Kustomize is template-free overlays — perfect for the dev/stage/prod pattern.

```
base/                  # shared manifests
├── deploy.yaml
├── svc.yaml
└── kustomization.yaml
overlays/
├── dev/
│   ├── kustomization.yaml
│   └── replica-patch.yaml
├── staging/
│   └── kustomization.yaml
└── prod/
    └── kustomization.yaml
```

Both ArgoCD and Flux speak Kustomize natively.

### Secrets in GitOps
"Just put the YAML in Git" is the right answer for everything except secrets. Three viable patterns:

**1. Sealed Secrets** — encrypt secrets to a cluster-specific public key; store the encrypted blob in Git; the controller decrypts on apply.
- Pros: simple, git-native.
- Cons: per-cluster keys; rotation is fiddly; no integration with cloud KMS.

**2. External Secrets Operator (ESO)** — store secrets in AWS Secrets Manager / Azure Key Vault / GCP Secret Manager; ESO syncs them into K8s `Secret` objects.
- Pros: cloud-native rotation, audit, IAM.
- Cons: external dependency; circular bootstrap (ESO needs creds to talk to the cloud — IRSA/Workload Identity solves this).

**3. SOPS + age/KMS** — encrypt the YAML file itself in Git; Flux has built-in SOPS decryption.
- Pros: works for any YAML, not just Secrets.
- Cons: ArgoCD support is plugin-based.

**Default recommendation:** External Secrets + cloud-native KV. Sealed Secrets only for clusters that *can't* reach a cloud KV (true air-gap).

### Multi-Environment Management
Three common shapes:

**Folder-per-env (most common):**
```
clusters/
├── dev/
├── staging/
└── prod/
```
Promote by merging the same change PR into each folder (or by templating).

**Branch-per-env:** A `dev`, `staging`, `prod` branch. **Avoid this.** Long-lived branches diverge; promotion becomes merge hell.

**Tag-per-env:** Releases tagged `v1.2.3`; environments pin to tags. Good for rigid release cadences.

### Best Practices
- Use Kustomize overlays *inside* a Helm chart, not instead of it, when you need both templating and per-env tweaks.
- Store cluster-level config (CRDs, controllers) in a separate repo from app config. Different cadences, different reviewers.
- Pre-render Helm charts in CI for prod — review the actual YAML, not the template.

### Common Mistakes
- Encrypting secrets with a single org-wide key. Rotate it once and you're locked out of every cluster.
- Long-lived `dev` branch that has 80 commits ahead of `prod`. The promotion PR is unreviewable.
- ESO with cleartext static AWS keys mounted in a Secret. (Use IRSA.)

### Pro Tips
- `kubeseal --raw` lets you seal a single value; great for one-off secrets without redoing the whole file.
- Validate all rendered manifests against the API server in CI: `helm template ... | kubectl apply --dry-run=server -f -`.
- For environment differences, prefer **a few well-named patches** over **many small ones**. Reviewers thank you.

---

## PART B — HANDS-ON: Helm + Kustomize + Sealed Secrets via ArgoCD

### Objective
Deploy the same app to **dev** and **prod** environments from one repo, using Helm for the chart and Kustomize for env overlays. Manage one secret with Sealed Secrets.

### Repo Layout
```
gitops-demo/
├── charts/
│   └── web/                          # Helm chart
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/...
├── envs/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── values.yaml               # Helm overrides
│   │   └── sealed-secret.yaml
│   └── prod/
│       ├── kustomization.yaml
│       ├── values.yaml
│       └── sealed-secret.yaml
└── argocd/
    ├── web-dev.yaml
    └── web-prod.yaml
```

### Files

#### `envs/dev/values.yaml`
**What:** Per-environment Helm overrides.
**Why:** Same chart, different shape per env.

```yaml
replicaCount: 2
image:
  tag: 1.27.2
ingress:
  host: web.dev.example.com
resources:
  requests: { cpu: 50m,  memory: 64Mi }
  limits:   { cpu: 250m, memory: 128Mi }
```

#### `envs/prod/values.yaml`
```yaml
replicaCount: 6
image:
  tag: 1.27.2
ingress:
  host: web.example.com
resources:
  requests: { cpu: 200m, memory: 256Mi }
  limits:   { cpu: 1,    memory: 512Mi }
```

#### `envs/dev/sealed-secret.yaml`
**What:** A SealedSecret — encrypted YAML safe to commit.
**Why:** Lets you keep secrets in Git without exposing them.
**Where:** Per-environment (each cluster has its own sealing key).
**When:** Anywhere you'd otherwise commit a `kind: Secret`.

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: web-db
  namespace: web
spec:
  encryptedData:
    DATABASE_URL: AgB7c3a8...long-base64-blob...==
  template:
    metadata:
      name: web-db
      namespace: web
    type: Opaque
```
**Key lines:**
- `encryptedData` — encrypted with the cluster's public key; only this cluster's controller can decrypt.
- `template` — defines the resulting `Secret`'s shape after decryption.

To create one: `kubeseal --format yaml < my-secret.yaml > sealed-secret.yaml`.

#### `argocd/web-dev.yaml` — Helm-aware Application
**What:** ArgoCD Application that renders a Helm chart with env-specific values.
**Why:** Same chart, two Applications (dev + prod), different `valueFiles`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/gitops-demo.git
    targetRevision: main
    path: charts/web
    helm:
      valueFiles:
        - ../../envs/dev/values.yaml      # path relative to source.path
  destination:
    server: https://kubernetes.default.svc
    namespace: web-dev
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```
**Key lines:**
- `helm.valueFiles` — list of values files to layer over `values.yaml`. Last one wins.
- `ServerSideApply=true` — better merge semantics with operators that mutate fields.

#### `envs/dev/kustomization.yaml` — for the SealedSecret
**What:** A Kustomize bundle that just deploys the SealedSecret (kept separate from the app chart so it has its own ArgoCD App).
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: web-dev
resources:
  - sealed-secret.yaml
```

### Commands
```bash
# Install Sealed Secrets controller (one-time, per cluster)
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Install kubeseal CLI (Mac shown; use brew/curl/apt accordingly)
brew install kubeseal

# Create a sealed secret
echo -n 'postgres://user:pass@host/db' | \
  kubectl create secret generic web-db --dry-run=client \
    --from-file=DATABASE_URL=/dev/stdin -o yaml | \
  kubeseal --format yaml > envs/dev/sealed-secret.yaml

# Commit and push — ArgoCD picks it up
git add envs/dev/sealed-secret.yaml
git commit -m "dev: add db credential"
git push

# Apply both Applications
kubectl apply -f argocd/web-dev.yaml -f argocd/web-prod.yaml

# Verify
argocd app list
argocd app sync web-dev
kubectl -n web-dev get secret web-db    # decrypted by the controller
```

### Promotion Workflow (dev → prod)
```bash
# 1. Test passes in dev
# 2. Open PR that bumps the image in envs/prod/values.yaml
git checkout -b promote-1.27.3
sed -i 's/tag: 1.27.2/tag: 1.27.3/' envs/prod/values.yaml
git commit -am "prod: promote web 1.27.3"
git push -u origin promote-1.27.3
# 3. PR review + merge -> ArgoCD reconciles prod within ~3 min
```

### Advanced Thinking Layer
- **Why a separate ArgoCD Application for the SealedSecret?** Different lifecycle. Secrets rotate; app deploys daily. Separating them means one doesn't trigger reconcile of the other.
- **Trade-off — Helm at sync time vs render-at-PR:** Sync-time is ergonomic; render-at-PR shows reviewers the *exact* YAML that will hit the cluster. Big-bank prod usually picks render-at-PR.
- **Anti-pattern:** Reusing one SealedSecret across clusters. Sealing keys are per-cluster. Use ESO instead.

---
---

# LEVEL 3 — ADVANCED

## PART A — CONCEPTS

### Multi-cluster GitOps
Two architectural patterns:

**Hub-and-spoke (recommended for most):**
```
   GitOps repo
       |
       v
   "Hub" cluster (runs ArgoCD/Flux)
       |
       +--> sync to spoke 1 (EKS prod us-east-1)
       +--> sync to spoke 2 (AKS prod eastus)
       +--> sync to spoke 3 (GKE prod uscentral1)
```
Pros: single pane of glass, single set of credentials to manage.
Cons: hub becomes critical; hub outage halts reconciliation everywhere.

**Per-cluster controllers:**
```
   GitOps repo
       |
       +--> ArgoCD/Flux in EKS  -> local cluster only
       +--> ArgoCD/Flux in AKS  -> local cluster only
       +--> ArgoCD/Flux in GKE  -> local cluster only
```
Pros: no single point of failure; tighter blast radius.
Cons: more controllers to upgrade; no global view (use a UI federation tool).

### ArgoCD Architecture (deeper)
```
+-------- ArgoCD (in cluster) --------+
| api-server          (gRPC + REST)   |
| repo-server         (renders mfsts) |
| application-controller (reconciles) |
| applicationset-controller (gen apps)|
| notifications-controller            |
| dex (optional SSO)                  |
| redis (cache + locks)               |
+-------------------------------------+
```
Key CRDs: `Application`, `ApplicationSet`, `AppProject`.

### Flux Architecture
```
+--------- Flux (in cluster) ---------+
| source-controller    (clones repos) |
| kustomize-controller (applies kustomization.yaml) |
| helm-controller      (renders HelmReleases) |
| notification-controller             |
| image-automation-controller (optional)|
+-------------------------------------+
```
Key CRDs: `GitRepository`, `Kustomization`, `HelmRepository`, `HelmRelease`, `OCIRepository`.

### ArgoCD vs Flux — the honest comparison

| | ArgoCD | Flux |
|---|---|---|
| UI | First-class web UI | Minimal (use Weave GitOps / capacitor) |
| Multi-tenancy | `AppProject` model | Tenancy via namespaces + RBAC |
| Multi-cluster | Hub model with `cluster` secrets | Each cluster runs its own |
| Helm | Renders inline | `HelmRelease` CRD |
| Image updates | `argocd-image-updater` (separate) | Built-in `image-reflector` + `image-automation` |
| Progressive delivery | Argo Rollouts (sister project) | Flagger (sister project) |
| Best for | Teams that want a UI; org-wide app catalog | Cluster-bootstrap; teams comfortable with CLI/CRDs |

Both are excellent. Pick one and commit. Switching costs are real but not catastrophic.

### CI/CD Integration with GitOps
Two repos:
- **App repo** — application source code; CI builds image and pushes to registry.
- **GitOps/config repo** — manifests; image tag is bumped here.

Two patterns to bridge them:

**A. Image-update bot** (Argo CD Image Updater, Flux Image Automation): the agent watches the registry, sees a new tag, opens a commit/PR to the GitOps repo.

**B. CI bumps the GitOps repo:** the app's CI pipeline opens a PR against the GitOps repo with the new image tag. More explicit, more reviewable, more popular.

```
PR (app repo) -> CI builds image:abc123 -> pushes to ECR/ACR/GAR
                                           |
                                           v
                                  CI opens PR (GitOps repo)
                                  bumping image tag to abc123
                                           |
                              merge -> ArgoCD/Flux sync -> rollout
```

### Progressive Delivery (Argo Rollouts / Flagger)
Replace Kubernetes' rolling update with smarter strategies:
- **Canary** — shift 10% → 25% → 50% → 100% of traffic to the new version, with metric checks at each step.
- **Blue-Green** — full new version up; flip traffic when validated.
- **Analysis** — automated rollback if error rate or latency exceeds SLO.

Argo Rollouts uses a `Rollout` CRD (replaces `Deployment`); Flagger orchestrates an existing Deployment + a service mesh.

### RBAC and Access Control
ArgoCD's `AppProject` controls which repos/clusters/namespaces an Application can target.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata: { name: team-payments, namespace: argocd }
spec:
  sourceRepos: ['https://github.com/payments-team/*']
  destinations:
    - namespace: 'payments-*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []        # no cluster-scoped resources
  namespaceResourceBlacklist:
    - { group: '', kind: ResourceQuota }
```
Then a `Role` in ArgoCD limits which users can create Apps in this Project.

### Best Practices
- One **AppProject per team**, one **Application per service per cluster**.
- Use **ApplicationSet** so adding a cluster doesn't mean copy-pasting Applications.
- Image updates by **CI bumping the repo**, not by image-update bots — better audit trail.

### Common Mistakes
- One ArgoCD instance managing 100+ clusters with default settings. Tune `controller-shard` count.
- ApplicationSet generators that fan out without `goTemplate` — debugging is hell.
- Argo Rollouts without proper analysis templates — you've replaced one deploy with a slower deploy.

### Pro Tips
- `argocd app diff <app>` is your friend — shows the live-vs-desired diff before sync.
- Flux's `flux trace` walks an object back to the Git commit that produced it. Audit gold.
- For multi-cluster, ApplicationSet's **cluster generator** + cluster labels lets you target by capability ("all GPU clusters get the inference workload").

---

## PART B — HANDS-ON: Multi-Cluster Fan-Out + Argo Rollouts (Canary)

### Objective
1. Use ArgoCD ApplicationSet to deploy the same app to three clusters (EKS, AKS, GKE) from one definition.
2. Replace the Deployment with an Argo Rollout doing a 4-step canary.

### Architecture
```
                      GitOps repo
                          |
                          v
         +--- Hub cluster running ArgoCD ---+
         |   ApplicationSet: web            |
         |     -> Application web-eks       |
         |     -> Application web-aks       |
         |     -> Application web-gke       |
         +----+--------------+-------------+
              |              |             |
              v              v             v
         +--------+    +--------+   +--------+
         |  EKS   |    |  AKS   |   |  GKE   |
         |  web   |    |  web   |   |  web   |
         | canary |    | canary |   | canary |
         +--------+    +--------+   +--------+
```

### Files

#### `argocd/cluster-secret-eks.yaml` (one per spoke cluster)
**What:** A K8s Secret of type `cluster` that ArgoCD recognizes as a registered cluster.
**Why:** Multi-cluster registration = creating these secrets in the hub.
**Where:** `argocd` namespace on the hub cluster.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-eks-prod
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    env: prod
    cloud: aws
    region: us-east-1
stringData:
  name: eks-prod-use1
  server: https://EKS_API_ENDPOINT
  config: |
    {
      "bearerToken": "<sa-token-from-spoke>",
      "tlsClientConfig": { "insecure": false, "caData": "<base64 ca>" }
    }
```
**Key lines:**
- `argocd.argoproj.io/secret-type: cluster` — magic label that tells ArgoCD to register it.
- `labels` — `env`, `cloud`, `region` — used by ApplicationSet generators to fan out selectively.
- `bearerToken` — comes from a ServiceAccount on the spoke cluster with appropriate RBAC.

(Easier: `argocd cluster add <kubeconfig-context>` does this for you.)

#### `argocd/web-appset.yaml`
**What:** ApplicationSet — generates one Application per matching cluster.
**Why:** "Add a 4th region" = label a new cluster; no YAML edits needed.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: web
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: prod
  template:
    metadata:
      name: 'web-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/gitops-demo.git
        targetRevision: main
        path: charts/web
        helm:
          valueFiles:
            - '../../envs/{{metadata.labels.cloud}}-{{metadata.labels.region}}/values.yaml'
      destination:
        server: '{{server}}'
        namespace: web
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: [CreateNamespace=true, ServerSideApply=true]
```
**Key lines:**
- `generators.clusters.selector` — selects cluster Secrets by label (`env=prod`).
- `template.source.helm.valueFiles` — interpolates per-cluster values file path. The `cloud` and `region` labels build the path.
- One Application is generated per matching cluster, automatically.

#### `charts/web/templates/rollout.yaml` — Argo Rollout (replaces Deployment)
**What:** A Rollout CRD that does canary deploys.
**Why:** Safer rollouts with metric-driven analysis.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: web
spec:
  replicas: 6
  strategy:
    canary:
      canaryService: web-canary
      stableService: web-stable
      trafficRouting:
        nginx:
          stableIngress: web
      steps:
        - setWeight: 10
        - pause: { duration: 2m }
        - setWeight: 25
        - pause: { duration: 5m }
        - analysis:
            templates: [{ templateName: success-rate }]
            args:
              - name: service-name
                value: web-canary
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
  selector:
    matchLabels: { app: web }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27.2
          ports: [{ containerPort: 80 }]
```
**Key lines:**
- `canaryService` / `stableService` — two Services; traffic-routing splits between them.
- `trafficRouting.nginx` — uses NGINX ingress to weight traffic; alternatives: Istio, ALB, SMI.
- `steps` — declarative canary recipe; `pause` waits for clock or for manual promotion.
- `analysis` — automated check; if it fails, Rollout aborts and reverts.

#### `charts/web/templates/analysistemplate.yaml`
**What:** Defines what "healthy canary" means.
**Why:** Without it, canary is just a slower deploy.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate }
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.99
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.observability:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",code=~"2.."}[2m])) /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```
**Key lines:**
- `successCondition` — pass if 99%+ of requests are 2xx.
- `failureLimit` — 3 consecutive failures abort the rollout.
- `provider.prometheus` — same idea works for Datadog, New Relic, Wavefront, web (any HTTP).

### Commands
```bash
# Register spoke clusters (or apply the cluster-secret YAMLs)
argocd cluster add eks-prod-use1
argocd cluster add aks-prod-eastus
argocd cluster add gke-prod-uscentral1

# Apply ApplicationSet
kubectl apply -f argocd/web-appset.yaml

# Verify it generated 3 Applications
argocd app list | grep ^web-

# Install Argo Rollouts (one-time per spoke cluster — install via GitOps in real life)
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f \
  https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Promote / abort canary
kubectl argo rollouts get rollout web -n web --watch
kubectl argo rollouts promote web -n web
kubectl argo rollouts abort web -n web

# Manual sync after a values change
argocd app sync web-eks-prod-use1
```

### CI/CD Bridge (snippet for app repo, GitHub Actions)
```yaml
- name: Build and push image
  run: |
    docker build -t ghcr.io/org/web:${{ github.sha }} .
    docker push ghcr.io/org/web:${{ github.sha }}

- name: Bump image in GitOps repo
  uses: peter-evans/repository-dispatch@v3
  with:
    token: ${{ secrets.GITOPS_REPO_TOKEN }}
    repository: org/gitops-demo
    event-type: image-bump
    client-payload: |
      { "service": "web", "image": "ghcr.io/org/web:${{ github.sha }}" }
```
A workflow in the GitOps repo listens on `repository_dispatch:image-bump` and opens a PR updating `envs/*/values.yaml`.

### Advanced Thinking Layer
- **Why ApplicationSet over a script that loops and creates Apps?** Because it's *declarative*. Removing a cluster label automatically removes the Application. Drift detection works on the meta-level.
- **Why Argo Rollouts over Kubernetes' rolling update?** Built-in metric-driven abort. A bad deploy that increases error rate gets reverted automatically — without a human paged.
- **Anti-pattern:** Adding `analysis` steps that query a Prometheus the *new app version* writes to. The deploy "succeeds" because the broken version reports 100% success on its own meaningless metrics. Always analyze against an external/golden signal.

---
---

# LEVEL 4 — EXPERT / ARCHITECT

## PART A — CONCEPTS

### Repo Strategy at Scale

Three patterns; pick by team topology, not by aesthetics.

**1. Mono-repo (everything in one)**
```
gitops/
├── platform/    # cluster bootstrap, controllers
├── shared/      # base charts, policies
└── apps/
    ├── web/
    └── payments/
```
Pros: atomic cross-cutting changes; one PR shows the whole picture.
Cons: blast radius; access control is harder; CI is slow.

**2. Multi-repo per team**
```
gitops-platform/      # owned by platform team
gitops-team-web/      # owned by web team
gitops-team-payments/ # owned by payments team
```
Pros: clear ownership; per-team CI/CD; granular RBAC.
Cons: cross-cutting changes touch N repos; bootstrap is more complex.

**3. App-of-apps (ArgoCD-specific)**
A "root" Application points at a folder of child Applications. Adds discoverability to either of the above.

**Recommended at scale:** Multi-repo, with a thin platform mono-repo for cluster bootstrap.

### Designing GitOps for 50+ Clusters
- **Sharded controllers.** ArgoCD's application-controller can shard (`ARGOCD_CONTROLLER_REPLICAS`); each shard owns N clusters.
- **Cache the repo.** `repo-server` clones every sync; use `--reposerver-allow-out-of-bounds-symlinks=false` and a generous `parallelism-limit`.
- **Webhook over polling.** Default poll is 3 min. With webhooks, sync starts in seconds.
- **Per-cluster bootstrap repo.** Cluster-specific config (CIDR, IRSA roles) lives in a small per-cluster repo; shared workloads live in the platform repo.

### Security at Scale

**Secret encryption:**
- Per-cluster sealing keys (Sealed Secrets) — rotate annually, with a documented procedure.
- For ESO, **IRSA / Workload Identity** so no static cloud creds exist anywhere.
- For SOPS, use cloud KMS with per-env keys; rotate via re-encrypt + commit.

**Access control:**
- Branch protection: require PR review on `main`, status checks for tests + policy.
- Code owners (`CODEOWNERS`) per directory; platform team owns `/platform`, app teams own their `/apps/<name>`.
- ArgoCD RBAC + AppProject so a payments engineer can't sync to the `kube-system` namespace of the `web` cluster.
- SSO via Dex (ArgoCD) or OIDC (Flux); no shared admin accounts.

**Supply chain:**
- Sign commits (`git commit -S`).
- Sign images (cosign).
- Verify signatures at admission (Kyverno, Connaisseur).

### Observability and Auditing
- **Git log = audit log.** Every change has author, time, commit message, PR.
- **ArgoCD events** → Prometheus + Grafana (Argo CD comes with dashboards).
- **Flux events** → `flux events`, Notifications controller → Slack/PagerDuty/MS Teams.
- **Sync history** kept for N days; integrate with SIEM if regulated.
- **Drift dashboards** — alert when an Application stays `OutOfSync` longer than 10 min.

### Disaster Recovery
The deepest GitOps benefit: **the cluster is derived data**.

DR runbook (any cloud, any cluster):
1. Provision a new cluster (Terraform — see your other guide).
2. Install the GitOps controller (one `kubectl apply` or `flux bootstrap`).
3. Apply your "root" Application / Kustomization pointing at the GitOps repo.
4. Wait. Cluster reconstitutes itself.

What you must have **outside Git**:
- The sealing key (Sealed Secrets) or KMS access (ESO/SOPS).
- Persistent volume snapshots / database backups (Velero handles this).
- DNS that you can repoint.

Test DR quarterly. Untested DR is theatre.

### Best Practices
- Pin everything: chart versions, image digests, controller versions.
- Render manifests in CI (`helm template`, `kustomize build`) and lint with `kubeconform` + OPA/Conftest.
- Treat the platform repo's `main` branch as **the** production state. No hot-fixes outside Git.
- Document the "break glass" procedure — when (rarely) a human needs to bypass GitOps in an outage.

### Common Mistakes
- One ArgoCD for everything, with `cluster-admin` everywhere.
- Cluster bootstrap that requires a human to run scripts. (Use Crossplane / Cluster API + GitOps.)
- "We'll fix the secret rotation later." Later is two years from now, after an audit.

### Pro Tips
- **App-of-apps with auto-generated children** scales better than a wall of hand-written Applications.
- **Renovate or Dependabot on the GitOps repo** for chart and image bumps — turns "did you update?" into a PR.
- For 100+ clusters, **fleet-style management** (Rancher Fleet, ArgoCD ApplicationSets w/ cluster generators) beats per-cluster reasoning.

---

## PART B — HANDS-ON: Production GitOps Reference (FluxCD edition)

We'll do the expert lab in **Flux** for variety. Same principles, different CRDs.

### Objective
1. Bootstrap Flux into a cluster from Git.
2. Manage app delivery via `Kustomization` and `HelmRelease`.
3. Wire SOPS for secrets.
4. Set up notifications to Slack on reconcile failure.
5. Show how the same model scales to multi-cluster.

### Repo Layout (the canonical Flux pattern)
```
flux-fleet/
├── clusters/
│   ├── eks-prod-use1/
│   │   ├── flux-system/        # bootstrapped automatically
│   │   ├── infrastructure.yaml # points at infrastructure/
│   │   └── apps.yaml           # points at apps/
│   ├── aks-prod-eastus/
│   └── gke-prod-uscentral1/
├── infrastructure/
│   ├── controllers/            # ingress-nginx, cert-manager, ESO
│   └── configs/                # ClusterIssuer, default policies
└── apps/
    ├── base/
    │   └── web/
    └── overlays/
        ├── eks-prod-use1/
        ├── aks-prod-eastus/
        └── gke-prod-uscentral1/
```

### Files

#### `clusters/eks-prod-use1/apps.yaml`
**What:** A Flux `Kustomization` pointing at the per-cluster app overlay.
**Why:** Tells Flux to apply everything under `apps/overlays/eks-prod-use1`.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  path: ./apps/overlays/eks-prod-use1
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: web
      namespace: web
```
**Key lines:**
- `interval` — reconcile every 10 minutes (plus event-driven on Git change via webhook).
- `prune: true` — same as ArgoCD's prune.
- `decryption.provider: sops` — Flux decrypts SOPS-encrypted YAML inline.
- `healthChecks` — Flux blocks reporting "ready" until these resources are healthy.

#### `apps/base/web/release.yaml`
**What:** A Flux `HelmRelease` — Flux's equivalent of `helm install`.
**Why:** Decouples the chart source from the install.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: web
  namespace: web
spec:
  interval: 10m
  chart:
    spec:
      chart: web
      version: '0.5.x'      # SemVer range; auto-update minor/patch
      sourceRef:
        kind: HelmRepository
        name: my-charts
        namespace: flux-system
  values:
    replicaCount: 3
    image:
      tag: 1.27.2
  install:
    remediation: { retries: 3 }
  upgrade:
    remediation: { retries: 3, remediateLastFailure: true }
```
**Key lines:**
- `chart.spec.version: '0.5.x'` — pin major+minor, allow patches. Conservative-but-not-stuck.
- `install.remediation.retries` — auto-rollback on failed install.

#### `apps/base/web/repository.yaml`
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: my-charts
  namespace: flux-system
spec:
  url: https://charts.example.com
  interval: 1h
```

#### `apps/overlays/eks-prod-use1/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: web
resources:
  - ../../base/web
patches:
  - target: { kind: HelmRelease, name: web }
    patch: |
      - op: replace
        path: /spec/values/replicaCount
        value: 8
      - op: add
        path: /spec/values/ingress/host
        value: web.us-east-1.example.com
```
**Why:** Same base, per-cluster overlay. Flux reads `kustomization.yaml`, layers patches, applies.

#### `apps/base/web/secret.enc.yaml` — SOPS-encrypted Secret
**What:** A regular Kubernetes Secret, but the `data` is encrypted with `sops`.
**Why:** Safe to commit; Flux decrypts at apply time.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-db
  namespace: web
type: Opaque
data:
  DATABASE_URL: ENC[AES256_GCM,data:abc123...,iv:...,tag:...,type:str]
sops:
  age:
    - recipient: age1qyqszq...
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        ...
        -----END AGE ENCRYPTED FILE-----
  lastmodified: '2026-05-04T12:00:00Z'
  version: 3.8.1
```
Created with: `sops --age <recipient> --encrypt --in-place secret.yaml`.
Decrypted by Flux using the `sops-age` Secret referenced in the Kustomization.

#### `infrastructure/controllers/notifications.yaml`
**What:** Wire Flux events to Slack.
**Why:** A failed reconcile that no one sees is a silent failure.

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: gitops-alerts
  secretRef:
    name: slack-url
---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata: { name: on-failure, namespace: flux-system }
spec:
  providerRef: { name: slack }
  eventSeverity: error
  eventSources:
    - kind: Kustomization
      name: '*'
    - kind: HelmRelease
      name: '*'
```

### Commands

```bash
# 1. Bootstrap Flux into a cluster
flux bootstrap github \
  --owner=your-org \
  --repository=flux-fleet \
  --branch=main \
  --path=clusters/eks-prod-use1 \
  --personal=false

# This commits the Flux controllers' manifests under clusters/eks-prod-use1/flux-system
# and installs them.

# 2. Add the SOPS decryption key as a secret
cat ~/.config/sops/age/keys.txt | \
  kubectl create secret generic sops-age -n flux-system \
    --from-file=age.agekey=/dev/stdin

# 3. Push your apps definitions and watch
git push
flux get kustomizations --watch
flux get helmreleases -A

# 4. Force a reconcile
flux reconcile source git flux-system
flux reconcile kustomization apps

# 5. Trace any object back to git
flux trace deployment/web -n web

# 6. Inspect events
flux events -A --since=10m

# 7. Suspend (pause) reconciliation during incident response
flux suspend kustomization apps
flux resume kustomization apps
```

### Multi-cluster bootstrap loop
```bash
for cluster in eks-prod-use1 aks-prod-eastus gke-prod-uscentral1; do
  kubectl config use-context $cluster
  flux bootstrap github \
    --owner=your-org --repository=flux-fleet \
    --branch=main --path=clusters/$cluster
done
```
Each cluster pulls only its own folder. Adding a cluster = creating a new folder + bootstrap.

### Operational Runbook (the cheat sheet)
```bash
# Is GitOps healthy?
flux check
flux get all -A
argocd app list

# Why is X stuck?
flux events --for Kustomization/apps --since=1h
argocd app get <name>
argocd app diff <name>

# Force re-sync
flux reconcile kustomization apps --with-source
argocd app sync <name> --prune

# Rollback
git revert <bad-commit> && git push          # The True GitOps Way
argocd app rollback <name> <revision>        # Bypass-git escape hatch (AVOID in prod)
flux suspend hr web -n web                    # Stop reconciler, then manually fix

# Emergency: stop all reconciliation
flux suspend kustomization --all -A
# Fix the issue in git, then:
flux resume  kustomization --all -A
```

### Disaster Recovery Drill
```bash
# 1. Provision new cluster
terraform -chdir=clusters/dr apply

# 2. Bootstrap Flux against the SAME repo, new path
flux bootstrap github --owner=your-org --repository=flux-fleet \
  --branch=main --path=clusters/eks-dr-usw2

# 3. Restore PVs from Velero
velero restore create --from-backup nightly-2026-05-03

# 4. Repoint DNS (Route53/Front Door/Cloud DNS)
# 5. Validate
flux get all -A
kubectl get pods -A | grep -v Running    # should be empty
```

### Advanced Thinking Layer
- **Why fully bootstrap Flux from Git?** Because the bootstrap *itself* is a GitOps action. Lose the cluster, re-run `flux bootstrap`, the cluster reconstitutes from the same repo. No special "first-time" scripts.
- **Why SOPS over Sealed Secrets at scale?** SOPS encrypts to cloud KMS keys you already have IAM around. Sealed Secrets keys are local — fine for one cluster, painful at fifty.
- **Why suspend instead of delete during an incident?** Suspended resources still exist; resume picks up where it left off. Deleting causes prune cascades you didn't want.
- **Anti-pattern: hot-fix in cluster, then "remember to update Git later."** You will not remember. The next reconcile will undo your fix. Always commit first, sync second.

---
---

# COMMANDS QUICK REFERENCE

### Git workflows
```bash
git checkout -b feat/web-bump-1.27.3
# edit files
git add . && git commit -m "web: bump to 1.27.3"
git push -u origin feat/web-bump-1.27.3
# open PR, get review, merge to main
# GitOps controller picks it up within 1-3 minutes
```

### kubectl (GitOps-relevant)
```bash
kubectl get applications -n argocd
kubectl get kustomizations,helmreleases -A
kubectl get gitrepositories -A
kubectl describe application web -n argocd
kubectl logs -n argocd deploy/argocd-application-controller
kubectl logs -n flux-system deploy/source-controller
```

### argocd CLI
```bash
argocd login <server>
argocd cluster add <kubectx>
argocd cluster list
argocd repo add https://github.com/org/gitops --username <u> --password <p>
argocd app create <name> \
  --repo https://github.com/org/gitops \
  --path apps/web \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace web \
  --sync-policy automated --auto-prune --self-heal
argocd app list
argocd app get <name>
argocd app diff <name>
argocd app sync <name> --prune
argocd app rollback <name> <revision>
argocd app history <name>
argocd app delete <name>
argocd appset list
```

### flux CLI
```bash
flux check --pre              # preflight
flux bootstrap github ...     # install + commit
flux get sources git
flux get kustomizations -A
flux get helmreleases -A
flux reconcile source git flux-system
flux reconcile kustomization apps --with-source
flux suspend kustomization apps
flux resume  kustomization apps
flux trace deployment/web -n web
flux events --for Kustomization/apps --since=10m
flux logs --all-namespaces --level=error
flux uninstall   # removes Flux but leaves workloads
```

---

# FINAL CHECKLIST — "Production-Grade GitOps" Means

- [ ] Git is the only writer to managed namespaces (documented + enforced).
- [ ] One Application/Kustomization per (service × cluster), generated from a template (ApplicationSet / loop).
- [ ] Auto-sync + selfHeal/prune ON in non-emergency operation.
- [ ] Secrets via External Secrets (preferred) or SOPS/Sealed Secrets (acceptable).
- [ ] No static cloud credentials anywhere — IRSA / Workload Identity / AAD WI.
- [ ] PR review + branch protection + signed commits on `main`.
- [ ] Rendered-manifest validation in CI (`kubeconform`, OPA/Conftest).
- [ ] Image bumps via PR (CI-driven), not in-cluster mutation.
- [ ] Notifications wired to chat for sync failures and drift.
- [ ] Multi-cluster managed by ApplicationSet / per-cluster Flux bootstrap.
- [ ] Disaster recovery drill executed in the last 90 days.
- [ ] Documented break-glass procedure for the rare time you must bypass GitOps.

If you can tick all twelve, you don't just *do* GitOps — you've made the cluster a function of the repository.
