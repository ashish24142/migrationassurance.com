---
layout: post
title: "I spent $40 fine-tuning a 14B model to beat frontier LLMs at contract review. It lost by 27 points. Except where it won."
description: "A $40 fine-tuned Qwen3-14B vs Claude Opus 4.8, Claude Opus 4.6, GPT-5.2 and GPT-4o on CUAD contract clause extraction, scored on the official metric. It loses to Claude overall and wins 11 of 40 categories against both GPT models. GPT-4o and GPT-5.2 tie, a full model generation apart. Every prediction is public and re-scorable."
---

Everyone in legal AI is paying frontier prices right now. The pitch behind my
latest experiment was simple: contract clause extraction is a narrow, repetitive,
convention-heavy task. Exactly the kind of thing a small specialist model should
be able to learn. So what does $40 of fine-tuning actually buy against models
that cost $165 per evaluation run?

The honest answer: not victory. Something more interesting. The specialist
could not lay a hand on the reigning giant, Claude. But against GPT-5.2 it won
11 of 40 categories outright, and against GPT-4o, another 11.

## The setup

The benchmark is [CUAD](https://www.atticusprojectai.org/cuad), 510 real
commercial contracts annotated by lawyers across 41 clause categories: governing
law, non-compete, IP assignment, termination, the clauses a lawyer scans for
first. The task: given a contract and a category, extract the exact supporting
text, or say the clause is absent. Scoring uses CUAD's official metric (area
under the precision-recall curve), not a metric I invented.

I measured four frontier models first, through one identical pipeline: Claude
Opus 4.8, Claude Opus 4.6, GPT-5.2, and GPT-4o. Then I fine-tuned Qwen3-14B, an
open Apache-licensed model, with LoRA on CUAD's training split. One A100 for an
afternoon. About $40 of GPU time.

Three rules were fixed before any training started, because a benchmark you can
quietly bend is worthless:

**First, the test set is untouchable.** A script proves zero overlap between
training and test contracts, in code, before every run. Checkpoint selection
happened on a held-out dev split. The test set was scored exactly once.

**Second, the thresholds were pre-committed.** Within 4 points of frontier:
green. 5 to 8 points below: yellow. Worse than that: red. Written down before
the GPU was rented.

