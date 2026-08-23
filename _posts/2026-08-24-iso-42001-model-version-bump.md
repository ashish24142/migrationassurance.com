---
layout: post
title: "What ISO 42001 change control means when the 'change' is a model version bump"
description: "A model deprecation is a significant change under ISO/IEC 42001 even when zero lines of your code change. Here is how to translate eval engineering into the audit evidence Clause 8 actually asks for."
---

Two emails arrive in the same week. The first is from your model vendor: the
model your AI system runs on will be retired in ninety days. The second is from
your compliance team: the ISO/IEC 42001 certification audit is scheduled for
next quarter, and the auditor wants to walk through your change management
process for the AI system.

Most teams treat these as unrelated emails. They are the same email.

## The change nobody's change process can see

Classic IT change management is built around changes you initiate: a ticket, a
review board, an approval, a rollback plan. A model version bump breaks every
assumption in that pipeline.

The change is initiated by an outside party, on their schedule. It often ships
with no code change on your side: same API, same endpoint, same prompt, and
quietly different behavior. Your change advisory board sees nothing, because
nothing in your repository changed. Your regression suite, if it exists, was
probably written for software whose outputs are deterministic.

So the honest answer many teams would give an auditor who asks "how did you
verify the AI system after the model changed?" is a Slack thread where someone
ran ten examples and wrote "looks fine."

ISO/IEC 42001 does not accept Slack threads.

## What the standard actually asks

Three parts of Clause 8 do the work here.

**Clause 8.1, operational planning and control**, requires you to plan,
implement, and control the processes for operating the AI system, with defined
criteria at each lifecycle stage, and specifically to control planned changes
and to review and mitigate the adverse effects of unintended changes. A vendor
deprecation you saw coming is a planned change. A silent behavior shift under a
stable API version is an unintended one. Both are covered, and both demand
documented control, not vibes.

**Clause 8.2, AI risk assessment**, requires reassessment not just at planned
intervals but when significant changes occur. **Clause 8.4, AI system impact
assessment**, has the same trigger. Which raises the question every AI
governance program eventually has to answer in writing:

*Is a model version bump a significant change?*

For a system that extracts obligations from contracts or facts from clinical
documents, the answer has to be yes. The model is not a component of the
system's behavior. On tasks like these, it mostly is the system's behavior.
Changing it changes everything the risk assessment assessed, while changing
nothing your source control can see.

Once you accept that framing, the compliance question becomes an engineering
question: what evidence shows the post-change system still meets its criteria?

## Translating eval engineering into audit evidence

Here is the translation table I wish more auditors and more engineers had. The
left column is what an evals engineer builds. The right column is what that
artifact is, in the language of a management system audit.

| Engineering artifact | What it is as audit evidence |
|---|---|
| Frozen ground-truth suite, agreed before testing | Acceptance criteria defined prior to the change, immune to after-the-fact negotiation |
| N runs per case, flip rate measured | Measurement system analysis: proof your test instrument is stable enough to detect the change |
| Layered matching, with the matching layer logged | A documented, repeatable measurement method |
| Paired significance test with a pre-defined pass rule | A statistically defended acceptance decision, not an opinion |
| Verdict report with per-case detail, in HTML and JSON | Retained documented information, the thing Clause 8.1 explicitly requires |

The middle rows deserve a word, because they are where LLM systems differ from
everything an audit program has seen before.

LLM outputs are nondeterministic. Run the same prompt on the same document with
the same model several times and some answers change. If your acceptance test
runs everything once, your measurement instrument has noise you never
characterized, and your "regression" findings are partly coin flips. Measuring
the flip rate per case, before and after the model change, is what makes the
rest of the evidence trustworthy. Auditors who come from manufacturing will
recognize this instantly: it is gauge repeatability, applied to a model.

And the pass rule matters more than the pass. A defensible rule looks like
this: the change fails acceptance only if the new model is worse and the
difference is statistically significant on paired per-case results. Defined
before the test runs. Written down. Applied without exception or with named,
bounded exceptions. That single sentence converts an argument about whether 93%
versus 94% "feels different" into a p-value and a decision.

## What to ask engineering for, starting Monday

If you own AI governance and a model deprecation is on the calendar, these are
the five asks. Each one maps to a Clause 8 obligation, and none of them is
exotic.

1. A frozen, ground-truth test suite for every AI system in scope, agreed with
   business owners now, before any migration starts.
2. Flip-rate measurement in the acceptance process: every case runs N times,
   instability is quantified per case, per model.
3. A written acceptance rule with a statistical bar, approved before testing.
4. A migration report per model change: verdict up front, per-case evidence
   behind it, retained as documented information.
5. A standing trigger: any model version change, planned or discovered, opens a
   risk assessment review per Clause 8.2. No silent swaps.

Teams that have this in place discover something pleasant at audit time: the
evidence already exists, because it was produced by the engineering process
itself, not assembled retroactively for the auditor.

## The tooling

The framework above is implemented in [signoff](https://github.com/ashish24142/signoff),
an open-source CLI I maintain: N-run A/B testing between model versions, layered
matching, flip rates, a paired sign test, and a report designed for exactly the
audience in this essay. It is vendor-neutral by design, because evidence that
only works in one vendor's favor is not evidence.

Model deprecations are not slowing down. Neither are certification audits. The
teams that will pass both are the ones that noticed these are the same email.

---

*I'm [Ashish Kumar Singh](https://www.linkedin.com/in/ashish-kr-singh-ml/), an engineering
leader building enterprise LLM systems (contract intelligence today, a decade of clinical AI
before that), and I still write the code. Migration validation, prompt regression testing, and
document-extraction evals are my daily work. This site is about migration assurance;
[signoff](https://github.com/ashish24142/signoff) is its toolkit. All views are my own,
drawn from personal experiments and open-source work, never from client or employer data.*
