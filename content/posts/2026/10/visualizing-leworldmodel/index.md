---
title: 'Visualizing LeWorldModel'
date: 2026-10-07
draft: true
categories: ['notes']
tags: ['world-model', 'jepa', 'visualization']
summary: 'Step through one training step of LeWorldModel, from pixels to the total loss, and see the shape of every tensor.'
ShowToc: false
---

{{< figure src="lewm_training_pipeline.png" alt="LeWorldModel Training Pipeline" caption="The LeWorldModel training pipeline. Figure from the [LeWorldModel paper](https://arxiv.org/abs/2603.19312)." >}}

[LeWorldModel (LeWM)](https://arxiv.org/abs/2603.19312) is a Joint-Embedding Predictive Architecture (JEPA) that learns a world model end to end from pixels. JEPA models learn wold model in latent space and also use it to predict and plan in latent space.

The key contribution of this paper is SIGReg (Sketched-Isotropic-Gaussian Regularizer), (I think SIGReg is actually proposed by a previous paper, I'm just not sure how to phase it here).

SIGReg aims to solve the problem of representation collapse. JEPA tries to embed the world into latent representations, and the model will learn constant (or fixed) representation (?) if no restrictions, which means the model can always encode the world into a fixed vector so the encodings from observation and prediction always exactly match. SIGReg (and approaches used in previous JEPA papers) aims to prevent that. The advantage of SIGReg over previous approaches is that it only introduces one extra hyper-parameter, while previous approaches introduces 6.

SIGReg's core approach is random projection to 1-D and then calculate the Epps-Pulley test statistics. Epps-Pulley calcuates character function of the projected probability distribution (can be seen as Fourier transformation on the probability distribution), then comapre the characteristic functions against the characteristic function of a Gaussian distribution (by calculating L2-norm between them), and use the result as another loss (the SIGReg loss) to the model.

Why projection to a single dimension randomly? Because it's hard to check wether a high-dimension distribution is Gaussian or not, and Epps-Pulley works for 1-D. Each training batch will project to 1024 random directions. Over the whole training process, it will cover most of the directions, which is imperically enough to prevent reprensentation collapse.

To help understand the training pipeline and SIGReg, I asked Claude Code create the following visualizations.

### Training Pipeline Visualization

Use the buttons in the visualizer to move from step to step. Each step shows the tensors and their shapes:

You can also open the details of the modules with a plus sign, for example one ViT block or one conditional block of the predictor, by clicking on the block.

{{< viz src="lewm_training_vis.html" height="900px" title="LeWM training step visualizer" >}}

### SIGReg Character Function Visualization

{{< viz src="sigreg_cf.html" height="900px" title="SIGReg characteristic function visualizer" >}}
