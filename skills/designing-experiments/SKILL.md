---
name: designing-experiments
description: Design an online controlled experiment (A/B test) before it starts — form a falsifiable hypothesis, choose variants, pick the randomization unit, run a power analysis to set sample size (n ≈ 16σ²/δ²), and set duration. Invoke for "design an A/B test", "how many users do I need", "how long should the experiment run", "user vs session vs page randomization", or "what should we test".
---

# Designing Experiments

Everything that must be decided BEFORE an experiment starts: the hypothesis, the variants, the randomization unit, the sample size, and the duration. The single most important principle: fix these decisions — hypothesis, metrics, practical-significance boundary, size, duration — before looking at any data, because you are almost certainly wrong about which ideas will win, and only a pre-committed design lets the data overrule you.

Why bother at all: it is nearly impossible to assess the value of an idea in advance. A low-priority Bing change that lengthened ad titles — a few days of one engineer's work — raised revenue 12%, worth over $100M/year in the US alone. Amazon moving a credit-card offer from the home page to the shopping cart (where purchase intent is clear) added tens of millions in annual profit, and its shopping-cart recommendations — which a senior VP forbade an engineer to work on — won "by such a wide margin that not having it live was costing Amazon a noticeable chunk of change." Google's test of 41 gradations of blue on search links, mocked by designers, measurably improved engagement; Bing's equivalent color tweaks were worth over $10M/year. None of these were predicted. Controlled experiments are the top of the hierarchy of evidence — the gold standard for causality — above observational studies, case studies, and the HiPPO (Highest Paid Person's Opinion).

## When to use

- You are about to build or run an A/B test and need to write the experiment design.
- Someone asks "how many users / how long do we need?" — power and duration analysis.
- Choosing between user-, session-, or page-level randomization, or between cookie / signed-in ID / device ID.
- Deciding what to test at all: which idea, how many variants, painted-door vs full build.
- Judging whether an experiment can even answer the question being asked (tactic vs strategy).
- Checking whether an organization/product meets the prerequisites for useful experimentation.

## Core principles

1. **Assume your idea will fail — that is the base rate.** Only about one third of ideas tested at Microsoft improved the metrics they were designed to improve; in well-optimized domains like Bing and Google the success rate is 10–20%; Slack sees ~30% of monetization experiments win; Netflix considers 90% of what it tries to be wrong. Design experiments to cheaply kill bad ideas, not to confirm good ones. Never trust intuition or the HiPPO over measurement — and never infer causality from correlation: Office 365 users who see error messages and crashes churn *less*, because heavy usage drives all three.

2. **Verify the necessary ingredients before designing anything.** (a) Experimental units (usually users) assignable to variants with little or no interference between them; (b) enough units — thousands at minimum; the larger the count, the smaller the detectable effect; (c) key metrics, ideally an OEC, agreed upon and practically measurable from reliable, cheap instrumentation; (d) changes that are easy to make (server-side changes are far easier to test than client-side). If these fail, use complementary techniques instead (see ../when-you-cant-ab-test/SKILL.md).

3. **State a falsifiable hypothesis tied to a specific metric and population.** "Adding a coupon-code field to checkout will degrade revenue-per-user for users who start the purchase process" is testable; "improve the checkout experience" is not. Locate the change in the user funnel and scope the metric's denominator to exactly the users who could be affected: all site visitors is valid but noisy (diluted by users who never reach checkout); only purchasers is *wrong* (if more users complete purchase, revenue-per-user can drop while total revenue rises); users who start checkout is right — all potentially affected users, no unaffected ones.

4. **Normalize key metrics by sample size and set a practical-significance boundary up front.** Use revenue-per-user, not total revenue — actual user counts differ between variants by chance even at equal allocation. Then decide what effect size is worth acting on *before* running: at Google/Bing scale a 0.2% revenue change is practically significant; a startup may not care about anything under 10%. This boundary (the minimum detectable effect, δ) drives sample size, and should reflect the cost of building/maintaining the feature and the cost of a wrong decision (a short-lived change tolerates lower bars).

5. **Test the idea separately from the implementation.** Run several Treatments of the same idea (e.g., coupon field inline vs as a popup) so a losing implementation doesn't kill a good idea, or vice versa. Use a painted-door (fake-door) test — build only the visible surface, e.g. a coupon field that always says "Invalid Coupon Code" — to measure the impact of an idea before paying for the full system. In the coupon example the painted door showed −2.8% and −7.8% revenue-per-user for the two UIs, killing the promotion-email business model for the cost of one form field.

