---
name: experimentation-principles
description: Apply the core principles of trustworthy A/B testing when designing, running, or judging any online controlled experiment. Invoke as the starting point whenever someone says "let's run an experiment", "should we A/B test this", "the test won, ship it", or asks how to measure whether a change worked. The deep-dive skills in this repo cover each area; this is the one-page operating doctrine.
---

# Experimentation Principles

The condensed doctrine of *Trustworthy Online Controlled Experiments*. Getting numbers is easy; getting numbers you can trust is hard. A controlled experiment is the gold standard for causality, but only when you actively defend its trustworthiness — the default state of an experiment result is "probably misleading in some way you haven't checked yet."

## When to use

- Someone proposes running an experiment and you need the design conversation to start right.
- A result is in and a ship/no-ship decision is about to be made.
- You're reviewing someone else's experiment plan or readout.
- You need the quick version before diving into a specialized skill in this repo.

## The principles

### 1. You are bad at guessing what works — that's why you experiment

At Google and Bing, only **10–20% of experiments produce positive results**; at Microsoft overall, roughly a third are positive, a third flat, a third negative. Ideas that experts loved have failed, and throwaway changes have been worth fortunes (Bing's ad-title change: ~$100M/year, initially deprioritized for six months). Never let confidence in an idea substitute for measurement, and never punish "failed" experiments — most of them are the system working.

### 2. Establish causality with randomization, nothing less

Correlational evidence ("users who use feature X churn less") is confounded by who chooses to do X. Only random assignment of users to variants makes Treatment and Control comparable in expectation on everything — observed and unobserved. If you can't randomize, you're in observational-causal territory with much weaker guarantees (`../when-you-cant-ab-test/SKILL.md`).

### 3. Decide the success criterion before you start (the OEC)

Agree in advance on an **Overall Evaluation Criterion**: few metrics (≤5 key ones), measurable within the experiment window, believed to *causally* drive long-term value. Watch for metrics that move "up" while the product gets worse — Bing's ranker bug raised queries/user +10% and revenue/user +30% by returning *worse* results. Sessions/user (satisfied users return) beats raw engagement counts. Every metric will be gamed under optimization pressure (Goodhart's law) — constrain it. Details: `../metrics-and-oec/SKILL.md`.

### 4. State a falsifiable hypothesis and size the test for it

Write down what you expect to move and by how much before launching. Define **practical significance** (the smallest effect worth shipping) and power the experiment to detect it: `n ≈ 16σ²/δ²` per variant for 80% power at p < 0.05. Online effects that matter are tiny — a 0.5% revenue move can be worth $10M+ — so most useful experiments need hundreds of thousands of users. Underpowered experiments produce "no significant difference" that means nothing. Details: `../designing-experiments/SKILL.md`.

### 5. Twyman's law: any figure that looks interesting is usually wrong

Surprisingly good results get celebrated and surprisingly bad ones get dismissed; investigate both identically. Extreme results are far more often instrumentation bugs, data loss, bot filtering, or computation errors than real effects. If a result would be a big deal if true, assume it's a bug until you've ruled that out — and **replicate** big wins before believing them.

### 6. Check trust guardrails before reading any metric

Run the checks in order and stop at the first failure:
- **Sample Ratio Mismatch**: if the observed user split deviates from design (p < 0.001), the result is invalid no matter how small the imbalance looks — 821,588 vs 815,482 on a 50/50 design is p ≈ 1.8e-6. ~6% of experiments at Microsoft have SRMs.
- **A/A tests**: run them continuously; each metric should be significant ~5% of the time with uniform p-values. A failing A/A means your platform, variance estimation, or assignment is broken.
- Telemetry fidelity, cache sharing, cookie writes, bot filtering.

Details: `../trustworthiness/SKILL.md`.

### 7. Interpret the statistics correctly

The p-value is P(data this extreme | no effect) — not the probability the treatment works. Peeking daily and stopping at significance inflates false positives 5–10×: fix duration in advance or use sequential methods. Testing many metrics/segments multiplies false positives (23% chance of one "significant" result among 5 independent metrics at α=0.05). Statistical significance ≠ practical significance: a significant +0.01% may not pay for its complexity, and a non-significant result with a wide CI is "underpowered," not "no effect." Details: `../analyzing-results/SKILL.md`.

### 8. Run at maximum power for a fixed cycle — usually one week at 50/50

