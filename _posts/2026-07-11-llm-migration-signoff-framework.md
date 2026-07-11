---
layout: post
title: "Is the new model safe to deploy? A sign-off framework for LLM migrations"
---

The email every AI platform team eventually gets arrived on a Tuesday: the model
our document-extraction system had been built on was being deprecated. We had a
few months to move a fleet of production prompts to the successor model — prompts
that extract facts from legal and healthcare documents, where a wrong answer
isn't an inconvenience, it's a liability.

The obvious question from leadership was simple: **"Is the new model safe to deploy?"**

The honest answer, at that moment, was: *"We don't know, and we don't even have a
way of knowing."* This essay is about the framework we built to answer that
question properly — and why the standard approach (run a few examples, eyeball
the outputs, ship it) would have quietly hurt us.

## Why eyeballing fails

Our system extracts dozens of attributes from documents — governing law,
effective dates, party names, monetary amounts, renewal terms. A migration
"spot check" looks like this: run twenty documents through the new model, diff
the answers, see mostly agreement, call it a day.

Three problems.

**First, the sample is too small to see rare regressions.** If the new model breaks
one attribute on a few percent of documents, twenty samples will miss it more often
than not — and a few percent of a production document flow is a lot of wrong
answers with a compliance label on them.

**Second, "the same" is not a well-defined concept for LLM output.** Is
`$250,000,000` the same as `250000000.00 USD`? Is `State of New York` the same
as `New York`? Naive exact-match scoring will tell you the new model "failed" on
a large share of cases that any human reviewer would score as identical. Scoring
is a pipeline, not a string comparison.

**Third — and this is the one almost everyone misses — the old model doesn't agree
with itself.** Run the *same* prompt on the *same* document with the *same* model
five times, and on some attributes you get different answers on different runs.
On our worst prompts, repeated runs disagreed with each other not occasionally
but routinely — the kind of instability UAT teams file as "the system changed
its mind on refresh." A single-run comparison between old and new models is
therefore partly measuring *luck*. You cannot tell a regression from a coin flip.

That third problem has a name — nondeterminism — and it reshaped the whole
validation design. Before you can compare two models, you have to measure how
unstable each one is on its own.

## The framework: N runs, layered matching, and a statistical verdict

Here is the shape of what we ended up building. None of it is exotic; the value
is in the discipline of doing all of it, every time.

**1. A frozen validation suite.** Real (anonymized) documents with ground-truth
answers for every attribute, agreed with the business owners *before* migration
testing started. If you negotiate what "correct" means after you've seen the new
model's outputs, you will negotiate your way into shipping it.

**2. N runs per prompt per model — not one.** We ran every case multiple times on
the old model and the new. From those runs, two numbers per case:

- the **majority answer** — what the model "really" says;
- the **flip rate** — how often runs disagree with that majority.

The flip rate is the diagnostic gold. A case that changed answers between models
but is unstable on the *old* model too isn't a migration regression — it's a
prompt that was never reliable to begin with. Our "regressions" bucket shrank
substantially once we could separate the two, and the unstable-prompt bucket
became its own workstream (fix the prompt, not the model).

**3. Layered matching instead of exact match.** Each answer is scored through
escalating layers: exact match → normalized match (case, whitespace,
punctuation) → numeric equivalence (`$1,000` vs `1000`) → semantic/LLM-judge
comparison for the genuinely fuzzy cases. Every match records *which layer*
passed, and the judge layer is used sparingly and audited by humans on a sample —
because "a model said the model is right" is not, by itself, evidence.

**4. A paired statistical test, not a vibes delta.** Suppose the new model scores
94% on your suite and the old one 93% — improvement, or noise? With per-case
paired results you can run an exact sign test on the discordant pairs and get a
p-value instead of an argument. Our rule: the migration **fails** sign-off only
if the new model is worse *and* the regression is statistically significant.
Everything else is either a pass or a named, bounded exception.

**5. A report a risk committee can read.** The deliverable was never a Jupyter
notebook. It was a short report: verdict up front, per-attribute table (accuracy
old/new, flip rates, match layers used), the list of every regressed case with
its documents, and the methodology in an appendix. The methodology section is
what turns "the team says it's fine" into *evidence* — the thing you can hand an
auditor when they ask, a year later, why you approved the swap.

## What the framework changes

Once it was in place, the migration verdict came back genuinely mixed — which is
exactly why the framework matters:

- Most attributes cleared cleanly: new model equal or better, sign-off unambiguous.
- A meaningful share of *apparent* regressions turned out to be instability —
  high flip rates on both models. Those became prompt fixes, not migration blockers.
- And a small number of real, statistically significant regressions were caught
  before production — the cases an eyeball check would have shipped.

That last bucket is the entire ROI of the framework. One real regression caught
before deployment pays for the whole validation effort — in domains like ours,
several times over.

## The tooling, open-sourced

I've generalized the core of this framework into **[signoff](https://github.com/ashish24142/signoff)** —
an open-source CLI that runs a YAML-defined suite on two models, N runs each,
scores through layered matching, computes flip rates and the sign test, and emits
an HTML + JSON report. It's vendor-neutral by design: it judges an OpenAI→Anthropic
migration with exactly the same rigor as the reverse.

If you're facing a forced migration this year — and with current model
deprecation schedules, you probably are — the checklist version of this essay is:

1. Freeze a ground-truth suite before you start.
2. Run everything N times; measure flip rates before you interpret diffs.
3. Score with layered matching; log which layer matched.
4. Demand statistical significance before calling something a regression — or a win.
5. Write the report for the person who has to defend the decision, not the
   person who ran the notebook.

"Is the new model safe to deploy?" deserves a better answer than "probably."
It deserves a sign-off.

---

*I'm [Ashish Kumar Singh](https://in.linkedin.com/in/ashish-kr-singh-ml) — an engineering
leader building enterprise LLM systems (contract intelligence today, a decade of clinical AI
before that), and I still write the code. Migration validation, prompt regression testing, and
document-extraction evals are my daily work. This site is about migration assurance;
[signoff](https://github.com/ashish24142/signoff) is its toolkit. All views are my own,
drawn from personal experiments and open-source work, never from client or employer data.*
