---
name: when-you-cant-ab-test
description: Estimate causal impact when a controlled experiment is impossible, unethical, or premature. Covers complementary techniques (logs analysis, human evaluation, UER, focus groups, surveys, external data) and observational causal study designs (interrupted time series, interleaving, regression discontinuity, instrumental variables, propensity scores, difference-in-differences), plus their failure modes. Invoke for "we can't A/B test this", quasi-experiments, natural experiments, correlation-vs-causation questions, or sizing an opportunity before building.
---

# When You Can't A/B Test

Randomized controlled experiments are the gold standard for causality; everything in this skill is second-best by construction. Use complementary techniques to generate ideas, size opportunities, and validate metrics; use observational causal study designs when you must estimate a causal effect without randomization. The single most important principle: every observational method rests on untestable assumptions, and the dominant failure mode is an unanticipated confound — treat any causal claim from observational data as provisional until an experiment confirms it.

## When to use

- The causal action isn't under your control (e.g., what happens when a user switches from iPhone to Samsung).
- There are too few units to randomize (an M&A event, a Superbowl ad, ~210 US DMAs for TV campaigns — too few for power even with pairing).
- The Control's opportunity cost is too high, or the outcome takes too long (a car repurchase 5 years out).
- The change is expensive relative to its perceived learning value (forcibly signing out all users; removing all ads).
- The treatment is unethical or illegal to withhold or impose.
- You need ideas for experiments, a proxy metric for something unmeasurable (satisfaction, trust), or an upper-bound sizing before investing in a build.
- Someone presents a correlation from logs and wants to act on it as if causal.

## Core principles

1. **The basic identity of causal inference: observed difference = treatment effect + selection bias.**
   `Outcome(treated) − Outcome(untreated) = [effect of treatment on the treated] + [selection bias]`. Randomization drives the expected selection-bias term to zero; every method below is an attempt to argue that term is small without randomization. State explicitly, for any observational design, what assumption zeroes out selection bias — and what would break it.

2. **Pick techniques by the scale-vs-depth tradeoff.**
   Methods form a spectrum from deep-but-small to shallow-but-huge: UER field/lab studies (tens of users, richest detail, direct observation of "why") → focus groups → surveys → human evaluation → external data → logs-based analysis (all users, actions but no "why"). Scale buys generalizability (external validity); depth buys mechanism and intent. Early in a product cycle with too many ideas, go qualitative (UER, focus groups); once you have quantitative candidates, move to observational analysis and experiments.

3. **Triangulate — build a hierarchy of evidence.**
   No single method replicates another's result; use several to bound the answer. Canonical loop for an unmeasurable concept like "user happiness with recommendations": observe a handful of users in a UER study and collect candidate behavioral signals → run a large logs analysis to check the signals hold at scale and relate to business metrics → run an on-screen survey to reach more users with simple questions → run learning experiments that perturb recommendations to see how the proxy metrics move with business metrics and the OEC.

4. **Logs-based (retrospective) analysis sizes the opportunity before you build.**
   Use logs to build intuition (distributions of sessions-per-user, click-through rate, by-segment differences, trends), to characterize candidate metrics (variance, correlation with existing metrics, behavior on past experiments), to find funnel drop-offs that suggest experiments, and to compute an upper bound on impact before investing — e.g., before making email attachments easier to use, count how many attachments are sent today. Caveat: logs only project the past forward. Low current usage may itself be caused by bad usability, which logs won't reveal; pair logs with user research.

5. **Human evaluation gives calibrated labels, not user behavior.**
   Paid raters answer tasks ("prefer side A or B?", "how relevant is this result?"); assign multiple raters per task and resolve disagreements by voting, since rater quality on pay platforms like Mechanical Turk varies with incentives and payment. Raters are not your end users and miss local context — the query "5/3" is arithmetic (1.667) to most raters but a bank query to users near Fifth Third Bank — so personalized systems are hard to rate. The flip side: raters can be trained to spot spam and harm users don't perceive. Bing and Google run human-eval programs fast enough to sit alongside online experiment results in launch decisions; rated-poor results are also a goldmine for debugging rankers.

