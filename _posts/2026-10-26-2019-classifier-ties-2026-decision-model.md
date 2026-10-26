---
layout: post
title: "A 2019 classifier ties a 2026 decision model on agent guardrails"
description: "Zero-shot, an open 2026 decision model does no better than a 2019 NLI classifier on 244 agent-guardrail decisions, and neither beats always giving the same answer."
tag: Agent evals
image: /assets/img/decision-models-2019-vs-2026.png
image_alt: "Bar chart: BART-large-MNLI 2019 at 49.2%, Laya 2026 at 52.5%, always the same answer at 67.6%, with overlapping confidence intervals"
---

In September 2026, TypeSafe AI launched **Jev**, a "decision model". You give it the state of a task and a question with a fixed set of options, and it returns typed answers (a choice, a score, a yes/no), each with a probability, in a single pass. Vercel added it to its AI Gateway within days and called it the fastest-adopted model in the gateway's history. The open-source **Laya** works the same way.

The pitch is that this is a new kind of model for agents. I wanted to know something narrower and testable: **is it better at agent decisions than what we already had?**

So I put Laya head-to-head with a zero-shot classifier from 2019 on 244 agent-guardrail decisions.

**The short version:**

- **Laya (2026) and BART-large-MNLI (2019) are statistically tied:** 52.5% vs 49.2% mean accuracy, with overlapping 95% confidence intervals.
- **Neither beats a model that reads nothing** and always gives the most common answer (67.6%).
- **They fail in different places.** Laya is better at spotting destructive commands; the 2019 model is better at recognising a finished task.
- **I couldn't test Jev** (no API access). It has the same interface, so the same three checks apply before you trust it. They're at the end, and they take about ten minutes.

## Why a 2019 classifier is the fair opponent

A decision model takes an input, a question and a fixed set of answers, and returns one probability per answer from a single forward pass. That's a classifier. The vocabulary maps one-to-one:

| Decision-model term | Classic term |
|---|---|
| Choice | multiclass classification |
| Noul (yes/no) | binary classification |
| Options passed in the request | zero-shot classification |
| Probability per option | softmax / sigmoid output |

The closest ancestor is zero-shot classification through natural-language inference (NLI), which has been one line in the `transformers` library since 2019. You pass the text and the candidate answers; the model scores how strongly the text supports each one. No training, options supplied at request time: the same contract Jev and Laya offer.

So if decision models are a real step forward, they should clearly beat a 2019 NLI classifier on the job they're built for. That's the test.

To be fair to the new wave: the product is genuinely better. A typed API with probabilities beats coaxing JSON out of an LLM, and one forward pass is far cheaper than generating text. TypeSafe says Jev is up to 194× faster and 445× cheaper than LLMs in its own evaluations. This post isn't about speed or ergonomics. It's about whether the *decisions* are better.

## The setup

