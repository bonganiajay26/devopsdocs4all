# Kubernetes Multi-Cloud Mastery

**A progressive, production-grade learning guide — Beginner to Expert**
**Author Persona:** Senior Kubernetes Architect | Multi-cloud (AWS EKS, Azure AKS, GCP GKE)

---

## How To Use This Guide

Four levels, two parts each:

- **Part A — Concepts:** the mental models, best practices, and traps.
- **Part B — Hands-on:** real manifests, real commands, file-by-file breakdowns.

Don't skip the labs. Kubernetes only clicks when you watch pods crash, read events, fix the YAML, and watch them go Ready.

Cluster prereqs (any one is fine for early levels):
- **EKS:** `eksctl create cluster --name demo --region us-east-1 --nodes 2`
- **AKS:** `az aks create -g demo-rg -n demo --node-count 2 --generate-ssh-keys`
- **GKE:** `gcloud container clusters create demo --num-nodes=2 --zone=us-central1-a`
- **Local:** `kind create cluster` or `minikube start`

---
---

# LEVEL 1 — BEGINNER

## PART A — CONCEPTS

### Kubernetes in One Paragraph
Kubernetes is a control loop system. You declare *desired state* in YAML; controllers continuously reconcile reality toward that state. If a pod dies, the Deployment controller notices the gap and creates a new one. Everything else — services, scaling, rollouts — is the same pattern applied to different objects.

### Architecture (text diagram)
```
+----------------------- CONTROL PLANE -----------------------+
|                                                             |
|   kube-apiserver  <----- you (kubectl) and controllers      |
|         |                                                   |
|         v                                                   |
|   etcd (state store)                                        |
|         ^                                                   |
|         |                                                   |
|   controller-manager  +  scheduler  +  cloud-controller     |
+-----------------------------------------+-------------------+
                                          |
                                          v (API)
+--------------------- WORKER NODES -----------------------+
|  kubelet  +  kube-proxy  +  container runtime (containerd)|
|     |                                                     |
|     v                                                     |
|  Pods (1+ containers, shared network/IPC/storage)        |
+----------------------------------------------------------+
```

| Component | Role |
|---|---|
| **kube-apiserver** | The only thing that talks to etcd. All clients go through it. |
| **etcd** | Strongly-consistent KV store for cluster state. |
| **scheduler** | Decides *which node* a pending pod should run on. |
| **controller-manager** | Runs the reconcile loops (Deployment, Job, Node, etc.). |
| **kubelet** | Node agent. Talks to the runtime, reports status. |
| **kube-proxy** | Programs iptables/IPVS to implement Service routing. |

### Core Objects
- **Pod** — Smallest deployable unit. One IP, shared volumes. Usually one app container + maybe sidecars.
- **ReplicaSet** — Maintains N pod replicas. You almost never write one directly.
- **Deployment** — Manages ReplicaSets to do rolling updates and rollbacks. **This is the workhorse.**
- **Service** — Stable virtual IP/DNS in front of a set of pods.
  - `ClusterIP` — internal only (default).
  - `NodePort` — exposes a port on every node (dev only, mostly).
  - `LoadBalancer` — provisions a cloud load balancer (ELB/ALB on AWS, Azure LB, GCP NLB).
- **Namespace** — Logical partition. Use one per team/env, not one per microservice.
- **ConfigMap / Secret** — Non-sensitive / sensitive config injected as env vars or files.
- **Probes** — `liveness` (restart if fails), `readiness` (remove from Service if fails), `startup` (delay other probes during slow boot).

### Resource Requests vs Limits
- **request** = what the scheduler reserves (capacity planning).
- **limit** = the hard ceiling enforced at runtime.
- Memory over limit → **OOMKilled**. CPU over limit → **throttled** (not killed).

### Best Practices
- Always set `requests` *and* `limits`.
- Use **Deployments**, never raw Pods, for stateless workloads.
- Use a **non-default namespace** per app/env.
- Pin image tags to **immutable digests** (`@sha256:...`) in production, not `:latest`.
- Add **liveness + readiness** probes to every workload.

### Common Mistakes
- Running as `root` inside containers (set `runAsNonRoot: true`).
- Using `:latest` and being surprised when pods change behavior overnight.
- Setting CPU limits too aggressively → unexplained latency from throttling.
- Confusing `kubectl exec` debugging with production observability.

### Pro Tips
- `kubectl explain deployment.spec.strategy` — interactive schema. Better than Googling.
- `kubectl get events --sort-by=.lastTimestamp` — your first stop when something is wrong.
- Set `terminationGracePeriodSeconds` to match how long your app needs to drain.

---

## PART B — HANDS-ON: Your First App on Kubernetes

### Objective
Deploy an `nginx` web app, expose it inside the cluster, then expose it externally on a cloud LoadBalancer. Wire in a ConfigMap and probes.

### Architecture
```
   Internet
      |
      v
+--------------------+
|  Cloud LB (ELB/    |
|  Azure LB / GCP NLB)|
+---------+----------+
          |
          v
+--------------------+
|  Service (type=    |
|  LoadBalancer)     |
+---------+----------+
          | round-robin to ready pods
          v
   +------+------+------+
   | Pod  | Pod  | Pod  |   <- managed by Deployment (3 replicas)
   +------+------+------+
```

