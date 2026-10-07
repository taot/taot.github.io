---
title: 'Hello World: A Sample Post'
date: 2026-10-07
draft: true
tags: ['meta']
summary: 'A sample post that shows math, code, footnotes, and an interactive visualization.'
---

This sample post shows the features of this site. Replace it with your first article.

## Math

Inline math: the characteristic function of $X$ is $\varphi_X(t) = \mathbb{E}[e^{itX}]$.

Display math:

$$
\hat{f}(\xi) = \int_{-\infty}^{\infty} f(x)\, e^{-2\pi i x \xi}\, dx
$$

## Code

```python
import torch

def sigreg_loss(z: torch.Tensor, t: torch.Tensor) -> torch.Tensor:
    cf = torch.exp(1j * z[:, None] * t[None, :]).mean(0)
    return (cf - torch.exp(-t**2 / 2)).abs().pow(2).mean()
```

## Interactive visualization

{{< viz src="viz/sigreg_cf.html" height="760px" title="SIGReg characteristic function" >}}

## Footnotes

A claim with a source.[^1]

[^1]: Author et al., *Title of the Paper*, 2025.
