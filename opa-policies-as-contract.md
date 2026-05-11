# OPA Policies as the Contract Between Platform and Product Teams: Lessons from 18 Months of DevSecOps

> *A senior DevSecOps practitioner's unfiltered account of what Open Policy Agent actually looks like after a year and a half in production — the wins, the failures, and what we'd do differently.*

---

## Preface: Who This Is For

This is not an OPA tutorial. The official documentation covers Rego syntax better than I can. This is a field report for engineering leaders, platform engineers, and security practitioners who are considering OPA adoption or are six months in and wondering why things aren't going as smoothly as the conference talks suggested they would.

If you want the theory, read the docs. If you want to understand the organizational dynamics, the failure modes, and the tradeoffs that don't make it into vendor slide decks, keep reading.

---

## 1. The Problem: Misalignment Between Platform and Product Teams

Every organization above a certain size develops a version of the same problem.

The platform team — responsible for cluster stability, security posture, compliance, and cost — publishes standards. A wiki page titled "Kubernetes deployment guidelines." A Confluence doc called "Security requirements for production services." A PDF from the security team labeled "Container hardening checklist v2.3 (FINAL) (FINAL2)."

Product teams read these documents. Or they don't. Or they read an outdated version. Or they read the current version and interpret it differently than the platform team intended. Or they know the rules but face a deadline and make a pragmatic exception that they intend to fix later.

Six months later: a production incident exposes a container running as root. A compliance audit finds 40% of workloads missing mandatory labels. A security review reveals that three product teams are pulling images from an unapproved registry because the approved registry was slow and nobody updated the guidance when the mirror was set up.

The platform team's response is typically more process: mandatory review gates, sign-off requirements, architecture review boards. Each of these slows product teams down. Product teams push back. Tensions rise. The platform team is accused of being a bottleneck. The product teams are accused of cutting corners on security.

This dynamic is not a people problem. It is an information architecture problem. The contract between platform and product teams exists — it just lives in documents that cannot be enforced, in tribal knowledge that leaves with the engineers who hold it, and in manual processes that scale with headcount rather than with deployment frequency.

When we started looking at Open Policy Agent, we were not primarily motivated by security. We were motivated by wanting to turn an adversarial relationship into a collaborative one. The insight that changed our thinking: **if the rules are in code, they can be tested, versioned, debated like code, and enforced without requiring a human in the loop for every deployment**.

---

## 2. Why We Treated OPA Policies as a Contract

The word "contract" is deliberate. In software engineering, a contract defines what a caller must provide and what a system guarantees in return. It is explicit, testable, and version-controlled. When a contract changes, both parties know. When a caller violates a contract, they get a clear error, not a vague rejection email from a review board.

Platform teams impose requirements on product teams constantly. They just rarely formalize them as contracts. The result is asymmetric accountability: the platform team holds informal power to block deployments based on criteria that may shift based on who reviews it and what mood they're in. Product teams have no clear specification of what "correct" looks like, so they cannot confidently know whether their service will pass review until it fails.

OPA inverts this. When a policy is written in Rego and enforced at admission time, the contract is:

- **Explicit**: the policy says exactly what is required
- **Testable**: product teams can run `opa test` against their manifests before submitting a PR
- **Deterministic**: the same manifest produces the same result every time, regardless of which reviewer is on duty
- **Auditable**: every policy decision is logged with the input, the decision, and the matching rule

This is what we mean by policy as contract. The platform team's job shifts from "reviewing and approving deployments" to "writing, maintaining, and communicating policies." The product team's job shifts from "convincing the platform team" to "satisfying the contract."

That shift sounds simple. In practice, it took 18 months to actually achieve, and we are still not fully there.

---

## 3. Implementation Journey

### Months 1–3: Deployment and Denial

We started where most teams start: Kubernetes admission control. We deployed OPA Gatekeeper into a non-production cluster, wrote five policies that reflected our most critical requirements, and declared success.

The policies we started with:

```rego
# Policy 1: Containers must not run as root
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.securityContext.runAsNonRoot
  msg := sprintf(
    "Container '%v' in Pod '%v' must set securityContext.runAsNonRoot: true",
    [container.name, input.request.object.metadata.name]
  )
}
```