### Files

#### `01-namespace.yaml`
**What:** Creates an isolated namespace.
**Why:** Avoid colliding with `default`; lets you scope RBAC, quotas, network policies.
**When:** Always — first object in any environment.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web
  labels:
    env: dev
    team: platform
```
**Key lines:**
- `metadata.name` — referenced by every other manifest as `namespace: web`.
- `labels` — used later by NetworkPolicies and selectors.

#### `02-configmap.yaml`
**What:** Stores non-sensitive config (the HTML page nginx will serve).
**Why:** Decouples configuration from the image; same image runs in dev/staging/prod.
**Where:** Mounted as a volume into the pod.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-content
  namespace: web
data:
  index.html: |
    <html><body>
      <h1>Hello from Kubernetes</h1>
      <p>Served from a ConfigMap.</p>
    </body></html>
```
**Key lines:**
- `data` — keys become filenames when mounted as a volume.
- The `|` block scalar preserves newlines.

#### `03-deployment.yaml`
**What:** The workload spec.
**Why:** Manages replicas, rolling updates, and self-healing.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: web
  labels: { app: web }
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels: { app: web }
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101            # nginx user in the official image
        fsGroup: 101
      containers:
        - name: nginx
          image: nginx:1.27.2     # pinned, never :latest
          ports:
            - containerPort: 8080
              name: http
          resources:
            requests: { cpu: "50m",  memory: "64Mi" }
            limits:   { cpu: "250m", memory: "128Mi" }
          readinessProbe:
            httpGet: { path: /, port: http }
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet: { path: /, port: http }
            periodSeconds: 10
            failureThreshold: 3
            initialDelaySeconds: 15
          volumeMounts:
            - name: content
              mountPath: /usr/share/nginx/html
            - name: nginx-conf
              mountPath: /etc/nginx/conf.d
      volumes:
        - name: content
          configMap: { name: web-content }
        - name: nginx-conf
          configMap:
            name: web-content
            items:
              - key: nginx.conf
                path: default.conf
                # (You'd add a key 'nginx.conf' to the ConfigMap to listen on 8080)
```
**Key lines:**
- `selector.matchLabels` **must** equal `template.metadata.labels`. This is the source of 50% of "why is nothing happening?" tickets.
- `maxUnavailable: 0` — zero-downtime; ramp surge before tearing down old pods.
- `runAsNonRoot` — security baseline.
- Distinct `readiness` (controls traffic) and `liveness` (controls restart) — they answer different questions.

#### `04-service.yaml`
**What:** Stable internal endpoint + cloud load balancer.
**Why:** Pods come and go; the Service IP/DNS does not.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: web
  annotations:
    # AWS NLB (uncomment on EKS):
    # service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    # Azure internal LB (uncomment on AKS for internal-only):
    # service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    # GCP NEG (recommended on GKE for container-native LB):
    # cloud.google.com/neg: '{"ingress": true}'
spec:
  type: LoadBalancer
  selector: { app: web }
  ports:
    - name: http
      port: 80
      targetPort: http
```
**Key lines:**
- `selector` — *not* a label, it's a *query*. Matches pods, not Deployments.
- `targetPort: http` — references the named container port. Refactor-safe.
- Cloud-specific annotations select LB flavor (NLB vs ALB, internal vs public).

### Commands
```bash
# Apply
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-configmap.yaml -f 03-deployment.yaml -f 04-service.yaml

# Watch pods come up
kubectl -n web get pods -w

# Get the external IP/DNS (cloud-dependent)
kubectl -n web get svc web

# Inspect
kubectl -n web describe deploy web
kubectl -n web logs deploy/web --tail=50

# Exec in
kubectl -n web exec -it deploy/web -- sh

# Roll out a new image
kubectl -n web set image deploy/web nginx=nginx:1.27.3
kubectl -n web rollout status deploy/web

# Rollback
kubectl -n web rollout undo deploy/web

# Scale
kubectl -n web scale deploy/web --replicas=5

# Cleanup
kubectl delete ns web
```

### Expected Output
Three Ready pods, a Service with an external IP/DNS, and an HTML page when you `curl` it. `kubectl get events -n web` should show no warnings.

### Advanced Thinking Layer
- **Why a Deployment instead of a StatefulSet for nginx?** Stateless workload. Pod identity doesn't matter.
- **Why `maxUnavailable: 0`?** Trades rollout speed for zero downtime — the right default for user-facing services.
- **Anti-pattern:** Setting `livenessProbe` with the same threshold as `readinessProbe`. A slow dependency will *kill* your pods instead of just removing them from the LB.

---
---

# LEVEL 2 — INTERMEDIATE

## PART A — CONCEPTS

### Helm — The Package Manager
Helm packages a set of templates + default values into a **chart**. You install a chart with overrides per environment.

```
mychart/
  Chart.yaml         # metadata
  values.yaml        # default values
  templates/         # Go-templated manifests
  charts/            # subchart deps
```

