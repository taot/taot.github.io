---
title: 'Strata Benchmark: Qwen3.8-Flash-Next Q2 on an RTX 5090'
date: 2026-10-19T07:54:25-04:00
draft: false
categories: ['experiments']
tags: ['llm', 'benchmark', 'strata', 'local-models']
summary: 'Quality and speed of Qwen3.8-Flash-Next (Q2_0) served by Strata on one RunPod RTX 5090, with thinking on and off.'
ShowToc: false
---

[Strata](https://github.com/Niko1221/Strata) runs Qwen3.8-Flash-Next, a mixture-of-experts model, on one GPU plus system RAM. I set it up on a RunPod RTX 5090 pod (see the [setup notes](https://github.com/taot/local-small-models/tree/master/strata)) and ran my benchmark tool, `smolbench`, against it. This post gives the results.

**Summary:**

- With thinking on, the quality is good: IFEval 0.92, MMLU-Pro 0.90, LiveCodeBench-Pro (easy) 0.83, BFCL 0.67.
- Decode stays near **89 tok/s** with prompts of up to 32K tokens. Prefill reaches 2,700 tok/s at 32K.
- With thinking off, coding falls to **0.50**. In most failures, the model writes its reasoning as plain text and then stops before it writes the code.

## 1. Setup

| | |
|---|---|
| Model | Qwen3.8-Flash-Next, Q2_0 quant |
| Engine | Strata 0.1.45 (commit `477ac13c`) |
| Hardware | RunPod RTX 5090 32 GB, 108 GB RAM |
| Connection | SSH tunnel to the pod, one request at a time |
| Benchmark level | `quick` (small subsets, 1 repeat) |
| Max output tokens | 16,384 |
| Thinking | on and off, set with `chat_template_kwargs.enable_thinking` |
| Date | 2026-10-19, total run time about 2 h 20 min |

The full config is in [config.yaml](config.yaml). All 8 evals completed. No sample had an error, and all smoke checks passed (chat, tool call, thinking on, thinking off, no hidden system prompt).

## 2. Quality

For reference, the table also shows a run of DeepSeek-v4.1-flash through an API on 2026-10-17. That is a large hosted model, so use it only as a rough guide.

| Test (samples) | Thinking on | Thinking off | DeepSeek-v4.1-flash (on / off) |
|---|---|---|---|
| IFEval (100) | **0.919** | 0.890 | 0.955 / 0.947 |
| MMLU-Pro (50) | **0.900** | 0.820 | 0.860 / 0.800 |
| LiveCodeBench-Pro, easy (30) | **0.833** | **0.500** ⚠️ | 0.767 / 0.700 |
| BFCL (100) | 0.670 | 0.570 | 0.690 / 0.710 |

The subsets are small, so the standard error is 4 to 9 points. Do not read too much into differences of a few points. The raw scores and all sub-metrics are in [evals.json](evals.json).

Thinking on is better on every test. The gain is largest for coding (+33 points). The cost is time: the thinking-on evals took about 1.5 h, and the thinking-off evals took about 0.8 h.

## 3. Speed

One request at a time, 256 output tokens, thinking off:

| Prompt length | Time to first token | Prefill | Decode |
|---|---|---|---|
| 1K | 1.9 s | 536 tok/s | 106 tok/s |
| 8K | 4.8 s | 1,695 tok/s | 88 tok/s |
| 32K | 12.1 s | 2,712 tok/s | 89 tok/s |

Decode speed does not fall much as the context gets longer. A first "cold" request gave only 35 tok/s decode, but it includes warm-up, so I do not count it. The raw numbers are in [speed.json](speed.json).

## 4. Findings

### Coding with thinking off: the model stops before it writes the code

LiveCodeBench-Pro with thinking off failed 15 of 30 problems. In 11 of those 15, the output has no C++ code at all:

- **6 problems stopped normally with no code.** The model reasons in the answer text, ends with a line such as "Let's write." or "Now produce final code.", and then the turn ends. These outputs are short (800 to 8,000 tokens), so the token limit is not the cause.
- **5 problems hit the 16,384-token limit** while the model still reasoned in plain text.

The other 4 failures had code that was wrong or too slow.

So with thinking off, the model still "thinks", but in the visible answer, and it often loses the step where it writes the final code. I do not know yet if the Q2 quant or the server causes this. A run with a higher quant (IQ3_S) can help to find out.

### Tool calls with thinking off: too many calls

In BFCL, the "irrelevance" categories test whether the model correctly does *not* call a tool. With thinking off, the model scored 0.25 and 0.47 on them. With thinking on, it scored 0.50 and 0.76.

### BFCL categories with a score of zero

Some BFCL sub-categories have a score of 0: `sql`, `simple_java` and `multi_turn_composite`. DeepSeek has a score of 0 in the same categories, so the cause is probably the scorer or the subset, and not the model.

## 5. Limits of this run

- **Memory use is not measured.** The benchmark measures memory on the machine that runs it. Here, that was my local PC at the other end of the SSH tunnel, not the pod. So the GPU and RAM numbers in `speed.json` are not valid.
- **Reasoning tokens are not reported.** Strata returns 0 reasoning tokens also with thinking on. The output token counts include the thinking.
- **Some LiveCodeBench-Pro test cases are missing.** Hugging Face gave "not found" errors for a number of problems. The eval still scored 30 problems, but the missing test cases can change which problems go into the subset.
- **Small samples and one repeat.** This is the `quick` level. A larger run is necessary for exact comparisons.
