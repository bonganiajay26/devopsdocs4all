# How I Built Our SRE On-Call Program From Zero to 99.95% SLO — Without Burning Out the Team

---

## Title Variations

1. **"How I Built Our SRE On-Call Program From Zero to 99.95% SLO — Without Burning Out the Team"** *(chosen — specific, outcome-driven, addresses the real fear)*
2. **"The On-Call Program We Built After Our Third Engineer Quit Because of Pages"**
3. **"From Alert Chaos to 99.95% SLO: A Practical SRE Playbook for Engineering Leaders"**
4. **"What We Got Wrong About On-Call (And How We Fixed It Before It Broke the Team)"**
5. **"Building Sustainable On-Call: The Framework That Got Us to 99.95% Without Burning Anyone Out"**

---

## TL;DR — Executive Summary

We inherited a system where everyone was on-call, nobody was accountable, and 11pm pages were routine. Three engineers had left in six months citing on-call as a contributing factor. Reliability was a secondary concern at best.

Over 18 months, we redesigned the entire on-call program from the ground up: defined ownership, established SLIs/SLOs with error budgets, rebuilt alerting from noise down to signal, introduced structured incident management with blameless postmortems, and implemented explicit burnout prevention policies.

The results: 99.95% availability (up from ~99.1%), mean time to recovery down from 47 minutes to 9 minutes, on-call pages per engineer per week down from 11 to 2.4, and zero attrition attributable to on-call conditions in the following 14 months.

This article is the playbook. Not the idealized version — the one with the mistakes included.

---

## Main Article

---

### 1. The Starting State: Organized Chaos

When I joined, the on-call rotation was a shared PagerDuty schedule with 14 engineers on it. Everyone was on-call, always. Every alert went to everyone. There was no primary/secondary distinction, no escalation path, and no definition of what "on-call" actually meant in practice.

The oncall schedule was inherited from a startup phase where four engineers running everything made informal coordination work. It had never been redesigned as the team grew to 14. Nobody had asked the fundamental questions: what are we protecting, who owns what, and what's an acceptable level of disruption to someone's evening?

The alert inventory told the story clearly. In a single week:

- 847 PagerDuty notifications fired
- 312 woke at least one engineer outside business hours
- Of those 312, our postmortem analysis later determined that **fewer than 40 required human action that night**
- The remaining 272 were noise: self-healing conditions, false positives, and informational alerts misconfigured as pages

Three engineers had resigned in the preceding six months. In exit interviews, all three cited on-call as a contributing factor. One said something I kept returning to throughout the redesign: *"I don't mind being woken up when it matters. I mind being woken up at 2am to watch a dashboard until something resolves itself."*

That sentence contains the entire philosophy of what we needed to build.

---

### 2. Why Change Was Necessary — and Why Now

The proximate cause was attrition. The underlying cause was that we had never made explicit decisions about reliability. We had implicit SLOs (users expected the product to work), implicit on-call responsibilities (whoever saw the page first dealt with it), and implicit expectations about team health (engineers were professionals who could handle disruption).

Implicit agreements break down at scale. They produce inconsistent behavior, unequal burden distribution, and resentment that compounds quietly until someone leaves.

The business case for change was straightforward: three engineers at fully-loaded cost is significant. On-call was a recruitment problem, a retention problem, and increasingly a reliability problem. We were spending more time managing incident chaos than investing in reliability improvements. The system was consuming engineering capacity that should have been building the product.

There is a version of this story where leadership had to be convinced that reliability investment was worth it. We got lucky — after the third resignation, the case was obvious. If you are making this argument in a healthier organization, the math is simpler than you might think: track hours spent on incidents, multiply by loaded engineering cost, compare to the investment needed to reduce those incidents. Reliability investment almost always wins on a 12-month horizon.

One constraint I want to be honest about: we had organizational authority to make this change. An SRE team with no organizational mandate to enforce on-call standards will struggle with the structural changes described here. The technical practices are table stakes. The organizational authority to require them is the actual prerequisite.

---

### 3. Designing the On-Call Model

