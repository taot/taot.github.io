---
title: 'LeWorldModel on PushT: Second Experiments'
date: 2026-10-10
draft: false
categories: ['experiments']
tags: ['world-model', 'jepa']
summary: ''
ShowToc: false
---

# LeWM Push-T: My Second Experiment vs Paper (arXiv 2603.19312v3)

This run fixes the two main differences from the [first experiment](/posts/2026/10/leworldmodel-pusht-experiment-1/): it trains for 10 epochs instead of 3, and it uses 224×224 images instead of 112×112, for both training and eval.

**Legend:** ✓ = matches · ≈ = close / likely same · ⚠️ = differs, likely affects results · — = not stated in paper

## 1. Training parameters

| | Mine | Paper | |
|---|---|---|---|
| Epochs | 10 | 10 | ✓ |
| Image size | 224×224 | 224×224 | ✓ |
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
| Train / val split | 0.9 / 0.1 | — | — |
| Dataset | `librakevin/lewm-pusht` | DINO-WM Push-T, 20,000 expert episodes | ≈ (not verified) |
| Training seeds | 1 (seed 3072) | 3 | ⚠️ |

The run took 6 h 40 min on one RTX 5090: 13,933 steps per epoch and 139,330 steps in total. The full training log is in [train.log](train.log).

```bash
python train.py data=pusht img_size=224 trainer.max_epochs=10
```

Final metrics after epoch 10:

| | Total loss | Pred loss | SIGReg loss |
|---|---|---|---|
| Train (last logged step) | 0.0887 | 0.0077 | 0.898 |
| Validation | 0.1230 | 0.0032 | 1.332 |

## 2. Eval parameters

| | Mine | Paper | |
|---|---|---|---|
| Checkpoint | `weights_epoch_010.pt` | final | ✓ |
| Episodes per seed | 50 | 50 (same 50 for all models) | ✓ |
| Eval budget | 50 steps | 50 steps | ✓ |
| Goal offset | 25 steps | 25 steps | ✓ |
| Start states | sampled at random from the dataset | sampled at random from the dataset | ✓ |
| CEM samples | 300 | 300 | ✓ |
| CEM iterations | 30 | 30 (Push-T) | ✓ |
| CEM top-k | 30 | 30 | ✓ |
| CEM initial variance | 1.0 | 1 | ✓ |
| Planning horizon | 5 (= 25 env steps) | 5 (= 25 env steps) | ✓ |
| Receding horizon | 5 (execute full plan) | 5 (execute full plan) | ✓ |
| Eval image size | 224 | 224 | ✓ |
| What the seed varies | eval start states (1 model) | training (3 models) | ⚠️ |
| Eval seeds | 0–9 | — | — |

```bash
python eval.py --config-name=pusht.yaml \
  policy=pusht/2026-10-10_120411_img224_s3072/weights_epoch_010.pt \
  eval.img_size=224 seed=<0..9>
```

Each 50-episode eval takes about 1 minute on the RTX 5090.

## 3. Eval Result (Push-T success rate)

**All 10 seeds together (same epoch-10 checkpoint, 500 episodes in total):**

| Eval seed | Success rate |
|---|---|
| 0 | 92.0% |
| 1 | 92.0% |
| 2 | 94.0% |
| 3 | 98.0% |
| 4 | 88.0% |
| 5 | 96.0% |
| 6 | 90.0% |
| 7 | 96.0% |
| 8 | 92.0% |
| 9 | 94.0% |
| **Mean ± std** | **93.2 ± 2.9** |

Overall, 466 of 500 episodes succeeded. The ± is computed the same way as the paper's. The sample std is ±3.0.

| | Success rate |
|---|---|
| Experiment 1 (3 epochs, 112 px) | 80.6 ± 5.2 |
| **Experiment 2 (10 epochs, 224 px)** | **93.2 ± 2.9** |
| Paper, LeWM (Tab. 5) | 96.0 ± 2.83 |

## Notes

1. **The seed-to-seed spread is sampling noise.** At a 93% success rate, 50 episodes give a binomial std of about ±3.6 points. The observed spread across seeds (±3.0) is within that.
2. **The same seed doesn't give the same result twice.** Rerunning seeds 0–2 gave 94 → 92, 90 → 92 and 94 → 94. The seed fixes the start states, and the CEM solver also gets `seed`, but a run still isn't bit-reproducible. GPU nondeterminism is a likely cause, but I haven't confirmed it.
3. **The remaining ~3-point gap to the paper** could come from training-run variation (I trained only 1 seed; the paper trained 3), from which 50 trajectories the paper evaluated on (not published), or from differences in the dataset.
