---
layout: post
title: "I changed one thing about my $40 contract-review model. It gained ten points. My prediction still missed."
description: "The sequel to the nightwing experiment: swapping generative fine-tuning for an extractive span head moved CUAD AUPR from 0.291 to 0.389 at the same size, data and budget. The model now beats GPT-4o in 25 of 40 categories. The pre-registered 0.50 prediction missed, and the scaling curve from 0.5B to 14B is nearly flat. Every number is public and re-scorable."
---

Last week I published [an experiment]({% post_url 2026-07-13-forty-dollar-specialist-vs-frontier %})
where a $40 fine-tuned 14B model lost to frontier LLMs at contract clause
extraction by 27 points, and I ended it with a bet, written down before spending
another dollar: reframe the task from generation to extraction, and the same
model clears 0.50 on the official CUAD metric.

The experiment ran. Here is the honest scoreboard first, because that is the
deal with pre-registered predictions: you publish the result you got, not the
one you wanted.

| Model | CUAD AUPR (full test split) |
|---|---:|
| Claude Opus 4.8 | 0.561 |
| Claude Opus 4.6 | 0.498 |
| GPT-5.2 | 0.423 |
| GPT-4o | 0.421 |
| nightwing v2 (extractive) | **0.389** |
| nightwing v1 (generative) | 0.291 |

0.389 is not 0.50. The prediction missed and I will get to why. But look at the
two nightwing rows. Same base model. Same training data. Same one-epoch budget,
about thirty dollars of GPU time. The only difference is what I asked the model
to do with the contract.

## The one change

Version one treated contract review the way everyone treats LLMs: as text
generation. Show the model a contract chunk and a question, train it to write
out the answer clause. Version two treats it as what the benchmark actually is:
span extraction. The model does not retype the clause. It points at where the
clause starts and where it ends.

That is the entire change. Pointing instead of retyping. Ten points.

Generation was paying a double tax. The model had to reproduce long legal spans
verbatim, where one paraphrased word breaks the span-overlap scoring, and it had
to do that from a decoder trained to be fluent rather than faithful. Extraction
removes both failure modes at once: the span it selects is by construction
verbatim, and the scoring threshold stops punishing honest reformulation.

## Where the ten points landed

The headline gap to Claude is still real. What moved is everything underneath
it. Against the GPT frontier models, the specialist now wins a majority of
categories: **25 of 40 against GPT-4o, 22 of 40 against GPT-5.2**. In version
one it won 11. It beats all four frontier models simultaneously in six
categories now, up from two, and the margins got silly: Agreement Date 0.829
against a best frontier score of 0.168.

The part I did not expect: the reasoning-heavy categories recovered too. In
version one I concluded that cheap specialists learn annotation conventions
while real semantics stay with the giants, and the collapse categories backed
that up. Version two took Covenant Not To Sue from 0.107 to 0.539, Joint IP
Ownership from 0.184 to 0.500, Anti-Assignment from 0.224 to 0.497. Those are
not date-format conventions. The generative framing was not just taxing the
easy categories; it was hiding capability the model already had.

## The other experiment hiding inside this one

Because the pipeline was cheap, I trained the identical recipe at four sizes.
Dev scores: 0.5B gets 0.284, 1.5B gets 0.289, 7B gets 0.300, 14B gets 0.303.

Twenty-eight times more parameters, two points. The half-billion-parameter
model, which trains overnight on a consumer laptop, already matches what my
14B generative version needed an A100 to reach. If you read the first essay,
this should rhyme: GPT-4o and GPT-5.2, a full frontier generation apart, tied
on this task to within two thousandths of a point. Same lesson from the other
side of the fence. On convention-bound extraction work, scale is almost
decorative. Framing is load-bearing.

## Why the prediction missed

The 0.50 target was not pulled from air. A 0.9B extractive model from the
original CUAD paper scores about 0.47 published. My 14B extractive scored 0.389.
The difference is that the published number comes from the full training recipe:
multiple epochs, every negative example, tuned hyperparameters. Mine was
deliberately lean, one epoch, negatives subsampled two-to-one, first-guess
learning rate, zero tuning, because the question this round was about framing,
not recipe, and I kept every other variable at its cheapest setting.

That is a real miss against a target I set knowing the reference point, so it
counts as a miss. But it localizes the remaining gap precisely: not
architecture, not scale, recipe. Version three changes only that, and the
prediction carries forward with better information behind it.

## What this means if you buy LLM systems

The cheapest lever in this entire series was not a bigger model, more data, or
more GPU hours. It was asking the model for the right kind of output. Before
you benchmark vendors against each other, benchmark the task framing against
itself: the same model, asked two different ways, just produced a ten-point
swing that dwarfs the gap between two frontier generations.

And a note on predictions. Writing 0.50 down in public before spending made
this essay harder to write and easier to trust. My model lost to my own
forecast, in print, and the experiment is still the most useful thing this
project has produced. Vendor decks do not work this way. That is worth
remembering when you read them.

Everything is public and re-scorable at
[github.com/ashish24142/nightwing](https://github.com/ashish24142/nightwing):
raw predictions for every model at every size, the full run journal including
the four dependency failures and the two silent bugs that free local testing
caught before they could burn the cloud budget, and the exact runbook that
produced the numbers. The trained model itself is on
[Hugging Face](https://huggingface.co/ashish24142/nightwing-14b-cuad-extractive)
if you want to run it on your own contracts.

---

*I'm [Ashish Kumar Singh](https://www.linkedin.com/in/ashish-kr-singh-ml/), an
engineering leader building enterprise LLM systems, and I still write the code.
The nightwing code and every prediction file are on
[GitHub](https://github.com/ashish24142/nightwing); the trained model is on
[Hugging Face](https://huggingface.co/ashish24142/nightwing-14b-cuad-extractive);
the paper is at [doi.org/10.5281/zenodo.21530247](https://doi.org/10.5281/zenodo.21530247).*

*Views my own. Not legal advice, and the model still isn't a lawyer.*
