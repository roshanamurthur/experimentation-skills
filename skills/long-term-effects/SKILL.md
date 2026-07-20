---
name: long-term-effects
description: Use when a short-term A/B result may not predict long-term value — novelty/primacy spikes, user learning, adaptation, network effects, ads blindness, or metrics that accumulate over months. Triggers on "novelty effect", "primacy effect", "will this last", "long-term impact", "holdout", "holdback", "reverse experiment", "post-period analysis", "time-staggered treatment", "learning effect", "cookie churn".
---

# Measuring Long-Term Treatment Effects

Most experiments run 1–2 weeks and the short-term effect is stable and generalizes — but not always. The long-term effect is the *asymptotic* effect of a Treatment (practically, 3+ months, or defined by number of exposures). When short- and long-term diverge, shipping on the short-term number is a trap: raising prices lifts short-term revenue but bleeds long-term revenue; showing more low-quality ads lifts short-term clicks but erodes long-term clicks and searches. This skill covers *why* they diverge and the measurement methods (long-running holdouts, post-period difference-in-difference, time-staggered treatments) plus their failure modes.

## When to use

- A Treatment shows a strong early win and you suspect **novelty** (users try it because it's new) or **primacy/change aversion** (users resist because they're primed on the old experience).
- The metric that truly matters **accumulates slowly**: retention, renewal on annual contracts, offline arrival (Airbnb/Booking book now, travel months later).
- The change touches a **network or marketplace** where effects propagate or supply lags demand (Live Video adoption, LinkedIn PYMK, two-sided markets).
- You need to **attribute** long-term financial impact to an experiment for forecasting or team goals.
- You want **institutional learning**: how big is the short-vs-long gap and what causes it?
- There is **launch pressure** and you need a cheap way to keep measuring after rollout (holdback/reverse experiment).
- Someone proposes reading the **last week of a long-running experiment** as "the long-term effect" — check the caveats first.

## Core principles

1. **Long-term ≠ short-term for identifiable reasons — diagnose which apply.** The divergence isn't random; it comes from specific mechanisms (below). If none apply, the short-term effect is a fine proxy and you should ship on it. Measuring long-term is expensive; only do it when a real mechanism threatens the extrapolation.

2. **User-learned effects: behavior changes as users adapt.** Users learn and reach an equilibrium. A single crash won't churn a user, but frequent crashes teach them to leave. Users adjust their ad-click rate once they notice ads are low quality. Discoverability cuts both ways — a useful feature may take time to find (slow uptake, then heavy engagement), and users primed on the old UI need time to adapt. This is the engine behind novelty (initial over-exploration that fades) and primacy (initial resistance that fades).

3. **Network and marketplace effects take time to propagate and can settle LOWER.** Features spread through networks (you use Live Video after friends do), so full effect lags. In two-sided marketplaces (Airbnb, eBay, Uber) a feature can spike demand but supply lags, delaying the revenue impact. Recommendation/connection systems (LinkedIn PYMK) often start high on novelty/diversity and reach a *lower* long-term equilibrium as finite supply ("people you know") is exhausted. See `../interference/SKILL.md` for leakage between variants.

4. **Delayed experience/measurement decouples the online action from the outcome.** There can be a long gap between the online experience and the outcome that matters — months between booking and arrival, a full year before an annual contract's renewal decision. Retention and renewal metrics are driven by that delayed offline experience, which a 2-week window can't see.

5. **The ecosystem shifts underneath a long experiment.** Other teams launch interacting features (more products embedding Live Video makes it more valuable; a first push-notification win erodes as every team starts notifying). Seasonality (gift cards at Christmas), competitors copying the feature, government policy (GDPR changing ad-targeting data), ML concept drift on stale models, and software rot all move the baseline. These are *exogenous* — isolate them when your goal is a generalizable/time-extrapolated effect.

6. **Know your purpose — it dictates the method.** Attribution (what does the world look like long-term with vs. without this feature now?) is hard because it mixes endogenous learning with exogenous shifts, and compounding launches build on each other. Institutional learning (how big is the gap, and why?) — a big novelty effect signals a suboptimal experience (slow discovery → add in-product education; try-once-then-abandon → likely clickbait/low quality). Generalization (measure long-term on some experiments to predict it for others, or build a short-term metric predictive of long-term) requires isolating from exogenous shocks unlikely to repeat.

