---
name: instrumentation-and-clients
description: Get trustworthy data out of products — client-side vs server-side instrumentation tradeoffs, lossy beacons, clock skew, joining logs across sources, and the instrumentation culture. Also covers mobile/thick-client experiment specifics: app-store release cycles, feature flags and dark features, offline caching, delayed logging, and their experiment-design implications. Invoke when adding logging, debugging missing/mismatched events, or designing an experiment in a native mobile or desktop app.
---

# Instrumentation and Client-Side Experiments

Covers how to log what users and systems actually do, and how experiments differ when code ships inside a thick client (native mobile/desktop app) rather than being served from your servers. The single most important principle: instrumentation is part of the feature — nothing ships without it, and broken instrumentation is a broken feature. Flying a plane with broken panel instruments isn't safe just because the plane still flies.

## When to use

- Deciding what to log, and whether to log it client side or server side.
- Debugging missing events, click undercounts, duplicate events, or impossible timestamps.
- Joining logs from browser, mobile app, and server into one analysis stream.
- Designing an experiment that runs in a native mobile or desktop app.
- Planning feature flags, dark features, or server-side parameters for a client release.
- Interpreting early results from a client experiment (weak signal, biased population).
- Establishing team norms around instrumentation quality.

## Core principles

### Instrumentation

1. **Client-side instrumentation shows what the user experienced; server-side shows what the system did. You need both.** Client side captures: user actions (clicks, hovers, scrolls, times, actions that never hit the server — hover help text, form-field errors, slideshow flips), perceived performance (time to display / become interactive), and errors and crashes (JavaScript errors are common and browser-dependent). Server side captures: response-generation time and slowest component, 99th-percentile performance, request/retry rates, exceptions, cache hit rates. Client-side data is uniquely able to catch things like client-side malware overwriting what the server sent.

2. **Client-side instrumentation costs the user and loses data.** It burns CPU cycles, network bandwidth, and battery; large JavaScript snippets slow page load, and that added latency reduces both engagement on the visit and return likelihood. Web beacons are lossy: when a user clicks to a new site, the beacon can be cancelled by the navigation race, with loss rates varying by browser. Forcing the beacon out first (synchronous redirect) cuts loss but adds latency and increases click abandonment. Choose per application: ad clicks tie to payments and compliance, so they take the reliable-but-slower path; ordinary engagement clicks can accept lossiness for speed.

3. **Never trust the client clock.** Users and systems change client clocks manually or automatically, so client timestamps are not synchronized with server time. Never subtract a client time from a server time — they can be significantly off even after time-zone adjustment. Servers need syncing too: one server can serve a request while another logs the beacon, producing timestamp mismatches within your own fleet.

4. **Server-side logs are less about the user but lower variance and richer in "why."** Time-to-generate-HTML isn't polluted by network noise, so server metrics are more sensitive. Server logs can carry internal state you can never get from the client: ranking scores explaining why search results appeared, which server/data center handled the request (for finding bad equipment or stressed data centers).

5. **Design logs from multiple sources to join.** You will have logs from multiple client types (browser, mobile), from servers, and per-user state (opt-ins/opt-outs). Requirements: a common identifier in all logs as the join key, indicating which events belong to the same user/randomization unit; additionally per-event join keys where a client event ("user saw screen X") pairs with a server event ("why we chose screen X") — two views of the same shown event. Use a shared format: common fields (timestamp, country, language, platform) that become analysis and targeting segments, plus customized fields.