The first decision, and the most important one: **what are we actually protecting, and who owns it?**

We mapped every production service to an owning team. Not a list of everyone who touches it — a single team that is the last line of defense. This is harder than it sounds. Six of our fourteen services had no clear owner. Teams would point at each other during incidents. Resolving ownership conflicts before they became incident conflicts was three weeks of uncomfortable conversations.

The ownership map became the foundation for the rotation design. We built three separate rotations:

**Primary rotation: Product Engineering**
Teams closest to the application layer own their services. They carry the pager for anything in their domain. This is not SRE owning everything — that model doesn't scale and it removes accountability from the teams who make the code decisions.

**Secondary/escalation rotation: Platform SRE**
Infrastructure layer: Kubernetes clusters, databases, networking, shared platform services. Also serves as escalation path when a product team primary cannot resolve within 30 minutes.

**Staff on-call: Engineering management**
Not a technical rotation. One engineering manager per week available to handle escalations that require organizational decisions — vendor calls, major customer communication, cross-team coordination. This protects ICs from carrying organizational problems that aren't theirs to solve alone.

The tiered structure solved several problems simultaneously. Product engineers became experts in their own systems rather than generalists drowning in unrelated alerts. The SRE team focused on platform reliability rather than debugging application behavior. Management had clear accountability without being in the technical weeds.

**Rotation design principles we established:**

- Minimum rotation depth: 5 engineers. With fewer, any individual is on-call too frequently. We negotiated this as a hard constraint — teams below 5 engineers paired with a neighboring team until they grew.
- Maximum shift length: 7 days primary, then mandatory minimum 2 weeks off-call.
- Follow-the-sun where possible: for teams with engineers in multiple time zones, we handed off the rotation at business day boundaries. For single-timezone teams, we accepted that overnight on-call was unavoidable but committed to compensating for it explicitly (more on this in Section 7).
- No on-call without adequate runbooks. Engineers should not be paged for conditions they have no documentation to respond to.

**What we got wrong in the rotation design:**

We initially made the rotation voluntary — engineers opted into the schedule rather than it being a team obligation. This created an immediate equity problem. Senior engineers, who had more context, volunteered more heavily. Junior engineers opted out more frequently, which was individually understandable but collectively unsustainable. It also meant that the engineers most in need of developing incident response skills (juniors) weren't developing them.

We moved to an expectation model: on-call is a professional responsibility for engineers above a certain level, with explicit exceptions for individuals in specific circumstances (mental health leave, family emergencies, etc.). Junior engineers participate in a shadowing role — they receive pages alongside a senior engineer and are expected to take the lead on low-severity incidents.

---

### 4. Defining SLIs, SLOs, and Error Budgets

This is where most teams spend too much time in abstraction. The goal is a small number of SLIs that customers actually care about and SLOs that are honest about current capability rather than aspirational.

**Our SLI selection process:**

We started by asking product and customer success: what does the product need to do for a customer to consider it "working"? We got a list of about 40 things. We then asked: which of these does the customer directly notice when it fails? That cut the list to 11. We then asked: which of these can we actually measure accurately at the system boundary? That cut the list to 4.

Four SLIs. Not 40. Fewer SLIs, precisely measured, are more actionable than comprehensive SLIs with questionable data quality.

Our production SLIs:

```
SLI 1: Request success rate
  Definition: Percentage of HTTP requests returning non-5xx responses
  Measurement window: 5-minute rolling window
  Numerator: count(requests where status < 500)
  Denominator: count(all requests)
  Exclusions: Health check endpoints, known canary traffic

SLI 2: Request latency
  Definition: Percentage of requests completing within 500ms at p99
  Measurement window: 5-minute rolling window
  Measurement point: Load balancer egress (not application-internal)

SLI 3: Data pipeline completeness
  Definition: Percentage of expected pipeline runs completing successfully
  within their scheduled window
  Measurement: Job success rate × schedule adherence

SLI 4: Payment authorization success rate
  Definition: Percentage of payment authorization requests completing
  successfully (excluding known card declines)
  Measurement: Payment service response codes, excluding issuer declines
```

