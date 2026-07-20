---
name: variance-and-sensitivity
description: Use when computing confidence intervals or p-values for ratio/percent-delta metrics, when an experiment lacks power to detect the effect you expect, or when applying CUPED, outlier capping, or triggered analysis. Triggers on "variance is too high", "not enough power", "delta method", "ratio metric variance", "CUPED", "cap outliers", "triggering", "counterfactual logging", "dilute the effect".
---

# Variance Estimation and Improved Sensitivity

Variance drives every statistical output you care about — p-values, confidence intervals, power, and CIs. Get variance wrong and every conclusion is wrong: overestimated variance yields false negatives, underestimated variance yields false positives. This skill covers the two jobs that make an experiment trustworthy AND sensitive: computing variance correctly (especially for ratio and percent-delta metrics) and reducing variance to squeeze more power out of the same traffic (CUPED, capping, triggering).

## When to use

- You are reporting a **relative/percent change** (Δ%) and need its confidence interval.
- Your metric is a **ratio** (CTR = clicks/pageviews, revenue-per-click) where the analysis unit differs from the randomization unit.
- Your experiment **cannot detect** an effect you believe is real — you need more sensitivity without more traffic.
- You have **pre-experiment data** on the same users and want ~50% variance reduction via CUPED.
- **Outliers, bots, or spam** are inflating variance and killing your t-statistic.
- Only a **subset of users could have been affected** by the change and you want to filter out the rest (triggering).
- You need to **translate a triggered effect into an overall/site-wide effect** (dilution).

## Core principles

1. **The i.i.d. assumption breaks when analysis unit ≠ randomization unit.** The simple formula `var(Ȳ) = σ̂²/n` assumes samples are i.i.d. (or at least uncorrelated), which holds only when the analysis unit equals the experiment/randomization unit. User-level metrics (clicks-per-user) are fine when you randomize by user. But page-level or click-level metrics (CTR, revenue-per-click) put multiple correlated observations from the same user into the sample — "within-user correlation" — and the naive formula is biased. This is the single most common variance mistake.

2. **Use the delta method for ratio metrics.** Write the ratio as `M = X̄/Ȳ` (ratio of two user-level averages). Because X̄ and Ȳ are jointly bivariate normal in the limit, M is asymptotically normal, and:
   `var(M) = (1/Ȳ²)·var(X̄) + (X̄²/Ȳ⁴)·var(Ȳ) − 2·(X̄/Ȳ³)·cov(X̄, Ȳ)`
   This correctly accounts for the covariance term the naive approach drops.

3. **Percent delta (Δ%) has its own variance — don't divide var(Δ) by Ȳc².** Δ% = Δ/Ȳc is the standard business-friendly reporting unit (a "1% session increase" is legible; "0.01 more sessions" is not). The common mistake is `var(Δ)/Ȳc²`, which is WRONG because Ȳc is itself a random variable. Because Treatment and Control are independent:
   `var(Δ%) = (1/Ȳc²)·var(Ȳt) + (Ȳt²/Ȳc⁴)·var(Ȳc)`
   When Treatment and Control means differ meaningfully, this diverges substantially from the naive estimate.

4. **Percentiles and other non-mean statistics need density estimation or bootstrap.** Metrics like the 90th/95th percentile page-load-time cannot be written as a ratio of two user-level means. Use bootstrap (resample with replacement, estimate variance from many simulations) — powerful and broadly applicable but computationally expensive as data grows. Cheaper: if the statistic is asymptotically normal, its variance is a function of the density (estimate the density → estimate variance). Since most time-based metrics are event/page-level while randomization is user-level, combine density estimation WITH the delta method.

5. **Outliers hurt variance more than they hurt the mean — cap them.** A single positive outlier raises the Treatment mean a little but raises the standard deviation much more; simulation shows the t-statistic collapses from very significant to not significant as one outlier grows. Bots and spam (excessive clicks/pageviews) are the usual source. Cap observations at a reasonable threshold — e.g., a human is unlikely to search >500 times or have >1,000 pageviews in a day. Capping is critical *before* estimating variance.

6. **Reduce variance to gain sensitivity — many levers, pick by cost.** Lower-variance metric with similar information (searchers vs. searches; conversion Boolean vs. purchase amount — Kohavi 2009 cut required sample size by 3.3× using conversion rate instead of spend). Transform: cap, binarize, or log (Netflix uses "streamed > x hours?" Boolean instead of raw streaming hours; log helps heavy long tails when interpretability isn't required — but don't log revenue if the business goal is revenue). Triggered analysis (see below). Stratification / control-variates / CUPED. Randomize at a finer unit (per-page/per-query boosts sample size for page-load metrics). Paired designs (interleaving of ranked lists removes between-user variability). Pool controls across experiments into a shared, larger Control.