6. **Make instrumentation a cultural requirement, not an engineering afterthought.** The hardest part is getting engineers to instrument at all — the payoff is delayed (code written now, logs examined later) and functionally dissociated (the feature author usually isn't the log analyst). Counter it: (a) cultural norm that nothing ships without instrumentation — put it in the spec, and give broken instrumentation the same priority as a broken feature; (b) test instrumentation during development — engineers see the resulting events before submitting, code reviewers check; (c) monitor raw logs for quality — event counts by key dimensions, invariants (timestamps in valid ranges), outlier detection — and fix detected problems immediately, org-wide.

### Client-side experiments

7. **The client release cycle is not yours.** Server-side changes deploy continuously and take effect instantly with no user action; the variant a user sees is fully server-controlled. Thick-client code ships through three parties: app owner → app store review (can take days) → the end user, who may delay or ignore the upgrade for weeks. Enterprises may block updates; some software (e.g., Exchange in sovereign clouds) can't call unapproved services; OS-style updates requiring reboots can't ship often. So multiple app versions are always live simultaneously, and each update costs users bandwidth and annoyance. Prefer server-side implementation wherever possible — the more logic lives in services, the more agile and consistent experimentation is across clients.

8. **Anticipate changes early: ship every variant with the build, gated by parameters.** All experiments — every variant, including future bug fixes to a variant — must be coded into the current app build; anything new waits for the next release. Consequences: (a) unfinished features ship gated off by feature flags ("dark features"), turned on later when ready; (b) features are built server-configurable so a bad variant can be reverted instantly without a client release — users aren't stuck with a faulty app for weeks; (c) fine-grained parameterization creates new variants post-release without new code: parameterize the number of feed items fetched, or ML model parameters tuned from the server. Windows 10 parameterized the taskbar search-box text and ran experiments over a year after shipping; the winner raised engagement and Bing revenue by millions of dollars. Read app-store policies on shipping dark features and disclose appropriately.

9. **Expect delayed logging and a delayed effective start time.** After activating a client experiment: devices may be offline or bandwidth-limited and never fetch the new configuration; configs fetched at app-open often apply only from the *next* session (don't change a live session), which delays light users — a once-a-week user may start a week late; and many devices still run old app versions — initial adoption takes about a week to stabilize, varying by population and app type. Net effects on analysis: early signals look weaker (small sample) and are selection-biased toward frequent users and Wi-Fi users (early adopters), so extend experiment duration. Treatment and Control can have different effective start times — shared-Control setups may have Control live earlier, with a different population (selection bias) and warmed caches (faster responses), biasing comparisons. Choose the comparison time window carefully.

10. **Telemetry transmission is a tradeoff triangle: connectivity, battery/CPU, storage.** Users can be offline for days; most apps send telemetry over Wi-Fi only, delaying server receipt with country-level heterogeneity in bandwidth and cost. More frequent transmission drains battery (and low-battery mode restricts what apps may do); on-device aggregation saves bandwidth but costs CPU on low-end devices; caching to wait for Wi-Fi costs storage — and bigger apps see reduced downloads and more uninstalls. These tradeoffs affect both your visibility into the client and user engagement itself — worth experimenting on, and worth double-checking when interpreting results.

11. **Build failsafes for offline and first-run.** Cache the experiment assignment on device so the next app-open works offline and stays consistent. If the server doesn't respond with configuration, fall back to a default variant. For OEM-distributed apps, set up the first-run experience: fetch configs that take effect at next startup, and use a randomization ID that is stable before and after sign-up/login.

12. **Triggered analysis needs client-side assignment tracking.** Clients typically fetch assignments for all active experiments at once (e.g., at app start) to save round trips — so logging "assignment fetched" over-triggers massively. Instead, send the assignment/tracking event from the client when the feature is actually used. Watch the event volume: high-volume tracking events cause latency and performance issues.

13. **Guard device- and app-level health, not just engagement.** A Treatment can burn CPU and battery without moving engagement during the experiment window; extra push notifications can drive users to disable notifications in device settings — small now, sizable long-term. Track app size (bigger apps get fewer downloads, more uninstalls), bandwidth consumption, battery usage, and crash rate. For crashes: log a clean exit, so a missing clean-exit flag lets the next app start send crash telemetry.

14. **Evaluate whole-app releases with quasi-experimental methods.** Not every change fits behind an A/B parameter. Truly randomizing the whole app means bundling both versions in one binary — usually impractical (doubles app size). App-store staged rollouts (both Google Play and Apple support them) select eligible users at random but *cannot* be analyzed as randomized experiments: the owner sees who *adopted*, not who was *eligible*. Since old and new versions coexist during adoption, you get an A/B comparison only after correcting for adoption bias — see Xu and Chen (2016) for de-biasing techniques.

15. **Watch multi-device users and cross-platform interactions.** Different devices expose different IDs, so the same user may land in different variants on different devices. Devices interact: "Continue on desktop/mobile" sync, and email links that open the app vs the mobile website. If an experiment causes or suffers such interactions, evaluate user behavior holistically across platforms, not app performance in isolation. Trap: the app experience is usually better than mobile web, so shifting traffic app→web depresses total engagement — a confound, not a treatment effect.

## Workflow

### Adding instrumentation to a feature

1. Write the instrumentation spec alongside the feature spec — what the user sees/does (client), what the system does and why (server).
2. Assign every event the common join key (user/randomization-unit ID) and per-event join keys linking client and server views of the same shown event.
3. Use the shared log format: common fields (timestamp, country, language, platform) + custom fields.
4. Decide the fidelity/latency tradeoff per event class: compliance-critical events (ad clicks, payments) take the reliable path even at latency cost; bulk engagement events can be lossy.
5. For experiments: stamp events with experiment, variant, and iteration IDs; send client-side assignment events at feature use for triggering.
6. Test the instrumentation during development — verify events appear correctly before code review sign-off.
7. Set up raw-log quality monitors: event counts by key dimension, invariants, outlier detection. Treat regressions as ship-blocking bugs.
8. In downstream processing: never mix client and server clocks arithmetically; handle duplicates and timestamp anomalies in cleaning; check whether cleaning removes events unevenly across variants (SRM risk).

### Running a client-side experiment

1. Enumerate every variant (and plausible bug-fix parameter changes) *before* the release; ship them all in the build behind feature flags with server-controlled parameters and safe defaults.
2. Prefer server-side implementation of any logic that can live in services.
3. Parameterize aggressively — counts, text, model parameters — so new variants can be created post-release by config alone.
4. Check app-store policy on dark features; disclose as required.
5. Implement failsafes: cached assignment for offline opens, default variant on config-fetch failure, stable randomization ID across sign-in, first-run config path for OEM installs.
6. Plan client-side assignment tracking at feature use (not config fetch) for triggered analysis.
7. Add device/app health guardrails: battery, CPU, app size, bandwidth, crash rate (clean-exit logging), notification-disablement.
8. Launch, then wait out adoption: expect ~1 week to stable adoption; extend duration beyond your server-side norm.
9. Discount early reads — small samples biased toward heavy/Wi-Fi users. Align Treatment and Control effective start times and comparison windows; beware shared Controls that started earlier (population bias + warm caches).
10. Check for multi-device contamination and cross-platform traffic shifts before attributing effects.
11. If reverting: shut the feature off server side immediately — never wait for a client release.

## Pitfalls

| Pitfall | Why it happens | Detect / avoid |
|---|---|---|
| Undercounted clicks | Navigation race cancels web beacons; loss rate varies by browser | Accept for low-stakes events; synchronous redirect for payment/compliance events (accepting latency) |
| Latency from instrumentation itself | Large JS snippets, chatty telemetry | Budget instrumentation like any latency cost; batch/aggregate; it affects engagement (Chapter-5 effect) |
| Nonsense durations / negative times | Client clock skew vs server; two servers with unsynced clocks | Never subtract client and server timestamps; sync server clocks; validate timestamp invariants in log monitors |
| Can't join client and server events | No common join key planned upfront | Common user-level key in all logs + per-event join keys; shared common fields |
| Features shipped without logging | Time lag + functional dissociation between author and analyst | "Nothing ships without instrumentation" norm; instrumentation in the spec; reviewers check; broken instrumentation = broken feature priority |
| Silent instrumentation rot | No one watches raw logs | Monitor event counts by dimension, invariants, outliers; org-wide immediate fixes |
| Experiment variant can't be fixed for weeks | Bug fix requires a client release through app-store review + user adoption | Server-controlled feature flags; revert by config, not release |
| New variant idea requires new release | Change wasn't parameterized | Parameterize anticipated knobs pre-release (Windows 10 search-box text: experiments a year post-ship) |
| Early client-experiment results look weak or weird | Delayed config delivery, next-session activation, old versions still live | Expect ~1 week adoption ramp; extend duration; don't decide off early biased samples |
| Biased Treatment vs Control comparison | Shared Control started earlier: different population, warm caches | Align effective start times; choose comparison windows after both variants stabilize |
| Inconsistent experience offline | No cached assignment; config server unreachable | Cache assignment on device; default variant fallback; stable pre/post-login randomization ID |
| Over-triggering in analysis | Assignments fetched in bulk at app start, logged as exposure | Log assignment at actual feature use, from the client; watch tracking-event volume |
| Hidden long-term harm despite flat engagement | Battery drain, notification disablement don't show in short engagement metrics | Device/app health guardrails: battery, CPU, app size, crashes, notification settings |
| Missing crash telemetry | Crashed app can't send | Log clean exits; send crash report on next start |
| Staged rollout analyzed as an A/B test | Store rollouts randomize eligibility, but owner only observes adoption | Treat as quasi-experiment; correct adoption bias (Xu & Chen 2016) |
| Same user in different variants on different devices | Different IDs per device/platform | Understand available IDs; analyze cross-device users holistically |
| App→web traffic shift misread as engagement drop | Web experience is worse than app | Flag cross-platform routing changes as confounds; measure across platforms |

## Rules of thumb

- App-store review: days. User adoption of a new version: weeks; ~1 week to reach a stable adoption rate (varies by app and population).
- Extend client-side experiment durations beyond server-side norms to cover config-delivery delay, next-session activation, and version adoption.
- Default telemetry policy in most apps: send over Wi-Fi only — expect corresponding server-side data delay, heterogeneous by country.
- Payment/compliance events (e.g., ad clicks): reliable delivery over low latency. Everything else: usually the reverse.
- Never compute client-time minus server-time. Ever.
- Every log stream: one common user-level join key + shared common fields (timestamp, country, language, platform).
- Ship every variant and every likely parameter knob in the build; assume zero post-release code changes until the next cycle.
- Light users may enter a client experiment up to a session-interval late (a week for weekly users) when configs apply next-session.
- Client-side assignment tracking fires at feature use, not config fetch.
- Bigger app size → fewer downloads, more uninstalls; treat app size as a guardrail metric.
- Broken instrumentation gets the same priority as a broken feature.

## Related skills

- [experimentation-platform](../experimentation-platform/SKILL.md) — the platform components that consume these logs; variant assignment and counterfactual logging
- [trustworthiness](../trustworthiness/SKILL.md) — SRM checks that catch lossy/uneven logging; trust guardrails
- [variance-and-sensitivity](../variance-and-sensitivity/SKILL.md) — triggering, which depends on correct client-side assignment tracking
- [latency-and-performance](../latency-and-performance/SKILL.md) — why instrumentation-induced latency matters for engagement
- [designing-experiments](../designing-experiments/SKILL.md) — duration and power adjustments for delayed client starts
- [ramping](../ramping/SKILL.md) — staged rollouts and ring-based release, including app-store staged rollout limits
- [analyzing-results](../analyzing-results/SKILL.md) — segment analysis over the common fields these logs provide

## Source

Distilled from *Trustworthy Online Controlled Experiments* (Kohavi, Tang & Xu, 2020), chapters 12 and 13.
