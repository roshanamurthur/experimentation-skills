---
name: metrics-and-oec
description: Choose, formulate, and evaluate metrics for an organization or experiment — goal/driver/guardrail taxonomy, combining key metrics into an Overall Evaluation Criterion (OEC), weighting tradeoffs, and defending against gaming (Goodhart's law). Invoke when picking success metrics, defining an OEC, debating "which metric should we optimize", or auditing metrics for gameability.
---

# Metrics and the Overall Evaluation Criterion (OEC)

This skill covers how to pick and formulate metrics — for the organization (goals, drivers, guardrails) and for experiments (the OEC). The single most important principle: the OEC must be measurable in the short term (an experiment's duration) yet believed to *causally* drive long-term strategic objectives. Correlation in historical data is not enough — Goodhart's law guarantees that a correlational metric under optimization pressure stops being a good measure.

## When to use

- Choosing or defining success metrics for a product, team, or company (OKRs, "true north" metrics).
- Deciding what metric(s) an A/B test should be judged on — defining or refining an OEC.
- A stakeholder proposes a single metric to optimize ("time-on-site", "queries", "CTR", "short-term revenue") and you need to sanity-check it.
- Combining multiple conflicting key metrics into a single decision criterion, or assigning weights.
- Auditing metrics for gameability, perverse incentives, or vanity-metric status.
- A metric moved in a "good" direction but you suspect the underlying user experience got worse.
- Deciding which business metrics can be reused for experimentation vs. which need surrogates.

## Core principles

1. **Classify every metric as goal, driver, or guardrail — and know which role it plays for which team.**
   Goal metrics (success / true-north metrics) reflect what the organization ultimately cares about; articulate the goal in *words* first (tied to the mission), then accept that the metric is an imperfect proxy. Driver metrics (signpost / surrogate / predictive metrics) are shorter-term, faster-moving, more sensitive, and encode a causal hypothesis about what drives the goal. Guardrail metrics protect against violated assumptions and constraints (e.g., latency, crash rate) — plus a second class that guards experiment trustworthiness (see `../trustworthiness/SKILL.md`). The same metric plays different roles for different teams: latency is a guardrail for a product team but the *goal* metric for an infrastructure team (which then uses the product team's goal metrics as its guardrails).

2. **Goal metrics must be simple and stable; driver metrics must be aligned, actionable, sensitive, and hard to game.**
   Simple: easily understood and broadly accepted by stakeholders. Stable: you should not need to redefine the goal metric every time a feature launches. Drivers must be validated as actual drivers of the goal (run experiments expressly to test this), be levers teams can act on, and be sensitive enough to detect impact from most initiatives. Anticipate the behavior each metric incentivizes before adopting it.

3. **Build quality into metric definitions.**
   A raw count is rarely enough. A search-result click followed by an immediate back-button press is a *bad* click; a signup is *good* only if the user then engages; a LinkedIn profile is *good* only if it has enough info (education, positions) to represent the user. Penalize quick-backs, require success qualifiers ("good/successful session" rather than raw time-on-site), and use human evaluation to counterbalance clickbait when optimizing CTR. Unqualified time-on-site as an optimization target leads to interstitial pages and a slow site — short-term gains, long-term abandonment.

4. **Measure what you don't want when it's easier than measuring what you do.**
   User dissatisfaction is often more precisely measurable than satisfaction: a short visit from a search result correlates with unhappiness, while a long visit is ambiguous (finding what they need vs. struggling in frustration). Negative metrics work well as guardrail or debug metrics. Bounce rate is a canonical example — use surveys/UER to hypothesize the behavior pattern, then large-scale log analysis to pin the exact threshold (1 pageview? 20 seconds?).

5. **Keep statistical models in metrics interpretable and re-validated.**
   LTV computed from a predicted survival probability is fine — until the survival function is too complicated to get stakeholder buy-in or debug a sudden drop. Netflix uses *bucketized* watch hours as drivers precisely because they are interpretable and indicative of long-term retention. Re-check prediction error on model-based metrics over time.

6. **Experiment metrics need properties beyond business metrics: measurable, attributable, sensitive, and timely.**
   Attributable means you can tie the metric value to a variant (third-party-provided data often can't be). Sensitivity spans a spectrum: company stock price is uselessly insensitive to a product change; "is the new feature showing?" is maximally sensitive but says nothing about value; feature click-through is sensitive but localized and misses cannibalization of the rest of the page. Good middle-ground metrics: whole-page click-through penalized for quick-backs, a success measure (e.g., purchase), and time-to-success. For yearly-renewal subscriptions, don't wait a year — use surrogates like usage as early indicators of the satisfaction that leads to renewal. For ads revenue, a few very expensive clicks inflate variance; keep raw revenue for business reporting but add a truncated/capped revenue metric for experiment sensitivity.

7. **Prefer a single weighted OEC over a pile of key metrics.**
   No single natural metric captures a business (the jet-cockpit argument: a pilot needs airspeed AND altitude AND fuel), but a *composite* single score is achievable and valuable — like a basketball scoreboard or a FICO score (multiple inputs, one 300–850 number). An agreed OEC makes success explicit, aligns the organization on tradeoffs, enables consistent decisions without management escalation, and enables automated shipping and parameter sweeps. Roy's recipe: normalize each key metric to a predefined range (e.g., 0–1), assign weights, OEC = weighted sum.

8. **If you can't weight yet, use the four-case decision rule and accumulate decisions until weights emerge.**
   (1) All key metrics flat or positive, at least one positive → ship. (2) All flat or negative, at least one negative → don't ship. (3) All flat → don't ship; increase power, fail fast, or pivot. (4) Some positive, some negative → decide on tradeoffs — and log these judgment calls; enough of them lets you fit weights.

9. **Limit key metrics to about five (the Otis Redding problem).**
   "Can't do what ten people tell me to do, so I guess I'll remain the same." Too many key metrics cause cognitive overload and get ignored — and inflate false positives: under the null, with k independent metrics at p < 0.05, the chance at least one is "significant" is 1 − 0.95^k — 23% at k = 5, 40% at k = 10. Keep hundreds of *debug/diagnostic* metrics on the scorecard, but only a handful of key ones.

10. **The metric can move "up" and still be bad — decompose before you trust direction.**
    Bing's ranker bug: a Treatment showed very poor search results, and two key org metrics improved sharply — distinct queries/user up over 10%, revenue/user up over 30%. Worse results forced users to re-query (inflating queries) and click ads instead (inflating revenue). If those were the OEC, a search engine should deliberately degrade quality. Decompose monthly distinct queries: `users/month × sessions/user × distinct-queries/session`. Users/month is fixed by experiment design (a 50/50 split puts ~equal users in each variant) — unusable in an OEC. Distinct queries/task should be *minimized* (users finishing faster), but its measurable surrogate, queries/session, is subtle: up may mean re-querying, down may mean abandonment — so decrease it only while verifying task success/no increased abandonment. That leaves **sessions/user as the metric to increase: satisfied users come back more often.** Revenue/user likewise needs constraints — e.g., cap average ad pixels over queries and maximize revenue within that constraint optimization.

11. **Charge gamed short-term wins their long-term cost (the Amazon email OEC).**
    Amazon's programmatic email campaigns were credited with click-through revenue — a metric monotonically increasing in email volume (true even comparing Treatment recipients vs. Control non-recipients), so it rewarded spam. Traffic-cop constraints ("one email per X days") just created a scheduling contest. The fix: recognize an unsubscribe kills all future email opportunity, and put its lifetime loss into the OEC: `OEC = (Σᵢ Revᵢ − s × unsubscribe_lifetime_loss) / n` where i ranges over recipients, s = unsubscribes, n = users in the variant. With just a *few dollars* assigned to unsubscribe lifetime loss, more than half the campaigns showed a negative OEC. Bonus insight: since unsubscribes are so costly, redesign the unsubscribe page to default to leaving only the campaign family, not all Amazon email — drastically cutting the cost per unsubscribe.

12. **Assume the metric will be gamed; constrain it (Goodhart's law, Campbell's law, Lucas critique).**
    Goodhart: "When a measure becomes a target, it ceases to be a good measure." Campbell: the more a quantitative indicator drives decisions, the more it corrupts the process it monitors. Lucas critique: historical correlations are not structural — the Phillips curve's inflation/unemployment correlation (UK, 1861–1957) broke when policy pushed on it (1973–75 US: both rose). Harford's version: "Fort Knox has never been robbed, so sack the guards" — incentives shift when policy changes. Practically: unconstrained metrics are gameable. Short-term revenue → raise prices / plaster ads → LTV declines. "Queries with no results" → return bad results. Constrain revenue by page space or quality. Prefer metrics of *user value and user actions* (Facebook Likes: a user action that's core to the experience) over vanity counts of *your* actions (banner-ad count). Grove's rule: measure output (orders), not activity (calls).

13. **Evaluate metrics continuously and expect them to evolve.**
    Before adding a metric, check it adds information over existing ones. Re-validate model-based metrics (LTV prediction error). Periodically audit heavily-used experiment metrics for induced gaming (did a threshold cause disproportionate effort moving users just across it?). The hardest evaluation is causal validation of driver → goal; approaches: triangulate with surveys/focus groups/UER; use observational studies to *invalidate* hypotheses (they rarely prove causation — see `../when-you-cant-ab-test/SKILL.md`); borrow validation from other companies (published speed→revenue and app-size→downloads studies); run experiments whose primary purpose is metric evaluation (e.g., slow-rollout a loyalty program to test retention/LTV impact); use a corpus of well-understood historical experiments as golden samples for checking sensitivity and causal alignment. Metrics also change because the business evolves (adoption → engagement/retention focus; early adopters unrepresentative of the long-term user base), the environment evolves (competition, privacy, policy), or your understanding evolves. Drivers/guardrails/data-quality metrics evolve faster than goal metrics. As the org grows, build infrastructure for metric changes: evaluation of new metrics, schema changes, backfills.

## Workflow

1. **Articulate success in words first.** Why does the product exist? What does success look like? Get leadership to answer; tie to mission. Only then translate to metrics, acknowledging the translation is a proxy.
2. **Pick 1 or a very small set of goal metrics** that are simple and stable. Accept they move slowly and small per-initiative.
3. **Derive driver metrics from a causal model of success.** Use frameworks to enumerate candidates: HEART (Happiness, Engagement, Adoption, Retention, Task success), PIRATE/AARRR (Acquisition, Activation, Retention, Referral, Revenue), or user funnels. Check each driver for: alignment (validated causal link to goal), actionability, sensitivity, gaming resistance.
4. **Define guardrails**: sensitive metrics measuring phenomena known to affect goals that most teams shouldn't move — latency, HTML response size per page, JavaScript errors per page, revenue-per-user (or its more sensitive variants: revenue indicator per user, revenue capped at $X, revenue-per-page), pageviews-per-user (denominator of many per-page metrics — an unexpected change there requires careful review; won't work when testing infinite scroll), client crash rate (add a did-user-crash indicator — lower variance, significant sooner).
5. **Formulate precisely.** Use small-scale methods (surveys, UER) to generate hypotheses, then large-scale log analysis to fix thresholds and definitions. Build in quality qualifiers (good clicks, successful sessions, good profiles). Add negative/dissatisfaction metrics. Decompose key metrics into debug metrics (e.g., CTR → ~20 region-level click metrics; revenue → purchase indicator × conditional revenue, so you can tell whether more people purchased vs. bigger baskets).
6. **Filter for experiment-worthiness**: measurable within experiment duration, attributable to variant, sensitive, timely. Substitute surrogates where needed (usage for renewal; truncated revenue for raw revenue).
7. **Augment the experiment scorecard** with surrogates, granular feature-level metrics, trustworthiness/data-quality guardrails, and diagnostic metrics — expect a few key metrics plus hundreds to thousands of others, segmentable by browser, market, etc.
8. **Combine key metrics into an OEC.** Normalize each to 0–1, weight, sum. If weights aren't agreed yet, run the four-case decision rule (principle 8) and record every case-4 tradeoff decision to fit weights later. Cap key metrics at ~5.
9. **Red-team the OEC for gaming.** For each component ask: what's the cheapest way to move this without creating user value? (Degrade relevance? Spam? Raise prices? Plaster ads?) Add constraints or lifetime-cost terms (Amazon unsubscribe model) until the cheap exploits go negative.
10. **Validate causality** of drivers/OEC against goals using the methods in principle 13, then **schedule periodic re-evaluation** — for gaming, prediction drift, and business/environment change.
11. **Once the OEC is trusted, automate**: ship decisions and parameter sweeps can run against it without escalation.

## Pitfalls

| Pitfall | Why it happens | Detect / avoid |
|---|---|---|
| Optimizing a metric that goes up when the product gets worse (Bing queries/user, revenue/user under a ranker bug) | Metric conflates re-querying/ad-clicking friction with engagement | Decompose the metric; prefer sessions/user; require task-success checks alongside queries/session |
| Click-through revenue OEC rewards spam (Amazon email) | Metric monotonically increases with volume, even Treatment-vs-Control | Add lifetime-cost terms for user-hostile events (unsubscribe lifetime loss) |
| Time-on-site as unqualified target | Long visits ambiguous: success or frustration; slow pages inflate it | Qualify with good/successful-session definitions; watch abandonment |
| CTR optimization breeds clickbait | Metric is a proxy with failure cases | Counterbalance with human-evaluated relevance; penalize quick-backs |
| Vanity metrics (count of banner ads, calls made) | Counts your activity, not user value/output | Use user-action metrics (clicks on ads, Likes, orders) |
| Unconstrained metrics gamed (chicken efficiency → cook-to-order and business death; rat/cobra bounties → farming; ER wait time → patients held in ambulances; weightlifter breaking records grams at a time; low-inventory bonus → operations shutdown; falsely certifying orphans as mentally ill for $2.25/day vs $0.70; fire departments funded per call → less prevention) | Humans are ingenious when rewards attach to numbers | Constrain the metric (revenue per page space/quality); think through incentives before adopting; audit thresholds for bunching |
| Too many key metrics (Otis Redding problem) | Each stakeholder adds one | Cap at ~5; remember 23% false-positive chance at k=5, 40% at k=10; demote the rest to debug metrics |
| Treating correlation as causation in the OEC (Phillips curve, Fort Knox guards) | Historical regularities aren't structural; incentives shift under policy | Causal validation: dedicated experiments, golden historical experiments, triangulation |
| Driver metric never validated against goal | Causal model is just a hypothesis | Test it: experiments for metric evaluation, observational invalidation, cross-company studies |
| Complicated model-based metric (opaque LTV survival model) | Sophistication over interpretability | Prefer interpretable forms (bucketized watch hours); track prediction error over time |
| Insensitive metric as experiment OEC (stock price) or hyper-local one (feature shown/feature CTR) | Wrong point on the sensitivity spectrum; misses cannibalization | Use whole-page CTR with quick-back penalty, success, time-to-success; truncate outlier-heavy revenue |
| Guardrail misapplied (pageviews-per-user during infinite-scroll test) | Feature legitimately changes the guardrail | Re-derive which guardrails apply per experiment |
| Metrics frozen while business/users change | Early adopters unrepresentative; environment shifts | Scheduled metric reviews; expect drivers to evolve faster than goals; build backfill/schema infra |
| Watermelon metrics — green outside, red inside | Team-level targets diverge from customer experience | Align every team's goal/driver/guardrail set to company strategy; discuss role swaps explicitly |

## Rules of thumb

- Limit key experiment metrics to **~5**; with k independent metrics, P(≥1 false positive at 0.05) = 1 − 0.95^k (k=5 → 23%, k=10 → 40%).
- OEC construction: normalize each key metric to **0–1**, weight, sum (Roy 2001).
- Amazon email OEC: `(Σ Revᵢ − s × unsubscribe_lifetime_loss)/n`; even a few dollars of unsubscribe loss flipped >50% of campaigns negative.
- Bing decomposition: `distinct queries/month = users/month × sessions/user × queries/session`; optimize **sessions/user up**, queries/session down only with task-success checks; users/month is fixed by design.
- Revenue guardrail variants when raw revenue/user is too noisy: revenue indicator (0/1), revenue capped at $X, revenue-per-page. Decompose revenue = purchase indicator × conditional revenue.
- Crash metrics: prefer a did-user-crash indicator over crash counts — lower variance, significant earlier.
- Sensitivity spectrum anchors: stock price (never sensitive enough) ↔ feature-shown (sensitive, meaningless). Aim between: penalized whole-page CTR, success, time-to-success.
- Goal metrics: simple + stable. Drivers: aligned + actionable + sensitive + gaming-resistant. Guardrails: more sensitive than goals/drivers.
- When a metric definition includes a threshold, periodically check for bunching just across it — evidence of gaming.
- Four-case ship rule: any positive & none negative → ship; any negative & none positive → no; all flat → no (repower/pivot); mixed → tradeoff call, and record it.

## Related skills

- `../designing-experiments/SKILL.md` — power/sample size for the metrics you pick
- `../analyzing-results/SKILL.md` — multiple-testing corrections when the scorecard has hundreds of metrics
- `../trustworthiness/SKILL.md` — trustworthiness-type guardrail metrics (SRM, A/A tests)
- `../variance-and-sensitivity/SKILL.md` — capped metrics, CUPED, making metrics more sensitive
- `../latency-and-performance/SKILL.md` — establishing the causal impact of latency (the canonical guardrail) on goals
- `../long-term-effects/SKILL.md` — measuring the long-term value your OEC is meant to proxy
- `../when-you-cant-ab-test/SKILL.md` — observational methods for metric validation
- `../culture-ethics-memory/SKILL.md` — historical-experiment corpora as golden samples

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapters 6, 7.