**Third, everything is auditable.** Every model's raw predictions are committed
to the [public repo](https://github.com/ashish24142/nightwing). You can re-score
every number in this essay yourself.

## The scoreboard

| Model | CUAD AUPR (full test split) | Cost per evaluation |
|---|---:|---:|
| Claude Opus 4.8 | 0.561 | ~$165 |
| Claude Opus 4.6 | 0.498 | ~$154 |
| GPT-5.2 | 0.423 | ~$13 |
| GPT-4o | 0.421 | ~$58 |
| nightwing-14b (my fine-tune) | 0.291 | ~$0 |

Red band. Twenty-seven points below the best frontier model. If I were selling
you a fine-tuning success story, this essay would end here and you should close
the tab.

Look at the two GPT rows before you scroll on. GPT-4o scores 0.421. GPT-5.2, a
full model generation later, scores 0.423. Two thousandths of a point of
improvement across a generation of frontier progress, on a task worth billions to
the legal industry. Hold that thought.

The per-category table is where it gets interesting.

## Where the $40 model beat all three frontier models

| Category | nightwing-14b | Best frontier | GPT-5.2 |
|---|---:|---:|---:|
| Agreement Date | **0.687** | 0.168 | 0.054 |
| Effective Date | **0.369** | 0.105 | 0.070 |

Not close wins. On Agreement Date the fine-tune scored four times the best
frontier model, and twelve times GPT-5.2. Against GPT-5.2 specifically, the
specialist won 11 of 40 categories outright, including Document Name (0.711 vs
0.471) and Third Party Beneficiary (0.585 vs 0.355).

## The GPT-5.2 scoreline

Category-by-category against GPT-5.2 alone, the $40 model went 11 for 40:

| Category | nightwing-14b | GPT-5.2 |
|---|---:|---:|
| Agreement Date | **0.687** | 0.054 |
| Effective Date | **0.369** | 0.070 |
| Document Name | **0.711** | 0.471 |
| Third Party Beneficiary | **0.585** | 0.355 |
| Expiration Date | **0.633** | 0.589 |

(Plus six narrower wins; GPT-5.2 still takes the overall score, 0.423 to 0.291.)
More than a quarter of the categories, against a frontier model, for the price
of a nice dinner. And the same is true of GPT-4o: 11 of 40 categories again,
with even larger margins (Agreement Date 0.687 vs 0.079, Third Party Beneficiary
0.585 vs 0.102). The gap between "frontier" and "specialist" is not a wall. It is
category-shaped.

Now back to that thought I asked you to hold. GPT-4o and GPT-5.2, a generation
apart, land two thousandths of a point apart on this task. A full cycle of
frontier scaling moved the needle essentially zero. That is the tell: contract
clause extraction is not bottlenecked by reasoning power that more scale would
supply. It is bottlenecked by knowing where a lawyer would draw the span, which
is a convention you learn from examples, not a capability you scale into. That is
exactly the seam a $40 specialist reaches through.

**Why dates, of all things?** Because CUAD scores by span overlap against what
human annotators marked, and date questions are annotation-convention questions.
When you ask for an agreement date, where does the "correct" span start and end?
Just the date? The whole sentence? The clause naming the parties and the date?
Frontier models answer correctly in the colloquial sense and still lose, because
they draw different boundaries than the annotators did. The fine-tune never had
to guess. It learned the convention from 368 contracts of examples. No amount of
frontier-model scale fixes this, because the information simply is not in the
prompt.

And where did the specialist collapse? The categories that need actual reasoning
over meaning. Identifying all parties to a contract: 0.247 vs Claude's 0.954.
Source code escrow, a rare clause it barely saw in training: 0.100 vs 0.800.
The pattern across all 40 categories is clean. Convention-driven fields favor
the cheap specialist. Semantics-driven fields favor scale.

## What this means if you buy LLM systems

The interesting number is not 0.291. It is the shape of the table. A model that
costs nothing per run beat three frontier models simultaneously on the fields
where learned convention matters, and lost everywhere reasoning matters. Nobody
running "which model should we use" evaluations at the whole-benchmark level
would ever see this. Category-level measurement is the whole game. This is the
same lesson as [`signoff`](https://github.com/ashish24142/signoff), from the
other direction: aggregate scores hide exactly the behavior differences you are
accountable for.

There is also a warning here about benchmark readings in vendor decks. My RED
result and my two knockouts come from the same table. Pick your rows carefully
enough and either story sells.

## What's next

The 27-point gap is probably not a capacity limit. A 0.9B-parameter extractive
model from the original CUAD paper scores about 0.47 published, nearly GPT-5.2
territory, at one-fifteenth the size of my model. The difference is task
framing: it points at spans instead of generating them. Version two of this
experiment puts an extractive head on the 14B backbone. Pre-registered
prediction, written down before spending: it clears 0.50. The repo, the run
journal with every failure along the way, and all predictions are public at
[github.com/ashish24142/nightwing](https://github.com/ashish24142/nightwing).

*Update, July 18: version two ran. The prediction missed, and the framing
change still gained ten points. The full result is in
[the sequel]({% post_url 2026-07-18-one-change-ten-points %}).*


---

*I'm [Ashish Kumar Singh](https://in.linkedin.com/in/ashish-kr-singh-ml), an
engineering leader building enterprise LLM systems, and I still write the code.
The nightwing code and every prediction file are on
[GitHub](https://github.com/ashish24142/nightwing); the trained model is on
[Hugging Face](https://huggingface.co/ashish24142/nightwing-14b-cuad-extractive).*

*Views my own. Not legal advice, and the model isn't a lawyer either.*
