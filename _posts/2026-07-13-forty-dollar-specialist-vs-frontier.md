---
layout: post
title: "I spent $40 fine-tuning a 14B model to beat frontier LLMs at contract review. It lost by 27 points. Except where it won."
---

Everyone in legal AI is paying frontier prices right now. The pitch behind my
latest experiment was simple: contract clause extraction is a narrow, repetitive,
convention-heavy task. Exactly the kind of thing a small specialist model should
be able to learn. So what does $40 of fine-tuning actually buy against models
that cost $165 per evaluation run?

The honest answer: not victory. Something more interesting.

## The setup

The benchmark is [CUAD](https://www.atticusprojectai.org/cuad), 510 real
commercial contracts annotated by lawyers across 41 clause categories: governing
law, non-compete, IP assignment, termination, the clauses a lawyer scans for
first. The task: given a contract and a category, extract the exact supporting
text, or say the clause is absent. Scoring uses CUAD's official metric (area
under the precision-recall curve), not a metric I invented.

I measured three frontier models first, through one identical pipeline: Claude
Opus 4.8, Claude Opus 4.6, and GPT-5.2. Then I fine-tuned Qwen3-14B, an open
Apache-licensed model, with LoRA on CUAD's training split. One A100 for an
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
|---|---|---|
| Claude Opus 4.8 | 0.561 | ~$165 |
| Claude Opus 4.6 | 0.498 | ~$154 |
| GPT-5.2 | 0.423 | ~$13 |
| nightwing-14b (my fine-tune) | 0.291 | ~$0 |

Red band. Twenty-seven points below the best frontier model. If I were selling
you a fine-tuning success story, this essay would end here and you should close
the tab.

But the per-category table is where it gets interesting.

## Where the $40 model beat all three frontier models

| Category | nightwing-14b | Best frontier | GPT-5.2 |
|---|---|---|---|
| Agreement Date | **0.687** | 0.168 | 0.054 |
| Effective Date | **0.369** | 0.105 | 0.070 |

Not close wins. On Agreement Date the fine-tune scored four times the best
frontier model, and twelve times GPT-5.2. Against GPT-5.2 specifically, the
specialist won 11 of 40 categories outright, including Document Name (0.711 vs
0.471) and Third Party Beneficiary (0.585 vs 0.355).

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

*Views my own. Not legal advice, and the model isn't a lawyer either.*