Equal splits maximize power (variance ∝ 1/(q(1−q))). Run whole weeks to capture day-of-week effects; don't stop early on a good day. Ramp in stages before that for safety, not measurement (`../ramping/SKILL.md`), and plot the treatment effect over time — a trend signals novelty or primacy effects, meaning the short-term number won't hold long-term (`../long-term-effects/SKILL.md`).

### 9. Make sure the variants don't contaminate each other

The t-test assumes one user's experience doesn't depend on another's assignment (SUTVA). It breaks in social networks, marketplaces with shared inventory or budgets, shared caches and infrastructure. If units interact, standard analysis is biased — use isolation, geo/cluster randomization, or switchbacks. Details: `../interference/SKILL.md`.

### 10. Squeeze sensitivity from the traffic you have

Before demanding more users: analyze only **triggered** users (those who could have seen the change) and dilute the effect back to overall; apply **CUPED** with pre-experiment data (~50% variance reduction on key metrics); cap outliers (Bing capped revenue/user at $10/user/week); use the delta method when the analysis unit differs from the randomization unit. Details: `../variance-and-sensitivity/SKILL.md`.

### 11. Instrument first — no telemetry, no experiment

Instrumentation is part of the feature, not an afterthought; a feature with broken logging is a broken feature. Client and server logs disagree (loss, clock skew — never subtract a client time from a server time); know which you're reading. Speed itself is a feature: +100ms of latency cost Amazon ~1% of sales and Bing ~0.6% of revenue — guard performance in every experiment. Details: `../instrumentation-and-clients/SKILL.md`, `../latency-and-performance/SKILL.md`.

### 12. Experiment ethically and remember what you learned

Ask: is this a minimal-risk change any user would reasonably expect (UI variants), or does it alter something consequential without consent? Weigh risk vs. benefit; escalate anything touching emotions, money, health, or deception. Record every experiment — hypothesis, scorecard, decision, rationale — because the meta-analysis of hundreds of past experiments (which categories pay off, which metrics are sensitive) is often worth more than any single result. Details: `../culture-ethics-memory/SKILL.md`.

## Quick workflow (design → decision)

1. Write the hypothesis, the OEC, guardrails, and the practical-significance threshold.
2. Choose the randomization unit (default: user) and compute required sample size and duration (whole weeks).
3. Instrument, add counterfactual logging if the change is triggered, and pre-register the analysis.
4. Ramp for safety → hold at 50/50 (or max feasible) for the full measurement window.
5. Check SRM and trust guardrails. Fail → debug, don't read metrics.
6. Analyze: t-test, CIs, triggered analysis + dilution, segments (with multiple-testing skepticism), effect-over-time plot.
7. Decide with both significances: statistically + practically significant → ship; neither → don't; ambiguous CI → add power or judgment call, and log the rationale.
8. Surprising result → apply Twyman's law and replicate before shipping.
9. Record the experiment and its decision in the institutional-memory system.

## Rules of thumb

- ~10–33% of experiment ideas win; design your process for mostly-negative results.
- `n ≈ 16σ²/δ²` users per variant (80% power, α = 0.05).
- Run whole weeks; one week at 50/50 is the standard measurement phase.
- SRM check at p < 0.001 gates everything; user ratios outside [0.99, 1.01] on a 50/50 design are almost certainly bugs.
- Metrics should be significant ~5% of the time in A/A tests — much more or less means broken stats.
- Peeking inflates false positives 5–10×; five metrics at α=0.05 give a 23% familywise false-positive rate.
- +100ms latency ≈ −1% Amazon sales / −0.6% Bing revenue: always guard speed.
- CUPED ≈ 50% variance reduction; triggering can shrink required samples by an order of magnitude.
- Cap ≤5 key metrics; keep hundreds of debug metrics.
- Replicate any result that would change your roadmap.

## Related skills

Every deep-dive in this repo: `../designing-experiments/SKILL.md`, `../metrics-and-oec/SKILL.md`, `../analyzing-results/SKILL.md`, `../trustworthiness/SKILL.md`, `../variance-and-sensitivity/SKILL.md`, `../ramping/SKILL.md`, `../interference/SKILL.md`, `../long-term-effects/SKILL.md`, `../latency-and-performance/SKILL.md`, `../when-you-cant-ab-test/SKILL.md`, `../experimentation-platform/SKILL.md`, `../instrumentation-and-clients/SKILL.md`, `../culture-ethics-memory/SKILL.md`.

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), synthesizing all chapters.