### Persistent Volumes
- **PV** — cluster-level storage object (managed by admin or dynamic provisioner).
- **PVC** — namespace-level claim ("I need 10Gi of fast SSD").
- **StorageClass** — defines provisioner + parameters (gp3 on AWS, premium-ssd on Azure, pd-ssd on GCP).
- Dynamic provisioning: PVC → StorageClass → cloud disk created → PV bound.

### Networking
- **CNI** — pod networking plugin (AWS VPC CNI, Azure CNI, Calico, Cilium).
- **kube-proxy** — implements Service IPs (iptables/IPVS).
- **CoreDNS** — `web.web.svc.cluster.local` style DNS resolution.
- **Service-to-pod path:** ClusterIP → kube-proxy DNAT → pod IP → CNI delivery.

### Ingress
A Service of type=LoadBalancer per app gets expensive. **Ingress** is a single LB + reverse proxy that routes by host/path to many services.
- Needs an **Ingress Controller** (nginx-ingress, ALB controller on EKS, AGIC on AKS, GCE Ingress on GKE).

### RBAC
Role = a list of permissions in a namespace. ClusterRole = same, cluster-wide.
RoleBinding/ClusterRoleBinding = grants that role to a subject (user, group, ServiceAccount).
**Default ServiceAccount has no permissions** — give workloads their own SA when they need API access.

### Multi-environment
- **Namespace per env** in one cluster — fine for dev/staging.
- **Cluster per env** — recommended for prod (real isolation, blast radius).
- Use Helm `values-dev.yaml` / `values-prod.yaml` or Kustomize overlays.

### Best Practices
- One Helm chart per app, environment-specific values files.
- Use `helm diff upgrade` before applying.
- Storage: pick the right `volumeBindingMode` (`WaitForFirstConsumer` for AZ-affinity).
- RBAC: deny-by-default; grant per workload, not per cluster.

### Common Mistakes
- Hard-coding values in templates instead of `.Values.xxx`.
- PVCs that fail because pods land in an AZ where the disk doesn't exist (use `WaitForFirstConsumer`).
- Granting `cluster-admin` to a CI/CD bot. Then forgetting.

### Pro Tips
- `helm template . | kubectl apply --dry-run=server -f -` — server-side validation without installing.
- `kubectl auth can-i get pods --as=system:serviceaccount:web:web-sa -n web` — debug RBAC like a pro.
- Cilium gives you eBPF observability + NetworkPolicies that actually scale. Worth considering on day one for new clusters.

---

## PART B — HANDS-ON: Helm Chart + Persistent Storage + RBAC + Ingress

### Objective
Convert the Level 1 app into a Helm chart, add a Postgres-style stateful component using a PVC, expose it through nginx-ingress, and restrict its API permissions with RBAC.

### Architecture
```
Internet
   |
   v
+----------------+
|  Ingress LB    |  (single cloud LB)
|  (nginx)       |
+--+-----+-------+
   |     |
   |     +-> /api  -> Service api -> Deployment api (3x)
   +-> /     -> Service web -> Deployment web (3x)
                                       \
                                        -> ServiceAccount with limited RBAC
                                        -> PVC -> StorageClass -> cloud disk
```

### Files

#### `mychart/Chart.yaml`
**What:** Chart metadata.
```yaml
apiVersion: v2
name: web
description: Hello-world web app
type: application
version: 0.1.0      # chart version
appVersion: "1.27.2" # the app's version
```

#### `mychart/values.yaml`
**What:** Default values overridable per environment.
**Why:** Single source of truth for "what's tunable."
```yaml
replicaCount: 3
image:
  repository: nginx
  tag: 1.27.2
service:
  type: ClusterIP
  port: 80
ingress:
  enabled: true
  className: nginx
  host: web.example.com
  path: /
resources:
  requests: { cpu: 50m, memory: 64Mi }
  limits:   { cpu: 250m, memory: 128Mi }
persistence:
  enabled: false
  size: 5Gi
  storageClassName: ""   # use cluster default
rbac:
  create: true
serviceAccount:
  create: true
  name: ""
```

#### `mychart/templates/_helpers.tpl`
**What:** Reusable label and name helpers.
**Why:** DRY — define labels once.
```gotemplate
{{- define "web.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "web.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

#### `mychart/templates/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "web.fullname" . }}
  labels: {{- include "web.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels: {{- include "web.labels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "web.fullname" . }}
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports: [{ name: http, containerPort: 80 }]
          resources: {{- toYaml .Values.resources | nindent 12 }}
          readinessProbe: { httpGet: { path: /, port: http }, periodSeconds: 5 }
          livenessProbe:  { httpGet: { path: /, port: http }, periodSeconds: 10, initialDelaySeconds: 15 }
          {{- if .Values.persistence.enabled }}
          volumeMounts:
            - name: data
              mountPath: /var/lib/data
          {{- end }}
      {{- if .Values.persistence.enabled }}
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: {{ include "web.fullname" . }}
      {{- end }}
