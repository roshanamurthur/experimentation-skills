---
name: analyzing-results
description: Analyze the results of an online controlled experiment (A/B test) — run the two-sample t-test, interpret p-values and confidence intervals correctly, separate statistical from practical significance, make the launch/no-launch decision, handle multiple testing, segment analysis, Simpson's paradox, and combine replications with Fisher's meta-analysis. Invoke when reading experiment scorecards, deciding whether a result is real, or deciding whether to ship.
---

# Analyzing Experiment Results

Covers the statistics of turning an experiment's data into a trustworthy launch/no-launch decision: the two-sample t-test, p-values and confidence intervals (and their common misreadings), statistical vs practical significance, normality checks, multiple testing, segment drill-downs, and Simpson's paradox. The single most important principle: a statistical result is not a decision — combine the measured effect and its uncertainty with a pre-declared practical-significance boundary, and only then decide.

## When to use

- An experiment finished (or is running) and you need to decide launch vs no-launch.
- Someone asks "is this result significant?" or "the p-value is 0.06, what do we do?"
- A metric moved but the confidence interval straddles the practical-significance boundary.
- Many metrics or segments are being scanned and some are "unexpectedly significant."
- A revenue-like (highly skewed) metric is being tested, especially with small samples.
- Segment-level results disagree with the aggregate, or both segments improve while the total declines.
- An experiment ran through a ramp-up (traffic percentage changed mid-flight) and you're combining periods.
- A surprising result needs replication, or several replications need to be combined into one conclusion.

## Core principles

1. **Test the difference of means with a two-sample t-test.** For metric Y with Treatment and Control samples, the null hypothesis is `mean(Yt) = mean(Yc)`. Compute the delta `Δ = Ȳt − Ȳc` (an unbiased estimator of the shift in means) and the t-statistic `T = Δ / sqrt(var(Δ))`, where independence of the samples gives `var(Δ) = var(Ȳt) + var(Ȳc)`. Larger |T| means stronger evidence against the null. Always use per-unit normalized metrics (e.g., revenue-per-user, not total revenue) since realized sample sizes differ by chance even under equal allocation.

2. **Choose the metric's denominator to include all potentially affected users and no one else.** For a change on the checkout page, revenue-per-user over *all site visitors* is valid but noisy — users who never start checkout cannot be affected and dilute the effect. Revenue-per-user over *purchasers only* is wrong: if the treatment changes the fraction who purchase, revenue-per-user can drop while total revenue rises. The right denominator is users who *start* the purchase process — everyone the change could touch, nobody it couldn't. Generalize: analyze the population at and below the funnel step where the change lives.

3. **The p-value is P(data this extreme | null is true) — never P(null is true | data).** Declaring significance at p < 0.05 means that when there is truly no effect, you correctly conclude "no effect" 95 times out of 100; p < 0.01 is very significant. By Bayes rule, `P(H0 | Δ) = [P(H0)/P(Δ)] · p-value` — converting a p-value into "probability the treatment did nothing" requires a prior on the null. A treatment that cannot possibly work (transmuting lead to gold) produces 100% false positives among its rejections regardless of p-value.

4. **Know the full list of p-value misinterpretations — each one leads to a wrong decision.**
   - "p = 0.05 means the null has a 5% chance of being true." Wrong — the p-value is computed assuming the null is true.
   - "p > 0.05 means there is no difference." Wrong — the data are consistent with zero *and* with every other value inside the confidence interval; zero is not privileged. The experiment may simply be underpowered (a review of 115 A/B tests at GoodUI.org found most were underpowered).
   - "p = 0.05 means this data would occur only 5% of the time under the null." Wrong — the p-value includes results *equal to or more extreme than* the observed one.
   - "p = 0.05 means a 5% false-positive rate if you reject." Wrong — the false-positive rate depends on the prior probability that the null is true.
   - The definition also silently assumes the data were collected as claimed (random assignment, no peeking, analysis not chosen post hoc). Picking the analysis that yields the smallest p-value, or reporting a p-value *because* it was small, invalidates it.

5. **A 95% confidence interval is Δ ± 1.96 standard errors, and it is the dual of the p-value.** The CI does not contain zero ⟺ p < 0.05. Two more traps: (a) overlapping *per-variant* CIs do NOT imply non-significance — the Control and Treatment CIs can overlap by as much as 29% while the delta is still significant (non-overlap does imply significance); always test the CI of the *delta*. (b) A specific computed interval either contains the true effect or it doesn't — the 95% refers to how often intervals from repeated studies would contain it, not a 95% chance for this one.