7. **CUPED exploits pre-experiment data for ~50% variance reduction.** CUPED (Controlled-experiment Using Pre-Experiment Data) is the online-experiment application of stratification/control-variates that emphasizes pre-period covariates. Stratification divides the sampling region into strata (platform, browser, day-of-week), samples within each, and recombines — usually lower variance. Post-stratification does this retrospectively during analysis (cheaper than stratified sampling at runtime; behaves like stratified sampling at large sample sizes, weaker when samples are small and variable). Control-variates uses the covariates as regression variables instead of building strata. Using the *same metric measured pre-experiment* as the covariate is what makes CUPED so effective.

8. **Finer randomization units cut variance but cost user-level measurement.** Randomizing per-page or per-query multiplies sample size and shrinks variance — but a sub-user unit means (a) the same user can get inconsistent UI (bad experience for noticeable UI changes) and (b) you lose the ability to measure any user-level-over-time effect like retention. Only go sub-user when the metric is genuinely event-level and no user-level story matters.

## Triggering

Triggering improves sensitivity by removing users who **could not have been affected** — their true Treatment effect is zero, so they only add noise. As experimentation maturity grows, more experiments are triggered.

**A user is triggered** if there is (potentially) some difference between the variant they were in and the counterfactual (any other variant). Always log triggering events at runtime so the triggered population is identifiable, and always run the analysis step for at least all triggered users.

**Triggering examples (increasing complexity):**
- *Intentional partial exposure*: change only ships to US users → analyze only US users. Include "mixed" users who could have seen the change, and count all their activity after exposure (even outside the US) because of residual effects. Any well-defined-on-pre-experiment-data segment works (Edge-browser users, a zip code, "heavy users who visited ≥3× last month").
- *Conditional exposure*: change to checkout → trigger only users who started checkout. Weather-answer change → only users whose query returned a weather answer. Collaboration/co-editing change → only users who collaborated.
- *Coverage increase*: free shipping threshold drops from $35 to $25 → only users with a cart between $25 and $35 are affected. Don't trigger users >$35 or <$25 (same behavior in both arms), and don't trigger the intersection who see the identical offer.
- *Coverage change* (not just increase): both Control and Treatment must evaluate the counterfactual "other" condition and mark a user triggered only if the two variants differ.
- *Counterfactual triggering for ML models*: a new classifier/recommender agrees with the old one for most users → zero effect for them. To know, generate the counterfactual: Control runs BOTH models, exposes the Control output, logs both; Treatment runs BOTH, exposes the Treatment output, logs both. Trigger where actual ≠ counterfactual. Cost: inference roughly doubles; latency differences between models won't show up in the experiment since both always execute.

**Optimal vs. conservative triggering.** Optimal = trigger only users for whom the two compared variants differed. With multiple Treatments, ideally log the actual + all counterfactuals (costly — every model must run). Conservative triggering (including MORE users than optimal) does NOT invalidate the analysis — it only loses some power. If the conservative set isn't much bigger than the ideal set, take the simplicity. Examples: log just a Boolean "did the variants differ" instead of every output; or after broken counterfactual logging, fall back to "user initiated checkout" (identifies more than the ideal set but still drops the 90% who never entered checkout).

**Dilution — translating triggered effect to overall effect.** Improving revenue 3% for 10% of users does NOT mean 10%×3% = 0.3% overall. The true overall impact is anywhere from 0% to 3%. Definitions: ω = overall universe, θ = triggered population; Δθ = MθT − MθC (absolute triggered effect); δθ = Δθ/MθC (relative triggered effect); triggering rate τ = NθC/NωC (percent of users triggered). Two equivalent correct formulas for diluted (site-wide) percent impact:
- `(Δθ · NθC) / (MωC · NωC)` — absolute effect over the total.
- `(Δθ / MωC) · τ` — triggered absolute effect relative to the *untriggered/overall* metric, times the triggering rate.

**The dilution pitfall:** using `(Δθ / MθC) · τ` (dividing by the *triggered* Control metric instead of the *overall* metric) is only correct when the triggered population is a random sample. When the triggered population is skewed (it usually is), this is off by a factor MωC/MθC. Example: a change to very low spenders (10% of average spend) improving their revenue 3% → 3% × 10% (share) × 10% (spend ratio) = 0.03%, negligible. Conversely, if the only path to revenue is checkout and you changed checkout, triggered and overall effects are equal — no dilution needed. Ratio metrics need more refined dilution formulas and can trigger Simpson's paradox (triggered ratio improves while diluted global ratio regresses).

**Trustworthy-triggering checks:**
1. **SRM on the triggered population.** If the overall experiment has no Sample Ratio Mismatch but the triggered analysis does, bias is being introduced — usually counterfactual triggering done wrong.
2. **Complement (never-triggered) analysis.** Generate a scorecard for never-triggered users; it should look like an A/A test. If more metrics are significant than expected by chance, the trigger condition is wrong — you influenced users you excluded.

## Workflow