6. **UER goes deep on few users; always validate its output at scale.**
   Field and lab studies (at most tens of users) observe people doing real tasks, with instruments logs can't provide: eye tracking, diary studies capturing intent and offline activity. Use them to spot struggle (purchases taking too long, users hunting for coupon codes) and to generate metric ideas by correlating true intent with what instrumentation sees. Never ship a conclusion from a UER study alone — validate with observational analysis or an experiment.

7. **Focus groups elicit reactions, not preferences — beware group-think.**
   Guided group discussion scales better than UER and tolerates ambiguous, open-ended questions, but covers less ground and converges on fewer opinions. What people say is not what they choose: Philips ran a focus group where teenagers praised yellow boom boxes and called black "conservative" — then, offered a free boom box on the way out, most took black. Use focus groups for ill-formed hypotheses, branding, and emotional reactions; never as a proxy for revealed preference.

8. **Surveys measure what instrumentation can't, but are never directly comparable to logged data.**
   Question wording, ordering, and priming change answers; answers are self-reported and may be untruthful even anonymously; respondents are a biased sample (response bias — unhappy users answer more). Prefer relative comparisons over time to absolute numbers, and freeze the questionnaire if you want a time series. Good uses: offline behavior, opinions, trust and satisfaction (including months post-purchase), and long-run trends to correlate with aggregate business metrics — which can justify investment in a broad area even if it generates no specific ideas.

9. **External data validates and benchmarks; trust trends, not absolute numbers.**
   Sources: panel-based site-traffic providers (comScore/Hitwise style), per-user segment vendors, commissioned or published surveys, academic papers, and crowd-sourced pattern libraries. You don't control their sampling or methods, so absolute numbers rarely match yours — compare time-series trends and seasonality instead. Papers can validate scalable proxies for unscalable metrics (e.g., search-task duration correlates with self-reported satisfaction; eye-tracking studies calibrate how representative click data is). Smaller companies can also inherit conclusions with strong external evidence — e.g., latency matters — without rerunning the foundational experiments, running their own only to tune product-specific tradeoffs.

10. **Observational causal studies are a last resort with a bad track record — design accordingly.**
    Of six highly cited observational causal studies Ioannidis evaluated, five failed to replicate. Young & Karr compared 52 observational claims from 12 medical papers against randomized trials: zero replicated, and 5 of 52 were significant in the opposite direction — their conclusion: "Any claim coming from an observational study is most likely to be wrong." If you must run one, pre-register the assumption that identifies the effect, hunt for confounds before believing the estimate, and treat activity/engagement as the default suspected common cause in any online study.

## Workflow