7. **The last week of a long-running experiment is NOT a clean long-term effect.** Measure percent-delta in the first week (pΔ₁ = short-term) and last week (pΔT = "long-term") — but pΔT is contaminated by dilution (multi-device/multi-entry-point users increasingly experience both arms; the longer it runs the worse it gets), cookie churn (a Treatment cookie gets re-randomized into Control), and network leakage cascading over time. Survivorship bias: users who dislike Treatment abandon, so pΔT reflects only survivors (this should also fire an SRM alert). Interacting feature launches erode the win. And you cannot attribute pΔT − pΔ₁ to the Treatment alone — seasonality and population change can produce the whole gap.

## Measurement methods

**Method 1 — Cohort analysis.** Fix a stable cohort of users *before* the experiment (e.g., by logged-in user ID) and analyze only them short- and long-term. Effective against dilution and survivorship *if* the cohort is stable — fails when the ID is a high-churn cookie. Watch external validity: a logged-in-only cohort may not generalize to all users; correct with stratification-based weighting (stratify by pre-experiment engagement, weight by population distribution) — with the same limitations as observational studies.

**Method 2 — Post-period (difference-in-difference) analysis.** After running Treatment for time T, turn it off (or ramp Treatment up to 100%) so *both* groups get the identical experience, then measure the T-group vs. C-group difference in the post-period (T to T+1). This isolates the **learning effect**: what users/systems carried over. Two kinds:
- *User-learned effect*: users adapted (e.g., ad-load changing click behavior — Hohnhold et al. 2015).
- *System-learned effect*: the system "remembers" — profile updates persist, email opt-outs persist, ML personalization keeps showing more ads to users who learned to click more.
Extrapolation from short-term to long-term is valid when the system-learned effect is zero (a true A/A post-period where both arms see identical features). It's non-zero for permanent state changes (persistent personalization, opt-outs, unsubscribes, impression caps). Strength: isolates exogenous/seasonality/interaction effects and explains *why* short ≠ long. Still suffers dilution and survivorship — combine with cohort analysis or adjust.

**Method 3 — Time-staggered treatments.** To answer "how long is long enough?" without guessing off a noisy trend line (Treatment effect is rarely stable — day-of-week and big events swamp the long-term trend). Run two copies of the same Treatment with staggered starts: T₀ starts at t=0, T₁ at t=1. At any t>1, T₀ and T₁ form an A/A test differing only in exposure duration. Two-sample t-test on T₁(t) − T₀(t); when the difference is small, the treatments have converged and you can apply the post-period method from there. Prefer a lower Type II error rate than the usual 20% even at the cost of Type I > 5%. Assumes T₁(t) − T₀(t) shrinks over time, and needs enough gap between starts — if learning is slow and the starts are too close, there's no detectable difference at T₁'s start.

**Method 4 — Holdback and reverse experiment.** When there's launch pressure and a long Control is too expensive (opportunity cost). *Holdback*: launch Treatment to 90%, keep 10% in Control for weeks/months. It's a long-running experiment with a small Control → less power, so confirm the reduced sensitivity still lets you learn what you need. *Reverse experiment*: launch Treatment to 100%, then ramp 10% *back* into Control after weeks/months. Benefit: everyone has had Treatment for a while, so the network/marketplace has reached the new equilibrium before you measure — ideal when adoption depends on network effects or constrained supply. Downside: for a visibly different feature, pulling users back to Control confuses them.

## Workflow

