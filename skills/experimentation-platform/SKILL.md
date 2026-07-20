---
name: experimentation-platform
description: Build or evaluate an experimentation platform — experiment definition and deployment, variant assignment with consistent hashing, concurrent experiments and layers, analysis pipelines (data cooking, metric computation, scorecards, near-real-time alerting and auto-shutdown), and the build-vs-buy decision. Invoke when designing A/B testing infrastructure, choosing a vendor, or scaling experiment analysis.
---

# Experimentation Platform

Covers the infrastructure that makes experiments self-service and trustworthy at scale: the four platform components, variant assignment mechanics, concurrent-experiment designs, and the automated analysis pipeline. The single most important principle: the goal of a platform is to minimize the incremental cost of running a *trustworthy* experiment — a platform that accelerates untrustworthy results accelerates bad decisions.

## When to use

- Designing or reviewing the architecture of an A/B testing platform.
- Deciding whether to build an in-house platform or buy a third-party solution.
- Implementing variant assignment: hashing, bucketing, traffic allocation, layers.
- Scaling beyond a handful of experiments — users need to be in multiple experiments at once.
- Building the analysis side: log cooking, metric computation, scorecards, dashboards.
- Adding near-real-time monitoring, alerting, or automatic shutdown of bad experiments.
- Questions like "how do Google/Bing/LinkedIn run 20,000 experiments a year?"

## Core principles

1. **A platform has four high-level components — build all of them, not just assignment.** The components shared by Bing, LinkedIn, and Google platforms: (a) experiment definition, setup, and management via UI/API, stored in the experiment system configuration; (b) experiment deployment covering variant assignment and parameterization, server- and client-side; (c) experiment instrumentation; (d) experiment analysis — metric definitions, computation, and statistical tests (p-values, confidence intervals). Teams routinely underinvest in (c) and (d); without them the platform is a feature-flagging system, not an experimentation platform.

2. **Variant assignment must be random yet deterministic, and independent across users.** Assignment is a pseudo-random hash of an ID — `f(user_id)` — so the same user always gets the same variant. Knowing one user's assignment must tell you nothing about another user's assignment. Typical mechanics: hash the user ID into 1,000 disjoint buckets (`f(UID) % 1000`) and allocate bucket ranges to variants — 200 buckets = 20% traffic. Any two buckets in the same treatment must be statistically similar: roughly equal user counts (also within slices like country, platform, language) and roughly equal key-metric values. Monitor bucket characteristics continuously — Google and Microsoft both found randomization-code bugs this way.

3. **Re-randomize buckets between experiments to kill carry-over effects.** Prior experiments can taint buckets for the next experiment (users habituated or damaged by an earlier treatment). Shuffle bucket-to-user mapping with every new experiment so buckets are no longer contiguous. Iterations of the *same* experiment should share the same hash ID so users get a consistent experience across iterations.

4. **Scale concurrency with layers, not manual traffic negotiation.** Single-layer (numberline) assignment — each user in exactly one experiment — is fine early on but caps the number of concurrent experiments and forces manual traffic haggling (LinkedIn negotiated ranges by email; Bing had a program manager whose office was packed with people begging for traffic). To scale, move to overlapping experiments: multiple layers, each behaving like a single-layer system, with the layer ID added to the hash (`f(UID, layer_id) % 1000`) so layers are orthogonal. Production code and instrumentation must then handle a *vector* of variant IDs per request.

5. **Choose a layer design that prevents bad interactions.** A full-factorial design (every user in every experiment, one layer per experiment) scales in a decentralized way and maximizes power, but cannot prevent collisions — e.g., blue text from Experiment 1 plus blue background from Experiment 2 is unusable for users in both treatments, and each experiment's measured result is wrong if interactions are ignored. Two safer designs: **nested** (partition system parameters into layers — UI header, page body, backend, ranking — so conflicting experiments are structurally in the same layer and never co-assigned) and **constraints-based** (experimenters declare conflicts; a graph-coloring algorithm keeps conflicting experiments off the same user). Google, LinkedIn, Microsoft, and Facebook all use variants of these. Microsoft also runs automated interaction detection. Full factorial is still preferable when the power loss from splitting traffic outweighs interaction risk — and you can detect interactions post hoc.