1. **Confirm you actually can't experiment.** Re-check the blockers above. If the treatment is triggered by an algorithmic score threshold, note that this setting also lends itself to a randomized experiment near the threshold (or a hybrid) — prefer that to pure RDD. If the goal is only idea pruning for an expensive build, you need sizing (logs) not causality.
2. **Write the causal question as treated-vs-counterfactual.** Identify the treatment, the outcome, the unit, and what "untreated" would have looked like. Write down the selection-bias term and the leading candidate confounds — for online data, start with user activity level, seasonality, and population mix.
3. **Choose the design that matches your data structure:**
   - You control the change but can't split the population → **Interrupted Time Series (ITS)**: model the metric pre-intervention, use the model's post-intervention forecast as the counterfactual, and estimate the effect as the average gap between actual and predicted (Bayesian Structural Time Series is a standard online implementation). Strengthen it by turning the treatment on and off repeatedly — burglaries fell each time police helicopter surveillance was deployed and rose each time it was withdrawn. Key assumption: no time-based confound (seasonality, concurrent system changes). Extra risk: users may notice the experience flipping back and forth, and the measured effect becomes the effect of inconsistency.
   - You're comparing two ranking algorithms → **Interleaving**: mix results x1, y1, x2, y2, ... with duplicates removed and compare click-through on each algorithm's contributions. Very sensitive, but requires homogeneous results; breaks when the first position takes more space or results affect other page areas.
   - Treatment is assigned by a sharp threshold on a score → **Regression Discontinuity Design (RDD)**: compare units just above vs just below the cutoff (scholarship at 80%: 79.9% vs 80.1% students are assumed exchangeable). Key assumption: units can't manipulate their score (violated by "mercy passing"), and nothing else changes at the same threshold — the age-21 mortality spike (~100 extra deaths on a baseline of ~150/day, absent at the 20th and 22nd birthdays) is a clean drinking-age effect only if legal gambling starting at 21 doesn't contaminate it.
   - Something quasi-random influences who gets treated → **Instrumental Variables (IV) / natural experiments**: find an instrument that approximates random assignment and affects treatment uptake without directly affecting the outcome — the Vietnam draft lottery for military service, charter-school seat lotteries for attendance, notification-queue and message-delivery order for measuring notification impact on engagement. Estimate with two-stage least squares. Key assumption: the instrument affects the outcome only through the treatment (exclusion restriction) — untestable.
   - You can only match populations → **Propensity Score Matching (PSM)**: stratify or match treated and untreated units on a modeled probability of treatment so the comparison isn't driven by population mix (used e.g. for online ad campaign evaluation). Key assumption: "strong ignorability" — all confounds are observed and in the model. This is the weakest design: hidden covariates leave hidden bias, and King & Nielsen argue PSM often increases imbalance, inefficiency, model dependence, and bias.
   - You have a comparable untreated group over the same period → **Difference-in-Differences (DiD)**: effect = (Treatment post − pre) − (Control post − pre). The Control's change absorbs external factors (seasonality, economy, inflation). Standard for geo experiments (run TV ads in one DMA, compare against a matched DMA) and works for exogenous changes too (NJ minimum-wage rise vs eastern Pennsylvania fast-food employment). Key assumption: parallel trends — groups may differ in level but must move together absent treatment.
4. **Attack your own result before reporting it.** For each candidate confound, ask whether it could produce the effect with zero true causal impact. Check robustness: placebo periods/thresholds, alternative control groups, on-off reversals for ITS, sensitivity of PSM/regression estimates to specification.
5. **Report with calibrated humility.** State the identifying assumption, the confounds you ruled out and the ones you couldn't, and frame the estimate as lower-trust evidence in the hierarchy. Where the decision is important, propose the eventual controlled experiment (or hybrid) that would confirm it.

## Pitfalls