6. **Randomize by user unless you have a specific reason not to.** The granularity options are page-level, session-level (visit, ends after ~30 min inactivity), query-level (search engines), user-day, and user-level. Two questions decide it: does the user notice the change (a font change flickering per page is a broken experience — coarser unit needed), and which metrics matter? Finer units give more samples, hence lower variance and more power — but only sub-user metrics remain valid, and users noticing different variants across pages violates SUTVA (the stable unit treatment value assumption: units must not interfere). User-level is the most common choice: it preserves experience consistency and enables long-term metrics like retention and sessions-per-user.

7. **The randomization unit must be the same as, or coarser than, the analysis unit of every metric you care about.** Randomize by user, analyze sessions-per-user: fine. Randomize by user, analyze page-level CTR: workable but needs the delta method or bootstrap for correct variance, and is skewable by a single bot with 10,000 pageviews under one ID (cap per-user contributions or use average-CTR-per-user). Randomize by page, analyze revenue-per-user: *invalid* — each user's experience mixes variants, so user-level metrics are meaningless. If user-level metrics are in your OEC, you cannot randomize below user. Randomizing across a dependency is also invalid: with page-level randomization and personalization, a bad Treatment result on query 1 pushes the reformulated query 2 into Control.

8. **Choose the user identifier by required scope and longevity.** Signed-in ID: stable across devices, platforms, and time — best when available. Cookie (or Apple idFA/idFV, Android Advertising ID): per-platform, erasable, less persistent — but the right choice when the flow crosses the sign-in boundary (e.g., new-user onboarding). Device ID: immutable, longitudinally stable, no cross-device consistency. IP address: avoid except for infrastructure tests (e.g., hosting-location latency) that can only be controlled at IP level — IPs both split single users (home vs work) and merge thousands behind one firewall, wrecking power and adding outliers. For interference-prone settings, randomize coarser still: tenant for enterprise (Office), advertiser or advertiser clusters for auctions, friend clusters for social networks.

9. **Size the experiment with a power analysis: n ≈ 16σ²/δ² per experiment (Treatment and Control equal size).** σ² is the sample variance of the metric, δ the minimum practically significant delta. This targets the industry standard of 80% power (design for 80–90%) at α = 0.05. Power is relative to effect size — an experiment powered for a 10% effect is *not* powered for a 1% effect. Levers: a lower-variance metric (purchase-yes/no indicator instead of revenue-per-user) shrinks n; a larger δ shrinks n; a stricter p-value threshold (0.01) grows n. Also check the normality of the mean: rule of thumb, ≥ 355·s² samples per variant where s is the metric's skewness coefficient — Bing capping Revenue/User at $10/user/week cut skewness from 18 to 5 and the required sample from 114k to 10k. Capping/transforming skewed metrics buys sensitivity cheaply.

10. **Overpower rather than exactly power.** You will want to read segments (country, platform) and multiple key metrics, each of which needs its own power; being powered for the overall effect does not mean being powered for Canada. Overpowering also hedges against triggered-analysis changes to σ² and δ. If truly traffic-starved, an underpowered experiment can be replicated on orthogonal randomization and the p-values combined via Fisher's meta-analysis. With many Treatments, consider a Control larger than each Treatment.

11. **Run at least one full week, in whole-week multiples.** Weekend users differ from weekday users, and the same user behaves differently by day — the design must span the weekly cycle. Longer runs add users, but sub-linearly (day-one users return, so 2 days < 2N users), and some accumulating metrics (e.g., sessions-per-user) grow in variance over time, so running longer eventually stops buying power. Watch for: seasonality/holidays (gift cards sell in December — external validity limits generalization); novelty effects (a flashy new button gets clicked, then decays) and primacy effects (users primed on the old design need time to adopt — big site redesigns commonly fail even to reach parity with the old site) — extend the run if the treatment effect trends over time.

12. **Match the design to where the question sits on the strategy–tactics hierarchy.** Experiments directly test tactics; a strategy is only tested through the portfolio of tactics under it. One failed experiment means a poor tactic, not a bad strategy — but if many tactics under a strategy keep failing, the strategy is the problem: Bing spent over $25M and two years on social-network integration before abandoning it when experiment after experiment showed no impact (treat it as sunk cost and decide forward-looking). Keep a portfolio: mostly incremental hill-climbing near the current product, plus a few radical bets — most big jumps fail, but the rare wins pay for many failures. Radical/marketplace changes need longer, larger, or alternative designs (country-level experiments; the ice-cube analogy — a small marketplace perturbation may show nothing until it crosses a melting point).

