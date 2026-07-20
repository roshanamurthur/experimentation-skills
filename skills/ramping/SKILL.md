---
name: ramping
description: Plan and execute a staged rollout of an experiment from 0% to 100% exposure. Invoke when deciding traffic allocation over time, how fast to ramp, how long to hold each ramp stage, when to use holdouts/holdbacks or reverse experiments, or when balancing launch speed against measurement quality and user risk. Trigger keywords: ramp, rollout, staged launch, traffic allocation, MPR, 50/50, holdout, holdback, canary, exposure.
---

# Ramping Experiment Exposure (SQR Framework)

Roll a Treatment out gradually — controlled exposure — instead of flipping to 100% on day one. Every ramp plan is a three-way tradeoff between **Speed** (how fast you launch), **Quality** (how precisely you measure), and **Risk** (how much damage a bad Treatment does before you catch it). The single most important principle: get through the risky low-traffic stages fast, then hold at the **maximum power ramp (MPR — usually 50/50) for a full week** to measure, and treat everything after that as short operational formality.

## When to use

- Deciding the traffic-allocation schedule for a new feature launch.
- Someone proposes launching straight to 100% ("it's just a small change").
- Someone proposes measuring at 5% or 95% "to be safe" instead of 50/50.
- Choosing how long to sit at each ramp stage, or whether a week at 50/50 is really necessary.
- Deciding whether a long-term holdout/holdback is justified, and at what allocation.
- Planning what happens after 100%: code cleanup, reverse experiments, replication of surprising results.
- Designing platform tooling/automation that enforces ramp policy at scale.

## Core principles

1. **Ramp because launch risk is unknown, not because measurement requires it.** If measurement were the only goal, you would run the whole experiment at MPR from minute one — that is the fastest, most precise design. You start small purely to contain damage from bugs, scaling failures, and bad user experiences. Healthcare.gov is the canonical failure: launched to 100% of users on day one, and the site collapsed under load — a rollout by geography or by last name A–Z would have caught it cheaply.

2. **MPR = the allocation that maximizes statistical power given available traffic.** For a two-sample t-test, variance is proportional to `1/(q(1−q))` where `q` is the Treatment traffic fraction — minimized at q = 50%. So with 100% of traffic available and one Treatment, MPR is 50/50. With only 20% of traffic available to the experiment, MPR is 10/10. With four variants splitting 100%, give each variant 25%. Never confuse "safe" with "underpowered": measuring at 5% or at 95% both cost you sensitivity for the same reason.

3. **Each ramp phase has exactly one primary goal — don't blend them.** Phase 1 (pre-MPR): mitigate risk, trade speed vs. risk. Phase 2 (MPR): measure, trade speed vs. quality. Phase 3 (post-MPR): resolve operational/scaling concerns only. Phase 4 (long-term holdout): learn about long-term effects, only when justified. A ramp stage that isn't serving its phase's goal is wasted time.

4. **Ramping too slowly wastes time and resources; ramping too fast hurts users and produces bad decisions.** Slow ramps delay launches and tie up experimentation traffic. Fast ramps skip the safety rings (shipping bugs to everyone) or cut the MPR week short (biasing measurement toward heavy users and weekday users, and missing novelty/primacy trends). Both failure modes get worse as experimentation scales — encode the policy in tooling, don't rely on experimenter judgment each time.

5. **One week at MPR is the default; precision gains beyond a week are small.** A one-day experiment is biased toward heavy users; weekday and weekend users differ, so you need full weekly cycles. Variance keeps shrinking with duration but with sharply diminishing returns — after a week the extra precision is usually negligible unless novelty or primacy effects are present, in which case run longer (see `../long-term-effects/SKILL.md`).

6. **Long-term holdouts are opt-in, not a default ramp step.** Withholding a known-superior experience from paying users has real cost and can be unethical. Run a holdout (e.g., 5–10% of users kept on Control for ~two months) only when: (a) long-term effect may differ from short-term — novelty/primacy area, or an impact so large it must be validated for financial forecasting, or a believed delayed effect via adoption/discoverability; (b) an early-indicator metric moved but the true-north metric is long-term (e.g., one-month retention); or (c) there is a variance-reduction benefit from holding longer.

7. **If the short-term effect is too small to detect at MPR, hold out at MPR — not at 90/10.** The common assumption that holdouts should run 90% or 95% Treatment is wrong for small effects: sensitivity lost by moving from 50/50 to 90/10 usually exceeds sensitivity gained by running longer. Keep the holdout at MPR when the whole point is detecting a small delayed effect.

8. **Replicate surprising results before believing them.** Rerun with a different set of users or an orthogonal re-randomization. If the result holds, trust it. Replication kills spurious errors cheaply, and after many iterations of an experiment the final iteration's estimate is biased upward (winner's curse / multiple testing) — a replication run gives an unbiased estimate.

9. **A launch is not finished at 100% — clean up.** With code-fork architectures, delete the dead code path after launch: an unmaintained dead path that accidentally executes during an experiment-system outage can be disastrous. With parameter systems, promote the new value to the default. Fast-moving teams skip this; don't.

## Workflow

1. **Compute MPR** from available traffic and variant count: with fraction `T` of traffic available and `k` variants (incl. Control), MPR gives each variant `T/k`. One Treatment on full traffic → 50/50.
2. **Phase 1 — pre-MPR safety rings (hours to a couple of days total).** Expose successive rings, watching real-time or near-real-time guardrail metrics at each:
   a. Whitelisted individuals (the implementing team) — verbatim feedback.
   b. Company employees — forgiving of bad bugs.
   c. Beta users / insiders — vocal, loyal, opt-in early adopters.
   d. A single data center at small traffic (Bing: 0.5–2%) — isolates resource-interaction bugs like slow memory leaks or heavy disk I/O; once one data center looks healthy at decent traffic, ramp all data centers.
   Treat early-ring measurements as qualitative or directional only: traffic is too small for power, and insiders are a biased population. Expect to find most bugs here.
