---
title: 'Understanding LeWorldModel'
date: 2026-10-07
draft: true
tags: ['world-model', 'jepa', 'visualization']
summary: 'Step through one training step of LeWorldModel, from pixels to the total loss, and see the shape of every tensor.'
ShowToc: false
---

[LeWorldModel (LeWM)](https://arxiv.org/abs/2603.19312) is a Joint-Embedding Predictive Architecture (JEPA) that learns a world model end to end from pixels. This page follows **one training step** through the whole model.

{{< figure src="lewm_training_pipeline.png" alt="LeWorldModel Training Pipeline" caption="The LeWorldModel training pipeline. Figure from the [LeWorldModel paper](https://arxiv.org/abs/2603.19312)." >}}

Use the buttons in the visualizer to move from step to step. Each step shows the tensors and their shapes:

1. A training batch of frames and actions.
2. The **encoder** (ViT-tiny) turns each frame into an embedding (the CLS token), and a projector MLP maps it to the latent space.
3. The **action encoder** embeds the actions.
4. The **predictor** takes the context embeddings and the actions, and predicts the embeddings of the next frames.
5. The **prediction loss** compares the predictions with the target embeddings.
6. **SIGReg** keeps the embeddings close to an isotropic Gaussian, so that the model does not collapse.
7. The **total loss** adds the two terms.

You can also open the details of each module, for example one ViT block or one conditional block of the predictor.

{{< viz src="lewm_training_vis.html" height="900px" title="LeWM training step visualizer" >}}

The code for the model is in my fork of the official repository: [taot/le-wm](https://github.com/taot/le-wm).