```rego
# Policy 2: Images must come from approved registries
package kubernetes.admission

approved_registries := {
  "registry.company.internal",
  "gcr.io/our-project",
  "ghcr.io/our-org"
}

deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  image := container.image
  not any_approved(image)
  msg := sprintf(
    "Container image '%v' is not from an approved registry. Approved: %v",
    [image, approved_registries]
  )
}

any_approved(image) {
  approved := approved_registries[_]
  startswith(image, approved)
}
```

```rego
# Policy 3: All deployments must have mandatory labels
package kubernetes.admission

required_labels := {"team", "environment", "cost-center", "service-tier"}

deny[msg] {
  input.request.kind.kind == "Deployment"
  provided := {label | input.request.object.metadata.labels[label]}
  missing := required_labels - provided
  count(missing) > 0
  msg := sprintf(
    "Deployment '%v' is missing required labels: %v",
    [input.request.object.metadata.name, missing]
  )
}
```

These policies were technically correct and organizationally catastrophic. We had not told product teams they were coming. We had not provided a migration path for existing workloads. We had not explained what to do when a legitimate use case violated the policy. We had not given product teams a way to test their manifests locally.

The first week in the non-production cluster generated 847 policy violations across 12 product teams. We did not have the bandwidth to explain each one. Product teams started treating OPA as an obstacle rather than a guardrail.

**Lesson learned immediately:** Enforcement without migration is just blocking. Start in audit mode, not enforce mode. Give teams 30 days to see their violations before you start denying.

### Months 3–6: Building the Contract Infrastructure

We backed up, slowed down, and did the work we should have done first.

We ran Gatekeeper in audit mode for 8 weeks across all clusters. We built a dashboard that showed each team's policy violation count, trending over time. We ran working sessions with each product team to explain the policies and, critically, to listen to the cases where our policies were wrong.

Two of our initial five policies were wrong. Not wrong in intent, but wrong in specificity.

The `runAsNonRoot` policy broke three legitimate workloads: a database sidecar that required root to bind to port 80, a log shipping agent that needed to read protected log files, and an in-house monitoring tool that had never been updated to support non-root operation. We had a policy with no exception mechanism. The only options were "violate the policy" or "block the workload."

We rewrote the policy to support annotated exemptions with mandatory justification:

```rego
# Policy 1 (v2): Containers must not run as root, with audited exemptions
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.securityContext.runAsNonRoot
  not has_exemption(input.request.object, container.name)
  msg := sprintf(
    "Container '%v' must set runAsNonRoot: true. To request an exemption, add annotation 'policy.company.io/root-exemption' with a Jira ticket reference.",
    [container.name]
  )
}

has_exemption(pod, container_name) {
  annotation := pod.metadata.annotations["policy.company.io/root-exemption"]
  # Annotation must reference a valid Jira ticket (PLAT-XXXXX format)
  regex.match(`^PLAT-[0-9]+$`, annotation)
}
```

The exemption mechanism was a significant design decision with real tradeoffs:

**Pros:** Legitimate exceptions have a path forward. Teams are not blocked. The annotation creates an audit trail — we could query all exemptions in the cluster and review them quarterly.

**Cons:** Teams will abuse exemptions if the review process is not enforced. An annotation on a manifest is trivial to add. Without a process to review exemptions, you have effectively made your policy optional.

We handled this by building a quarterly exemption review into our security governance calendar. Any exemption annotation older than 90 days without a renewed approval gets automatically removed by a controller we wrote. The policy then re-evaluates — either the workload has been fixed or it blocks.

This is the kind of organizational infrastructure that OPA documentation doesn't mention.

### Months 6–12: Expanding the Perimeter

By month six, Kubernetes admission was mostly working. Product teams had a 95% compliance rate on mandatory labels, 89% on image registry, and 71% on security context policies. The 71% figure on security context was not a failure — it represented legitimate exemptions working their way through the remediation queue.

We expanded OPA into three new domains:

**Terraform plan validation in CI/CD:**

```rego
# Block Terraform plans that open security groups to 0.0.0.0/0
package terraform.security

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group_rule"
  resource.change.actions[_] == "create"
  cidr := resource.change.after.cidr_blocks[_]
  cidr == "0.0.0.0/0"
  resource.change.after.from_port != 443
  resource.change.after.from_port != 80
  msg := sprintf(
    "Security group rule '%v' opens port %v to 0.0.0.0/0. Only ports 80 and 443 are permitted to be world-accessible. Reference: SEC-POLICY-023",
    [resource.address, resource.change.after.from_port]
  )
}
```