6. **Pick one of three deployment architectures deliberately — it's a triggering-vs-debt-vs-performance tradeoff.** (a) Code fork on `getVariant(userId)`: assignment happens next to the change, so triggering (identifying affected users) is easy, but forked code paths accumulate technical debt. (b) Parameterized system: every testable change is controlled by an experiment parameter; less debt, still easy triggering. (c) Assignment done once early in the flow; a full config with all parameter values is passed down (`config.getParam("buttonColor")`); every parameter has a default and treatments only override deltas. (c) is the most performant at hundreds-to-thousands of parameters but makes triggering and counterfactual logging harder. Google moved from (a) to (c) for performance and to escape code-path-reconciliation pain; Bing uses (c); Office uses parameterized code forks plus a bug ID passed as an experiment parameter that fires an alert after three months to remind engineers to delete stale experimental code paths.

7. **One shared assignment implementation, and decide atomicity explicitly.** The Variant Assignment Service can be a separate server or a library embedded in client/server — but there must be a single implementation, or divergent copies will drift and introduce bugs. Decide whether all servers must switch to a new iteration atomically: a single web request can fan out to hundreds of servers, and inconsistent assignment (e.g., different ranking algorithms on disjoint index shards of one query) corrupts the user experience. Fix: the parent service assigns once and passes the assignment down to children.

8. **Constrain where in the flow assignment happens.** Assignment can occur at the traffic front door, client side, or server side. Ask: at what point do you have all attributes needed for targeting (user ID, language, device are available at request time; account age or visit frequency need a lookup, pushing assignment later)? In Walk/early Run phases, allow assignment at exactly *one* point — multiple assignment points require orthogonality guarantees so earlier assignment doesn't bias later assignment.

9. **Instrument every request with variant and iteration IDs, and log counterfactuals where needed.** The platform must auto-generate experiment IDs, variant IDs, and iteration numbers and stamp them into instrumentation. Iteration matters because at start/ramp, not all servers or clients switch simultaneously. In the Run/Fly phases, log the counterfactual — what a Treatment user *would* have seen under Control — which is essential for triggered analysis but hard in architecture (c) and expensive at runtime; set guidelines for when it's required. User-entered feedback must be logged with variant IDs.

10. **Automate analysis end to end: cook, compute, visualize.** Data processing ("cooking"): sort and join logs from multiple systems by user ID and timestamp, sessionize, clean (bot/fraud heuristics, duplicate events, timestamp fixes), and enrich (parse user agents, add per-session aggregates, annotate experiment transitions). Computation: metrics by segment (country, language, device/platform), p-values/confidence intervals, trustworthiness checks like SRM, and automatic interesting-segment detection. Visualization: scorecards showing relative change with statistical-significance color-coding. Test the cooking and computation pipelines themselves — trust in results starts with trust in the pipeline. And when reading results, always check trustworthiness tests (SRM) *before* looking at the OEC or segments; Microsoft's ExP hides the entire scorecard if key trust tests fail.

11. **Choose a computation architecture: materialized per-user stats vs integrated per-experiment computation.** Option 1: materialize per-user statistics (pageviews, impressions, clicks per user) and join to a user→experiment mapping table — reusable for general business reporting, not just experiments. Option 2: compute per-user metrics on the fly inside experiment analysis — more per-experiment flexibility and can save machine/storage cost, but requires shared metric/segment definitions across pipelines to prevent drift. Either way: maintain a single source of metric definitions, enforce implementation consistency (common implementation or ongoing comparison tests), and plan change management — redefining an existing metric is harder than adding one; decide whether and how far to backfill.

