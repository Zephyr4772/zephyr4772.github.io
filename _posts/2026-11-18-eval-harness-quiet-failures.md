---
layout: post
title: "Eval harnesses fail in two ways, and the quiet one is worse"
description: "Fixing bugs in three open-source LLM and VLM evaluation harnesses: a math metric that marked 7 = 7 as wrong, results that vanished while the run reported success, prompts that showed every image twice, and benchmarks whose data had quietly disappeared from the Hub."
tag: Evaluation
image: /assets/img/eval-harness/cover.png
image_alt: "lighteval on Windows scores 7 = 7 as wrong and fails 233 of 233 answer-matching tests. Four quiet failures: math metric run finishes but returns False; saved results reported success but 0 bytes on disk; chat prompt run finishes with stray <image> text; 32 tasks listed but pointing at missing data"
---

Over the past few weeks I sent fixes to three open-source evaluation harnesses: **lighteval** (Hugging Face), **lmms-eval** (LMMs-Lab) and **evalscope** (ModelScope). These are the tools that turn "run benchmark X on model Y" into a number in a paper, a leaderboard or a model card.

Most of the bugs were loud. Something crashed, I read the traceback, I fixed it. A few were quiet: the run finished, a number came out, and the number was wrong or the results were gone. **The quiet ones are the ones worth writing about**, because nothing tells you they happened.

**The short version:**

- **A wrong score.** On Windows, lighteval's math metric marked every answer as wrong, including `"7"` against `"7"`. All 233 of its answer-matching tests fail on Windows. Nothing crashes, and the score is just low.
- **Results that vanish.** On Windows, lighteval reported success after saving per-sample results for tasks like `truthfulqa:mc|0`. The file on disk was empty. The data was hidden in an NTFS side channel that the loader never reads.
- **A wrong prompt.** lmms-eval's chat path sent each image to the model twice: once as the processor's image token and once as a literal `<image>` string. Some models crash on that. Others get stray text in the prompt and answer anyway.
- **Data that disappeared.** Benchmarks point at datasets on the Hugging Face Hub, and those datasets change. I checked every task config in two harnesses against the Hub (52,776 tasks in lighteval, about 1,800 configs in lmms-eval). I found 18 lmms-eval tasks and 14 lighteval tasks pointing at data that doesn't exist. Some of it never existed at all.

## Quiet failure 1: a metric that marks the right answer wrong

lighteval's math metrics parse the model's answer and compare it to the gold answer with `sympy`. Parsing can hang on adversarial input, so both steps run under a `timeout` decorator.

On Linux and macOS, `timeout` uses `signal.alarm`. Windows has no `SIGALRM`, so there it starts a `multiprocessing.Process` instead. Windows starts processes with `spawn`, which has to pickle the function to send it to the child. The function was defined inside the decorator, and a local function can't be pickled:

```
AttributeError: Can't get local object 'timeout.<locals>.decorator.<locals>.wrapper.<locals>.run_func'
```

That would be a loud failure, except the callers catch the exception and treat it as "couldn't parse" or "doesn't match". So every comparison returns `False`:

| on Windows 11, Python 3.12 | before | after |
|---|---|---|
| `compare_gold_target(["7"], ["7"])` | `False` | `True` |
| extract `$\boxed{\frac{1}{2}}$` | `[]` | `[1/2]` |
| lighteval's own answer-matching tests | **233 failed** | 233 passed |

The evaluation finishes and reports a math score near zero. If you only look at the number, a low score on a small model looks plausible.