```rego
# Enforce mandatory resource tags in Terraform
package terraform.tagging

required_tags := {"environment", "team", "cost-center", "managed-by"}

deny[msg] {
  resource := input.resource_changes[_]
  resource.change.actions[_] == "create"
  taggable_resource(resource.type)
  provided := {tag | resource.change.after.tags[tag]}
  missing := required_tags - provided
  count(missing) > 0
  msg := sprintf(
    "Resource '%v' is missing required tags: %v",
    [resource.address, missing]
  )
}

taggable_resource(type) {
  taggable_types := {
    "aws_instance", "aws_s3_bucket", "aws_rds_instance",
    "aws_eks_cluster", "aws_lambda_function"
  }
  taggable_types[type]
}
```

**CI/CD pipeline checks (via OPA as a sidecar in GitHub Actions):**

```rego
# Enforce that container images are pinned to digest, not tag
package cicd.image_pinning

deny[msg] {
  container := input.containers[_]
  image := container.image
  not contains(image, "@sha256:")
  not is_exempt(container)
  msg := sprintf(
    "Container image '%v' must be pinned to a digest (sha256:...) not a mutable tag. Mutable tags break reproducibility and create supply chain risk.",
    [image]
  )
}

is_exempt(container) {
  # Development environments may use tags for iteration speed
  input.environment == "development"
}
```

**Secrets management enforcement:**

```rego
# Prevent hardcoded secrets in environment variables
package kubernetes.secrets

suspicious_env_patterns := [
  "(?i)password",
  "(?i)secret",
  "(?i)api.?key",
  "(?i)token",
  "(?i)credential"
]

deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  env := container.env[_]
  # Variable name matches a suspicious pattern
  pattern := suspicious_env_patterns[_]
  regex.match(pattern, env.name)
  # AND the value is a direct string (not a secretKeyRef or configMapKeyRef)
  env.value
  not env.valueFrom
  msg := sprintf(
    "Container '%v' has environment variable '%v' with a direct string value. Secrets must be sourced from Kubernetes Secrets or an external secrets manager, not hardcoded in the manifest.",
    [container.name, env.name]
  )
}
```

The Terraform integration was, unexpectedly, easier to get adoption on than Kubernetes admission. Infrastructure engineers understood the risk model immediately. The feedback loop was clean: the CI pipeline told you exactly which Terraform resource violated which policy and why, with a reference to the specific policy document.

**The unexpected lesson from Terraform integration:** when you give engineers specific, actionable error messages with references to the policy — not just "this is denied" — they fix the issue rather than escalating to the platform team. The reference number (e.g., `SEC-POLICY-023`) was critical. Engineers could look up the policy, understand the intent, and often self-serve the fix.

### Months 12–18: The Policy Sprawl Crisis

By month 12, we had 87 active policies across four policy domains. We had a problem that none of our initial architectural thinking had anticipated: **policy sprawl**.

Policies had been written by four different team members over 12 months. Naming conventions were inconsistent. Some policies used package `kubernetes.admission`, others used `k8s.admission`, others `k8s.security`. Some policies were tested exhaustively; others had no tests at all. Some policies referenced the same underlying logic without sharing it. The `has_exemption` function was reimplemented in at least nine different policies with nine slightly different implementations.

We also had 23 policies in "audit mode" that had been in audit mode for over 6 months. Nobody had driven remediation. They were effectively dead policies that were generating noise without generating compliance.

The sprawl problem had a direct cost: when we needed to update our mandatory labels to add a new `data-classification` tag, we had to update 11 policies that referenced the required labels set independently. We missed updating 3 of them. For 6 weeks, our label enforcement was inconsistent depending on which policy evaluated first.

---

## 4. Key Lessons Learned

### Lesson 1: Rego is Not Self-Documenting

We had engineers who could write complex Rego logic. We also had engineers who, six months later, could not explain what the complex logic they wrote was doing. Rego's data-flow evaluation model is unfamiliar to engineers who think procedurally. Functions that compose elegantly in Rego can be completely opaque to someone reading them for the first time.

