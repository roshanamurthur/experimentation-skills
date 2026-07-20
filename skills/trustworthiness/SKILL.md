---
name: trustworthiness
description: Validate that A/B experiment results can be trusted before acting on them. Invoke when results look surprisingly good or bad, before shipping a winner, when checking for SRM (sample ratio mismatch), when setting up or interpreting A/A tests, or when auditing an experimentation platform for internal/external validity threats (SUTVA, survivorship bias, novelty effects, redirects, bot filtering).
---

# Experiment Trustworthiness

Validate results before believing them. The organizing principle is Twyman's law: **"Any figure that looks interesting or different is usually wrong."** Extreme results are more often caused by instrumentation errors, data loss/duplication, or computation bugs than by genuinely huge effects. Treat trust checks like asserts in defensive programming — run them on every experiment, and refuse to read the metrics scorecard until they pass.

## When to use

- A key metric moved surprisingly far in either direction and someone wants to ship or celebrate.
- Before making any ship/no-ship decision on experiment results.
- The ratio of users between variants looks off, or an SRM alert fired and needs debugging.
- Standing up or auditing an experimentation platform (A/A test design, p-value distribution checks).
- Results are being generalized to a new market, platform, or a long time horizon.
- A treatment effect is drifting over time (suspected novelty or primacy effect).
- Deciding whether a redirect-based, email-based, or trigger-based experiment design is sound.

## Core principles

1. **Apply Twyman's law asymmetrically-aware.** When results are surprisingly positive, the natural inclination is to build a story and celebrate; when surprisingly negative, to find a flaw and dismiss. Resist both. Investigate every extreme result the same way: assume instrumentation error, data loss/duplication, or computational error until ruled out. Good data scientists are skeptics.

2. **An SRM invalidates everything, even when tiny.** If the observed ratio of users between variants deviates from the design ratio with a low p-value (e.g., < 0.001), do not read any other metric except to debug. Concrete example: a 50/50 design yielding Control = 821,588 vs Treatment = 815,482 users is a ratio of 0.993 — looks negligible, but p = 1.8e-6, less than 1 in 500,000 by chance. With large samples, a ratio below 0.99 or above 1.01 for a 1.0 design almost certainly indicates a serious bug. Tiny imbalances flip conclusions because the missing users are rarely random — they are usually the extremes (heaviest users or zero-activity users). At MSN, an experiment with ratio 0.992 (~800k users per arm, p = 7e-7) initially showed a significant *drop* in engagement; the true cause was that treatment increased engagement so much that the heaviest users got classified as bots and filtered out. After fixing bot filtering, the result reversed to +3.3% engagement. At Bing, an SRM traced to old-Chrome users plus a misclassified bot made five success metrics look significantly positive; on the properly balanced 96% of users, all movement vanished. At Microsoft, ~6% of experiments exhibit an SRM — this failure mode is common, not exotic.

3. **Run A/A tests continuously, not just once.** An A/A test splits users like an A/B test but serves identical experiences. If the system is correct, each metric should be statistically significant ~5% of the time at p < 0.05, and the p-value distribution across repeated trials should be uniform. A/A tests validate Type I error control, variance estimation, unbiased assignment (catching carry-over effects from prior experiments, as Bing does with continuous A/A tests), and data agreement with the system of record (if you run 20%/20%, do you see ~20% of total users in each arm, or are you leaking users?). Run them in parallel with production A/B tests forever — they catch regressions, new broken metrics, and emerging outliers.