```
**Key lines:**
- `{{- include "web.labels" . | nindent 4 }}` — render shared labels indented 4 spaces.
- Conditional volume blocks — same template, multiple shapes per env.

#### `mychart/templates/pvc.yaml`
```yaml
{{- if .Values.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "web.fullname" . }}
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: {{ .Values.persistence.size }} } }
  {{- if .Values.persistence.storageClassName }}
  storageClassName: {{ .Values.persistence.storageClassName }}
  {{- end }}
{{- end }}
```
**Why:** PVC binds to a PV created dynamically by the StorageClass. `ReadWriteOnce` works on all three clouds; `ReadWriteMany` requires EFS / Azure Files / Filestore.

#### `mychart/templates/serviceaccount.yaml`
```yaml
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "web.fullname" . }}
{{- end }}
```

#### `mychart/templates/rbac.yaml`
```yaml
{{- if .Values.rbac.create }}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "web.fullname" . }}-reader
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "web.fullname" . }}-reader
subjects:
  - kind: ServiceAccount
    name: {{ include "web.fullname" . }}
roleRef:
  kind: Role
  name: {{ include "web.fullname" . }}-reader
  apiGroup: rbac.authorization.k8s.io
{{- end }}
```
**Why:** Workload can read its own ConfigMaps/Secrets via the K8s API but can't list pods or modify anything.

#### `mychart/templates/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "web.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/instance: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      name: http
```

#### `mychart/templates/ingress.yaml`
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "web.fullname" . }}
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: Prefix
            backend:
              service:
                name: {{ include "web.fullname" . }}
                port: { name: http }
{{- end }}
```

#### `values-prod.yaml`
```yaml
replicaCount: 6
image: { tag: 1.27.2 }
ingress:
  host: web.prod.example.com
persistence:
  enabled: true
  size: 50Gi
  storageClassName: gp3   # AWS; use 'managed-premium' on AKS, 'standard-rwo' on GKE
resources:
  requests: { cpu: 200m, memory: 256Mi }
  limits:   { cpu: 1,    memory: 512Mi }
```

### Commands
```bash
# Lint and dry-run
helm lint mychart/
helm template mychart/ -f values-prod.yaml | less

# Install nginx ingress controller (one-time per cluster)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace

# Install the app (per env)
helm upgrade --install web mychart/ -n web --create-namespace -f values-prod.yaml

# Diff before upgrade (requires helm-diff plugin)
helm diff upgrade web mychart/ -n web -f values-prod.yaml

# Rollback
helm rollback web 1 -n web

# Uninstall
helm uninstall web -n web

# RBAC sanity check
kubectl auth can-i list pods --as=system:serviceaccount:web:web-web -n web
# expect: no
kubectl auth can-i get configmaps --as=system:serviceaccount:web:web-web -n web
# expect: yes
```

### Advanced Thinking Layer
- **Helm vs Kustomize vs raw YAML?** Helm wins when you need parameterization across many envs and a release lifecycle (history, rollback). Kustomize wins when overlays are simple and you hate templating. Raw YAML wins for cluster-bootstrap manifests.
- **Why `WaitForFirstConsumer`?** With `Immediate` binding, a PVC may be created in AZ-a but the pod scheduled to AZ-b → unschedulable. `WaitForFirstConsumer` defers PV creation until the pod is scheduled.
- **Anti-pattern:** Granting `*` verbs in a Role "for now." Forty CVEs later, "for now" is still in prod.

---
---

# LEVEL 3 — ADVANCED

## PART A — CONCEPTS

### Multi-cluster strategies
- **Active-Passive DR:** Primary cluster serves; standby cluster ready in another region/cloud. RTO measured in minutes.
- **Active-Active:** Both clusters serve via global LB (Route53 latency, Azure Front Door, GCP Global LB). Higher cost, lower RTO.
- **Cluster-per-tenant:** SaaS isolation; complexity scales with tenants.
- **Burst-to-cloud:** Primary on-prem, overflow to cloud. Often a trap — moves complexity to networking.

### GitOps (ArgoCD / Flux)
Git is the source of truth. A controller in the cluster pulls and reconciles. Benefits:
- Audit trail = git log.
- Easy rollback = git revert.
- No CI runner needs cluster credentials.

ArgoCD `Application` CRD points at a repo path; Flux uses `Kustomization`/`HelmRelease` CRDs. Either works; pick one and stick.

### CI/CD Integration
Common pipeline:
```
PR  -> build image -> scan (Trivy) -> push to registry
                    -> render manifests (helm template / kustomize build)
                    -> open PR to GitOps repo with new image tag
Merge GitOps PR -> ArgoCD sync -> rollout
```
Image tag should be **immutable digest** in production manifests.

### Autoscaling
- **HPA** — scales pod replicas based on CPU/memory/custom metrics.
- **VPA** — adjusts pod requests/limits over time. Don't combine HPA on CPU + VPA on CPU.
- **Cluster Autoscaler** — adds/removes nodes when pods can't schedule.
- **Karpenter** (AWS, now multi-cloud experimental) — faster, more flexible alternative to CA. Picks instance types per pending pod.

