---
name: culture-ethics-memory
description: Build the organizational layer of experimentation — maturity phases (crawl/walk/run/fly), leadership behaviors and cultural tenets, ethics review for experiments on real users (risk, consent, data sensitivity, PII/anonymity), and institutional memory with meta-analysis. Invoke for questions about scaling an experimentation culture, whether an experiment is ethical to run, HiPPO-driven decisions, capturing past experiment results, or mining historical experiments for patterns.
---

# Culture, Ethics, and Institutional Memory

Experiments run on real people inside real organizations: the platform is necessary but not sufficient. This skill covers how organizations mature into "test everything," the leadership behaviors and cultural tenets that make it stick, the ethical review every experiment deserves, and the institutional memory that turns thousands of past experiments into a compounding asset. The single most important principle: the organization must accept that it cannot predict which ideas will win — at well-optimized products only 10–20% of ideas succeed — so the value is in cheap, trustworthy, ethical iteration and in learning from every result, especially failures.

## When to use

- Assessing where an org sits on the experimentation maturity curve and what to build next.
- Advising leadership on goals, incentives, or processes for a data-informed culture ("should we set goals as metrics or features?").
- Deciding whether a proposed experiment is ethical: deception, worse-experience treatments, sensitive data, vulnerable users.
- Answering "do we need consent for this?" or "is this data anonymous?"
- Designing what to record about each experiment, or mining past experiments for patterns, metric sensitivity, or prioritization.
- A postmortem, retro, or "what have all our experiments actually bought us?" question.

## Core principles

### Maturity and culture

1. **Organizations mature through four phases — crawl, walk, run, fly — roughly 4–5x more experiments per phase.**
   Crawl: ~monthly experiments (~10/year); build the prerequisites — instrumentation and basic data science to compute summary statistics for hypothesis testing — and get a few *successful* experiments (ones that meaningfully guide decisions) to generate momentum. Walk: ~weekly (~50/year); define standard metrics, get more teams experimenting, and build trust via instrumentation validation, A/A tests, and SRM checks. Run: ~daily (~250/year); comprehensive metrics, agreed metric sets or a codified OEC capturing tradeoffs, most new features evaluated through experiments. Fly: thousands/year; every change runs as an experiment, feature teams analyze straightforward experiments without data scientists, and the focus shifts to automation and institutional memory. Google, LinkedIn, and Microsoft each run 20,000+ controlled experiments/year.

2. **Leadership cultures evolve from hubris to fundamental understanding — expect the Semmelweis reflex.**
   Stage 1 is hubris: no measurement needed because the HiPPO (Highest Paid Person's Opinion) is confident. Stage 2 is measurement and control, but norms still reject contradictory knowledge (the Semmelweis reflex); paradigm shifts happen only after normal practice visibly fails. Only persistent measurement and experimentation gets an org to the stage where causes are understood and models actually work. Leadership buy-in matters most in Crawl and Walk, when goals are being aligned.

3. **Leaders must change what they reward, not just fund a platform.** Concretely:
   - Set goals as metric improvements, not "ship features X and Y." The deep shift is from "ship unless it hurts key metrics" to "do NOT ship unless it improves key metrics."
   - Engage in establishing shared goal and guardrail metrics, ideally codifying tradeoffs into an OEC.
   - Empower teams to innovate within guardrails; expect most ideas to fail; show humility when their own ideas fail. Establish failing fast as normal.
   - Demand proper instrumentation and high data quality.
   - Review results, enforce interpretation standards (minimize p-hacking), and make decision-making transparent.
   - Maintain a portfolio of high-risk/high-reward projects alongside incremental ones, knowing most will fail.
   - Fund experiments run purely to learn or establish ROI (e.g., slowdown and long-term-effect experiments), not only ship/no-ship decisions. A sequence of experiments can settle strategy: Bing abandoned its Facebook/Twitter integration after two years of experiments showed no value.
   - Shorten release cycles for a fast feedback loop, backed by sensitive surrogate metrics.

4. **Just-in-time process beats mass training.**
   A pre-experiment checklist ("What is your hypothesis? How big a change do you care about?" plus a link to a power calculator) up-levels experimenters without teaching everyone power analysis — Google's search org eventually retired the checklist once the org matured. Experiment review meetings examine trustworthiness first (instrumentation issues are common for first-timers), then produce launch/no-launch recommendations; they establish metric tradeoffs feedable into an OEC, spread learning across teams, and become effective around late Walk/Run — but only among teams sharing a product, metrics, and OEC. Experienced experimenters need hand-holding only their first few times, then become reviewers.

5. **Culture of intellectual integrity: the learning matters most, not whether you ship.**
   Compute many metrics and keep OEC/guardrail metrics highly visible on dashboards so teams cannot cherry-pick. Publicize surprising failures and successes via newsletters, curated feeds, or a discussion layer on the platform (Booking.com attaches a social network to theirs). Make it hard to launch a treatment that hurts important metrics — a spectrum from warning the experimenter, to notifying metric owners, to blocking launch (blocking outright can be counterproductive; open discussion of controversial calls is healthier). Celebrate learning from failed ideas: a third of experiments move key metrics positively, a third negatively, a third not at all (Microsoft; LinkedIn similar) — without experiments you'd ship the positives and negatives alike and they'd cancel.

### Ethics

6. **Ground reviews in the Belmont Report / Common Rule principles: respect for persons, beneficence, justice.**
   Respect = transparency, truthfulness, voluntariness (choice and consent). Beneficence = properly assess and balance risks and benefits. Justice = participants aren't exploited; risks and benefits are fairly distributed. These frameworks came from medicine after Tuskegee and Milgram; online risk is usually far lower, but judgment is still required — there are rarely unambiguous answers. Cautionary anchors: the Facebook/Cornell emotion-contagion study (manipulating post positivity/negativity to measure users' subsequent emotional expression) and OKCupid telling pairs they were 30/60/90% matches independent of their true computed match. Deception and power-of-suggestion experiments on relationships between people are a categorically higher ethical risk than testing features, text, algorithms, or infrastructure.