12. **Run two analysis paths: near-real-time for safety, batch for truth.** Bing, Google, and LinkedIn process terabytes of experiment data daily; all started with ~24-hour-delayed daily scorecards (Monday's data by end of Wednesday) and now run NRT paths alongside batch. The NRT path uses simple metrics (sums, counts), minimal statistical testing, no spam filtering, and often reads raw logs directly; its job is catching egregious problems — misconfigured or buggy experiments — and it triggers alerts and automatic experiment shut-off. The batch pipeline produces the trustworthy scorecard. Also support safety features on the management side: near-real-time monitoring/alerting, automated detection and shutdown of bad experiments, and automated ramp-up (pre-defined rings: team developers → all employees → growing external percentages).

13. **The management UI is a trust and safety surface, not just CRUD.** Required functionality: draft/edit/save specifications (owner, name, description, start/end dates); diff a draft iteration against the running one; view full experiment history/timeline; auto-generate IDs; validate specs (config conflicts, invalid targeting); start/stop with status checks. Permissions are asymmetric by design: only owners or specially permissioned people can *start* an experiment, but *anyone* can stop one (harming users is worse than a false alarm) — with alerts so owners find out. Gate go-live with pre-deployment test code or expert approval workflows.

14. **Measure the platform's own cost.** The experimentation platform itself has performance implications. Run some traffic entirely outside the platform as a holdout — that is itself an experiment measuring the platform's impact on latency, CPU utilization, and machine cost.

15. **Build vs buy is an ROI decision — interrogate the vendor's limits before committing.** Questions that decide it: Does the vendor cover your experiment types (frontend/backend, server/client, mobile/web)? JavaScript-tag solutions can't do backend experiments, don't scale to many concurrent experiments, and add page-load latency that itself hurts engagement. Can it compute your metrics — sessionized metrics and percentiles (critical for latency tails) are often unsupported? What randomization units are possible, and what user data can legally leave your walls? Is data logged to the vendor accessible, or do clients dual-log — and who reconciles when summary statistics diverge? Can you join external data (purchases, returns, demographics)? Is there NRT for stopping bad experiments? Institutional memory? Do WYSIWYG-built features require re-implementation post-experiment (a real bottleneck at scale)? Then weigh the cost to build, your *anticipated* experiment volume (build for where the momentum leads, not current count), and whether experimentation must integrate with your configuration/deployment system. A pragmatic path: use a vendor to demonstrate impact, then build when volume justifies it.

16. **Design the scorecard for thousands of metrics and non-experts.** As orgs reach Run/Fly, metrics grow into the thousands. Group them by tier (LinkedIn: companywide / product-specific / feature-specific) or function (Microsoft: data quality / OEC / guardrail / local-feature-diagnostic). Combat multiple-testing confusion ("why did this irrelevant metric move?") with p-value thresholds below 0.05 to filter to the most significant movers, plus Benjamini-Hochberg-style corrections. Auto-surface "metrics of interest" (combining importance, significance, false-positive adjustment) and show related metrics (CTR up — because clicks rose or pageviews fell? high-variance revenue alongside trimmed revenue). Make scorecards readable by marketers and executives, hiding debug metrics from non-technical audiences. Provide per-metric views across all experiments so metric owners see which experiments impact their metrics; let people subscribe to metric digests; and optionally force an approval conversation between experiment owner and metric owner before ramping an experiment that hurts a metric beyond a threshold. Show many metrics prominently (OEC, guardrails) so teams cannot cherry-pick results.

## Workflow

Building or evaluating a platform, in order:

1. **Decide build vs buy.** Walk through the vendor questions in principle 15 against your experiment types, metrics, privacy constraints, and anticipated (not current) volume.
2. **Define the experiment spec schema.** Owner, name, description, start/end dates, targeting, layer/constraint declarations, variants with parameter overrides, iteration tracking. Support multiple iterations per experiment (bug-fix evolution, ring-based ramp), one active iteration at a time.
3. **Build the management surface.** Draft/edit/diff/history, validation, auto-generated IDs, start/stop with asymmetric permissions (restricted start, universal stop + alerts), pre-launch checks (test code or approval workflow).
4. **Implement variant assignment.** Hash user ID into ~1,000 buckets; deterministic, independent, re-randomized per experiment. Single shared implementation (service or library). One assignment point in the flow initially. Decide atomicity handling (parent assigns, passes down).
5. **Choose the deployment architecture** — code fork, parameterized, or early-assignment config — using the triggering/debt/performance tradeoff. Give every parameter a default; treatments specify only deltas. Add a stale-experiment-code reminder mechanism (e.g., Office's 3-month bug-ID alert).
6. **Plan for concurrency.** Start single-layer; when traffic negotiation becomes painful, move to layers with `f(UID, layer_id)`. Pick nested or constraints-based design for interaction safety; make code and logs handle variant-ID vectors.
7. **Wire instrumentation.** Stamp every request/interaction with experiment, variant, and iteration IDs. Decide counterfactual-logging policy. Log feedback with variant IDs.
8. **Build the analysis pipeline.** Cooking (sort/join/clean/sessionize/enrich) → computation (metrics × segments, p-values/CIs, SRM and trust checks, interesting-segment detection) → visualization (relative changes, significance color-coding, trust-check gating). Test the pipeline itself. Centralize metric definitions with change management.
9. **Add the NRT safety path.** Simple counts on raw logs, alerting, automatic shutdown of egregiously bad experiments. Keep batch as the source of truth.
10. **Measure platform overhead** with a permanent outside-the-platform holdout.
11. **Scale the scorecard** as metrics grow: tiers, tighter p-value filters, metrics-of-interest, related metrics, per-metric cross-experiment views, subscriptions, ramp-approval workflows.

## Pitfalls

| Pitfall | Why it happens | Detect / avoid |
|---|---|---|
| Randomization bugs (uneven buckets) | Hash function or bucketing code errors | Continuously monitor bucket sizes and metric values across buckets and slices; A/A tests; SRM checks. Google and Microsoft both caught real bugs this way |
| Carry-over effects tainting results | Reusing contiguous buckets across experiments | Re-randomize (shuffle) buckets with every new experiment |
| Interacting experiments corrupting each other's results | Full-factorial overlap without conflict prevention | Nested or constraints-based layer design; automated interaction detection (Microsoft) |
| Inconsistent assignment within one request | Fan-out to many servers, non-atomic iteration switchover | Parent service assigns once and passes the assignment to child services |
| Divergent assignment logic | Multiple reimplementations of the hash/allocation code | Single shared library or service |
| Later assignment biased by earlier assignment | Multiple assignment points without orthogonality guarantees | One assignment point until platform maturity supports overlap guarantees |
| Escalating technical debt from code forks | Architecture (a) with no cleanup discipline | Parameterized architecture; expiry alerts on experimental code paths (Office: 3 months) |
| Counterfactual logging missing or too costly | Early-assignment architecture makes it hard; runtime expense | Set explicit guidelines for when counterfactual logging is required; budget its cost |
| Cherry-picked results | Only favorite metrics visible on the scorecard | Compute many metrics; make OEC and guardrails prominently visible to all |
| Reading results before trust checks | Scorecard shows everything at once | Gate or order the scorecard: SRM/trust checks first; hide scorecard on failure (Microsoft ExP) |
| Cleaning-induced SRM | Filtering removes more events from one variant than the other | Compare filter rates per variant; run SRM after cleaning |
| Metric definitions drifting between pipelines | Experiment pipeline and business-reporting pipeline evolve separately | Shared definition store; common implementation or ongoing cross-checks |
| Slow scorecards delaying decisions | Resource-intensive computation at scale | Efficient computation architecture; NRT path for urgent signals; intra-day batch updates |
| Bad experiment runs for hours unnoticed | No live monitoring | NRT alerting plus automated shutdown |
| Vendor platform can't grow with you | Bought for today's volume, not the trajectory | Decide on anticipated volume; plan the migration cost of switching |
| Platform itself slows the product | Assignment/config/logging overhead | Permanent holdout traffic outside the platform to measure its cost |

## Rules of thumb

- Bucket granularity: hash user ID into ~1,000 disjoint buckets; 200 buckets = 20% traffic.
- Maximum power for a single experiment: 50%/50% split over all users.
- Maturity cadence (rough): Crawl ~10 experiments/year → Walk ~50/year → Run ~250/year → Fly thousands/year (~4–5× per phase). Google, LinkedIn, and Microsoft each exceed 20,000/year.
- After reaching >1 experiment/day, expect roughly an order-of-magnitude growth over the next four years; eventually the bottleneck becomes converting ideas to code, not the platform (Office grew experiments >600% in 2018 on an already-built platform).
- Historical scorecard latency was ~24 hours (Monday's data by Wednesday EOD); modern platforms pair that batch path with a near-real-time path for alerting and auto-shutdown.
- One assignment point in the flow until Walk/early-Run maturity; one active iteration per experiment at a time.
- Anyone can stop an experiment; only owners/permissioned users can start one.
- Stale experimental code paths: alert for removal after ~3 months (Microsoft Office's bug-ID mechanism).
- With thousands of metrics, filter scorecards using p-value thresholds stricter than 0.05.
- Experiment data scale at large companies: terabytes per day.

## Related skills

- [instrumentation-and-clients](../instrumentation-and-clients/SKILL.md) — the logging that feeds this platform; client-side assignment and caching specifics
- [trustworthiness](../trustworthiness/SKILL.md) — SRM, A/A tests, and the trust checks the pipeline must automate
- [analyzing-results](../analyzing-results/SKILL.md) — the statistics the computation step implements (p-values, CIs, multiple testing, segments)
- [metrics-and-oec](../metrics-and-oec/SKILL.md) — metric tiers, guardrails, and OEC the scorecard presents
- [ramping](../ramping/SKILL.md) — automated ramp-up and staged rollout the platform should support
- [variance-and-sensitivity](../variance-and-sensitivity/SKILL.md) — triggering and counterfactual logging interplay
- [culture-ethics-memory](../culture-ethics-memory/SKILL.md) — institutional memory, meta-analysis, and the cultural half of platform maturity
- [designing-experiments](../designing-experiments/SKILL.md) — power and traffic-allocation decisions that drive layer design

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapters 4 and 16.