6. **Statistical significance is not practical significance — set the practical boundary before the experiment, from business context.** Practical significance (minimum effect worth acting on) depends on scale and cost: for Google/Bing-sized businesses a 0.2% revenue change is practically significant; a startup hunting for 10%+ improvements may ignore 2%. Set the boundary higher when launch/maintenance cost is high (a painted-door test is cheap, the full feature is not); set it near zero when the change is already fully built and costless to launch; lower both bars when the decision is short-lived (a headline that runs for a few days). Also weigh metric tradeoffs (engagement up but revenue or CPU cost down) and the asymmetric cost of wrong decisions.

7. **Control Type I and Type II errors via power, and remember power is relative to effect size.** Type I error: declaring a difference when there is none (capped at 5% by the p < 0.05 rule). Type II error: missing a real difference; power = 1 − Type II error, industry standard ≥ 80%. Sample size for 80% power at significance 0.05: `n ≈ 16σ²/δ²` per variant, where δ is the smallest practically significant delta (the minimum detectable effect) — you don't know the true δ, so power for the smallest δ you'd care to detect. An experiment powered to detect 10% is not powered to detect 1%. For small samples, also consider Type S (sign) errors — probability the estimate has the wrong direction — and Type M (magnitude) errors — the exaggeration ratio of the effect size.