7. **Place each experiment on the risk spectrum; use the ship litmus test.**
   Minimal risk = harm or discomfort no greater than ordinary daily life (physical, psychological, emotional, social, or economic harm all count). Litmus test: if organizational standards would let you ship the change to 100% of users without an experiment, running it at 50% first is at least as defensible — shipping code *is* an experiment, just an inefficient uncontrolled one. Resisting the experiment while accepting the ship is the "A/B illusion." But testing beats assuming even for well-intentioned changes: a 401(k) mailing telling employees how many peers had enrolled — expected to boost savings via peer effects — *decreased* savings when actually experimented. Equipoise (genuine expert uncertainty between treatments) marks the comfortable zone; deliberately-worse experiences (slowdowns, more ads, disabling recommendations) violate equipoise but are usually justifiable when risk is minimal, there is no deception, and the goal is quantifying tradeoffs that improve the experience for everyone — the analogy is drug toxicity studies. As online treatments reach offline life (shipping physical goods, ride sharing), risk and consequentiality rise; recalibrate.

8. **Consent scales with risk; consider choices participants have.**
   Full informed consent per participant is standard in medicine but prohibitively expensive and annoying at online scale — and most online experiments are low-risk enough not to need it. Treat consent as a range: none needed for minimal-risk changes within user expectations; presumptive consent (ask a smaller representative group how they'd feel about a class of studies, generalize if they agree) for the middle; real informed consent when risk is substantive. Experiments should honor the expectations the product itself sets in its UI and communications. Weigh participants' alternatives: switching search engines is nearly free, so acceptable risk is low-stakes; other services have high switching costs in time, money, and data; in cancer trials the alternative is death, which justifies high risk *with* informed consent.

9. **Know your data-collection answers before running, and classify identifiability honestly.**
   Be able to answer: What is collected, and do users understand that? How sensitive is it (financial, health)? Could it discriminate against users? Can it be tied to an individual? What is it used for, by whom, and is collection necessary — how soon can it be aggregated or deleted? What harm (health, psychological/emotional, social status, financial) if it leaked? What confidentiality do users expect, what safeguards enforce it (access limited, logged, audited), and what redress if guarantees fail? Classify the data: **identified** (stored with PII — names, IDs, phone numbers; HIPAA lists 18 identifiers; device IDs often count; GDPR treats anything linkable to an individual as personal data), **pseudonymous** (stored against a random ID like a cookie), **anonymous** (no PII at all). Pseudonymous/anonymous does not mean re-identification is impossible — **anonymized** data is data where re-identification risk has been analyzed and bounded (Safe Harbor, k-anonymity, differential privacy), and even those quantify rather than eliminate risk. EU practice now distinguishes only personal vs anonymized data. Scale protection — confidentiality, access control, security, monitoring, auditing — with sensitivity and re-identification risk.

10. **Ethics is a cultural process, not an expert checkbox.**
    Everyone from leadership down should understand the questions; introspection is the point. Institutionalize it: cultural norms and education so the questions surface in product and engineering reviews; an IRB-equivalent process that assesses risk/benefit, ensures transparency, and approves/modifies/denies studies; tooling so all data is stored securely with time-limited, logged, audited access and clear acceptable-use policies; and a clear escalation path for anything above minimal risk or with data-sensitivity issues.

### Institutional memory

11. **Capture every experiment as a page in a digital journal.**
    For each experiment record: owner, start date and duration, description and screenshots for visual changes, the hypothesis, results as a definitive scorecard (both triggered and overall impact on key metrics), and the decision made and why. A centralized platform where all changes are tested makes this nearly free. Institutional memory becomes increasingly valuable in the Fly phase — it is the record that lets thousands of experiments compound.

12. **Meta-analysis pays off in five ways.**
    (a) *Culture*: quantify experimentation's contribution to org goals (Bing Ads showed 2013–2015 revenue gains attributable to hundreds of incremental experiment wins); share big or surprising results regularly — people remember examples, not statistics; publish success rates (10–20% at optimized domains; Microsoft's thirds split) to instill humility; track which teams experiment most, what fraction of features launch through experiments, and which outages trace to unexperimented changes — postmortems asking that question change culture, because experiments become a visible safety net. (b) *Best practices*: audit whether experiments follow recommended ramp schedules and are adequately powered; break down by team for accountability; invest in automation for the biggest gaps — LinkedIn found experiments lingering in early ramp phases (or skipping internal beta) and built auto-ramp. (c) *Future innovation*: a catalog of what worked and failed keeps newcomers from repeating mistakes; failed ideas may deserve retries after macro changes; patterns emerge (winning UI patterns, per-country heterogeneity, predictable impact of spacing/bolding/thumbnail changes on a results page) that shrink the search space for new experiments. (d) *Metrics*: measure metric sensitivity — a metric no experiment can move significantly is a bad experiment metric (DAU is notoriously hard to move short-term); build a corpus of trusted past experiments to evaluate new metric definitions; find related metrics and early indicators that lead slow-moving decision-critical metrics; use historical metric movements as Bayesian priors for mature products (rapidly evolving areas make past distributions unreliable). (e) *Empirical research*: historical experiment corpora enable studies like LinkedIn's analysis of 700 People You May Know experiments (2014–2016) showing connections balancing strength and diversity best help people land jobs, Airbnb's correction for the selection bias of launching only significant winners, and using experiment randomization itself as an instrumental variable.

