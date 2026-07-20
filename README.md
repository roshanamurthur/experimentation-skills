# experimentation-skills

Agent skills for designing, running, and analyzing **trustworthy online controlled experiments** (A/B tests), distilled from *Trustworthy Online Controlled Experiments: A Practical Guide to A/B Testing* by Ron Kohavi, Diane Tang, and Ya Xu (Cambridge University Press, 2020).

Each skill is a self-contained operating manual — trigger conditions, core principles, a step-by-step workflow, pitfalls, and numeric rules of thumb — written for an AI coding agent (Claude Code skill format) or a human engineer.

> These are distillations in original words, not reproductions of the book. Buy the book — it is the definitive text and every skill here cites its source chapters.

## The skill map

Start with **[experimentation-principles](skills/experimentation-principles/SKILL.md)** — the whole book's doctrine on one page, linking into the deep dives below.

The book's 23 chapters are re-chunked into 13 deep-dive skills organized around the **experiment lifecycle**, not the book's chapter order:

### Before the experiment
| Skill | Covers | Book chapters |
|---|---|---|
| [designing-experiments](skills/designing-experiments/SKILL.md) | Hypothesis, variants, randomization unit, power & sample size, duration | 1, 2, 14, 17 |
| [metrics-and-oec](skills/metrics-and-oec/SKILL.md) | Goal/driver/guardrail metrics, the Overall Evaluation Criterion, Goodhart's law | 6, 7 |

### While it runs
| Skill | Covers | Book chapters |
|---|---|---|
| [ramping](skills/ramping/SKILL.md) | Speed/Quality/Risk framework, four ramp phases, maximum power ramp | 15 |
| [trustworthiness](skills/trustworthiness/SKILL.md) | Twyman's law, A/A tests, sample ratio mismatch, validity threats | 3, 19, 21 |

### Analyzing results
| Skill | Covers | Book chapters |
|---|---|---|
| [analyzing-results](skills/analyzing-results/SKILL.md) | t-tests, p-values & CIs done right, multiple testing, segments, Simpson's paradox | 2, 3, 17 |
| [variance-and-sensitivity](skills/variance-and-sensitivity/SKILL.md) | Delta method, CUPED, capping, triggering & dilution | 18, 20 |
| [long-term-effects](skills/long-term-effects/SKILL.md) | Novelty/primacy, holdouts, post-period analysis, time-staggered treatments | 23 |

### Special situations
| Skill | Covers | Book chapters |
|---|---|---|
| [interference](skills/interference/SKILL.md) | SUTVA violations, network/marketplace leakage, cluster & geo randomization | 14, 22 |
| [latency-and-performance](skills/latency-and-performance/SKILL.md) | Slowdown experiments, speed → revenue numbers | 5 |
| [when-you-cant-ab-test](skills/when-you-cant-ab-test/SKILL.md) | Complementary techniques, observational causal inference and its pitfalls | 10, 11 |

### Infrastructure & organization
| Skill | Covers | Book chapters |
|---|---|---|
| [experimentation-platform](skills/experimentation-platform/SKILL.md) | Platform components, variant assignment, scaling analysis pipelines | 4, 16 |
| [instrumentation-and-clients](skills/instrumentation-and-clients/SKILL.md) | Client vs server logging, log joining, mobile experiment quirks | 12, 13 |
| [culture-ethics-memory](skills/culture-ethics-memory/SKILL.md) | Maturity models, ethics review, institutional memory & meta-analysis | 4, 8, 9 |

## Install (Claude Code)

Symlink or copy the skills into your skills directory:

```bash
# all skills, globally
for d in skills/*/; do ln -s "$(pwd)/$d" ~/.claude/skills/$(basename "$d"); done

# or per-project
cp -r skills/* your-project/.claude/skills/
```

Then invoke by name (`/designing-experiments`) or let the agent auto-trigger them from the `description` frontmatter.

## The one-paragraph version of the book

Getting numbers is easy; getting numbers you can trust is hard. Randomized controlled experiments are the gold standard for establishing causality, but only if you defend their trustworthiness: check sample ratios before reading metrics, run continuous A/A tests, treat any surprising result as probably a bug (Twyman's law), size experiments for the tiny effects that matter (a 0.5% revenue move can be worth $10M+), pick an OEC that measures long-term value rather than what's easy to game, and let the disagreements between your data and your intuition compound into institutional memory. Most ideas fail — that is why you experiment.