13. **The organizational tenets must hold for results to matter.** (1) The organization wants data-driven decisions and has formalized an OEC — measurable within one to two weeks, sensitive, and causally predictive of long-term goals ("percent of plan delivered" is not it; short-term profit is gameable by raising prices). (2) It will invest in infrastructure and trustworthiness — getting numbers is easy; getting numbers you can trust is hard. (3) It accepts that it is poor at assessing the value of ideas — otherwise losing results get explained away. Guardrail metrics encode what the organization refuses to trade away (crashes, latency): strategy "requires you to choose what not to do."

## Workflow

1. **Check prerequisites.** Enough units (thousands+), assignable without interference, agreed metrics, instrumentation in place, change is cheap to make. If not → ../when-you-cant-ab-test/SKILL.md.
2. **Write the hypothesis.** Name the change, the funnel step it touches, the expected direction, and the metric: "Doing X will move metric M for population P." Make it falsifiable.
3. **Pick the OEC / goal metrics.** Normalize by units (per-user, not totals). Scope the denominator to exactly the potentially-affected population — all affected, none unaffected. See ../metrics-and-oec/SKILL.md.
4. **Set the practical-significance boundary δ** from business context: build + maintenance cost, decision reversibility, lifespan of the change. High cost → high δ; near-zero launch cost → low δ.
5. **Choose variants.** Control = existing experience. Add multiple Treatments when separating idea from implementation. Prefer a painted door over building the full system when the question is "does anyone want this?"
6. **Choose the randomization unit.** Default: user. Confirm (a) every OEC metric's analysis unit is at or below the randomization unit, (b) users won't notice cross-variant inconsistency, (c) no interference across units (else cluster: tenant/advertiser/friend-cluster). Pick the identifier (signed-in > cookie/device > never IP unless infra) by required scope and longevity.
7. **Decide targeting.** All users, or a filtered population (locale, region, platform, device). Remember: targeting shrinks n available.
8. **Run the power analysis.** Estimate σ² from historical data for the exact metric and population; compute n ≈ 16σ²/δ² per variant pair; check the 355·s² normality bound and cap skewed metrics if needed. Add headroom for segment-level reads. Verify traffic-sharing with concurrent experiments still leaves enough.
9. **Set the split and duration.** Near-equal splits (e.g., 34/33/33 for A/B/C); enlarge Control if Treatments are many. Convert n to days from the (sub-linear) user-accumulation rate; round up to a whole number of weeks, minimum one; avoid holidays unless they are the target; plan to extend if novelty/primacy shows up in the effect trend.
10. **Pre-register the decision rule.** Statistical threshold (p < 0.05 or stricter), the practical boundary, guardrail metrics that veto a launch, and what you'll do in each stat-sig × practical-sig quadrant — including "rerun with more power" when the CI is too wide to rule practical significance in or out. Then, and only then, start the experiment (see ../analyzing-results/SKILL.md and ../trustworthiness/SKILL.md).

## Pitfalls