## Workflow

**Assessing/advancing maturity:**
1. Locate the phase by cadence: ~10/year = Crawl, ~50 = Walk, ~250 = Run, 1000s = Fly.
2. Fix the current phase's goal before scaling: Crawl → instrumentation and a few decision-guiding wins; Walk → standard metrics, A/A tests, SRM checks; Run → OEC and near-universal coverage; Fly → automation, self-serve analysis, institutional memory.
3. Get leadership acting on principle 3 — especially metric-based goals — before pushing volume.
4. Install just-in-time education: design checklist with power-calculator link, expert-led result reviews (trustworthiness first), classes as demand grows.
5. Enforce transparency mechanics: visible OEC/guardrails on every scorecard, broadcast of surprising results, friction (warn → notify → discuss) against launching on negative key metrics.

**Ethics review before a sensitive experiment:**
1. Classify the change: feature/algorithm/infrastructure test vs deception or power-of-suggestion study. The latter needs the full IRB-equivalent path.
2. Apply the ship litmus test; check equipoise; enumerate possible harms (physical, psychological, emotional, social, economic) and whether they exceed daily-life risk.
3. State the benefit and who receives it: treatment users, all users via the learning, or the business indirectly. For deliberately-worse experiences, confirm minimal risk, no deception, and that the tradeoff being quantified serves users.
4. Check respect: does the experiment match expectations the product sets? What choices/alternatives do participants have? Decide the consent level (none / presumptive / informed).
5. Answer the data-collection question list (principle 9); classify identifiability; set protection, retention, and access controls to match.
6. Escalate anything above minimal risk through the defined path; document the decision.

**Building institutional memory:**
1. Require every experiment record to include owner, dates, hypothesis, description/screenshots, scorecard (triggered + overall), decision and rationale.
2. Schedule recurring meta-analyses: success rates and goal-contribution (quarterly), best-practice audits (ramp schedules, power), metric sensitivity reviews.
3. Feed findings back: newsletters of surprising results, a trusted-experiment corpus for metric evaluation, priors for Bayesian analysis, pattern libraries for idea prioritization.
4. Wire memory into onboarding — the catalog of past wins and failures is the fastest way for new people to build intuition and avoid repeated mistakes.

## Pitfalls

