---
name: latency-and-performance
description: Measure how site/app speed affects business metrics using slowdown experiments. Invoke when quantifying the ROI of performance work, deciding whether latency regressions from a new feature are acceptable, designing a slowdown/latency experiment, choosing page-load-time (PLT) or perceived-performance metrics, or evaluating claims like "the page got slower so revenue dropped."
---

# Latency and Performance Experiments

Quantify what speed is worth by deliberately slowing the product down in a controlled experiment and measuring the damage. You cannot easily speed a site up on demand — engineers would have done it already — but you can always add delay, and under a local linear approximation the measured harm from slowing down estimates the gain from speeding up. The repeated finding across companies: speed matters far more than intuition suggests (Bing 2017: every 100 ms of improvement was worth ~$18M/year in revenue).

## When to use

- Someone asks "is performance work worth funding?" or "what's the ROI of making the site faster?"
- A new feature slows pages down and you need to know if its metric gains survive the latency cost.
- Designing a slowdown experiment: where to inject the delay, how large, constant vs proportional.
- Choosing how to measure page load time or perceived performance for experiment metrics.
- Evaluating whether to embed a third-party blocking JavaScript snippet (personalization/optimization vendors).
- A claimed latency-driven metric change looks extreme ("half a second cost us 20%") or suspiciously null ("200 ms didn't matter").
- Setting latency as a guardrail metric for all experiments.

## Core principles

1. **Slow down, don't speed up.** To isolate latency as the only changed factor, add a controlled delay to Treatment. Speeding up requires engineering breakthroughs; slowing down is a one-line change that yields a clean causal estimate of latency's effect on any metric — revenue, satisfaction, clicks, queries.

2. **The local linear approximation is the key assumption — state it and verify it.** Assume the metric-vs-latency curve is roughly linear near today's operating point (a first-order Taylor approximation), so the harm measured from +100 ms estimates the gain from −100 ms. It is plausible because small deltas (±0.1 s) around the current point are unlikely to hit discontinuities — a 3-second delay could hit a cliff, a tenth of a second is unlikely to. Verify by sampling two delay points: Bing ran both +100 ms and +250 ms slowdowns, and the 250 ms deltas were ~2.5x the 100 ms deltas (within confidence intervals) on key metrics, supporting linearity.

3. **Know the canonical numbers.** Amazon: a 100 ms slowdown decreased sales by 1%. Bing (2012 study): every 100 ms speedup improves revenue by 0.6%. Bing/Google joint study (2009): performance significantly impacts distinct queries, revenue, clicks, satisfaction, and time-to-click. Bing (2015 follow-up, sub-second 95th-percentile server time): the percentage impact on revenue shrank somewhat, but revenue had grown so much that every 4 ms of improvement funded an engineer for a year; in 2017, each 100 ms was worth $18M in incremental annual revenue. Use latency as a guardrail metric on all experiments.

4. **Inject the delay at the right place.** Modern optimized pages ship a first chunk (chrome/frame: header, navigation, JS — independent of the query/URL) immediately, then compute and send the URL-dependent second chunk. Do NOT delay chunk1: it is served with no computation (no plausible "improvement" exists there) and it provides the user's feedback that the request was received, so delaying it measures the wrong thing. Delay the response after the server finishes computing chunk2, simulating a slower back end.

5. **Choose delay size by balancing three forces.** (a) Larger delay → bigger treatment effect → tighter confidence interval on the slope (measuring at 250 ms bounds the slope better than 100 ms with similar CI width). (b) Larger delay → linear approximation less accurate. (c) Larger delay → more real harm to users, since faster is better. Bing settled on 100 ms and 250 ms as reasonable. Use a constant delay when modeling a server-side improvement; a percentage/payload-based delay only if modeling network-relative effects (e.g., users in South Africa already see very slow loads, so a constant 250 ms feels smaller there).

6. **Measure PLT server-side with the beacon-interval trick; keep clocks honest.** User-experienced PLT is T6 − T0 (request initiation to browser onload), which the client can't reliably report — client clocks can be years off, and mixing client/server timestamps across time zones creates garbage (including negative durations; server clock skew does too, so sync server clocks aggressively). Approximate PLT as T7 − T1: server-receipt of the original request to server-receipt of the onload beacon. Both the request and the beacon are small payloads with similar transit times, so the two unknown legs cancel. W3C Navigation Timing values match this approximation well in modern browsers.