| Pitfall | Why it happens | Detect / avoid |
|---|---|---|
| Claiming causality from observational correlation | Feature users churn less → "feature reduces churn" | A confounder (usage) drives both; only randomized assignment isolates cause. Office 365 crash-viewers churn less. |
| Sloppy randomization | Convenience assignment, biased hardware/process | Any factor influencing assignment biases results (RAND's pulse machine needed re-randomization; VA streptomycin trial failed to physician selection bias). Hash-based, persistent, independent assignment only. |
| Total-sum OEC (total revenue) | Feels like the business goal | Variant user counts differ by chance; use per-unit metrics (revenue-per-user). |
| Wrong denominator | Including unaffected users, or only converters | Unaffected users dilute (noise); conditioning on completion biases (more purchasers can lower revenue-per-user). Scope to users reaching the changed funnel step. |
| Randomization unit finer than analysis unit | Chasing power via page-level randomization | User-level metrics become meaningless when one user sees mixed variants. Check every OEC metric's unit first. |
| Page/session randomization with cross-page dependencies | Personalization, query reformulation | Bad Treatment on page 1 contaminates the Control page 2. Use user-level. |
| Visible flicker between variants | Fine-grained randomization of noticeable changes | Inconsistent experience itself moves metrics and violates SUTVA. Coarsen the unit. |
| IP-address randomization | Only knob for infra changes | Splits users across home/work, merges firms behind one IP → skew, outliers, low power. Reserve for infra-only tests. |
| Bot skew in coarser-than-analysis-unit metrics | One ID contributes 10,000 pageviews | Cap per-user contribution or use per-user averaged metrics. |
| Underpowered experiment read as "neutral" | No power analysis, or δ never set | A CI spanning ±10% revenue is "unknown," not "no effect." Compute n ≈ 16σ²/δ² first; rerun with more power when CI exceeds the practical boundary. |
| Powered for overall, not for segments/metrics | Power computed once for one metric | Overpower; re-check power per key segment and key metric. |
| Skewed metric breaks normality | Revenue-style long tails | Need ≥ 355·s² samples per variant; cap values (Bing: $10/user/week cap → skewness 18→5, n 114k→10k). |
| Sub-week or non-whole-week runs | Impatience | Day-of-week effects bias the estimate; minimum one week, whole-week multiples. |
| "More weeks = more power" assumed indefinitely | Ignoring return visitors and accumulating variance | User growth is sub-linear and some metrics' variance grows with time; extra weeks may add little. |
| Novelty/primacy mistaken for the true effect | Short-term readout of a flashy or habit-breaking change | Plot the effect over time; extend the run; consider long-term designs (../long-term-effects/SKILL.md). |
| Holiday-window generalization | Running during atypical seasons | External validity: December results may not hold in March. |
| Judging a strategy by one tactic | Experiment tests hypotheses, strategy is broader | One failure ≠ bad strategy; a long streak of failures does (Bing social: 2 years, $25M+, abandoned). |
| No OEC, or gameable OEC | Measuring is costly; "percent of plan delivered" is easier | Short-term-measurable, sensitive, long-term-predictive OEC agreed before the experiment; guardrails for what must not degrade. |
| "Too good to be true" designs unchallenged | Excitement | Wire alerts for implausible wins (Bing's revenue-too-high alert); they usually mean double-logging or a broken page — occasionally a $100M/yr winner. |

## Rules of thumb

- Sample size: **n ≈ 16σ²/δ²** per variant (equal split, 80% power, α = 0.05); use the smallest practically significant δ (minimum detectable effect).
- Design for **80–90% power**; industry floor is 80%.
- Statistical significance: **p < 0.05** standard; p < 0.01 when a false positive is expensive; stricter thresholds need larger n.
- 95% CI ≈ observed delta ± **1.96 standard errors**; significance ⇔ CI excludes zero.
- Normality of the mean: **≥ 355·s²** samples per variant (s = skewness coefficient); cap skewed metrics to shrink s.
- Minimum **thousands of experimental units** to experiment usefully at all; startups start by hunting big effects.
- Duration: **minimum 1 week**, whole-week multiples; extend for novelty/primacy or seasonality.
- Default randomization unit: **user**; identifier preference: signed-in ID > cookie/device ID > IP (infra only).
- Expected win rate: **10–33%** of ideas succeed (Google/Bing 10–20%, Microsoft ~33%, Slack ~30%); plan for ~70% of work being thrown away.
- Typical winning effect size at mature products: **0.1–2%** on key metrics; a 5% win on a 10%-triggered population dilutes to ~0.5% overall.
- OEC must be measurable in **1–2 weeks** yet predictive of long-term goals.
- Splits: near-equal (e.g., 34/33/33); grow Control above per-Treatment size when Treatments are many.
- Overpowering is fine and recommended — segments and secondary metrics need it.

## Related skills

- ../metrics-and-oec/SKILL.md — choosing the OEC, guardrails, metric taxonomy
- ../analyzing-results/SKILL.md — p-values, CIs, multiple testing, segments after the run
- ../trustworthiness/SKILL.md — A/A tests, SRM, sanity checks before believing results
- ../variance-and-sensitivity/SKILL.md — CUPED and triggering to shrink required n
- ../ramping/SKILL.md — staged rollout once designed (safety ≠ final size)
- ../interference/SKILL.md — when units leak: networks, marketplaces, cluster randomization
- ../long-term-effects/SKILL.md — novelty/primacy, holdouts, long-term measurement
- ../when-you-cant-ab-test/SKILL.md — alternatives when prerequisites fail
- ../experimentation-platform/SKILL.md — variant assignment and traffic-sharing infrastructure

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapters 1, 2, 14, 17.
