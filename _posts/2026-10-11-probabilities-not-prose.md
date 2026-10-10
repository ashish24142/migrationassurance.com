---
layout: post
title: "I asked a probability model 41 questions about 102 contracts. Three bets hit. Two missed."
description: "A pre-registered experiment: can a probability-native model (Jev) do first-pass contract screening with usable calibration? Screening: yes, strikingly. Extraction QA gating: no, not as bet. All numbers, raw outputs, and misses published."
---

Most models answer questions with prose. A newer kind answers with probabilities:
ask "does this contract contain a non-compete clause?" and instead of a paragraph
you get 0.93. The pitch is that the number is calibrated, meaning that when the
model says 0.9 it is right about 90% of the time.

That claim is either very useful or marketing. For contract work it matters more
than raw accuracy, because first-pass contract screening is a triage problem:
nobody expects zero errors, they expect to know which decisions can be trusted
unattended and which need a human. A calibrated probability is exactly that
routing signal. An uncalibrated one is decoration.

So I tested it, the same way I test everything: predictions written down before
spending a rupee, public data, official labels, every raw response archived, and
the misses published at the same size as the hits.

## The setup

Dataset: the CUAD test split, 102 real commercial contracts annotated by lawyers
across 41 clause categories. That yields 4,182 yes/no screening decisions with
official ground truth (about 30% positive).

Model under test: Jev (TypeSafe), a probability-native model, pinned at version
1.13.0. Each contract was chunked and sent with all 41 category questions in
parallel; a category's contract-level probability is the max across chunks, a
rule fixed before any run. Baseline: a current frontier mini model (gpt-5.4-mini)
asked the same questions with the same chunking, answering plain yes/no.

Five predictions, registered before the first API call:

1. Screening AUROC at least 0.85.
2. Expected calibration error at most 0.08.
3. Some confidence threshold auto-clears at least 60% of decisions at 95%+
   precision on the cleared set.
4. As a QA gate for another system's extractions, confidence predicts
   correctness at AUROC 0.80+, and reviewing the least-confident 20% of
   extractions catches at least half of all real errors.
5. The whole experiment stays within a fixed token budget. (It did. Total spend
   was about nine dollars.)

## What hit

**Screening worked, and the calibration is real.** Across 4,182 decisions:
accuracy 86.5%, AUROC 0.942 against a bet of 0.85, and expected calibration
error of 0.045 against a bet of 0.08. When this model says 0.9, it is right
roughly nine times in ten. The reliability curve tracks the diagonal closely at
the confident extremes, which is where real decisions concentrate.

![Calibration curve: when Jev says p it is right about p of the time](/assets/img/act1_calibration_web.png)

![Triage curve: share of decisions auto-cleared vs accuracy among cleared](/assets/img/act1_triage_web.png)

**The triage bet cleared with room to spare.** At a confidence threshold of
0.85, the model auto-clears 73% of all screening decisions, and those cleared
decisions are 96.0% accurate. The bet was 60% at 95%. In workflow terms: for
every hundred clause-presence checks, seventy-three resolve unattended at
near-reviewer accuracy, and twenty-seven go to a person, pre-sorted by exactly
how unsure the model is.

## The twist

The frontier baseline was more accurate than Jev. gpt-5.4-mini, answering the
same questions over the same chunks, scored 88.8% to Jev's 86.5%.

And it does not matter, because the baseline's answers are bare yes/no. There is
no way to know which 88.8% to trust, so every one of its 4,182 answers needs the
same human skepticism. The calibrated model loses the leaderboard and wins the
workflow: it is the difference between "mostly right" and "tells you when it's
right." If your metric for screening systems is raw accuracy, you are measuring
the wrong thing.

## The 3x question

After the main results were in, I added one more arm (disclosed as a post-registration
extension in the prereg file): gpt-6-luna, a probability model served through the same API
at three times the token price. Before running it I wrote down a guess: it would edge Jev's
AUROC by less than 0.02 and not justify the price. The guess was half wrong. It did not
edge Jev at all.

First, the operational part. Luna's input validator rejected 7 of the 41 lawyer-written
category questions outright with an unexplained error, and rephrasing recovered only 3, at
which point I stopped, because tuning questions until a vendor's filter accepts them is not
a benchmark. On the 34 categories it accepted, it then also rejected 3% of requests based on
the document content, unpredictably. The runs below treat every rejection as a data point.

On those same 34 categories, scored identically: Luna reached 86.0% accuracy to Jev's 85.0%,
but with worse ranking (AUROC 0.905 vs 0.929) and roughly double the calibration error
(ECE 0.092 vs 0.055). At the 0.85 confidence threshold Luna clears 89% of decisions, but at
only 90.0% accuracy among the cleared, which no screening workflow should accept. It is
overconfident, and overconfidence is the one flaw a triage system cannot tolerate.

So the 3x spend bought one point of raw accuracy, worse calibration, and a model that
sometimes refuses to answer at all. For this workload, the cheaper model is simply the
better instrument. Price is not a quality signal in this market yet.

## What missed

The second act failed its bets. I tested Jev as a QA gate for extractions made
by other systems: given a contract excerpt and an extracted answer (from
published frontier-model runs with known correctness), the question was whether
Jev's confidence predicts which extractions are wrong.

Weakly, is the honest answer. Confidence predicted correctness at AUROC 0.723
against a bet of 0.80, and reviewing the least-confident 20% of extractions
caught 35% of real errors against a bet of 50%. Better than chance, below the
bar I set, and not good enough to build a QA workflow on in this form. Judging
the presence of a clause appears to be a much easier task than judging the
correctness of someone else's extraction of it, and I under-rated that gap when
I registered the predictions.

![Error-catch curve: fraction of extractions reviewed vs fraction of real errors caught](/assets/img/jev-act2-chart.png)

Both numbers are in the repository with everything else. A finding you only
publish when it flatters the thesis is an advertisement.

## What this changes in practice

For CLM-style screening pipelines, the evidence here supports a two-tier
pattern: a calibrated prober handles clause presence at scale with an explicit
confidence threshold, and humans receive a queue sorted by uncertainty rather
than a firehose sorted by document order. The 73%-at-96% operating point was
achieved zero-shot, with official category definitions as the only prompt
engineering.

For extraction QA, keep human review sampling for now; confidence gating as
tested here does not earn its keep yet.

The experiment code, the pre-registration, every raw API response, the charts
and the per-category results are public at
[github.com/ashish24142/jev-clm](https://github.com/ashish24142/jev-clm), and the
harness slots into
[signoff](https://github.com/ashish24142/signoff) as a judge option with a
confidence threshold and a needs-review bucket, shipping with the Act 2 caveat
documented. Not sponsored, no relationship with the vendor; they will learn
this essay exists when everyone else does.

Probabilities, not prose, turns out to be a real thing, in the place I could
measure it. The place I assumed it would also work, it didn't. Both facts are
the point.

---

*I'm [Ashish Kumar Singh](https://www.linkedin.com/in/ashish-kr-singh-ml/), an engineering
leader building enterprise LLM systems (contract intelligence today, a decade of clinical AI
before that), and I still write the code. Migration validation, prompt regression testing, and
document-extraction evals are my daily work. This site is about migration assurance;
[signoff](https://github.com/ashish24142/signoff) is its toolkit. All views are my own,
drawn from personal experiments and open-source work, never from client or employer data.*