1. **Decide if long-term even differs.** Walk principles 2–5: user learning, network/marketplace, delayed experience, ecosystem shift. If none plausibly apply, ship on the short-term effect.
2. **State your purpose** — attribution, institutional learning, or generalization — because it picks the method and what to isolate.
3. **If keeping the experiment running long**, measure pΔ₁ (first week) and pΔT (last week) but treat pΔT skeptically: check for dilution, cookie churn, survivorship (run an SRM check), interacting launches, and seasonality before calling it "the long-term effect."
4. **Pick a method:** stable cohort → cohort analysis; want to isolate learning and strip exogenous noise → post-period diff-in-diff; unsure how long to wait → time-staggered treatments to detect convergence, then post-period; under launch pressure → holdback (small Control) or reverse experiment (equilibrium first).
5. **Diagnose the learning effect** in the post-period: is it user-learned (behavior adaptation) or system-learned (persistent state)? Extrapolation to long-term is only clean when system-learned = 0.
6. **Interpret novelty/primacy** as signal, not just noise: novelty that fades to nothing = clickbait/low quality; slow-building gains = discoverability problem fixable with in-product education.
7. **Guard against exogenous contamination** when generalizing — isolate big one-off shocks (seasonality, competitor moves, policy changes) unlikely to repeat.

## Pitfalls

| Pitfall | Why it happens | Detect / avoid |
| --- | --- | --- |
| Reading last week of a long experiment as clean long-term | Dilution, cookie churn, leakage accumulate over time | Use cohort/post-period methods; don't trust pΔT alone |
| Treatment-effect dilution | Users span multiple devices/entry points; only a fraction of their time is truly in-Treatment | Longer runs dilute more; use a stable-ID cohort |
| Cookie churn | Cookies churn or get clobbered; a Treatment user re-randomizes into Control | Longer runs churn more; cohort analysis fails on high-churn cookie IDs |
| Survivorship bias | Users who dislike Treatment abandon; pΔT captures only survivors | Should trigger an SRM alert; compare survival rates across arms |
| Interpreting pΔT − pΔ₁ as Treatment-caused | Seasonality/population change can produce the whole gap | Don't attribute without further study; isolate exogenous factors |
| Interacting feature launches | Other teams ship features that erode the win (notifications, embeds) | Expect erosion; post-period isolates interactions |
| Network leakage over time | Imperfect isolation lets Treatment leak to Control, cascading | Longer runs leak more; see interference skill |
| Reading a noisy trend line to decide "converged" | Day-of-week and big events swamp the long-term signal | Use time-staggered treatments (A/A on exposure duration) instead |
| Assuming system-learned effect is zero | Persistent personalization, opt-outs, impression caps carry over | Post-period extrapolation only valid when A/A post-period truly identical |
| Cohort not representative | Logged-in-only cohort differs from full population | External-validity risk; stratify + weight to population |
| Holdback under-powered | Small (10%) Control has low sensitivity | Confirm reduced power still detects the effect you need |
| Reverse experiment confuses users | Pulling users from a visible feature back to Control | Only use when the change isn't jarring to revert |
| Time-staggered starts too close together | Slow learning needs a gap to manifest a T₁−T₀ difference | Ensure enough gap between staggered start times |

## Rules of thumb

- Default experiment duration: **1–2 weeks**; short-term usually generalizes.
- Long-term threshold: **3+ months**, or defined by exposures (e.g., ≥10 exposures to the feature).
- Amara's law framing: overestimate technology impact short-run, underestimate it long-run.
- Post-period = **difference-in-difference**: both arms get identical features; the residual gap is the pure learning effect.
- Time-staggered convergence: prefer **Type II < 20%** even if Type I > 5% — missing a real remaining difference is the costly error here.
- Holdback: keep **~10% in Control** for weeks/months after a 90% launch. Reverse: ramp ~10% back to Control after a 100% launch.
- A divergent short-vs-long gap with abandonment should show up as an **SRM** — always check.

## Related skills

- `../trustworthiness/SKILL.md` — SRM, A/A tests, novelty/primacy as threats to validity
- `../interference/SKILL.md` — network/marketplace leakage between variants
- `../variance-and-sensitivity/SKILL.md` — dilution mechanics, survivorship, triggered-population SRM
- `../ramping/SKILL.md` — staged rollout, holdbacks/holdouts in the ramp process
- `../metrics-and-oec/SKILL.md` — building a short-term OEC that causally predicts long-term goals
- `../when-you-cant-ab-test/SKILL.md` — observational methods when a clean long-term A/B isn't feasible

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapter 23.