4. **The shape of the A/A p-value histogram diagnoses the failure.** Simulate ~1,000 A/A tests (replay last week's logs with fresh randomization hash seeds — cheap, no user impact), histogram the p-values per metric, and run a goodness-of-fit test (Anderson-Darling or Kolmogorov-Smirnov) against uniform. Failure signatures: (a) skewed distribution → variance estimation is wrong, usually because analysis unit ≠ randomization unit (e.g., CTR computed per-pageview while randomizing per-user violates independence) — fix with the delta method or bootstrap; or the metric is highly skewed and normal approximation fails (minimum sample size can exceed 100,000 users — cap the metric); (b) mass near p ≈ 0.32 → a single huge outlier dominates (t-statistic ≈ ±1) — investigate or cap; (c) a few point masses with gaps → near-single-valued metric with rare non-zero events; the t-test is inaccurate but less dangerous.

5. **Absence of significance is not absence of effect.** A non-significant result may simply be underpowered (an audit of 115 public A/B tests found most underpowered). Define practical significance up front and power the experiment to detect it. If a change affects only a small user subset, analyze that triggered subset — a large effect diluted across everyone can be undetectable.

6. **Interpret p-values and confidence intervals correctly.** The p-value is P(result this extreme or more | null is true) — not the probability the null is true, not the false-positive rate. Peeking at p-values continuously biases false positives by 5–10x; either use always-valid sequential p-values or fix the duration in advance (e.g., one week). Choosing the smallest p-value across metrics, time points, segments, or reruns is the multiple-comparisons problem — control it with False Discovery Rate methods. Overlapping 95% CIs for Control and Treatment do NOT imply no significant difference (CIs can overlap up to ~29% with a significant delta); non-overlapping CIs do imply significance. A specific 95% CI does not have a 95% chance of containing the truth — the truth is either in it or not.

7. **Check internal validity: SUTVA, survivorship, intention-to-treat.** SUTVA (Stable Unit Treatment Value Assumption) says units don't interfere; it breaks in social networks, communication tools (Skype peer-to-peer calls), co-authoring tools, two-sided marketplaces (treatment price changes affect Control's auctions), and shared resources (a treatment memory leak that crashes a machine takes Control users down with it, masking the delta). Survivorship bias: analyzing only users active for N months is the WWII-bomber mistake — armor the places with no bullet holes, because planes hit there never came back. Intention-to-treat: analyze by initial assignment, not by who actually adopted the treatment; analyzing only participants (e.g., only advertisers who accepted a campaign-optimization offer) is selection bias that overstates the effect.

8. **Check external validity: novelty and primacy effects.** Standard analysis assumes a constant treatment effect over time. Novelty effect: a flashy change attracts clicks initially, then decays (MSN changed an Outlook.com link to open a Mail app; clicks jumped +28% — not delight, but confused users clicking repeatedly; the effect visibly decayed within days). Primacy effect: users (or ML models) need time to adapt, so the effect grows. Detect both by plotting the treatment effect over time — any trend is a red flag; also plot the effect over time for only first-day-or-two users. Generalizing across populations/countries is questionable but cheap to resolve: rerun the experiment in the new market. Time-based generalization is harder — consider long-running holdouts.

9. **Never trust redirect-based implementations by default.** Redirecting Treatment to a new page consistently fails A/A tests and causes SRMs: (a) the redirect adds hundreds of ms of real-user latency (fast in the lab, 1–2 s in remote regions); (b) bots handle redirects differently — some don't follow, some deep-crawl the "new" site; (c) redirects are asymmetric — users bookmark/share the Treatment URL and land there without proper assignment, contaminating variants. Prefer server-side variant assignment; if a redirect is unavoidable, redirect both Control and Treatment so the penalty is symmetric.

10. **Monitor other trust guardrails beyond SRM.** Telemetry fidelity (web-beacon click logging is lossy; if Treatment changes the loss rate, results misstate reality — use internal referrers or dual logging to estimate loss). Cache hit rates (shared LRU caches favor the larger variant in unequal splits and violate SUTVA — include experiment ID in cache keys; prefer 10%/10% at runtime over 10%/90% if caches are shared). Cookie write rate ("cookie clobbering" — one Bing experiment writing an unused random cookie on every response showed massive fake degradations in sessions, queries, and revenue per user due to browser bugs). Quick queries (≥2 searches from one user within a second) — Google and Bing both treat treatment-driven shifts in this rate as marking results untrustworthy.

## Workflow

Pre-decision trust checklist — run in order; stop and debug at the first failure.

