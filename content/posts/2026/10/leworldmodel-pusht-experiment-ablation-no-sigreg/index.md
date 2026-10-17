---
title: 'Leworldmodel Pusht Experiment Ablation No Sigreg'
date: 2026-10-16T07:57:07-04:00
draft: true
categories: ['experiments']
tags: ['world-model', 'jepa']
summary: ''
---

# Summary

In this post, I train LeWM without SIGReg and compare its embeddings with normal runs.

## Q: What's the purpose of this experiment?

I want to see what SIGReg does. To do this, I compare the embeddings of models trained with and without SIGReg, and look for representation collapse in the model without it.

The non-SIGReg models is trained with the same setup as my baseline (image size 112, 3 epochs), except that the SIGReg weight ($\lambda) is set to 0.

In addition, I also look at the embeddings of another model trained with image size 224 and 3 epochs.

## Q: What types of collapse am I looking for?

- Complete collapse. All images map to almost the same point and the spread becomes very small.
- Dimensional collapse. The points spread out, but only along a few directions. The other directions are not used.

## Q: What is the input data?

I created a script that encodes a fixed set of frames with each checkpoint and saves the embeddings (`cls`, `emb`) and the true `state` and `proprio` of each frame.

I use all 3 checkpoints from each of these runs:
- pusht/2026-09-28_114752_img112_s3072 (image size 112, 3 epochs)
- pusht/2026-10-14_115337_img112_s3072_nosigreg (same as above but without SIGReg)

and all 10 checkpoints from:
- pusht/2026-10-10_120411_img224_s3072 (image size 224, 10 epochs)

The frames are in two groups: 20,000 random frames from the whole dataset, which I use for the statistics, and 1,094 frames of 10 random episodes, which I use to draw trajectories in the PCA plots. That is 21,094 frames in total.

## Q: For each test, what metrics are compared?

- Scale: how big is the point cloud.
  - length of mean vector (|mean|)
  - spread
  - spread / |mean|
- Direction: are all the points in the same directions?
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
  - does any principal component match the position or angle of the T block?

## Q: What does the result say? What's the restriction or drawback of SIGReg?

Without SIGReg, the embeddings collapse completely: the spread is about 10⁻⁵, compared with about 14 with SIGReg.

With SIGReg, there is no complete collapse, but there is some dimensional collapse: the effective rank is only about 69 of 192 at epoch 3.

Longer training moves the distribution closer to N(0, I), but it does not get very close. In the 10-epoch run, the effective rank stops at about 98. (This run also uses a larger image size, so the comparison is not clean.)

## Q: Future experiments I want to run?

The PushT scene has only about 5 degrees of freedom, but the latent has 192 dimensions. This can limit how many dimensions the encoder can use. A more complex scene may show if this is the cause.

# Experiment Details
