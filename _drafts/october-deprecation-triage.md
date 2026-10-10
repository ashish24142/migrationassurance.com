---
layout: post
title: "The October purge: a five-day triage for the model deprecation wave"
description: "GPT-4.1 retires October 14 on Azure, GPT-4.1-nano follows on the OpenAI API, and 25+ models shut down this month. A triage plan for teams with no eval infrastructure, and the one piece of evidence you cannot collect after shutdown."
---

This is the week the bill comes due. GPT-4.1 retires on Azure on October 14,
with GPT-5 as the designated replacement. GPT-4.1-nano follows on the OpenAI
API on October 23. They are the tail end of the largest deprecation wave the
ecosystem has seen: more than twenty-five models, including GPT-4, o1 and
o4-mini, gone by the end of this month.

If your migration is done and validated, close this tab. If you're like most
teams I talk to, somewhere between "we think we moved everything" and "we were
going to get to it," this is the five-day version of migration assurance.
Imperfect, but a world better than nothing, and built around one fact that
makes this week genuinely special:

**After shutdown, the old model stops being measurable. Forever.**

Every other part of a migration can be done late. You can validate the new
model next month. You can fix regressions as users report them, painfully. But
the baseline, how the old model actually behaved on your real workload, can
only be captured while the old model still answers. On Wednesday morning, that
evidence is gone, and every future argument about "the system got worse after
the switch" becomes unresolvable. When someone asks in January why extraction
quality changed, "we have no record of the old behavior" is the answer nobody
wants to give an auditor, a client, or their own leadership.

So here is the triage, in priority order. If you only do one day's worth, do
day one and day two.

## Day 1: inventory the exposure

Find every prompt pinned to a dying model. The obvious ones are in your config.
The dangerous ones are not: fallback chains that name a legacy model, batch
jobs, scheduled reports, notebooks that became production, and vendor
integrations that call a deprecated model on your behalf. Search your codebase
for model identifiers, then search your API usage dashboards for the same,
because the dashboard knows about callers your repository forgot.

## Day 2: capture the baseline while it exists

For each exposed workload, assemble a small suite of real, representative
inputs. Fifty cases with known-correct answers beats five hundred without.
Then run every case against the old model N times, five at minimum, and store
every raw response. Two numbers per case matter: the majority answer, and the
flip rate, how often repeated runs disagree with that majority. The flip rate
is what stops you from blaming next week's model for instability your system
always had. This is the step with the hard deadline. Everything else on this
list survives Tuesday. This does not.

## Day 3: run the same suite against the replacement

Same cases, same N runs, on the new model. Score both sides with layered
matching, exact match, then normalized, then numeric equivalence, then a
model-assisted comparison for the genuinely fuzzy cases, logged by layer. Watch
for format drift specifically: answers that are semantically right but shaped
differently break downstream parsers while scoring as "correct."

## Day 4: decide with a rule, not a meeting

Because the results are paired, an exact sign test on the cases where exactly
one model is right gives you a p-value instead of an argument. The defensible
rule: the migration fails only if the new model is worse and the difference is
statistically significant. Define it before you look at the results. Everything
else is a pass, or a named exception with an owner and a follow-up date.

## Day 5: write it down and ship

Verdict up front, per-case evidence behind it, methodology in an appendix,
retained where your change records live. If you operate under ISO/IEC 42001 or
similar, this report is your Clause 8 documented information, produced as a
by-product of doing the migration properly. Then switch, before the platform
switches for you.

## After the switch

Keep the suite. Re-run it on a schedule and after any provider-side change.
The first post-migration week is where silent regressions surface, and a
re-run against your frozen suite finds them before your users do. You now own
something most teams lack: a permanent, versioned record of how your system
behaved across a model generation.

The tooling for all of this, N runs, flip rates, layered matching, the sign
test, the report, is open source: [signoff](https://github.com/ashish24142/signoff)
runs the whole pipeline from one YAML file, on either side of a migration, for
any provider. It exists because I needed it for exactly this situation.

Five days is not enough time to do this well. It is enough time to do it
honestly. And day two cannot wait: capture the baseline while there is still a
model to capture it from.

---

*I'm [Ashish Kumar Singh](https://www.linkedin.com/in/ashish-kr-singh-ml/), an engineering
leader building enterprise LLM systems (contract intelligence today, a decade of clinical AI
before that), and I still write the code. Migration validation, prompt regression testing, and
document-extraction evals are my daily work. This site is about migration assurance;
[signoff](https://github.com/ashish24142/signoff) is its toolkit. All views are my own,
drawn from personal experiments and open-source work, never from client or employer data.*