| Pitfall | Why it happens | How to detect / avoid |
|---|---|---|
| HiPPO decisions / hubris | Confidence substitutes for measurement | Publish success rates (10–20%); require experiments for launch claims; celebrate leaders whose ideas fail publicly. |
| Semmelweis reflex | Orgs reject knowledge contradicting entrenched norms | Expect resistance; accumulate persistent measurement until the paradigm shifts. |
| Feature-shipping goals | Teams graded on shipping X, not moving metrics | Reset goals to metric improvements; flip the default to "don't ship unless it improves key metrics." |
| Cherry-picked results | Teams report only favorable metrics | Compute many metrics; make OEC/guardrails prominent on a shared dashboard; common tracked metric definitions. |
| p-hacking | Pressure to show wins | Interpretation standards enforced in reviews; transparency on how results drive decisions. |
| Review meetings that don't work | Teams too diverse or tooling immature | Reviews need shared product, metrics, and OEC; expect effectiveness only from late Walk/Run. |
| Hard launch-blocks backfiring | Absolute gates kill legitimate judgment calls | Prefer warn → notify → open discussion over automatic blocking. |
| A/B illusion | Experimenting feels riskier than shipping to everyone | Apply the litmus test: willing to ship to 100% ⇒ 50% experiment is fine and safer. |
| Assuming benevolent intent = safe | Well-meant interventions can backfire | The 401(k) peer-information mailing decreased savings; test, don't presume. |
| Deception experiments treated as normal A/B tests | "It's just a variant" framing | Classify deception/power-of-suggestion studies separately (C/D, not A/B); higher-bar review and consent. |
| Calling data "anonymous" carelessly | Pseudonymous ≠ anonymized; re-identification is real | Distinguish anonymous vs anonymized; quantify re-identification risk (k-anonymity, differential privacy); GDPR may treat it as personal data anyway. |
| Collecting data "because we can" | Instrumentation defaults to everything | Require purpose and necessity; aggregate or delete as early as possible; time-limit, log, and audit access. |
| Ethics as expert-only checkbox | Delegation replaces introspection | Norms and education so everyone asks the questions in product/eng reviews; escalation path for >minimal risk. |
| No experiment record | Results live in decks and heads | Mandate the journal fields at experiment creation; centralized platform makes capture automatic. |
| Repeating past failures | No catalog of what was tried | Searchable institutional memory; onboarding through past experiments; deliberate retries only when the macro environment changed. |
| Underpowered / badly ramped experiments at scale | Best practices don't propagate themselves | Meta-analyze ramp schedules and power by team; automate (auto-ramp) the biggest gaps. |
| Selection bias in aggregate impact claims | Only significant winners get launched and summed | Apply a statistical correction when aggregating launched-experiment impact (Airbnb-style). |

## Rules of thumb

- Phase cadence: Crawl ~10 experiments/year, Walk ~50, Run ~250, Fly 1000s — each phase ≈ 4–5x the previous. Google/LinkedIn/Microsoft: 20,000+/year.
- Idea success rates: 10–20% at well-optimized domains (Bing, Google); roughly ⅓ positive / ⅓ negative / ⅓ flat at Microsoft.
- Minimal risk bar: harm/discomfort no greater than ordinary daily life or routine examinations.
- Ship litmus test: acceptable to ship to 100% without an experiment ⇒ acceptable to experiment at 50%.
- Consent range: minimal risk → none; middle → presumptive consent via a representative sample; substantive risk → informed consent.
- HIPAA: 18 identifiers count as PII; device IDs often qualify; GDPR: anything linkable to an individual is personal data.
- Deliberately-worse experiments (slowdowns, more ads, disabled features) are acceptable when: minimal risk + no deception + goal is quantifying tradeoffs benefiting all users.
- Experiment record minimum: owner, dates/duration, hypothesis, description + screenshots, triggered + overall scorecard, decision + rationale.
- A metric no experiment can move statistically significantly is a bad experiment metric; DAU rarely moves in short experiments.
- Checklists and expert reviews are training wheels — retire them once the org is up-leveled; reviewers should emerge from experienced experimenters.

## Related skills

- [experimentation-platform](../experimentation-platform/SKILL.md) — the infrastructure half of the maturity model
- [metrics-and-oec](../metrics-and-oec/SKILL.md) — goal/guardrail metrics and OEC that leadership must codify
- [trustworthiness](../trustworthiness/SKILL.md) — A/A tests and SRM checks that build trust in the Walk phase
- [ramping](../ramping/SKILL.md) — ramp best practices that meta-analysis audits
- [long-term-effects](../long-term-effects/SKILL.md) — learning experiments leadership must fund
- [when-you-cant-ab-test](../when-you-cant-ab-test/SKILL.md) — what to do when ethics rules out a controlled experiment
- [latency-and-performance](../latency-and-performance/SKILL.md) — slowdown experiments as the canonical equipoise-violating-but-ethical case

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapters 4 (maturity models section), 8, 9.