7. **`window.onload` is a broken finish line for modern pages; consider perceived performance.** An Amazon page renders above the fold in 2.0 s while onload fires at 5.2 s; Gmail's onload fires at 3.3 s with only a progress bar visible, and above-the-fold content lands at 4.8 s. Progressive rendering (header first) measurably helps. Alternatives, each with caveats: time-to-first-result (lists/feeds like Twitter); Above-the-Fold Time (AFT — needs heuristics and pixel-percentage thresholds for videos/animated GIFs/carousels); Speed Index (average display time of visible elements; robust to trivial late elements, still confused by dynamic above-fold content); Page Phase Time and User Ready Time (phase/essential-element definitions required). A definition-free option: time-to-click, or better time-to-successful-click (click with no bounce-back within 30 s) — robust, but only works when a click is expected; a perfect instant answer produces no click at all. There is no `perception.ready()`.

8. **Latency sensitivity differs by page element — measure per element.** Slowing Bing's algorithmic search results had material impact on revenue and user metrics. Slowing right-pane elements (loaded after `window.onload`) by 250 ms showed no statistically significant impact on key metrics with ~20 million users. Below-the-fold and right-pane latency is far less critical than above-the-fold primary content. Direct performance investment accordingly.

9. **Avoid third-party blocking snippets; assign variants server-side.** Vendor personalization/optimization snippets placed at the top of the page block rendering on a round trip plus tens of KB of JavaScript; placed lower, they cause page flashing. Per the slope you measured, the latency cost can offset any metric gain the tool delivers. Prefer server-side variant assignment and HTML generation.

10. **Be skeptical of extreme latency claims in both directions.** Google's 10→30 search results experiment lost ~20% of traffic and revenue, attributed to +500 ms of generation time — but multiple factors changed at once (30 results vs 10), so latency likely explains only a small share. Etsy reported a 200 ms delay "didn't matter" — more likely the experiment was underpowered than that users don't care; telling an org performance doesn't matter makes the site slow fast, until users abandon in droves. Rarely, too-fast responses reduce trust that work happened (hence fake progress bars). Replicate before believing; a result from one site may not transfer to yours.

## Workflow

1. **Define the question.** Immediate revenue impact per unit of latency? Long-term churn? Whether a specific feature's slowdown negates its gains? Which page elements deserve optimization? The output is a mapping: Δlatency → Δkey metrics.
2. **Pick the injection point.** Delay after the server computes the URL-dependent chunk (chunk2) — never the request-independent chunk1/frame. For element-level questions, delay just that element's render/load (e.g., right-pane after onload).
3. **Pick delay magnitudes.** Default to two points (e.g., 100 ms and 250 ms) so you can both estimate the slope tightly and test linearity by checking that effects scale proportionally (~2.5x for 2.5x delay). Use a constant delay for server-side modeling.
4. **Instrument PLT correctly.** Server-side T7 − T1 approximation via the onload beacon; synchronized server clocks; never mix client and server timestamps. Add perceived-performance metrics suited to the page (AFT/Speed Index for content pages, time-to-successful-click where clicks are expected).
5. **Choose metrics.** Revenue-per-user, sessions, distinct queries, clicks, satisfaction, time-to-click — plus your OEC and standard guardrails. Power the experiment for the small deltas involved (Bing used tens of millions of users; the right-pane null used ~20M).
6. **Run with standard trust checks.** Slowdowns are still A/B tests: check SRM, fixed duration, no peeking (see trustworthiness skill). Keep delays short-lived and bounded — you are deliberately harming Treatment users.
7. **Verify linearity.** Compare the two delay points' effects. If the ratio matches the delay ratio within CIs, report the per-100 ms slope. If not, the linear approximation fails at these magnitudes — shrink the delay.
8. **Translate to decisions.** Convert the slope into currency (e.g., "each 100 ms ≈ 0.6% revenue ≈ $X/year") to size performance teams, gate features that add latency ("this feature adds 80 ms; its conversion gain must exceed the latency cost"), and set latency as a guardrail metric on every experiment.
9. **Repeat periodically.** The slope changes as the product speeds up and revenue grows (Bing 2015 vs 2012). Re-run the slowdown experiment when the operating point moves materially.