The discipline we eventually imposed: every rule must have a comment block that explains the intent in plain English, a worked example of a compliant manifest, and a reference to the policy document or compliance control it satisfies.

```rego
# POLICY: Container Security Context — Non-Root Execution
# INTENT: Prevents containers from running as root (UID 0), which limits the
#         blast radius of container escape vulnerabilities.
# COMPLIANCE: CIS Kubernetes Benchmark 5.2.6, SOC 2 CC6.1
# POLICY DOC: https://wiki.company.io/security/policies/PLAT-SEC-004
# LAST REVIEWED: 2024-Q1
#
# COMPLIANT EXAMPLE:
#   securityContext:
#     runAsNonRoot: true
#     runAsUser: 1000
#
# EXEMPTION: Add annotation 'policy.company.io/root-exemption: PLAT-XXXXX'
#            where PLAT-XXXXX is an approved exemption ticket.

deny[msg] {
  # ... rule body
}
```

This comment discipline feels excessive until the engineer who wrote the policy leaves the company and you need to modify it under incident pressure.

### Lesson 2: The Policy Development Lifecycle Must Mirror Code Development

In our first year, policies went directly from "someone writes it" to "enforce mode." We had no:

- Policy design review (is this the right policy for this problem?)
- Policy testing requirements (does this policy do what you think it does?)
- Staged rollout (audit → warn → enforce over 30–60 days)
- Retroactive impact analysis (how many existing workloads does this break?)
- Deprecation path (how do we remove a policy that's no longer needed?)

We treated policies as configuration. They are not configuration. They are code that makes enforcement decisions. They deserve the same lifecycle as any other code.

By month 15, we had a policy lifecycle process that included all of these stages. The time from "policy proposed" to "policy in enforce mode" increased from days to 6–8 weeks. Product teams complained about the slower pace. We pushed back: the alternative was brittle policies that blocked legitimate work with no notice.

### Lesson 3: Warn Mode Is More Valuable Than Audit Mode

OPA/Gatekeeper has two non-enforcing modes: dry-run (audit existing resources, don't block new ones) and a warning mode that we implemented ourselves. The warning mode difference matters enormously for developer experience.

In audit mode, violations appear in a CRD status field that most engineers never look at.

In warn mode (which we implemented via a mutating webhook that added an annotation to violating resources), the engineer sees a warning in their terminal at deploy time:

```
Warning: Policy violation detected (non-blocking)
  Policy: require-resource-limits (PLAT-SEC-012)
  Container: api-server
  Message: Container has no CPU or memory limits set.
  This policy will move to enforce mode on 2024-03-15.
  Documentation: https://wiki.company.io/policies/PLAT-SEC-012
```

The warning at deploy time, tied to a specific future enforce date, created urgency without creating emergency. Engineer behavior changed dramatically when violations were visible in their normal workflow instead of buried in a CRD.

### Lesson 4: Policy Ownership Requires a Governance Model

After 18 months, we had 87 policies owned by... nobody, really. The platform team had written most of them. The security team had contributed some. A product team lead had written one to solve a problem in their domain and we had adopted it globally.

When a compliance audit required us to demonstrate that our Kubernetes policies mapped to specific SOC 2 controls, we could not produce that mapping in under two weeks because nobody had maintained it.

We built a policy registry: a simple YAML file in our policies repository that captured, for every policy:

```yaml
policies:
  - id: PLAT-SEC-004
    name: require-non-root-container
    domain: kubernetes.admission
    status: enforce
    owner: platform-security-team
    compliance_controls:
      - "CIS-K8S-5.2.6"
      - "SOC2-CC6.1"
    introduced: "2023-03-15"
    last_reviewed: "2024-01-10"
    review_due: "2024-07-10"
    exemption_count: 7
    policy_doc: "https://wiki.company.io/policies/PLAT-SEC-004"
```

This registry became the source of truth for our compliance reporting. Generating a SOC 2 evidence package dropped from a two-week manual effort to an afternoon scripting exercise.

### Lesson 5: Developer Experience Is Not Optional

The single biggest predictor of policy compliance was not policy severity — it was how well the error message explained what to do next.

We spent two months auditing our policy error messages and rewriting them. Before and after:

**Before:**
```
Error: Policy violation: require-non-root-container
```

**After:**
```
Policy violation: Container 'api-server' must not run as root.

Why this matters: Running as root inside a container significantly
increases the blast radius of a container escape. An attacker who
breaks out of a root container has root on the host kernel.

How to fix this:
  Add to your container spec:
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000  # Use a non-zero UID

If your container genuinely requires root (e.g., it binds to a
privileged port and cannot be changed), request an exemption:
  1. File a ticket at https://jira.company.io/...
  2. Add annotation: policy.company.io/root-exemption: PLAT-XXXXX

Policy reference: PLAT-SEC-004
Documentation: https://wiki.company.io/policies/PLAT-SEC-004
```

This is significantly more text. It is also significantly less likely to result in a Slack message asking "what does this error mean." The investment in error message quality pays back many times over in reduced support burden.

---

## 5. Anti-Patterns and Failures

### Anti-Pattern 1: Policies Without Tests

We had policies in enforce mode with zero tests. This sounds negligent. At the time, each policy "obviously worked" because we had tested it manually. Six months later, a refactor introduced a subtle Rego bug. The policy appeared to be evaluating correctly but was, in fact, always returning `deny` for a specific edge case.

The incident: a product team's deployment started failing with a cryptic policy violation. The manifest was clearly compliant. It took four hours to trace the issue to a policy that had been silently broken by an unrelated change.

**Minimum viable policy testing:**

```rego
# tests/kubernetes/admission/require_non_root_test.rego
package kubernetes.admission_test

import data.kubernetes.admission

# Test 1: Non-compliant pod with root container should be denied
test_root_container_denied {
  result := admission.deny with input as {
    "request": {
      "kind": {"kind": "Pod"},
      "object": {
        "metadata": {
          "name": "test-pod",
          "annotations": {}
        },
        "spec": {
          "containers": [{
            "name": "app",
            "image": "registry.company.internal/app:latest",
            "securityContext": {
              "runAsNonRoot": false
            }
          }]
        }
      }
    }
  }
  count(result) > 0
}

# Test 2: Compliant pod should not be denied
test_non_root_container_allowed {
  result := admission.deny with input as {
    "request": {
      "kind": {"kind": "Pod"},
      "object": {
        "metadata": {
          "name": "test-pod",
          "annotations": {}
        },
        "spec": {
          "containers": [{
            "name": "app",
            "image": "registry.company.internal/app:latest",
            "securityContext": {
              "runAsNonRoot": true,
              "runAsUser": 1000
            }
          }]
        }
      }
    }
  }
  count(result) == 0
}

# Test 3: Pod with valid exemption should not be denied
test_exempted_root_container_allowed {
  result := admission.deny with input as {
    "request": {
      "kind": {"kind": "Pod"},
      "object": {
        "metadata": {
          "name": "test-pod",
          "annotations": {
            "policy.company.io/root-exemption": "PLAT-1234"
          }
        },
        "spec": {
          "containers": [{
            "name": "legacy-app",
            "securityContext": {
              "runAsNonRoot": false
            }
          }]
        }
      }
    }
  }
  count(result) == 0
}
```

Rule: no policy merges without tests covering at least the happy path, the violation path, and any exemption logic.

### Anti-Pattern 2: Treating OPA as a Replacement for Architecture Conversations

We had a team that was pulling images from a public Docker Hub namespace because their internal image build pipeline was too slow. We wrote a policy to block Docker Hub. The team found a workaround. We tightened the policy. They found another workaround.

We had treated the symptom (unapproved registry) rather than the cause (build pipeline performance). The policy became an arms race.

The resolution was not a better policy — it was a platform investment to reduce internal build times from 18 minutes to 4 minutes. The Docker Hub policy then became non-controversial because the approved registry was no longer a meaningful burden.

**OPA enforces standards. It cannot compensate for platform infrastructure that makes compliance harder than non-compliance.**

### Anti-Pattern 3: Policy Sprawl Without a Deprecation Culture

We never deprecated a policy. By month 18, we had policies that reflected organizational requirements from 18 months ago that had since changed. We had policies that overlapped with each other, creating situations where a workload was evaluated against essentially the same rule by three different policies. We had policies in audit mode that had been in audit mode for so long that nobody knew whether they were intentionally parked or accidentally forgotten.

Introduce a quarterly policy review cycle. For every policy: Is it still required? Is the violation count trending down (good) or flat (perhaps not actionable)? Is it duplicated by another policy? Could it be simplified or consolidated?

### Anti-Pattern 4: Org-Wide Policies for Team-Specific Problems

One product team had a specific compliance requirement due to their product's regulatory context. They asked us to write a global policy that enforced their requirement for all teams. We obliged.

This was wrong. The policy introduced friction for teams with no exposure to that regulation. Rego supports namespaces and conditional evaluation — we could have scoped the policy to resources with a specific label or in a specific namespace. We didn't, because it was easier to write a global policy.

```rego
# Scoped policy: Only enforce PCI requirements on PCI-scoped workloads
package kubernetes.admission.pci

deny[msg] {
  input.request.kind.kind == "Pod"
  # Only evaluate pods in the pci-scope namespace or with the pci label
  is_pci_scoped(input.request.object)
  # ... PCI-specific checks
  msg := "..."
}

is_pci_scoped(pod) {
  pod.metadata.namespace == "pci-production"
}

is_pci_scoped(pod) {
  pod.metadata.labels["compliance-scope"] == "pci"
}
```

**Write policies at the narrowest scope that satisfies the requirement.**

---

## 6. Measurable Outcomes

After 18 months, we could measure the following with reasonable confidence:

**Compliance posture:**
- Mandatory label compliance across production workloads: 23% at start → 96% at 18 months
- Root container violations in production: 14 workloads at start → 2 (both with active exemptions)
- Image registry compliance: 67% at start → 99% at 18 months
- Terraform resource tagging compliance: not measured at start → 91% at 12 months (retroactive)

**Operational efficiency:**
- Manual deployment review requests to the platform team: ~30/week at start → ~4/week at 18 months. The remaining 4 are genuinely complex cases, not routine compliance questions.
- Mean time to identify a policy violation in CI: from "discovered in manual review" (~72 hours) to "flagged at PR creation" (~2 minutes)
- Security audit preparation time: 2 weeks → 2 days (due to policy registry and automated evidence generation)

**Developer experience (survey, n=47 engineers):**
- 78% reported that OPA error messages were clear enough to self-serve a fix (up from 31% at month 6)
- 64% reported that policies had not meaningfully slowed their deployment workflow
- 36% reported that at least one policy had been a source of significant friction in the past 6 months (this was the number we kept working on)

**The outcome we cannot cleanly measure** but believe to be real: the adversarial relationship between the platform team and product teams has significantly improved. The conversation has shifted from "why is the platform team blocking us" to "which policy is being violated and what's the fastest path to compliance." That shift has value that doesn't appear in any metric.

---

## 7. Recommendations for Other Teams

These are the things I would do differently from day one, and the things I would do again.

### Start with a Policy Charter, Not Policies

Before writing any Rego, define the governance model in a document that both platform and product teams sign off on:

- What decisions does OPA make autonomously (enforce mode)?
- What decisions does OPA flag for human review (warn mode)?
- Who can propose a new policy?
- Who approves a new policy?
- What is the exemption process?
- How are policies reviewed and deprecated?
- How are policy changes communicated to product teams?

Without this charter, you will answer all of these questions reactively during incidents. The answers will be inconsistent, and you will lose trust from both sides.

### Invest in Developer Tooling Before Enforcement

Product teams should be able to test their manifests against all applicable policies locally, before they push to CI. We built a small CLI wrapper called `platform-check` that ran the full policy suite against a local Kubernetes manifest:

```bash
$ platform-check deployment.yaml

✓  require-non-root-container     PASSED
✓  require-approved-registry      PASSED
✓  require-resource-limits        PASSED
✗  require-mandatory-labels       FAILED
   Missing labels: cost-center, data-classification
   Fix: Add the following to your deployment metadata.labels:
     cost-center: "your-cost-center-code"
     data-classification: "internal|confidential|restricted"
   Docs: https://wiki.company.io/labels

⚠  require-image-digest-pinning   WARN (enforce date: 2024-03-15)
   Image 'registry.company.internal/api:latest' uses a mutable tag.
   Start pinning to digest to avoid a blocking error after enforce date.
```

This tool eliminated a large class of CI failures. Engineers fixed issues before they reached the pipeline.

### Use OPA's Bundle Feature for Policy Distribution

Early on, we managed policies by deploying Gatekeeper `ConstraintTemplates` manually. As the policy count grew, this became unmanageable. We moved to OPA bundles — packaged policy sets hosted in a GCS bucket, pulled by OPA agents on a schedule.

This gave us:

- Atomic policy updates (all policies update together, or none do)
- Rollback (previous bundle version available by changing a path)
- Consistent policies across all clusters (no more "this cluster has the v2 policy, that one still has v1")

### Build a Policy Testing Culture, Not Just Tests

Tests are necessary but not sufficient. We introduced policy review sessions (monthly, 30 minutes) where the platform team and one rotating product team representative would review any new or changed policies together. The product team representative was not there to veto policies — they were there to:

1. Catch policies that would create unintended friction
2. Ask "what are we trying to prevent, and is this the right way to prevent it?"
3. Serve as a communication channel back to their team

This single practice did more for platform/product alignment than any technical investment.

### Never Launch in Enforce Mode Without Warning Mode First

The rollout sequence that worked:

1. **Audit mode** (weeks 1–4): Collect baseline data. No blocking. Publish a weekly report showing each team's violation count.
2. **Warn mode** (weeks 5–8): Violations appear at deploy time as non-blocking warnings with enforce date clearly stated.
3. **Enforce mode** (week 9+): Only after the violation count in warn mode has trended to near zero, or known exemptions are in place.

This is slower. It is also the only sequence that does not generate a wave of emergency escalations when you hit enforce.

---

## 8. Conclusion: Policy as Enablement, Not Control

The framing matters enormously.

If the platform team presents OPA as "the security team's enforcement mechanism," product teams will experience it as a constraint imposed on them by an external authority. Compliance becomes adversarial.

If the platform team presents OPA as "the contract we maintain on your behalf so that you don't have to carry the cognitive load of remembering every security requirement," product teams experience it as a service. Compliance becomes collaborative.

The policies themselves are identical in both framings. The organizational outcome is dramatically different.

After 18 months, the thing I believe most strongly is this: **the hard problem of policy-as-code is not the code**. Rego is learnable. The OPA ecosystem is mature. The integration points are well-documented.

The hard problem is the organizational design: who owns the policies, how they are communicated, how exceptions are handled, how product teams participate in policy evolution, and how the platform team maintains the discipline to keep the contract clear, maintained, and justified.

Get the organizational design right, and OPA becomes a force multiplier for both security and developer velocity. Get it wrong, and you have a sophisticated mechanism for generating friction and resentment.

Policy as code is not a technical solution to a security problem. It is an organizational mechanism for making implicit agreements explicit. The technology is the easy part.

---

## Summary: What We'd Do Differently

If we were starting over with what we know now:

1. **Write the policy charter before the first policy.** Define governance, ownership, exemption processes, and communication norms before any Rego is written.

2. **Build developer tooling first.** `platform-check` should exist before any policy hits warn mode.

3. **Never skip the audit → warn → enforce progression.** Even for policies that seem obviously correct.

4. **Invest in error message quality from day one.** Poor error messages are the primary source of support burden and developer frustration.

5. **Build the policy registry on day one.** You will need compliance evidence sooner than you expect. Make it easy to produce.

6. **Assign explicit policy ownership.** Every policy has an owner. Ownerless policies become nobody's problem — which means they become everybody's problem.

7. **Schedule quarterly policy reviews from the start.** Without them, you will accumulate dead policies, overlapping policies, and policies that no longer reflect current requirements.

8. **Resist the temptation to write policies to compensate for bad platform tooling.** Slow registries, painful image build pipelines, and missing infrastructure create workaround behavior. Policies cannot fix infrastructure gaps — they just make them more expensive.

---

*DevSecOps · Policy as Code · Open Policy Agent · Platform Engineering*

---

> **A note on tooling versions**: OPA's ecosystem evolves quickly. Gatekeeper's `ConstraintTemplate` API and bundle distribution mechanisms may have changed since this was written. Always refer to the official OPA and Gatekeeper documentation for current API specifics. The organizational patterns described here are tool-agnostic — they apply equally to Kyverno, Sentinel, or any other policy-as-code system.