### Observability
The "three pillars":
- **Metrics** — Prometheus + Grafana. Use the kube-prometheus-stack chart.
- **Logs** — Loki, Elastic, or cloud-native (CloudWatch, Azure Monitor, GCP Logging). Ship via Fluent Bit / Vector.
- **Traces** — OpenTelemetry → Tempo / Jaeger / cloud APM.

Plus: **events** (`kubectl get events`) and **audit logs** (control plane).

### Best Practices
- One source of truth (git). One reconciler in the cluster.
- HPA targets > 70% utilization rarely — leave headroom for spikes.
- Alert on **symptoms** (latency, error rate), not **causes** (CPU usage).

### Common Mistakes
- Pushing manifests directly with `kubectl apply` while ArgoCD is reconciling. Drift war ensues.
- Wiring HPA to a metric that *measures the result of scaling* (creating a feedback loop).
- Cluster Autoscaler with no pod disruption budgets — scale-down can take down a quorum.

### Pro Tips
- `kubectl top pod -n <ns>` — real-time usage. Requires metrics-server.
- ArgoCD ApplicationSets generate many Applications from a template — perfect for cluster fleets.
- For cost control, **Karpenter + Spot** + correct PDBs is the highest-leverage combo on AWS.

---

## PART B — HANDS-ON: GitOps + HPA + Observability

### Objective
Run ArgoCD, deploy the Helm chart from a git repo, attach an HPA, and install Prometheus/Grafana for visibility.

### Architecture
```
       Developer
         |
         v
   +-----------+        +---------------------+
   |  GitHub   | <----- | argocd-image-       |
   |  (apps/)  |        | updater (optional)  |
   +-----+-----+        +---------------------+
         |
         | poll/webhook
         v
+--------------------+      +-----------------+
|  ArgoCD            |----->|  Cluster        |
|  (in cluster)      | sync |  workloads      |
+--------------------+      +--------+--------+
                                     |
                           +---------v---------+
                           | Prometheus stack  |
                           | + Grafana + HPA   |
                           +-------------------+
```

### Files

#### `gitops/argocd-app.yaml`
**What:** Declarative ArgoCD `Application`.
**Why:** Even ArgoCD apps should be in git ("app of apps").

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/k8s-config.git
    targetRevision: main
    path: charts/web
    helm:
      valueFiles:
        - ../../envs/prod/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: web
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```
**Key lines:**
- `selfHeal: true` — out-of-band changes get reverted. This is the whole point.
- `prune: true` — deleted manifests in git → deleted in cluster.
- `ServerSideApply=true` — fewer "managed fields" surprises with operators.

#### `apps/hpa.yaml`
**What:** Horizontal Pod Autoscaler.
**Why:** Match capacity to demand automatically.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
  namespace: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-web
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 65 }
    - type: Resource
      resource:
        name: memory
        target: { type: Utilization, averageUtilization: 75 }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # don't flap
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0     # react fast
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
```
**Key lines:**
- `behavior.scaleDown` — smooth, conservative.
- `behavior.scaleUp` — aggressive, no stabilization window. Asymmetry matters.
- HPA needs **resource requests** on the pods to compute utilization.

#### `apps/pdb.yaml`
**What:** PodDisruptionBudget.
**Why:** Keeps a quorum during voluntary disruptions (node drains, autoscaler scale-down).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web
  namespace: web
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: web
```

#### `observability/values-kube-prom.yaml`
**What:** Override values for the kube-prometheus-stack chart.
**Why:** Production-friendly retention, persistent storage.

```yaml
prometheus:
  prometheusSpec:
    retention: 15d
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2,    memory: 6Gi }
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources: { requests: { storage: 100Gi } }

grafana:
  adminPassword: change-me-via-secret
  persistence: { enabled: true, size: 10Gi, storageClassName: gp3 }
  defaultDashboardsEnabled: true
```

### Commands
```bash
# Install ArgoCD
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd port-forward svc/argocd-server 8080:443

# Bootstrap your app
kubectl apply -f gitops/argocd-app.yaml
argocd app sync web

# Install observability
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kube-prom prometheus-community/kube-prometheus-stack \
  -n observability --create-namespace -f observability/values-kube-prom.yaml

# Install metrics-server (needed for HPA on some clusters)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Apply HPA + PDB (or commit to git for ArgoCD)
kubectl apply -f apps/hpa.yaml -f apps/pdb.yaml

# Inspect
kubectl -n web get hpa
kubectl -n web describe hpa web
kubectl top pods -n web

# Generate load (test scaling)
kubectl -n web run -it --rm load --image=busybox --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://web-web; done'
```

### Debugging Checklist
```bash
# A pod is CrashLoopBackOff?
kubectl -n web describe pod <name>          # look at events + last state
kubectl -n web logs <name> --previous       # logs from the last crashed instance
kubectl -n web get events --sort-by=.lastTimestamp

# A Service has no endpoints?
kubectl -n web get endpoints <svc>           # empty = label selector mismatch
kubectl -n web get pods -l app=web --show-labels

# DNS broken?
kubectl -n web run -it --rm dns --image=busybox --restart=Never -- \
  nslookup web.web.svc.cluster.local