1. **Identify your analysis unit vs. randomization unit.** If they match (user metric, user randomization) → simple `σ̂²/n`. If they differ (ratio metric) → delta method.
2. **For ratio metrics**, express as `M = X̄/Ȳ` of user-level averages and apply the delta-method variance (principle 2). Never use the naive per-observation formula.
3. **For percent delta**, use the Δ% variance formula (principle 3), NOT `var(Δ)/Ȳc²`.
4. **For percentiles / non-mean stats**, use bootstrap, or density estimation + delta method for event-level metrics.
5. **Cap outliers before estimating variance** at a defensible human-behavior threshold (e.g., ≤1,000 pageviews/day). Bots and spam first.
6. **If underpowered**, apply variance reduction in rough order of cost/benefit: pick a lower-variance metric → CUPED with a pre-experiment covariate (target ~50% reduction) → triggered analysis → finer randomization unit (only if no user-level metric needed) → pooled/shared control.
7. **If only a subset can be affected**, define the trigger condition on pre-experiment-safe data, log triggering events at runtime, and analyze at least all triggered users.
8. **Generate counterfactuals** when the difference isn't observable from exposure alone (ML models): run both variants, expose one, log both, trigger on actual ≠ counterfactual.
9. **Dilute the triggered effect** to a site-wide number with `(Δθ/MωC)·τ` (overall Control metric), never `(Δθ/MθC)·τ`.
10. **Run the two trust checks**: triggered-population SRM and never-triggered complement scorecard.

## Pitfalls

| Pitfall | Why it happens | Detect / avoid |
| --- | --- | --- |
| Naive variance on ratio metrics | Forgetting the i.i.d. assumption; multiple correlated observations per user | Use the delta method; check whether analysis unit = randomization unit |
| `var(Δ)/Ȳc²` for percent delta | Treating Ȳc as a constant when it's a random variable | Use the Δ% variance formula; gap grows as means diverge |
| Outliers inflating variance | Bots/spam produce extreme click/pageview counts; variance hit > mean hit | Cap at human-plausible threshold before variance estimation |
| Diluting by trigger rate wrong | Dividing by triggered Control metric (`Δθ/MθC·τ`) when population is skewed | Divide by overall metric (`Δθ/MωC·τ`); off by MωC/MθC otherwise |
| Assuming 10%×3% = 0.3% | Multiplying share by triggered effect naively | Overall effect ranges 0–3%; use proper dilution formula |
| SRM appears only in triggered analysis | Counterfactual triggering implemented incorrectly | Compare overall vs. triggered SRM; investigate the counterfactual logging |
| Never-triggered users show significant effects | Trigger condition too narrow — you affected excluded users | Run complement scorecard; it must look A/A |
| Tiny segment, huge relative win | Diluted value tiny when τ ≈ 0.001 even at 5% lift | Compute diluted value; only chase if the idea *generalizes* (MSN new-tab: 8.9% triggered → generalized to 5% on 12M users) |
| Triggered user not tracked forward | Only counting the triggering session/day | Once triggered, include the user for the rest of the experiment; a bad Treatment that suppresses visits is underestimated by per-day/per-session analysis |
| Counterfactual logging changes performance | Both arms run each other's (possibly slower) code, invisible in the experiment | Log per-model timing; run A/A'/B where A' = Control + counterfactual logging, alert if A ≠ A' |
| Shared controls + counterfactual logging | Shared controls run without code changes, so they can't log counterfactuals | Avoid pooling controls when counterfactual logging is required |
| Sub-user randomization loses retention | Per-page/per-query units can't measure user-over-time metrics; may show inconsistent UI | Only randomize sub-user for event-level metrics with no user-level story |

## Rules of thumb

- Sample size for a mean at 95% confidence / 80% power: `n ≈ 16·σ²/Δ²` per variant.
- CUPED with a good pre-experiment covariate: expect **~50% variance reduction**.
- Conversion (Boolean) instead of purchase amount cut required sample size by **3.3×** in a real case.
- Outlier cap examples: humans rarely search >500×/day or exceed >1,000 pageviews/day.
- Checkout-triggering example: purchase p=0.05 needs ~121,600 users; triggering on checkout (p≈0.5 among the 10% who start) needs only ~6,400 triggered users ≈ 64,000 total — roughly half the traffic, similar power in about half the time.
- Balanced Treatment/Control sizes converge to normality faster (relevant even when pooling a larger shared Control).
- Diluted percent impact = `(Δθ / MωC) · τ`, where τ = triggering rate. Never divide by the triggered Control metric for a skewed triggered population.

## Related skills

- `../analyzing-results/SKILL.md` — p-values, confidence intervals, multiple testing, Simpson's paradox
- `../trustworthiness/SKILL.md` — SRM, A/A tests, Twyman's law, trust guardrails
- `../designing-experiments/SKILL.md` — sample size, power, randomization unit choice
- `../metrics-and-oec/SKILL.md` — choosing lower-variance metrics, OEC, guardrails
- `../long-term-effects/SKILL.md` — dilution and survivorship over long horizons
- `../latency-and-performance/SKILL.md` — percentile latency metrics needing density/bootstrap variance

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapters 18 and 20.