1. **SRM check (gate everything on this).** Compute the ratio of randomization units per variant and its p-value vs the design ratio (t-test or chi-squared). If p < 0.001: declare SRM, hide/ignore the scorecard, go to the SRM debugging workflow below. Do not rationalize "it's only 0.6% off."
2. **A/A health.** Confirm the platform currently passes continuous A/A tests: per-metric p-value histograms uniform (goodness-of-fit passing), ~5% significance rate at α = 0.05. If a metric fails only in A/A, distrust that metric everywhere.
3. **Instrumentation/pipeline sanity.** User counts match the system of record for the allocation percentages. No unexpected data loss or duplication. Bot filtering behaves the same across variants. Telemetry loss rate, cookie write rate, cache hit rate, and quick-query rate show no treatment-correlated movement.
4. **Design validity.** Randomization unit ≥ analysis unit or delta method/bootstrap applied. No redirects (or symmetric redirects). Trigger conditions use only pre-experiment attributes. Users re-randomized after any bug-fix restart (residual/carryover effects). SUTVA plausible for this product; if not, note interference risk.
5. **Statistical validity.** Duration was fixed in advance (or sequential testing used) — no peeking-based stop. Multiple-testing correction applied across metrics/segments. Power was sufficient for the practical-significance threshold; a null result on an underpowered test is "unknown," not "no effect." Effect analyzed on the triggered population where applicable, with intention-to-treat assignment.
6. **Temporal validity.** Plot treatment effect by day. Flag any increasing/decreasing trend (primacy/novelty); if trending, extend the run until stable or reject the idea. Check first-day cohort's effect over time.
7. **Twyman review.** If any result still looks extreme, actively hunt for the error before accepting it: segment by browser/platform (a strongly negative single segment often means a JS bug), check for interpretation traps, and attempt a replication. Only after all checks pass, make the ship decision.

### SRM debugging workflow (where did the users go?)

Work through the pipeline systematically:

1. **Upstream of the randomization/trigger point.** Verify zero difference between variants before that point. If you analyze from the checkout page, nothing upstream (homepage copy, etc.) may differ. Watch for cross-surface leakage (Bing Image experiments changing regular web search results caused SRMs).
2. **Variant assignment itself.** Is randomization correct at the top of the pipeline? Complexity from ramp-up (1% → 50%), exclusions, isolation groups, and concurrent experiments causes bugs — e.g., a second experiment filtering on "font = black" steals users from only one variant of a font-color experiment. Check the hash function: Fowler-Noll-Vo failed under overlapping experiments at Yahoo!; use a cryptographic hash (MD5) or a strong non-cryptographic one (SpookyHash).
3. **Each data-pipeline stage, especially bot filtering.** Bots are >50% of Bing's US traffic and >90% in China/Russia. Treatment-induced behavior changes can push real users across bot heuristics thresholds (the MSN case) or change how bots interact (redirects).
4. **Start-time asymmetry.** Did both variants truly start together? Shared Controls, cache warm-up, app-push delays, and offline phones cause early-period skew — try excluding the initial period.
5. **Segment the sample ratio itself.** By day (did a ramp or a competing experiment "steal" traffic on one day?), by browser (one bad browser segment, as in the Bing scenario), by new vs returning users, and by intersection with other concurrent experiments (variant mix from other experiments should match across arms).
6. **Trigger conditions.** Was triggering based on an attribute the treatment can change (e.g., "dormant" users reactivated by the campaign disappear from the trigger)? Trigger on the attribute's pre-experiment/pre-assignment snapshot. ML-model-based triggers are especially suspect because models retrain mid-experiment.
7. **Residual/carryover effects.** Restarted after a bug fix without re-randomizing? Users who abandoned during the buggy period create an SRM. Carryover can also come from a prior successful experiment on the same users (LinkedIn People You May Know restart) or leftover frequency-cap cookies. Re-randomize.
8. **Decide: correct or rerun.** If the cause is fully understood and fixable in analysis (e.g., bot misclassification), fix and re-analyze. If traffic had to be removed (e.g., an entire browser excluded), some segments never got a valid treatment — rerun the experiment.

## Pitfalls