8. **The normality assumption is about the mean, not the metric — check it for skewed metrics.** The Central Limit Theorem makes Ȳ approximately normal even when Y is far from normal, but skewed metrics need more samples. Rule of thumb: at least `355·s²` samples per variant, where `s = E[(Y − E[Y])³] / [Var(Y)]^(3/2)` is the skewness coefficient — useful when |s| > 1. Reduce skewness by capping or transforming: Bing capped Revenue/User at $10 per user per week, dropping skewness from 18 to 5 and required samples tenfold, from 114k to 10k. The t-test on the *difference* needs fewer samples than either arm alone, especially with equal traffic allocation (the difference's null distribution is symmetric with zero skewness). If in doubt, validate once offline: shuffle labels across Treatment/Control to simulate the null distribution and compare to normal (Kolmogorov–Smirnov, Anderson–Darling), focusing on whether the Type I error rate stays under 0.05. If normality fails, use a permutation test — expensive at scale, but the cases that need it are small-sample anyway.

9. **Multiple testing inflates false positives — tier your significance levels.** With 100 independent metrics at α = 0.05, expect ~5 falsely significant metrics even for a do-nothing feature; the problem compounds across experiments, iterations, segments, and time (peeking). Bonferroni (0.05 / number of tests) is simple but too conservative; Benjamini–Hochberg controls the false discovery rate but is more complex. Practical rule: split metrics into (1) first-order — expected to be impacted, (2) second-order — possibly impacted (e.g., cannibalization), (3) third-order — unlikely to be impacted; apply tiered thresholds 0.05 / 0.01 / 0.001. The Bayesian rationale: the stronger your prior that the null is true, the lower the significance level you should demand.

10. **Segment the treatment effect, but treat surprising segments as leads to investigate, not conclusions.** Good segments: market/country (bad localization shows up here), device/platform and browser version (JavaScript incompatibilities), time of day / day of week, new vs existing users, account type. Segmenting the treatment effect (heterogeneous treatment effects / Conditional Average Treatment Effects) surfaces real bugs — e.g., a UI change slightly positive everywhere but strongly negative on Internet Explorer 7 because the JavaScript errored and blocked clicks. Segment differences in a raw metric can also be pure instrumentation artifacts: Bing mobile ad CTR differed wildly by OS because iOS/Windows Phone used redirect-based click tracking (high fidelity, slower) while Android used lossy web beacons — plus a Windows Phone bug counting swipes as clicks. Correct for multiple testing when scanning segments, and invoke Twyman's law on any striking segment result.

11. **Never trust segment-level movements when the treatment can move users between segments.** If a treatment changes who belongs to a segment, per-segment deltas mislead: with F-users averaging 20 sessions-per-user and non-users 10, a treatment that makes 15-session users stop using F raises the average in *both* segments while the aggregate can go up, down, or flat. When migration is possible, decide from the aggregate (non-segmented) treatment effect, and prefer segmenting only on attributes fixed before the experiment started.

12. **Simpson's paradox: combining periods or strata with different allocation percentages can reverse the sign.** During ramp-up (e.g., 1% treatment on Friday, 50% on Saturday), Treatment can beat Control in each period yet lose in the combined data — with 1M visitors/day, Treatment converting 2.30% vs 2.02% on Friday and 1.20% vs 1.00% on Saturday still shows 1.20% vs 1.68% combined, because the weighted average is dominated by the period with more treatment users. Same trap arises from non-uniform browser sampling, per-country allocation differences (US at 1%, Canada at 50%), carving out top-1% spenders at a different allocation, and staged data-center rollouts. Analyze periods/strata with different allocations separately or reweight — never naively pool. (Note this is distinct from segment-migration above, which involves users moving between segments.)

13. **Sensitivity comes from smaller standard errors — but growth in power is sub-linear over time.** Detecting smaller effects requires lower standard error of the mean, achieved by allocating more traffic or running longer as users accumulate. Running longer loses steam after the first couple of weeks: unique-user growth is sub-linear (day-one users return, so two days yields fewer than 2N users), and some accumulating metrics (e.g., sessions-per-user) have variance that grows over time. A binary purchase indicator has lower variance than revenue-per-user, so switching the OEC to it buys sensitivity at the cost of losing purchase-amount information. Overpowering is fine and recommended — you may have power to detect a revenue change over all users but not for the Canada-only segment you'll inevitably be asked about.

14. **Combine replications with Fisher's meta-analysis.** Replicating a surprising experiment (using orthogonal re-randomization or previously unallocated users) yields independent p-values; combine them with `X²(2k) = −2·Σ ln(pᵢ)`, which follows a chi-squared distribution with 2k degrees of freedom under the null. Two p-values each under 0.05 are much stronger evidence than one. Use Brown's extension when p-values are dependent. This is also the escape hatch for underpowered experiments: after exhausting max-power allocation and variance reduction, run two or more sequential orthogonal replications and combine.

## Workflow

1. **Confirm the analysis plan was fixed in advance**: OEC and key metrics, practical-significance boundary δ, significance threshold, and experiment duration were set before starting. If the duration was not predetermined and p-values were monitored continuously (peeking), significance claims are biased 5–10x — either use always-valid sequential p-values or restart the clock with a fixed duration.
2. **Run sanity checks before reading any results**: trust guardrails (sample ratio matches design, equal cache-hit rates) and organizational guardrails (latency and other metrics that should be invariant for this change). If they fail, debug the experiment — do not interpret metrics. (See the trustworthiness skill for SRM debugging.)
3. **Check the normality assumption** for skewed metrics: need ≥ 355·s² samples per variant. If short, cap/transform the metric or use a permutation test.
4. **Compute the t-test per metric**: Δ = Ȳt − Ȳc, var(Δ) = var(Ȳt) + var(Ȳc), T = Δ/√var(Δ); report p-value and the 95% CI `[Δ − 1.96·SE, Δ + 1.96·SE]` on the delta (report percent delta too — it is also approximately normal).
5. **Apply multiple-testing discipline**: tier metrics first/second/third-order with thresholds 0.05 / 0.01 / 0.001. Treat any "irrelevant metric went significant" through this lens before building a story.
6. **Place the result on the decision matrix** (CI of the delta vs the practical-significance boundary — see below) and make the launch/no-launch call.
7. **Drill into segments** (country, platform, browser, new/existing, day-of-week) for heterogeneous treatment effects; investigate outlier segments (Twyman's law), correct for the extra tests, and ignore per-segment deltas for any segment the treatment itself can move users into or out of.
8. **Check for Simpson's paradox** if allocation percentages varied across time, country, or stratum: analyze periods separately; never pool raw counts across different allocation ratios.
9. **Plot the treatment effect over time.** Standard analysis assumes a constant effect; a rising or falling trend flags novelty/primacy effects — run longer until it stabilizes (see long-term-effects skill).
10. **If the result is surprising or underpowered, replicate** on orthogonal randomization and combine p-values with Fisher's method rather than re-running until something is significant.
11. **Record the decision and its context** — the tradeoffs, thresholds, and reasoning — so future decisions have a basis, not just a local judgment call.

### Worked example: the coupon-code painted-door test

A commerce site tests whether merely adding a coupon-code field to checkout hurts revenue (painted door: the field is fake, every code returns "Invalid Coupon Code"). Two treatment UIs run against control at 34/33/33% for a full week (minimum four days for power; a week to capture day-of-week effects). Hypothesis: "Adding a coupon code field to the checkout page will degrade revenue-per-user for users who start the purchase process." Practical-significance boundary: 1% of revenue-per-user. After guardrails pass, results:

| Comparison | Treatment | Control | Difference | p-value | CI |
|---|---|---|---|---|---|
| Treatment 1 vs Control | $3.12 | $3.21 | −$0.09 (−2.8%) | 0.0003 | [−4.3%, −1.3%] |
| Treatment 2 vs Control | $2.96 | $3.21 | −$0.25 (−7.8%) | 1.5e-23 | [−9.3%, −6.3%] |

Both p-values are under 0.05 and both CIs sit entirely beyond the −1% practical boundary: statistically and practically significant revenue loss, driven by fewer users completing purchase. Decision: scrap the coupon-code business model — any coupon email campaign would have to recoup the coupon system's build cost *plus* this measured drag on all users. A cheap painted door killed an expensive idea; that is the analysis working as intended.

### The launch decision matrix

Compare the delta's confidence interval to the practical-significance boundary (dashed lines at ±δ_practical):

| Case | Statistically significant? | vs practical boundary | Decision |
|---|---|---|---|
| 1 | No | CI well inside the boundary | Change does nothing meaningful — iterate or abandon. |
| 2 | Yes | Effect beyond the boundary | Launch. |
| 3 | Yes | Effect confidently *below* the boundary | You're confident the magnitude is too small to outweigh costs — don't launch. |
| 4 | No | CI *extends beyond* the boundary on both sides | You cannot call this neutral — a ±10% revenue swing is possible. Underpowered: run a follow-up with more units. No launch decision can be made from this data. |
| 5 | No | Point estimate practically significant, CI crosses zero | Best guess is a meaningful effect, but no-effect is plausible. Repeat with more power. |
| 6 | Yes | Point estimate beyond boundary, CI crosses the boundary | Likely practically significant but not certain. Repeating with more power is best; launching is a reasonable call. |

**When the CI overlaps the practical-significance boundary (cases 4–6): the default answer is "collect more data," not "call it neutral."** A wide CI spanning the boundary in both directions (case 4) means the experiment could not distinguish "helps a lot" from "hurts a lot" — calling it neutral is the worst reading. Rerun with more units for greater power. If forced to decide anyway, be explicit about which factors (cost, downside asymmetry, lifespan of the change) you are trading off and how they set your thresholds — that record becomes the basis for future decisions, not just a local judgment call.

On peeking: continuously monitoring p-values and stopping at significance (as early Optimizely encouraged) biases significance claims by 5–10x. Two valid alternatives: (1) sequential tests with always-valid p-values or a Bayesian testing framework — the route Optimizely later implemented; (2) a predetermined experiment duration, such as one week, for declaring significance — the route used by the platforms at Google, LinkedIn, and Microsoft. Related cautionary tale: the surgeon who tried every analysis in the statistics software's drop-down menus and reported the one with the smallest p-value — that is multiple testing wearing a lab coat.

## Pitfalls

| Pitfall | Why it happens | Detect / avoid |
|---|---|---|
| Reading p-value as P(null is true) | Intuitive but backwards | It's P(data ≥ this extreme \| null). Posterior needs a prior (Bayes rule). |
| "Not significant" read as "no effect" | Ignoring power | CI shows the range of consistent effects; check power for δ_practical; most public A/B tests are underpowered. |
| Treating power as absolute | Forgetting power is relative to δ | Powered for 10% ≠ powered for 1%. State the δ every power claim refers to. |
| Peeking at p-values until significant | Continuous monitoring with fixed-horizon stats | Biases false positives 5–10x. Fixed duration, or sequential always-valid p-values / Bayesian framework. |
| Picking the test/analysis with the smallest p-value | Garden of forking paths | Pre-register the analysis; a p-value chosen for its size is invalid. |
| Comparing per-variant CIs for overlap | Feels equivalent to testing the delta | Per-variant CIs can overlap up to 29% with a significant delta. Test the delta's CI only. |
| "This specific CI has a 95% chance of containing the truth" | Confusing coverage with a single realization | 95% is the long-run coverage across repeated studies. |
| Sum-of-revenue as the metric | Ignores chance imbalance in sample sizes | Normalize per unit: revenue-per-user, sessions-per-user. |
| t-test on a heavily skewed metric with too few samples | Assuming CLT has kicked in | Require ≥ 355·s² per variant; cap values (Bing: $10/user/week cap cut required n from 114k to 10k); permutation test otherwise. |
| Declaring victory on 1 of 100 metrics | Multiple testing | ~5 of 100 will be false positives at α = 0.05. Tiered thresholds 0.05/0.01/0.001 by prior plausibility, or Benjamini–Hochberg. |
| Story-building on a striking segment result | Segments multiply tests; instrumentation differs by segment | Twyman's law: investigate first (e.g., per-OS click-tracking fidelity, IE7 JavaScript bug). Correct for multiple testing. |
| Celebrating when every segment improves | Treatment migrated users across segments | Sessions-per-user can rise in a segment and its complement while the aggregate falls. Use the aggregate; segment only on pre-experiment attributes. |
| Pooling data across ramp phases | Different allocation ratios weight periods unequally | Simpson's paradox: better in every period, worse combined. Analyze phases separately or use only the stable-allocation period. |
| Diluted analysis of a low-trigger feature | Including unaffected users | A large effect on a small subset vanishes in the aggregate — analyze the affected population (users who start checkout, not all visitors), and only unaffected users are excluded. |
| Re-running an experiment until p < 0.05 | Iteration as multiple testing | An A/A run 20 times will likely produce one p < 0.05. Replicate deliberately and combine with Fisher's method instead. |
| Shopping analyses for the smallest p-value | Forking paths / drop-down-menu statistics | Pre-register one analysis; a p-value selected for its size is not a p-value. |
| Calling a wide, boundary-spanning CI "neutral" | Absence of significance mistaken for evidence of no effect | If the CI admits ±10% revenue swings, you lack power — rerun bigger; make no launch decision from that data. |

## Rules of thumb

- Statistical significance default: p < 0.05; p < 0.01 is very significant.
- 95% CI ≈ observed delta ± 1.96 standard errors (≈ 2 SE); CI excludes zero ⟺ p < 0.05.
- Target power ≥ 80% (design for 80–90%); sample size `n ≈ 16σ²/δ²` per variant at 80% power, δ = smallest practically significant effect.
- Normality of the mean: ≥ 355·s² samples per variant (s = skewness coefficient); rule is informative for |s| > 1.
- Capping tames skew: Bing's $10/user/week revenue cap → skewness 18 → 5, required samples 114k → 10k.
- 100 metrics at α = 0.05 → ~5 false positives expected. Tiered α by expectation of impact: 0.05 (first-order) / 0.01 (second-order) / 0.001 (third-order).
- Per-variant CIs overlapping ≤ 29% can still mean a significant delta; non-overlap always means significant.
- Peeking at p-values inflates false-positive claims 5–10x.
- Practical significance scales with the business: ~0.2% matters at Google/Bing scale; startups often need 10%+.
- Fisher's combined test: `X²(2k) = −2·Σ ln(pᵢ)` ~ chi-squared with 2k df under the null.
- Run at least one full week to capture day-of-week effects; overpowering is fine and recommended (segment analysis needs it).
- CI overlapping the practical boundary → follow-up experiment with more power is the default; launching on case 6 (significant, point estimate beyond boundary) is reasonable.
- Denominator rule: include everyone the change could affect (users who start checkout), exclude everyone it couldn't (visitors who never reach it), and never condition on an outcome the treatment can move (purchasers).
- Unique-user growth over time is sub-linear (day 2 brings fewer than 2N users) — running much past two weeks buys little extra power for user-randomized experiments.
- Small-sample settings: report Type S (wrong-sign probability) and Type M (exaggeration ratio) alongside power.
- Peeking fix: sequential always-valid p-values (Optimizely's eventual approach) or a fixed predetermined duration (Google, LinkedIn, Microsoft).

## Related skills

- [designing-experiments](../designing-experiments/SKILL.md) — power analysis, sizing, and duration decisions made *before* the data arrives
- [metrics-and-oec](../metrics-and-oec/SKILL.md) — choosing the OEC and guardrail metrics being analyzed
- [trustworthiness](../trustworthiness/SKILL.md) — SRM and sanity checks that must pass before any of this analysis is valid
- [variance-and-sensitivity](../variance-and-sensitivity/SKILL.md) — CUPED and triggering to shrink var(Δ) and boost power
- [ramping](../ramping/SKILL.md) — the ramp phases whose changing allocations create Simpson's paradox
- [long-term-effects](../long-term-effects/SKILL.md) — novelty/primacy effects flagged by treatment-effect-over-time plots
- [culture-ethics-memory](../culture-ethics-memory/SKILL.md) — institutional memory and meta-analysis across historical experiments

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapters 2, 3, 17.