## Pitfalls

| Pitfall | Why it happens | Detect / avoid |
|---|---|---|
| Delaying chunk1 / the page frame | Seems like the natural single injection point | Overstates impact and measures an unimprovable path; delay after chunk2 computation instead |
| Trusting the linear approximation blindly | One delay point gives a slope with no linearity check | Run two delay magnitudes; confirm proportional effects (Bing: 250 ms ≈ 2.5x the 100 ms effect) |
| Extrapolating the slope far from today's operating point | Linearity is local (first-order Taylor) | Only claim effects for small deltas (~±100–250 ms); a 3 s change may hit a cliff |
| Using `window.onload` as "page ready" | Historical convention | Above-fold content can be ready far earlier (Amazon 2.0 s vs 5.2 s) or far later (Gmail 3.3 s vs 4.8 s); use AFT/Speed Index/time-to-successful-click |
| Mixing client and server timestamps | Both exist in logs | Time zones and dead-battery client clocks corrupt durations; use server-side T7 − T1 |
| Unsynchronized server clocks | Requests hit different servers | Negative durations in data; sync clocks frequently |
| Concluding "latency doesn't matter" from a null | Underpowered experiment (the Etsy 200 ms claim) | Check power; the right-pane null needed ~20M users to be meaningful |
| Attributing a multi-factor result to latency alone | Confounded changes (Google 10→30 results, −20%) | Only accept latency attributions from experiments where latency was the sole change |
| Believing non-experimental performance case studies | Marketing collections mix confounded changes | Prefer controlled slowdown experiments; replicate on your own site |
| Third-party blocking JS snippets at top of page | Vendor integration requirement | Round trip + tens of KB blocking render; use server-side assignment, or expect the latency cost to eat the gains |
| Treating all page elements as equally latency-sensitive | One global latency number | Run per-element slowdowns; below-fold/right-pane latency may not matter at all |
| Percentage delay when modeling server improvements | Seems fairer across geographies | A server-side fix is a constant absolute saving; use constant delay (payload-based only for network modeling) |
| Ignoring that too-fast can reduce trust | Rare UX effect | For "work being done" flows, users may distrust instant results (fake progress bars exist for this) |

## Rules of thumb

- Bing: 100 ms speedup → ~0.6% revenue improvement (2012); ~$18M/year per 100 ms (2017); 4 ms improvement funded one engineer-year (2015).
- Amazon: +100 ms → −1% sales.
- Reasonable slowdown magnitudes: 100 ms and 250 ms; always run two points to test linearity.
- Linearity check: effect at 250 ms should be ~2.5x the effect at 100 ms, within CIs.
- Constant delay for server-side modeling; delay injected after chunk2 (URL-dependent HTML) is computed.
- PLT ≈ T7 − T1 (server receipt of request → server receipt of onload beacon); matches W3C Navigation Timing.
- Right-pane / post-onload elements: +250 ms showed no significant metric impact even with ~20M users — prioritize above-the-fold latency.
- Expect huge sample sizes: latency effects per 100 ms are fractions of a percent on revenue.
- Make latency a guardrail metric on every experiment; a feature's gains must beat its latency cost using your measured slope.
- Onload-vs-perceived gaps to remember: Amazon above-fold 2.0 s vs onload 5.2 s; Gmail onload 3.3 s vs usable 4.8 s.

## Related skills

- `../trustworthiness/SKILL.md` — Twyman's law, SRM, A/A tests (slowdown experiments still need trust checks; redirects add asymmetric latency)
- `../designing-experiments/SKILL.md` — power and sample size for small deltas
- `../metrics-and-oec/SKILL.md` — guardrail metrics, OEC design
- `../analyzing-results/SKILL.md` — confidence intervals and slope estimation
- `../instrumentation-and-clients/SKILL.md` — client vs server logging, beacon loss
- `../experimentation-platform/SKILL.md` — server-side variant assignment infrastructure

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapter 5.