| Pitfall | Why it happens | Detect / avoid |
|---|---|---|
| Celebrating a surprising win without investigation | Story-building bias on positive results | Twyman's law: treat extreme results as probable bugs; run the full checklist first |
| Dismissing a surprising loss as "a flaw in the study" | Motivated reasoning on negative results | Apply identical scrutiny in both directions |
| Reading metrics despite a "small" SRM | 0.993 feels close to 1.0 | Compute the p-value; p < 0.001 → invalid; missing users are extreme users, so effects can fully reverse |
| "Not significant" read as "no effect" | Underpowered experiments | Power for the practical-significance bar; analyze triggered subset |
| p-value read as P(null is true) or false-positive rate | Common misconception | p = P(data this extreme \| null); Bayes with a prior is needed for P(null \| data) |
| Peeking / early stopping on significance | Dashboards update continuously; early tools encouraged it | 5–10x false-positive inflation; fixed duration or always-valid sequential p-values |
| Cherry-picking the smallest p-value across metrics/segments/reruns | Multiple comparisons | Pre-register primary metrics; FDR control; ~1 in 20 A/A reruns hits p < 0.05 by chance |
| "CIs overlap so it's not significant" | Wrong rule | CIs can overlap ~29% with a significant delta; test the delta directly |
| Redirect-based treatment implementation | Simple and elegant — and wrong | Causes latency asymmetry, bot divergence, bookmark contamination → SRM; go server-side or redirect both arms |
| Beacon/telemetry loss differing by variant | Web beacons are inherently lossy; treatment changes timing/placement | Telemetry-fidelity guardrail; dual logging for high-value clicks |
| Trigger attribute mutated by treatment | Attribute (e.g., dormant flag) evaluated at analysis time | Trigger on pre-assignment attribute snapshots |
| Restart without re-randomization | Desire not to flip visible experiences | Residual effects → SRM; run pre-experiment A/A and re-randomize |
| Shared resources across variants | LRU caches, machine crashes, memory leaks | Experiment ID in cache keys; cache-hit-rate guardrail; equal-size variants |
| Survivorship bias | Analyzing only long-active users | Include all assigned users (intention-to-treat) |
| Novelty/primacy misread as a stable effect | Constant-effect assumption | Plot effect over time; any trend → run longer or reject |
| Time-of-day sends confounding variants | Sequential sending (Control emails first, Treatment after) | Randomize send order; check open-time distributions |
| Bad hash function under concurrent experiments | Single-layer hash reused for overlapping layers | A/A tests across layers; cryptographic-quality hashing |
| Weak hardware "identical" fleets differing | Small hardware variation (Facebook service A/A failure) | A/A test across fleets before A/B |
| Segment migration misread (users of feature F vs complement both improve, aggregate flat/down) | Treatment moves users between segments | Judge on the aggregate metric; segment only by pre-experiment attributes |

## Rules of thumb

- SRM alarm: sample-ratio p-value < 0.001 → hide the scorecard; investigate before anything else.
- With large N, observed ratio outside [0.99, 1.01] on a 1:1 design ≈ certain bug.
- ~6% of experiments at Microsoft show SRMs — expect them routinely.
- A/A tests: ~5% of metrics significant at p < 0.05; p-value histogram uniform (Anderson-Darling/K-S to confirm); simulate ~1,000 by replaying a week of logs with new hash seeds.
- A/A p-value mass near 0.32 → single dominant outlier; skewed histogram → variance bug (analysis unit ≠ randomization unit → delta method/bootstrap).
- Peeking inflates false positives 5–10x.
- CIs can overlap up to ~29% and the delta is still significant at p < 0.05.
- Skewed metrics may need > 100,000 users for normal approximation — cap them.
- Bot traffic: > 50% of Bing US traffic, > 90% in China/Russia — bot filtering is a top SRM suspect.
- Prefer equal splits (e.g., 10%/10% at runtime) over unequal (10%/90%) when caches or shared resources are in play; you cannot fix this by discarding data post hoc.
- Always run continuous A/A tests alongside production experiments.

## Related skills

- `../analyzing-results/SKILL.md` — stats foundations, multiple testing, segments, Simpson's paradox
- `../designing-experiments/SKILL.md` — power, sample size, randomization unit
- `../variance-and-sensitivity/SKILL.md` — variance estimation, delta method, triggering
- `../ramping/SKILL.md` — staged rollout percentages (a common SRM source)
- `../interference/SKILL.md` — SUTVA violations and isolation
- `../long-term-effects/SKILL.md` — novelty/primacy and holdouts in depth
- `../metrics-and-oec/SKILL.md` — guardrail metric design
- `../instrumentation-and-clients/SKILL.md` — logging fidelity, client-side loss

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapters 3, 19, 21.