# HPA not scaling?
kubectl -n web describe hpa web              # look for "FailedGetResourceMetric"
# usually means metrics-server isn't installed or pods lack requests

# Image pull failing?
kubectl -n web describe pod <name> | grep -A5 Events
# look for ImagePullBackOff; check imagePullSecrets and registry creds
```

### Advanced Thinking Layer
- **GitOps trade-off:** You give up the ability to `kubectl apply` ad-hoc. That feels restrictive — until prod stops drifting.
- **Why combine HPA + Cluster Autoscaler?** HPA changes pod count; CA changes node count. They're complementary, not redundant. Karpenter often replaces CA but the principle is the same.
- **Anti-pattern:** Alerting on every pod restart. Pods restart. Alert on **availability SLO burn**, not on the implementation detail of restarts.

---
---

# LEVEL 4 — EXPERT / ARCHITECT

## PART A — CONCEPTS

### Multi-cloud Kubernetes — Honest Patterns

**1. Federation is (mostly) dead.** Don't reach for KubeFed first. Most "multi-cluster" needs are solved by:
- **Global DNS / global LB** — health-based routing (Route53, Front Door, Cloud DNS).
- **GitOps fleet** — one repo, ArgoCD ApplicationSet, N clusters.
- **Service mesh multi-cluster** — Istio multi-primary or Linkerd multi-cluster for cross-cluster service calls.

**2. Common production topologies**
```
Pattern A: Active-Active across regions, same cloud
  global LB --> EKS us-east-1 (3 AZs)
             \
              -> EKS us-west-2 (3 AZs)
   Shared: RDS global cluster, S3 CRR

Pattern B: Active-Active across clouds (high cost, real DR)
  global LB --> EKS (us-east-1)
             \
              -> AKS (eastus)
   Shared: replicated DB, object storage with sync

Pattern C: Per-tenant cluster (B2B SaaS)
  Tenant routing layer -> cluster-A
                       -> cluster-B
                       -> cluster-N (Karpenter for cost)
```

### High Availability & DR
- **Control plane HA:** managed by EKS/AKS/GKE — verify SLA (EKS 99.95%, GKE 99.95% regional, AKS varies).
- **Worker HA:** spread across **≥3 AZs**; honor PDBs; topology spread constraints.
- **DR primitives:**
  - **Velero** — backup + restore of cluster state (and PVs via snapshots).
  - **Cross-region replication** for object storage, registries, secrets.
  - **Runbook + game days** — DR you haven't tested isn't DR.

### Security Hardening
**Network policies (deny-all baseline)**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: web }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```
Then *allow* explicitly per workload. CNI must support NetworkPolicy (Calico, Cilium, Azure CNI w/ Calico, GKE Dataplane V2).

**Pod Security Standards** — enforce `restricted` baseline via namespace labels:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
```

**Secrets management**
- **Don't** store cleartext secrets in git. Period.
- **External Secrets Operator** pulls from AWS Secrets Manager / Azure Key Vault / GCP Secret Manager.
- **Sealed Secrets** for git-resident encrypted blobs.
- Use **IRSA (EKS)**, **Workload Identity (GKE)**, **AAD Workload Identity (AKS)** to avoid ever putting cloud credentials in pods.

### Cost Optimization
- **Right-size requests** with VPA recommender (in recommendation-only mode).
- **Spot/Preemptible** for stateless workloads — Karpenter or node groups + tolerations.
- **Cluster autoscaler downscale aggressively** — most clusters waste 30–50% of capacity.
- **Bin-packing** — fewer, larger nodes are cheaper than many small ones (above a threshold).
- Tools: **OpenCost**, **Kubecost** for chargeback.

### Service Mesh (Istio / Linkerd)
You need a mesh when:
- mTLS across services is a hard requirement.
- Fine-grained traffic shifting (canary, A/B) per request.
- Cross-cluster service discovery.
- Per-call observability with no app code changes.

You do **not** need a mesh when:
- You have <20 services and ingress + NetworkPolicy is enough.
- Your team is already drowning.

**Linkerd** — simpler, faster, smaller surface area. Great default.
**Istio** — more powerful, more complex. Pick when you need its specific features.

### Performance Tuning
- **CPU pinning**: `cpuManagerPolicy: static` for latency-sensitive workloads.
- **HugePages** for databases.
- **Topology spread constraints** to keep pods balanced across zones/nodes.
- **PriorityClasses** to ensure critical workloads preempt batch jobs.

### Pro Tips
- Run a **chaos test** (kill a node, drop an AZ) at least quarterly. Once a year you'll be glad.
- Treat the cluster as **cattle, not pets**: you should be able to rebuild from git within an hour.
- For multi-cluster GitOps, **ApplicationSet with cluster generators** is the magic incantation.

---

## PART B — HANDS-ON: Production Multi-Cluster Reference

### Objective
A reference production setup demonstrating: deny-all NetworkPolicy + explicit allows, IRSA-style workload identity (shown for EKS, with AKS/GKE equivalents), External Secrets, topology spread, PDB, PriorityClass, and an ArgoCD ApplicationSet that fans out to multiple clusters.

### Architecture
```
+-------------------- GitOps repo (single source) -----------------+
|  charts/web/                                                     |
|  envs/                                                            |
|    eks-prod-use1/values.yaml                                     |
|    aks-prod-eastus/values.yaml                                   |
|    gke-prod-uscentral1/values.yaml                               |
|  appsets/web.yaml      <- ArgoCD ApplicationSet                  |
+-----------+------------------------------------------------------+
            |
   +--------+--------+--------+
   |        |        |        |
   v        v        v        v
