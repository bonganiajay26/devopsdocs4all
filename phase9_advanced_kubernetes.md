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
