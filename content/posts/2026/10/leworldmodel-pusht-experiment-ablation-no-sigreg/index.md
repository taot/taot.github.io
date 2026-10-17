---
title: 'Leworldmodel Pusht Experiment Ablation No Sigreg'
date: 2026-10-16T07:57:07-04:00
draft: true
categories: ['experiments']
tags: ['world-model', 'jepa']
summary: ''
---

Ran ablation tests about SIGReg.

## Q: What's the purpose of this experiment?

To see the effect of SIGReg. The idea is to compare the embeddings from the models with and without SIGReg, and observe the representation collapse without SIGReg.

The model without SIGReg is trained with image size 112 in 3 epochs, having SIGReg removed (setting $\lambda$ to 0).

## Q: What types of collapse I'm looking for?

- Complete collapse. All images go to almost the same point. The spread becomes very small.
- Dimensional collapse. The points spread out, but only along a few directions. The other directions are not used.

## Q: What is the input data?

Created a script that saves raw embeddings (cls, emb), `state` and `proprio` (state of each frame).

All 3 checkpoints from these training runs:
- pusht/2026-09-28_114752_img112_s3072 (image size 112, 3 epochs)
- pusht/2026-10-14_115337_img112_s3072_nosigreg (same as above but without SIGReg)

All 10 checkpoints from
- pusht/2026-10-10_120411_img224_s3072 (image size 224, 10 epochs)

Two groups:
- Group 0: 20,000 random frames from the full dataset. We use them for all the statistics.
- Group 1: all the frames of 10 random episodes. We use them to draw trajectories in section 6.
So it's a total of 21094 frames.

## Q: For each test, what metrics are compared?

- Scale: how big is the cloud of points.
  - length of mean vector (|mean|)
  - spread
  - spread / |mean|
- Are all the points in the same directions?
  - histogram of 1 - cosine similarity
  - calculated between random frames
- Eigenvalue spectrum: how many directions are used?
  - eigenvalues of covariance matrix
  - effective rank
  - participation ratio
- Is the distribution close to N(0,I)?
  - SIGReg statistic
  - histogram along random directions
- Linear probe
  - train a ridge regression from latent to states (agent x, agent y, block x, block y, cos(block angle), sin(block angle))
- PCA
  - any principal component corresponds to position or direction of the T?

## Q: What does the result say? What's the restriction or drawback of SIGReg?

No-SIGReg collapses completely.

SIGReg does not complete collapse, but has some dimensional collapse.

Longer training makes the distribution more like N(0,I), but cannot make it very close to N(0,I).

## Q: Future experiments I want to run?

pushT only has 5 degrees of freedom which are encoded to 192 latent dimensions. This limits how many dimensions encoder can fill. We can try a more complex scene.