**Benchmark:** [agent-step-guard-bench](https://huggingface.co/datasets/sinhal/agent-step-guard-bench), 61 hand-written coding-agent transcripts, each asked four questions, for 244 decisions:

- `next_action`: continue, finish, ask a human, or stop
- `take_another_step`: yes / no
- `risky`: is the agent's next action destructive or irreversible?
- `complete`: is the task actually done, according to the evidence?

The transcripts are built around the ways agents go wrong: claiming "done" while tests fail, being about to run `DELETE FROM users;`, retrying the same failing command, getting blocked on a missing permission.

**Contenders**, all zero-shot (nobody sees a single label):

| System | Year | Notes |
|---|---|---|
| Always the most common answer | — | reads nothing; the floor |
| `facebook/bart-large-mnli` | 2019 | NLI zero-shot, using the benchmark's own question wording as hypotheses |
| Laya english / multilingual / typed-decisions | 2026 | predictions from [my earlier study](https://github.com/Zephyr4772/laya-zeroshot-agent-guard-eval), all three checkpoints |

**Scoring:** accuracy per question and the mean across the four; AUROC for the yes/no questions (0.5 is chance, below 0.5 means the score points the wrong way); 95% bootstrap confidence intervals over the 61 transcripts.

## Results

| System | Year | Mean accuracy [95% CI] | Risky AUROC | Complete AUROC | Another-step AUROC |
|---|---|---|---|---|---|
| Always most common answer | — | **67.6%** [64.3, 70.5] | 0.50 | 0.50 | 0.50 |
| BART-large-MNLI | 2019 | 49.2% [43.0, 54.9] | 0.53 | **0.65** | 0.39 |
| Laya english | 2026 | 45.9% [39.8, 51.6] | 0.59 | 0.38 | 0.39 |
| Laya multilingual | 2026 | 46.7% [41.0, 52.9] | 0.64 | 0.50 | 0.43 |
| Laya typed-decisions | 2026 | 52.5% [45.9, 58.6] | **0.73** | 0.55 | 0.32 |

Laya's best checkpoint is 3.3 points ahead of the 2019 model on mean accuracy. The intervals overlap almost entirely, so on 61 transcripts that gap is indistinguishable from noise.

Both are well below the constant baseline. On this benchmark, "always say *continue*, never risky, never complete" is right more often than either model.

## Where each one fails

The averages hide two different failure profiles.

**Laya is better at danger.** It separates destructive commands from safe ones better than the 2019 model (AUROC 0.73 vs 0.53). It rates `kill -9` on a dev server as high-risk. But it never turned that into an escalation: **0 of 8** risky transcripts got `ask_human`, on any checkpoint.

**The 2019 model is better at "done".** It recognises a genuinely completed task better (AUROC 0.65 vs 0.55). Laya's typed-decisions checkpoint, given a tool output of *12 passed, 2 failed* followed by the agent saying it's finished, chose `finish`. Across the six transcripts like that, it got **0 of 6** right: every time, it took the agent's word over the evidence.

**Both get "should the agent keep going?" backwards.** AUROC 0.32 to 0.43 means their confidence leans the wrong way: flipping the score would do better than using it.

## What this means for Jev

I can't tell you how Jev scores, and I won't guess. What these results do say is that the interface alone, typed answers with probabilities, doesn't guarantee better decisions than a 2019 classifier. The decision quality has to be measured, the same way it is for any classifier.

If you're adopting Jev or any decision model as an agent guard, run these three checks first.

**1. Beat the constant answer.** Count how often the most common label is correct on your data. If the model doesn't clearly beat that number, its probabilities aren't telling you anything.

**2. Beat the 2019 baseline.** It's one line:

```python
from transformers import pipeline
nli = pipeline("zero-shot-classification", model="facebook/bart-large-mnli")
nli(transcript, candidate_labels=["the agent should keep working", "the task is done",
                                  "a human must approve", "the agent is stuck"])
```

If the new model doesn't clearly beat this with non-overlapping confidence intervals, you're paying for the API and the speed, not for better judgment.

**3. Test on situations it hasn't seen.** Group your examples by *type* of situation (false success, destructive command, loop…) and evaluate one whole type at a time. Random splits flatter anything that's seen near-copies of the test case. I learned this the hard way below.

## A note on trained baselines

I also trained two old-school classifiers on the labels: TF-IDF with logistic regression, and a logistic regression on MiniLM sentence embeddings. With ordinary cross-validation, TF-IDF reached **73%**, the best number of the lot.

It didn't hold up. When a whole scenario type was held out, it fell to **56%** (and the MiniLM version to 55%), below the constant baseline. In this benchmark the scenario type almost decides the answer, so a trained model just memorises templates. That's why the headline comparison is zero-shot only, and why check 3 above exists.

The same limit applies to everything here: 61 templated transcripts are good at exposing failure modes, not at producing a leaderboard. The tie between Laya and BART-MNLI means "no measurable difference at this size", not "identical".

## Takeaways

1. **On agent-guardrail decisions, the 2026 open decision model doesn't measurably beat a 2019 zero-shot classifier:** 52.5% vs 49.2%, overlapping intervals.
2. **Neither is a guardrail out of the box.** Neither beats the constant answer, Laya never escalated a destructive command to a human, and its typed-decisions checkpoint trusted "done" over failing tests every time.
3. **Before trusting Jev or any decision model, run the three checks above** on your own transcripts: the constant answer, the one-line 2019 baseline, and held-out situation types.

Everything is public: the [benchmark](https://huggingface.co/datasets/sinhal/agent-step-guard-bench), the Laya predictions, and the [baseline scripts](https://github.com/Zephyr4772/laya-zeroshot-agent-guard-eval/tree/main/baselines). If you have Jev access, run it and send me the numbers. I'll add the row.