+------+ +------+ +------+ +------+
| EKS  | | AKS  | | GKE  | | DR   |   <- ArgoCD-managed clusters
+------+ +------+ +------+ +------+
   ^                ^                  Global routing via Route53 / Front Door
   |                |
   +------ mTLS via Linkerd multi-cluster ------+
```

### Files

#### `appsets/web.yaml` — ArgoCD ApplicationSet
**What:** Generates one `Application` per cluster from a single template.
**Why:** Onboarding a new cluster = adding a row, not copying files.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: web
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: eks-prod-use1
            url: https://EKS_API_ENDPOINT
            valuesFile: envs/eks-prod-use1/values.yaml
          - cluster: aks-prod-eastus
            url: https://AKS_API_ENDPOINT
            valuesFile: envs/aks-prod-eastus/values.yaml
          - cluster: gke-prod-uscentral1
            url: https://GKE_API_ENDPOINT
            valuesFile: envs/gke-prod-uscentral1/values.yaml
  template:
    metadata:
      name: 'web-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/k8s-config.git
        targetRevision: main
        path: charts/web
        helm:
          valueFiles: ['../../{{valuesFile}}']
      destination:
        server: '{{url}}'
        namespace: web
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

#### `policies/network-default-deny.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: web }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# Allow DNS to kube-system (otherwise nothing resolves)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-dns, namespace: web }
spec:
  podSelector: {}
  egress:
    - to:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
          podSelector: { matchLabels: { k8s-app: kube-dns } }
      ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
  policyTypes: [Egress]
---
# Allow ingress from the ingress-nginx namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-ingress-nginx, namespace: web }
spec:
  podSelector: { matchLabels: { app.kubernetes.io/name: web } }
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: ingress-nginx } }
      ports: [{ port: 80 }]
  policyTypes: [Ingress]
```
**Why:** Deny-all forces explicit declaration of every allowed flow. Compromise of one pod can't reach the database.

#### `security/external-secret.yaml` — External Secrets Operator
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata: { name: aws-sm, namespace: web }
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef: { name: web-eso }
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-creds, namespace: web }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-sm, kind: SecretStore }
  target: { name: db-creds, creationPolicy: Owner }
  data:
    - secretKey: DATABASE_URL
      remoteRef: { key: prod/web/db, property: url }
```
**Why:** Secret values live in AWS Secrets Manager (or Azure KV / GCP SM). The cluster pulls and refreshes them. No secrets in git, ever.

#### `security/sa-irsa.yaml` — IRSA on EKS (Workload Identity equivalent on GKE/AKS)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-eso
  namespace: web
  annotations:
    # EKS:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/web-eso
    # GKE Workload Identity equivalent:
    # iam.gke.io/gcp-service-account: web-eso@PROJECT.iam.gserviceaccount.com
    # AKS Workload Identity equivalent:
    # azure.workload.identity/client-id: <AAD-app-client-id>
```
**Why:** No long-lived cloud credentials in pods. The SA's projected JWT is exchanged for cloud creds at API-call time.

#### `workload/priorityclass.yaml`
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: business-critical }
value: 1000000
globalDefault: false
description: "Business-critical workloads — preempt lower-priority pods"
```
Reference it in your Deployment as `priorityClassName: business-critical`.

#### `workload/topology-spread.yaml` (snippet to add to a Deployment.spec.template.spec)
```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels: { app.kubernetes.io/name: web }
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels: { app.kubernetes.io/name: web }
```
**Why:** Don't let 6 pods land on 1 node or 1 AZ. AZ outage = ~33% loss instead of 100%.

#### `mesh/linkerd-inject.yaml` (namespace-level mTLS)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web
  annotations:
    linkerd.io/inject: enabled
  labels:
    pod-security.kubernetes.io/enforce: restricted
```
**Why:** Linkerd injects a sidecar that wraps all in/out traffic in mTLS. Zero application changes.

#### `dr/velero-schedule.yaml`
```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata: { name: nightly, namespace: velero }
spec:
  schedule: "0 2 * * *"     # 02:00 UTC daily
  template:
    includedNamespaces: ['*']
    excludedNamespaces: ['kube-system','velero']
    snapshotVolumes: true
    ttl: 168h0m0s            # keep 7 days
```

### Commands — Cluster Operator's Cheat Sheet

```bash
# Cluster identity / context juggling
kubectl config get-contexts
kubectl config use-context eks-prod-use1

# Node operations
kubectl cordon <node>                         # mark unschedulable
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>

# Top resource consumers
kubectl top nodes
kubectl top pods -A --sort-by=memory | head

# Find pods not ready
kubectl get pods -A --field-selector=status.phase!=Running