**SLO targets:**

We made a deliberate decision to set SLOs based on current capability plus a realistic improvement curve, not on what we thought we should aspire to. Setting an SLO of 99.99% when current availability is 99.1% is not a standard — it is a fantasy that produces constant error budget exhaustion and teams that stop caring about the metric.

```
Initial SLOs (set at program launch):
  Request success rate:              99.5% (30-day rolling)
  Request latency (p99 < 500ms):     99.0% (30-day rolling)
  Data pipeline completeness:        99.0% (7-day rolling)
  Payment authorization success:     99.8% (30-day rolling)

Target SLOs (12-month objective):
  Request success rate:              99.9%
  Request latency:                   99.5%
  Data pipeline completeness:        99.5%
  Payment authorization success:     99.95%
```

The gap between initial and target SLOs was the roadmap. Error budget = 1 - SLO. If you're at 99.5% and you have a budget of 0.5%, you have 216 minutes of allowable downtime per month. Every incident consumes from that budget. Running out of error budget triggers a reliability investment prioritization: no new features until you get back above the line.

**Error budget policy — the part most teams skip:**

The error budget is only useful if it drives decisions. We created an explicit policy:

- Error budget > 50% remaining: development velocity is unrestricted
- Error budget 25–50% remaining: one engineering sprint per quarter dedicated to reliability work
- Error budget < 25% remaining: feature development paused, reliability work takes priority
- Error budget exhausted: feature freeze until budget is restored, incident review with leadership

This policy had real consequences twice in the first year. Both times it was uncomfortable. Both times it was the right call. The error budget policy is what converts SLOs from metrics into accountability.

---

### 5. Alerting Philosophy — From Noise to Signal

The 847-alerts-per-week situation could not be fixed incrementally. We needed a rebuild from a clean slate.

**The principle we adopted:** An alert is a request for a human to take action right now. If a human does not need to take action right now, it is not an alert — it is a log entry or a metric that belongs on a dashboard.

This sounds obvious. In practice, every alert in an established system has a history, a creator, and someone who thought it was important enough to exist. Deleting alerts generates political friction. We handled this by creating a separate "alert graveyard" — we didn't delete the old alerts, we moved them to a "disabled" state with a 90-day retention policy. After 90 days without enabling them, they were deleted. This gave engineers a grace period if they realized they'd disabled something important.

**Our alerting rebuild methodology:**

