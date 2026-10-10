---
title: 'LeWorldModel on PushT: First Experiments'
date: 2026-10-08
draft: false
categories: ['experiments']
tags: ['world-model', 'jepa']
summary: ''
ShowToc: false
---

# LeWM Push-T: My Experiment vs Paper (arXiv 2603.19312v3)

**Legend:** ✓ = matches · ≈ = close / likely same · ⚠️ = differs, likely affects results · — = not stated in paper

## 1. Training parameters

| | Mine | Paper | |
|---|---|---|---|
| **Epochs** | **3** | **10** | ⚠️ |
| **Image size** | **112×112** | **224×224** | ⚠️ |
| Encoder | ViT-tiny, patch 14, 12 layers, 3 heads, hidden 192, from scratch | ViT-tiny, patch 14, 12 layers, 3 heads, hidden 192, from scratch | ✓ |
| Embedding | [CLS] → MLP projector (hidden 2048) + BatchNorm, dim 192 | [CLS] → 1-layer MLP + BatchNorm, dim 192 | ≈ |
| Predictor | transformer, 6 layers, 16 heads, dropout 0.1, AdaLN action conditioning, ~10M params | transformer ("ViT-S"), 6 layers, 16 heads, 10% dropout, AdaLN, ~10M params | ✓ |
| History size | 3 | 3 | ✓ |
| Sub-trajectory | 4 frames + 4 blocks of 5 actions | 4 frames + 4 blocks of 5 actions | ✓ |
| Frameskip | 5 | 5 | ✓ |
| Batch size | 128 | 128 | ✓ |
| SIGReg λ | 0.09 | 0.1 (λ ablation peaks near 0.09) | ≈ |
| SIGReg projections | 1024 | 1024 | ✓ |
| SIGReg knots | 17 | T nodes in [0.2, 4] (count not stated; insensitive) | — |
| Optimizer / LR / WD | AdamW / 5e-5 / 1e-3 | — | — |
| Precision / grad clip | bf16 / 1.0 | — | — |
| Dataset | `librakevin/lewm-pusht` | DINO-WM Push-T, 20,000 expert episodes | ≈ (not verified) |
| Training seeds | 1 (seed 3072) | 3 | ⚠️ |

## 2. Eval parameters

| | Mine | Paper | |
|---|---|---|---|
| Episodes | 50 | 50 (same 50 for all models) | ✓ |
| Eval budget | 50 steps | 50 steps | ✓ |
| Goal offset | 25 steps | 25 steps | ✓ |
| CEM samples | 300 | 300 | ✓ |
| CEM iterations | 30 | 30 (Push-T) | ✓ |
| CEM top-k | 30 | 30 | ✓ |
| CEM initial variance | 1.0 | 1 | ✓ |
| Planning horizon | 5 (= 25 env steps) | 5 (= 25 env steps) | ✓ |
| Receding horizon | 5 (execute full plan) | 5 (execute full plan) | ✓ |
| **Eval image size** | **112** | **224** | ⚠️ |
| Eval seed | 42 (default in `config/eval/pusht.yaml`) | — | — |

## 3. Eval Result (Push-T success rate)

**All 10 seeds together (same epoch-3 checkpoint, 500 episodes in total):**

| Eval seed | Success rate |
|---|---|
| 0 | 84.0% |
| 1 | 74.0% |
| 2 | 78.0% |
| 3 | 84.0% |
| 4 | 76.0% |
| 5 | 82.0% |
| 6 | 74.0% |
| 7 | 92.0% |
| 42 | 82.0% |
| 113 | 80.0% |
| **Mean ± std** | **80.6 ± 5.2** |

The ± is computed the same way as the paper's. The sample std is ±5.5.

## Differences

1. **Epochs: 3 vs 10.** Most likely the main cause of the gap.
2. **Image size: 112 vs 224**, for both training and eval.