# Find evicted pods
kubectl get pods -A | grep Evicted

# Find pending pods (often: no capacity)
kubectl get pods -A --field-selector=status.phase=Pending
kubectl describe pod <pending> | tail -20     # look at FailedScheduling events

# Force delete a stuck pod (last resort)
kubectl delete pod <name> --grace-period=0 --force

# Check API server health
kubectl get --raw='/readyz?verbose'

# Rotate kubelet certs (managed services usually do this)
# Audit RBAC
kubectl auth can-i --list --as=system:serviceaccount:web:web

# Validate manifests against the API server (without applying)
kubectl apply -f manifest.yaml --dry-run=server

# Diff what apply would change
kubectl diff -f manifest.yaml
```

### Debugging Real Failures

| Symptom | First check | Likely cause |
|---|---|---|
| Pod stuck `Pending` | `describe pod` events | No node has capacity / nodeSelector mismatch / unschedulable PVC |
| Pod `CrashLoopBackOff` | `logs --previous` | App error / wrong command / missing config |
| Pod `ImagePullBackOff` | `describe pod` events | Wrong image tag / no pull secret / private registry auth |
| Service has no endpoints | `get endpoints` | Selector mismatch / no Ready pods |
| 504 from Ingress | Ingress controller logs | Backend not ready / wrong path / TLS misconfig |
| HPA not scaling | `describe hpa` | metrics-server missing / no resource requests on pods |
| Cross-namespace traffic blocked | `get networkpolicy -A` | Deny-all policy with no matching allow rule |
| Slow DNS | CoreDNS logs + `nslookup` from a busybox | NDOTS / autopath / overloaded CoreDNS pods |
| OOMKilled but RAM looks fine | `dmesg` on the node | cgroup limit too low / memory leak / map[] growth |

### Advanced Thinking Layer

- **Why deny-all NetworkPolicies even with a service mesh?** Defense in depth. Mesh handles app traffic; NetworkPolicies stop a compromised sidecar from reaching the cluster API or another namespace.
- **Why ApplicationSet over per-cluster Applications?** A single edit propagates to N clusters. Onboarding a 4th region is a 5-line PR.
- **Trade-off of multi-cloud K8s:** You get DR diversity and avoid lock-in *at the platform layer*. You **don't** get app portability for free — IRSA, ALB Ingress, EBS volumes, Cloud SQL, etc. are all cloud-specific. Plan abstractions deliberately.
- **Anti-pattern:** "Lift and shift the data plane to be portable, then port the control plane to be portable, then port the apps." You'll spend two years building plumbing and ship nothing. Pick the cloud-specific abstraction that's good enough today and move.

---
---

# COMMANDS — kubectl QUICK REFERENCE

```bash
# Discovery
kubectl api-resources                  # what objects exist
kubectl explain pod.spec.containers    # interactive schema
kubectl get all -n <ns>                # all the common objects

# Apply / delete
kubectl apply -f file.yaml             # idempotent create/update
kubectl delete -f file.yaml            # remove
kubectl apply -k overlays/prod         # kustomize apply

# Pods / logs / exec
kubectl logs <pod> -c <container> -f --tail=100
kubectl logs -l app=web --max-log-requests=10
kubectl exec -it <pod> -- bash
kubectl debug <pod> --image=busybox --target=<container>   # ephemeral debug container

# Rollouts
kubectl rollout status deploy/web -n web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web --to-revision=3
kubectl rollout restart deploy/web      # bounce all pods

# Scaling
kubectl scale deploy/web --replicas=10
kubectl autoscale deploy/web --min=3 --max=20 --cpu-percent=70

# Port-forward (debug only)
kubectl port-forward svc/web 8080:80

# Cp files
kubectl cp <ns>/<pod>:/path/to/file ./local-file

# Wait
kubectl wait --for=condition=available deploy/web --timeout=120s

# Patch (surgical edit)
kubectl patch deploy web -p '{"spec":{"replicas":5}}'

# Label / annotate
kubectl label pod <pod> tier=frontend
kubectl annotate ns web owner=platform-team
```

---

# FINAL CHECKLIST — "Production-Ready" Means

- [ ] All workloads have requests, limits, readiness probe, liveness probe.
- [ ] All namespaces have a default-deny NetworkPolicy + explicit allows.
- [ ] All workloads run as non-root with restricted Pod Security.
- [ ] Secrets come from External Secrets / Sealed Secrets — none in git.
- [ ] Workload identity (IRSA / GKE WI / AKS WI) — no static cloud creds.
- [ ] PDBs exist for every Deployment ≥ 2 replicas.
- [ ] Topology spread across ≥ 2 AZs for everything user-facing.
- [ ] HPA + Cluster Autoscaler (or Karpenter) configured and tested under load.
- [ ] Prometheus + Grafana + alerting on **SLO burn**, not on CPU.
- [ ] Velero backups running, **and restored at least once**.
- [ ] GitOps reconciler is the only writer to the cluster.
- [ ] Disaster recovery runbook exists, tested in the last 90 days.

If you can tick all twelve, you don't just *use* Kubernetes — you operate it like a Senior/Staff platform engineer.