The fix runs the function in a thread with `join(timeout)` on systems without `SIGALRM`. The new tests run the Windows code path on Linux too, by forcing `os.name`, because lighteval's CI only runs on Linux. That's how this survived: [huggingface/lighteval#1494](https://github.com/huggingface/lighteval/pull/1494) (open; the bug report, #1108, is from December 2025).

## Quiet failure 2: results that save successfully and are gone

lighteval names files after tasks: `gsm8k|0`, `truthfulqa:mc|0`. Windows doesn't allow `|` or `:` in file names. That causes two failures:

- The sample cache, on by default, crashes every evaluation with `[WinError 123]`. That one is loud.
- `--save-details` writes `details_truthfulqa:mc|0_<date>.parquet`. That one is quiet.

NTFS reads `name:stream` as a file with an *alternate data stream*: a second, hidden stream of bytes attached to a file. So the run reports success, and what's left on disk is an empty file called `details_truthfulqa`. The parquet with every per-sample prediction is attached to it, invisible in Explorer and `dir`. `load_details_datasets` never finds it.

The fix percent-encodes task names when they become paths (`gsm8k%7C0`) and decodes them when reading back, so names stay the same everywhere else. An earlier report of the same problem was closed as completed in May 2025, but it still happened on `main`: [huggingface/lighteval#1497](https://github.com/huggingface/lighteval/pull/1497) (open).

## Quiet failure 3: a prompt that shows every image twice

lmms-eval supports two kinds of model wrappers. "Simple" models get a text prompt with `<image>` placeholders. "Chat" models get structured messages, where each image is its own item and the model's processor inserts its own image token.

Many tasks only define the simple prompt, so lmms-eval converts it for chat models. The conversion put all the images first and kept the literal `<image>` placeholders in the text. The model then sees each image twice:

- **Models whose image token is the string `<image>`** (LLaVA, Idefics3/SmolVLM) count four image tokens for two images and crash: `Found [4] <image> tokens and [2] images per sample.` That's loud.
- **Every other model** gets the images, plus the literal text `<image>` where each image used to be. That's quiet. The model answers and gets scored, on a prompt the benchmark authors never wrote.

STARE, MMT, Geometry3K, MathCanvas and PhysBench were affected. The fix places each image at its marker when the counts match, the way tasks with hand-written chat prompts already did: [EvolvingLMMs-Lab/lmms-eval#1671](https://github.com/EvolvingLMMs-Lab/lmms-eval/pull/1671) (merged).

That fix had a gap, which someone else then closed. I only handled prompts where the number of markers equals the number of images. Another contributor ran it on the full MMT dataset and found 559 multi-frame rows where it doesn't, and fixed those in a follow-up PR. My PR named MMT as fixed, so the gap was visible. Since then I check every affected task on real data before calling it fixed, not one example.

## Rot: benchmarks whose data is gone

Harnesses don't ship benchmark data. A task config says "load config X, split Y of Hub dataset Z", and the data is fetched at run time. Dataset owners rename splits, restructure repos and delete configs. Nobody tells the harness.

So I checked every task config against the Hub's dataset API, without running any of the harness code:

| harness | task configs checked | broken | example |
|---|---|---|---|
| lmms-eval | about 1,800 (pointing at about 370 Hub datasets) | 18 fixed, 3 flagged | MEGA-Bench's single-image configs were deleted from the Hub in May 2025 |
| lighteval | 52,776 | 14 fixed or removed | five WikiFact relations whose data files were empty from the very first upload |

Some details:

- **STARE (14 tasks):** the Hub copy only has a `train` split, so `test` fails. The fix points them at `train`.
- **MEGA-Bench single-image (2 tasks):** the configs were removed upstream. The fix pins the tasks to the last revision that has them.
- **lmms-eval video-tt (2 tasks):** their splits were deleted from the Hub *before* the tasks were added. They never worked. I flagged them for the maintainers rather than guess.
- **lighteval storycloze:2018:** the Hub repo never had 2018 data. **xcopa:en:** XCOPA has no English split.

These fail loudly when you run them. But in a harness with thousands of tasks, nobody runs all of them. A task can sit broken in the registry for months, listed as available. lmms-eval [#1667](https://github.com/EvolvingLMMs-Lab/lmms-eval/pull/1667) was merged. lighteval [#1500](https://github.com/huggingface/lighteval/pull/1500) is open.

## The loud ones: version churn

Loud failures still matter, but they get noticed and fixed quickly. Most of mine came from library updates:

- **transformers 5** rejects the old `audios=` processor argument. lmms-eval's generic chat model passed `audios=[]` on every request, so with transformers 5 every request failed ([#1668](https://github.com/EvolvingLMMs-Lab/lmms-eval/pull/1668), merged).
- **transformers 5.23** started calling `.items()` on each chat message. evalscope passed message objects, so local-checkpoint evaluation failed on every sample the day 5.23 came out ([modelscope/evalscope#2084](https://github.com/modelscope/evalscope/pull/2084), merged). The maintainer improved my fix before merging: my version dropped keys whose value was `None`, which changed the output of older chat templates that test for them.

## What I check now before trusting an eval number

1. **Score a known answer first.** Feed the metric the gold answer as the prediction. If it doesn't score 100%, nothing downstream means anything. That alone would have caught the Windows math bug.
2. **Read one rendered prompt per task**, exactly as the model receives it, after the chat template. Stray `<image>` text is obvious once you look.
3. **Confirm the per-sample results exist and load** before reading the summary score.
4. **Pin dataset revisions** in task configs, or check them against the Hub regularly. If you can't reproduce last month's run, the data may have changed.
5. **Run on a platform the CI doesn't cover.** Two of the three quiet bugs only show up on Windows, and lighteval's CI runs only on Linux.

## Status

| PR | what | status |
|---|---|---|
| [lighteval#1494](https://github.com/huggingface/lighteval/pull/1494) | math metric marks every answer wrong on Windows | open |
| [lighteval#1497](https://github.com/huggingface/lighteval/pull/1497) | details hidden in NTFS streams, cache crash on Windows | open |
| [lighteval#1500](https://github.com/huggingface/lighteval/pull/1500) | tasks pointing at Hub data that doesn't exist | open |
| [lmms-eval#1671](https://github.com/EvolvingLMMs-Lab/lmms-eval/pull/1671) | chat models shown every image twice | merged |
| [lmms-eval#1667](https://github.com/EvolvingLMMs-Lab/lmms-eval/pull/1667) | 18 tasks pointing at Hub data that doesn't exist | merged |
| [lmms-eval#1668](https://github.com/EvolvingLMMs-Lab/lmms-eval/pull/1668) | every request fails with transformers 5 | merged |
| [evalscope#2084](https://github.com/modelscope/evalscope/pull/2084) | local-model evals fail with transformers 5.23 | merged |

I used Claude Code to help find and write these fixes, and said so in each PR. I reproduced each bug and ran the checks myself.