3. **Auto-dial traffic up** to each target allocation rather than jumping. Even for a 5% target, taking an extra hour to reach 5% caps the blast radius of a bad bug at negligible schedule cost. Wire guardrail alerts to auto-ramp-down.
4. **Phase 2 — MPR for one week.** This is the measurement phase; all trustworthiness machinery applies here (SRM checks, A/A sanity, guardrails — see `../trustworthiness/SKILL.md`). Hold a full week to cover weekday/weekend cycles and dilute heavy-user bias. Extend beyond a week only if novelty/primacy trends are visible in the daily treatment-effect series.
5. **Decide** ship / abandon / iterate from the MPR read. If abandoning: ramp down to zero immediately — ramp-down is fast by design since the goal is limiting user impact, not measurement.
6. **Phase 3 — post-MPR operational ramps (each a day or less).** E.g., hold at 75% to confirm new services/endpoints scale. By now there should be zero end-user-impact concerns; these ramps exist only for infrastructure load. Cover peak-traffic periods with close monitoring, then go to 100%.
7. **Phase 4 — long-term holdout, only if justified** by principle 6. Typical shape: 5–10% held on Control for ~two months; use MPR allocation instead if the target effect is small (principle 7). Consider org-level alternatives: uber-holdouts withholding a traffic slice from *all* launches for a quarter to measure cumulative impact (Bing holds 10% of users out of all experiments to measure platform overhead), or a reverse experiment — put a small group back into Control weeks or months after the 100% launch.
8. **Post final ramp:** delete the dead code path (fork architecture) or promote the parameter default (parameter architecture). If results were surprising, schedule a replication with orthogonal randomization before institutionalizing the learning.

## Pitfalls

| Pitfall | Why it happens | Detect / avoid |
|---|---|---|
| Launching straight to 100% | "Low-risk change", deadline pressure | Never skip Phase 1; Healthcare.gov is the cautionary tale — even a geo or A–Z rollout would have caught the collapse |
| Measuring at small exposure (e.g., 5/95) and calling it the launch estimate | Conflating safety ramp with measurement ramp | Only the MPR read is the ship/no-ship measurement; variance ∝ 1/(q(1−q)) blows up away from the power-optimal split |
| Trusting metrics from employee/beta rings | Numbers exist, so people read them | Early-ring users are biased "insiders" with no power; use rings for bugs and qualitative feedback only |
| Cutting MPR short (1–3 days) | Impatience; results "look significant" | Day-level results over-weight heavy users and miss weekday/weekend differences; hold a full week |
| Lingering at MPR for weeks "for more precision" | More data feels safer | Diminishing returns after one week absent novelty/primacy; this is the ramping-too-slowly failure |
| Long-term holdout as a mandatory checklist step | Process cargo-culting | Costs traffic and can be unethical (withholding a superior experience from paying customers); require one of the three justifying scenarios |
| Running a small-effect holdout at 90/10 or 95/5 | "Holdouts mean most users get Treatment" | If the effect was undetectable at MPR, longer duration at 90/10 won't recover the lost 50/50 sensitivity — hold at MPR |
| Shipping the estimate from the final iteration of many | Multiple-testing / winner's curse bias inflates it | Replicate with orthogonal re-randomization for an unbiased estimate |
| Dead experiment code left in production | Cleanup skipped in fast-moving dev | Post-launch cleanup task in the launch checklist; dead paths executing during platform outages are disastrous |
| Expecting client/enterprise users to follow the ramp | Large enterprises control their own client-side updates | Treat them as effectively excluded from ramp exposure; see `../instrumentation-and-clients/SKILL.md` |
| Guardrail read too slow to matter | Batch pipelines only | Build real-time/near-real-time guardrail metrics; speed of risk detection sets speed of the whole pre-MPR phase |

## Rules of thumb

- MPR allocation: equal split of available traffic across variants; 50/50 for one Treatment on full traffic; 10/10 if only 20% of traffic is available; 25% each for four variants on full traffic.
- Variance in a two-sample t-test ∝ `1/(q(1−q))` — the quantitative case against measuring at any q far from 0.5.
- Hold MPR for **1 week** by default; longer only with novelty/primacy effects.
- Pre-MPR rings: whitelist → employees → beta/insiders → single data center at 0.5–2% → all data centers.
- Auto-dial to any new allocation over ~an hour rather than jumping, even for small targets like 5%.
- Post-MPR operational ramps: ≤1 day each, covering peak traffic, monitored.
- Long-term holdout: 5–10% of users, ~2 months — but at MPR allocation if chasing a small effect.
- Uber-holdout: a traffic slice (Bing: 10%) excluded from all launches for ~a quarter to measure cumulative impact.
- Ramp down a bad Treatment to zero immediately; ramping down is not a measurement exercise.
- Surprising result → replicate with orthogonal re-randomization before shipping the learning.

## Related skills

- `../designing-experiments/SKILL.md` — power/sample-size math that MPR feeds into
- `../trustworthiness/SKILL.md` — SRM and guardrails to run during the MPR week
- `../long-term-effects/SKILL.md` — novelty/primacy, holdouts, reverse experiments in depth
- `../interference/SKILL.md` — ramp rings (employees, single data center) as interference detectors
- `../variance-and-sensitivity/SKILL.md` — variance reduction when holdout power is tight
- `../experimentation-platform/SKILL.md` — automating ramp policy and auto-ramp-down at scale

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapter 15.