| Pitfall | Why it happens | How to detect / avoid |
|---|---|---|
| Unrecognized common cause | A third variable drives both "treatment" and outcome | Palm size predicts life expectancy — because gender drives both (women: smaller palms, ~6 years longer life in the US). In Office 365, users who see more errors churn *less* — usage is the common cause; heavy users see more errors and churn less. Default suspect online: activity level. |
| "Users of our feature churn less, so the feature reduces churn" | Same common cause — heavy users adopt more features and churn less | Run a controlled experiment; analyze new and heavy users separately. Never ship on this correlation. |
| Deceptive correlations from outliers | A few extreme points create an apparent relationship (energy drink vs athletic performance) | Plot the data; check whether removing outliers kills the effect. |
| Spurious correlations | Test enough hypotheses and strong correlations appear by chance | Spelling-bee winning-word length correlates r ≈ 0.86 with deaths by venomous spiders. If you lack the domain intuition to laugh it off, you'll believe your version of it. Demand a mechanism. |
| Time-based confounds in ITS | Comparisons span different time periods | Seasonality, concurrent launches, and underlying system changes masquerade as effects. Reverse the treatment multiple times; shade weekends/holidays in the analysis. |
| ITS inconsistency effect | Users notice the experience flipping | The measured effect may be irritation at inconsistency, not the change itself. |
| Threshold contamination in RDD | Multiple things change at the same cutoff | Enumerate everything else that switches at the threshold (drinking age 21 = gambling age 21). |
| Score manipulation in RDD | Units near the cutoff influence their own assignment | Check for bunching just above the threshold (teachers "mercy passing" students). |
| PSM hidden bias | Only observed covariates are matched | Assume unobserved confounds exist; run sensitivity analysis; prefer any design with a plausible source of quasi-randomness. |
| Parallel-trends violation in DiD | Control and Treatment were already diverging | Plot pre-period trends for both groups; if they aren't parallel before T1, the counterfactual is invalid. |
| Interleaving with heterogeneous results | First result takes more space / affects the page | Restrict interleaving to homogeneous ranked lists. |
| Overstated ad effectiveness | Exposure and outcome share the common cause of visiting at all | Yahoo! display-ad observational studies (50M users, three regression designs with controls) estimated 871%–1198% lift in brand searches; the controlled experiment measured 5.4% — two orders of magnitude less. A video-ad before/after study overstated the effect by 350% (being active on MTurk that day predicted being active on Yahoo!). A competitor-signup study found "exposed users signed up more" — the experiment showed identical lift for unexposed same-day visitors. Always include activity as a factor; distrust before/after on self-selected exposure. |
| Trusting refuted-study patterns | Observational medicine keeps teaching this lesson | Hormone replacement therapy looked protective against coronary heart disease observationally; randomized trials showed the opposite. Vitamin E supplementation looked cardioprotective; trials refuted it. Countries' chocolate consumption correlates strikingly with Nobel laureates per capita (r ≈ 0.79) — wealth confounds both. Use these as calibration anchors for how wrong careful-looking studies can be. |

## Rules of thumb

- UER studies: at most tens of users; focus groups: dozens; surveys and human evaluation: hundreds to thousands; logs and external panels: full population.
- Success base rate for well-optimized products is 10–20% of ideas — so an observational study "confirming" your idea works is fighting a low prior.
- Observational vs experimental replication: 5/6 highly cited observational studies failed replication (Ioannidis); 0/52 medical observational claims replicated in RCTs, 5/52 flipped sign (Young & Karr).
- Observational ad-lift estimates ran 871%–1198% vs 5.4% experimental truth — expect order-of-magnitude overstatement when exposure is self-selected.
- ~210 DMAs in the US — geo randomization is chronically underpowered; use DiD with matched markets and pairing.
- Assign 3+ raters per human-evaluation task and use a disagreement-resolution mechanism; rater quality tracks pay and incentives.
- Surveys: compare period-over-period (relative), not absolute; any change to wording or question order breaks the time series.
- External data: validate by overlaying time series and matching trend/seasonality; never expect absolute numbers to match.
- Before building an expensive feature, get an upper-bound sizing from logs first (e.g., attachment-send counts before improving attachments).
- If a threshold rule assigns treatment in your own software, prefer randomizing near the threshold over RDD.

## Related skills

- [designing-experiments](../designing-experiments/SKILL.md) — the preferred path whenever randomization is possible
- [metrics-and-oec](../metrics-and-oec/SKILL.md) — validating proxy metrics generated by these techniques
- [analyzing-results](../analyzing-results/SKILL.md) — stats foundations shared with observational analysis
- [trustworthiness](../trustworthiness/SKILL.md) — Twyman's law applies doubly to observational effects
- [long-term-effects](../long-term-effects/SKILL.md) — holdouts as an experimental alternative to long-horizon observational claims
- [culture-ethics-memory](../culture-ethics-memory/SKILL.md) — ethics constraints that sometimes force observational designs

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapters 10, 11.