Step 1: **Categorize every existing alert** into one of four buckets:
- Symptom-based (alerts on user impact): keep and review
- Cause-based (alerts on internal metrics that don't directly indicate user impact): candidate for deletion
- Informational (alerts on things that are interesting but not actionable): move to dashboard
- Duplicate/redundant: delete

Step 2: **Establish the signal baseline.** For every alert we kept, we required:
- A runbook link in the alert body
- A defined response time expectation (page immediately vs. next business day)
- A quarterly review annotation showing who last confirmed the alert was still appropriate

Step 3: **Implement multi-window, multi-burn-rate alerts** for SLO-based alerting.

The multi-burn-rate approach was the single most impactful technical change we made to alerting. Instead of alerting on instantaneous error rate spikes (high noise), we alert when error budget is burning at a rate that would exhaust the budget before the next natural review period.

```yaml
# Prometheus alerting rules — multi-burn-rate SLO alerting

groups:
  - name: slo_burn_rate
    rules:

      # Critical: fast burn — will exhaust monthly budget in 1 hour
      # Uses 5-minute and 1-hour windows to reduce false positives
      - alert: SLOBudgetBurnRateCritical
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[5m]))
              /
              sum(rate(http_requests_total[5m]))
            )
          ) > (14.4 * (1 - 0.999))
          and
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[1h]))
              /
              sum(rate(http_requests_total[1h]))
            )
          ) > (14.4 * (1 - 0.999))
        for: 2m
        labels:
          severity: critical
          page: "true"
        annotations:
          summary: "SLO budget burning at 14.4x — exhausts monthly budget in ~1 hour"
          runbook: "https://runbooks.company.io/slo-burn-rate-critical"

      # Warning: slow burn — will exhaust monthly budget in 6 hours
      - alert: SLOBudgetBurnRateWarning
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[30m]))
              /
              sum(rate(http_requests_total[30m]))
            )
          ) > (6 * (1 - 0.999))
          and
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[6h]))
              /
              sum(rate(http_requests_total[6h]))
            )
          ) > (6 * (1 - 0.999))
        for: 15m
        labels:
          severity: warning
          page: "false"   # Warning — notify, don't page outside hours
        annotations:
          summary: "SLO budget burning at 6x — exhausts monthly budget in ~6 hours"
          runbook: "https://runbooks.company.io/slo-burn-rate-warning"
```

The math behind burn rate multipliers: if our SLO is 99.9% (0.1% error budget = 43.2 minutes/month), a burn rate of 14.4x means errors are occurring at 14.4 times the sustainable rate. At this rate, the monthly budget exhausts in 43.2 / 14.4 = 3 hours. The multi-window confirmation (5m AND 1h) dramatically reduces false positives from transient spikes.

**Results of the alerting rebuild:**

- Week 1 baseline: 847 alerts/week
- Post-categorization (week 4): 312 alerts/week (disabled or moved 535)
- Post-burn-rate migration (week 12): 94 alerts/week
- 6-month steady state: 67 alerts/week

More importantly: actionability. At 847 alerts/week, our postmortem analysis showed 83% were non-actionable. At 67 alerts/week steady state, we measured 91% actionability — engineers were being paged for things that required a human response.

---

### 6. Incident Management Process and Runbooks

Incident response without a defined process is improv theater where people are actually stressed. The variance in response quality was enormous: when the right engineer happened to be on-call, incidents resolved in 15 minutes. When a less experienced engineer was primary with an ambiguous escalation path, the same class of incident took 90 minutes and involved five people in a Zoom call going in circles.

**We standardized three things:**

**Severity levels** — shared language before anything else:

| Severity | Definition | Response Time | Wake Up? |
|----------|------------|---------------|----------|
| SEV-1 | Customer-facing outage, revenue impact | Immediate | Yes |
| SEV-2 | Partial outage or significant degradation | 15 minutes | Business hours: yes. Off-hours: yes for primary. |
| SEV-3 | Minor degradation, workaround exists | Next business day | No |
| SEV-4 | Non-customer-facing issue, monitoring concern | Next sprint | No |

The SEV-3/SEV-4 distinction was the one that immediately reduced overnight pages. Many alerts that had been waking engineers were genuinely SEV-3 or SEV-4 conditions. Making the distinction explicit gave on-call engineers confidence to document, acknowledge, and defer without feeling like they were ignoring something important.

**Incident Commander role:**

Every SEV-1 and SEV-2 incident has an Incident Commander (IC). The IC is not necessarily the most senior technical person — they are the person responsible for keeping the incident organized. Their job:

- Maintain the incident timeline in the incident channel
- Keep communication moving (periodic updates to stakeholders)
- Ensure the technical responders have what they need (access, context, a quiet space to work)
- Make the call on escalation
- Run the initial postmortem kickoff

Separating the IC role from the technical responder role was a revelation. Before the separation, incidents were technical people trying to debug systems while simultaneously managing stakeholder Slack messages. The cognitive split was brutal. With a dedicated IC, technical responders could focus on the technical problem. The IC handled everything else.

**Runbook standards:**

A runbook that does not help an engineer who has never seen this system before is not a runbook — it is documentation for people who don't need it.

Our runbook template:

```markdown
# Runbook: [Alert Name]

## TL;DR
One sentence: what is happening and what is the first action to take.

## Impact
What users/systems are affected when this alert fires.
Expected vs. actual behavior.

## Likely Causes
1. [Most common cause] — Frequency: ~60% of pages
2. [Second most common cause] — Frequency: ~25% of pages
3. [Edge cases] — Frequency: ~15% of pages

## Diagnostic Steps
### Step 1: Verify the alert is genuine
  - Command/link: `kubectl get pods -n production | grep [service-name]`
  - Expected output if real: [what you'll see]
  - Expected output if false positive: [what you'll see]

### Step 2: [Next diagnostic step]
  - Command/link: [specific command]
  - If result is X: go to Step 3A
  - If result is Y: go to Step 3B

## Resolution Actions

### Resolution A: [For cause 1]
  1. [Specific step]
  2. [Specific step]
  3. Verify resolution: [how to confirm the issue is resolved]

### Resolution B: [For cause 2]
  1. [Specific step]

## Escalation
If not resolved within 30 minutes: page [team/person]
If this is a database issue: escalate to [DBA team contact]

## Related Alerts
- [Related alert name] — often fires alongside this alert for [reason]

## History
Last updated: [date] by [engineer]
Recent incidents: [link to postmortems]
```

The "Likely Causes" section with percentage estimates was the most valuable addition. It came from postmortem analysis of historical incidents. Giving the on-call engineer a probability-weighted starting point cut mean diagnostic time significantly.

**Postmortem process:**

We adopted blameless postmortems but added one requirement that most postmortem templates miss: a **counterfactual question**.

Standard postmortem questions focus on what happened. Our counterfactual question: "At what point, if a different decision had been made or a different condition had existed, would this incident not have occurred or have been materially less severe?" This frames the analysis as a systems problem rather than a person problem, without the abstraction of pure timeline-only postmortems that avoid naming specific decisions.

Every postmortem produced at minimum three action items, each with an owner and a deadline. Action item completion became a metric we tracked. The single most demoralizing pattern in postmortems is when the same action items appear in three consecutive postmortems for three separate incidents. We treated repeat action items as a signal that the priority level needed to increase.

---

### 7. Burnout Prevention and Sustainable Rotations

This section is the one I would have prioritized higher if I were doing this again. We got the technical practices mostly right in the first six months. We underinvested in the human side until we had a near-miss with engineer exhaustion in month eight.

**The near-miss:** One of our most experienced SREs had been primary on-call for two consecutive weeks (covering for a teammate on leave) and had multiple SEV-1 incidents in both weeks. She didn't raise the alarm — she continued to perform, handled every incident professionally, and submitted her availability for the next rotation cycle. In a one-on-one, almost as an aside, she mentioned she hadn't slept more than five hours on any night in the past two weeks. She was not about to quit or demand change. She was quietly grinding through it. That is the burnout pattern that is hardest to detect and most dangerous over time.

After that conversation, we implemented explicit protections:

**Protection 1: On-call hour tracking**

We built a simple dashboard that tracked hours of active incident response per on-call shift. Any shift with more than 10 hours of active response time in a week triggered an automatic conversation with the engineer's manager. Not a punishment — a check-in with a question: what do you need?

**Protection 2: Mandatory recovery time**

After any on-call shift involving a SEV-1 incident, the engineer received the equivalent number of recovery hours as their longest consecutive incident response block. If you were debugging a production incident for 4 hours overnight, you received 4 hours of protected recovery time during business hours that week.

This was initially controversial with engineering management — it felt like slack in the system. The framing that worked: would you rather this engineer take four hours off tomorrow, or have them make a critical error in a production system in two weeks because they've been running at 60% capacity?

**Protection 3: Alert quality reviews as a team activity**

Monthly, the on-call team reviewed the previous month's alerts together. Not to identify who was to blame for poor alert quality, but to categorize alerts collectively and make tuning decisions. This served two purposes: it improved alert quality continuously, and it gave engineers agency over the conditions of their on-call experience.

**Protection 4: A clear "I need help" protocol**

We made explicit that on-call engineers could and should hand off an incident mid-investigation if they hit a cognitive wall. The protocol: drop a message in the incident channel with a status summary and request a handoff. No justification required. We normalized this by having senior engineers model it publicly during less severe incidents.

**Protection 5: Follow-the-sun for teams that could support it**

We had engineers in three time zones. For the platform SRE rotation, we implemented a handoff at the business day boundary: the US East engineer took primary from 9am–5pm ET, handed to US West at 5pm ET (who covered until midnight ET), and handed to our European engineer at midnight ET (who covered until 9am ET next day). This eliminated overnight pages for all three engineers in most weeks.

The operational overhead was real: structured handoffs required discipline, and we lost some context transfer at shift changes. We handled this with a lightweight shift handoff document:

```markdown
## On-Call Handoff — [Date] [Time] [Timezone]

Handing off: [Name]
Taking over: [Name]

### Active incidents
- None / [Incident link and current status]

### Active alerts in warning state (watch these)
- [Alert name]: [What to watch for, what action if it fires]

### Known issues / degraded services
- [Service]: [Status and context]

### Planned maintenance in next 12 hours
- [Any maintenance windows or risky deployments]

### Anything you should know
[Freeform context that doesn't fit above]
```

**Rotation sizing as a burnout lever:**

The most structural burnout prevention is rotation depth. The math is simple: if you are in a 4-person rotation, you are on-call 25% of the time. In a 7-person rotation, you are on-call 14% of the time. The difference in psychological load between 25% and 14% is not proportional to the numbers — it is much larger, because on-call affects planning, sleep quality, and weekend anxiety even on off-call weeks.

We set a policy: no rotation with fewer than 6 engineers at steady state. Teams that could not reach 6 people paired with a neighboring team. This was organizationally uncomfortable but the right call.

---

### 8. Metrics, Results, and Lessons Learned

After 18 months, here is what we measured and what changed:

**Reliability metrics:**

| Metric | Baseline | 6 Months | 18 Months |
|--------|----------|----------|-----------|
| Monthly availability | ~99.1% | 99.7% | 99.95% |
| MTTR (mean time to recovery) | 47 min | 22 min | 9 min |
| MTTD (mean time to detect) | 18 min | 6 min | 2.5 min |
| Incidents/month (SEV-1+2) | 14 | 7 | 3 |

**On-call health metrics:**

| Metric | Baseline | 6 Months | 18 Months |
|--------|----------|----------|-----------|
| Pages/engineer/week | 11.2 | 4.8 | 2.4 |
| Actionable alert rate | 17% | 61% | 91% |
| Overnight pages (off-hours) | 312/week | 89/week | 23/week |
| Runbook coverage | ~20% of alerts | 67% | 94% |

**Team health (from quarterly engineering surveys, n varies 8–14):**

- Engineers rating on-call experience as "manageable" or better: 23% → 84%
- Engineers who would recommend the company to peers citing on-call as a positive: 8% → 61%
- On-call-related attrition in the 14 months following program launch: 0 (vs. 3 in preceding 6 months)

**The metric we stopped tracking — and why:**

We initially tracked "number of postmortems completed per month" as a health indicator. This was naive. Teams began closing postmortems quickly to hit the metric rather than thoroughly to improve the system. We replaced it with "postmortem action item completion rate" (% of action items closed by their stated deadline), which measured what actually mattered.

**Lessons that aren't in the metrics:**

The biggest cultural shift was in how engineers talked about incidents. Before: incidents were failures, something to move past quickly. After: incidents were information. They told us where the system was fragile and where our understanding was incomplete. The blameless postmortem process created the conditions for that shift, but it took about six months of consistent modeling from senior engineers before it took root.

The second cultural shift: on-call became a professional development activity rather than a burden to be minimized. Junior engineers in the shadowing role developed diagnostic skills that previously took years to acquire informally. We tracked this explicitly and used on-call participation as one input into the senior engineer promotion criteria.

---

## What I'd Do Differently

**1. Start with ownership mapping, not tooling.**
The first instinct when building an SRE program is to evaluate observability platforms and incident management tools. These decisions matter, but they're secondary to having clear service ownership. We spent weeks on tooling before we had an ownership map. The ownership map would have made the tooling decisions easier and the early alert routing much cleaner.

**2. Set honest initial SLOs, not aspirational ones.**
Our first instinct was to target 99.9% immediately. Our actual availability was 99.1%. Setting 99.9% as an initial target would have meant permanent error budget exhaustion and a metric that nobody trusted. We eventually went with 99.5% as the initial target, which we hit in month three, creating momentum and credibility. Start where you are, build incrementally.

**3. Build the runbook requirement into the alert creation process, not as a retrofit.**
We spent six weeks retroactively writing runbooks for existing alerts. If we had required a runbook link before any alert could be created in PagerDuty, we would have avoided the retrofit. Implementation: add a required field for runbook URL in your alert management system. If the runbook doesn't exist, the alert doesn't get created.

**4. Implement the IC role earlier.**
We ran the first four months of SEV-1 incidents without a formal Incident Commander model. Every incident involved senior engineers splitting attention between technical diagnosis and stakeholder management. The IC role was obvious in retrospect. We should have introduced it in the first month.

**5. Don't wait for burnout to be obvious before intervening.**
The near-miss with the SRE engineer in month eight was preventable. We had the data — her incident hours were visible in PagerDuty. We weren't reviewing it. Proactive review of on-call load data should happen weekly, not reactively.

**6. Involve product engineers in the on-call design, not just the SRE team.**
We designed the on-call model primarily within the SRE team and then rolled it out to product engineers. Several rotation design decisions would have been better if product engineers had been at the table earlier. The design felt imposed rather than collaborative, which created unnecessary friction in the early months.

---

## Key Takeaways Checklist

Use this as a self-assessment for your current on-call program:

**Foundation**
- [ ] Every production service has an explicit owning team (not a group, not "shared")
- [ ] On-call responsibilities are documented and communicated, not assumed
- [ ] Rotation depth is 6+ engineers at steady state; teams below this have a plan
- [ ] On-call expectations are part of the hiring and onboarding conversation

**SLOs and Error Budgets**
- [ ] You have 3–6 SLIs that represent genuine customer impact
- [ ] Your current SLOs are based on observed capability, not aspiration
- [ ] Error budget policy exists and has actually changed a prioritization decision at least once
- [ ] SLO compliance is reviewed monthly with leadership

**Alerting**
- [ ] You can state the actionability rate of your current alerts (% that required human action)
- [ ] You have eliminated or disabled alerts that don't require immediate human action
- [ ] Your critical alerts use multi-window burn rate logic, not instantaneous thresholds
- [ ] Every alert has a runbook link
- [ ] Alert quality is reviewed as a team on a regular cadence

**Incident Management**
- [ ] Severity levels are defined and shared across the organization
- [ ] SEV-1/SEV-2 incidents have a defined Incident Commander role
- [ ] Postmortems are blameless, systematic, and produce tracked action items
- [ ] Repeat postmortem action items trigger priority escalation

**Burnout Prevention**
- [ ] On-call hours per engineer are tracked and reviewed weekly
- [ ] Recovery time policy exists for engineers following high-intensity shifts
- [ ] A clear "I need help" handoff protocol exists and has been used
- [ ] Follow-the-sun is implemented where timezone coverage allows
- [ ] Engineers rate their on-call experience in regular surveys and the results are acted on

---

## Summary

---

We went from 847 alerts per week and 3 engineers citing on-call as a resignation factor, to 99.95% availability and zero on-call-related attrition in 14 months.

The technical changes: multi-burn-rate alerting, SLOs built on actual capability (not aspiration), tiered severity levels, and an Incident Commander model that separated communication from technical diagnosis.

The organizational changes were harder: clear service ownership before anything else, runbooks as a prerequisite for alert creation, mandatory recovery time after high-intensity incidents, and explicit error budget policies that actually changed feature prioritization twice.

The biggest lesson: the technical practices are learnable in weeks. The cultural shift — where incidents become information rather than failures — takes six months of consistent leadership modeling.

If you're building an SRE program, start with ownership mapping and honest SLOs. Everything else gets easier from there.

Happy to share specifics on any part of this. What's your current on-call page rate per engineer per week?

---

*Site Reliability Engineering · Incident Management · Engineering Leadership · Platform Engineering*
