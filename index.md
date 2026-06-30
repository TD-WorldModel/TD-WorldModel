---
layout: project
title: Temporal Discriminative World Models
subtitle:
description: A project page for Contrastive World Models.
date: June 30, 2026

# ---- Authors (the sup numbers map to the affiliations list below) ----
authors:
  - name: William Peng
    url: "#"
    affiliations:
    equal: true
  - name: Holger Molin
    url: "#"
    affiliations:
    equal: true

affiliations:
  - id:
    name: Stanford University

# ---- Status shown in place of the link buttons. ----
# When the paper is ready, delete `status` and add a `links:` list instead, e.g.
# links:
#   - text: Paper
#     url: "#"
#     icon: fas fa-file-pdf
status: Paper in progress

# ---- Teaser (optional). Drop a file in static/images/ and point here. ----
teaser: /static/images/patch_pc1_clean.png
teaser_caption: "First principal component of patch features across a manipulation rollout. Top: input frames. TD-WM (middle) concentrates representation on the arm and manipulated objects; LeWorldModel (bottom) spreads it across the static background."

# ---- Citation (optional). Paper in progress; add a BibTeX entry when ready. ----
# bibtex: |
#   @article{...}
---

> **Work in progress.** This page is an early draft; sections will be expanded and the paper is coming soon.

## Overview

**We beat LeWorldModel where it matters most.**

The short version: SIGReg plus MSE is misspecified for learning dynamics. [LeWorldModel](https://arxiv.org/abs/2603.19312) ends up rewarding the model for forgetting the hardest-to-predict parts of a scene — which are exactly the dynamics we care about. That is backwards.

Temporal Discriminative World Modeling (TD-WM) is a clean fix. It forces the model to discriminate dynamics rather than static appearance, and outperforms LeWorldModel on hard manipulation tasks.

How? We add a temporal discriminative objective inside JEPA with no latent regularization. The model no longer collapses, and it reaches a low SIGReg loss on its own without any direct supervision.

*Visual: latent feature rollout on the cube task — coming soon.*

## Motivating issues

The standard recipe for training JEPA-style world models is by now familiar: encode each frame into a latent, train a predictor to regress the next latent from the current one, and regularize the encoder against collapse. Planning then rolls the predictor forward and scores candidate action sequences by their L2 distance to a goal latent. [LeWorldModel](https://arxiv.org/abs/2603.19312), for instance, combines a next-embedding MSE objective, SIGReg for anti-collapse, an autoregressive predictor, and an L2 planning cost in latent space. When we tested this combination, however, it planned poorly — and the representation turned out to be the cause.

**SIGReg prevents collapse, but does not enforce information retention.** SIGReg constrains the geometry of the latent distribution by pushing embeddings toward a full-rank, decorrelated isotropic Gaussian. While this successfully avoids trivial collapse, it does not incentivize the encoder to retain physical information. This is exactly what we observe when we train and evaluate on OGBench's Scene environment: the cube's $$(x,y,z)$$ position and the 7-DOF arm were only weakly linearly represented within embeddings, even as the rest of the scene (drawer, window, buttons) was cleanly recoverable. Anti-collapse is necessary, but it does not necessarily produce informative latents.

![Probing object states from a SIGReg encoder on OGBench's Scene environment]({{ '/static/images/probe_states_cube_sigreg_e25.png' | relative_url }}){:.wide}
*Probing object states from a SIGReg-trained encoder. The cube ($$R^2 \approx 0.18$$) and arm ($$R^2 \approx 0.30$$) are only weakly recoverable, while the drawer, window, and buttons decode almost perfectly ($$R^2 \approx 0.95$$). Even nonlinear (MLP) probes and patch-level readouts recover cube position only partially.*

Motivated by these results, we trace the problem to the training objective itself. Next-embedding MSE is minimized by making the future predictable, and the surest way to achieve that is to stop representing the parts of it that are hard to anticipate. The small, fast-moving, control-relevant variables (the arm's precise pose, the cube) are exactly what a regressor would rather forget, because forgetting them lowers the loss. Clearly, the training objective needs to change.

## Temporal Discriminative World Modeling

A good world model should encode agent-relevant dynamics and predict future states conditioned on candidate actions. We can take advantage of the natural temporal structure of our data to contrast the current state against the future, pushing the encoder to pick out what separates the two. When co-trained alongside an action-conditioned predictor, the predictor has asymmetric information about dynamics. Intuitively, consider a robot carrying a box down a street. Having access to the robot's actions, the predictor knows exactly the cause of the box's movement; comparatively, it doesn't know the cause of leaves blowing in the wind. Hence the gradient points toward faithfully representing the box rather than the leaves — leaving us with an objective that incentivizes a high-quality world model.

### The objective

Following the basic JEPA recipe, we co-train an encoder $$f_\theta$$ and a predictor $$g_\phi$$. The encoder embeds each frame of a clip, $$z_i = f_\theta(o_i)$$, and from an $$H$$-frame history the predictor forms the embedding $$k$$ steps ahead using the intermediate actions:

$$\hat z_{t+k} = g_\phi\big(z_{t-H+1:t},\; a_{t:t+k-1},\; k\big).$$

Both encoder and predictor outputs are norm-constrained, so only their direction is free. Writing $$c_{ij}$$ for the cosine similarity between a prediction with target frame $$i$$ and an encoded frame $$j$$ of the same clip, we regress every such cosine similarity onto a target similarity that decays with the temporal gap $$\lvert i-j\rvert$$:

$$\mathcal{L}=\mathbb{E}_{i,j}\left[\left(c_{ij}-e^{-|i-j|/\sigma}\right)^2\right].$$

In a norm-constrained latent space this cosine regression is MSE-equivalent: at $$i=j$$ it recovers next-embedding prediction, while the partial-similarity targets at $$i \neq j$$ pull temporally distant frames toward orthogonality, ruling out collapse. Importantly, these targets are mutually consistent: the setpoint matrix $$S_{ij}=e^{-\lvert i-j\rvert/\sigma}$$ is a valid Gram matrix, so some configuration of unit vectors realizes every target cosine at once — the optimum is a geometrically attainable arrangement.

### Optimizations

Noting that 97% of training FLOPs are allocated to the encoder and that we sample long contiguous blocks of frames, we employ multi-horizon prediction to substantially increase gradient quality while only modestly increasing the compute cost. In addition, we find that a fused Muon–AdamW optimizer substantially accelerates convergence.

### Data

![Predictor fidelity under off-policy actions]({{ '/static/images/predictor_fidelity_ood.png' | relative_url }})
*Predictor fidelity under off-policy (noised) actions: our richer model degrades sharply out of distribution, while the less-informative SIGReg model changes little.*

Interestingly, when training on expert data alone, SIGReg out-plans our richer model. Probing both predictors with off-policy (noised) actions shows why: ours degrades sharply on actions outside the expert distribution, while SIGReg's — which encodes less of the scene — changes very little. We hypothesize that this robustness is mostly a consequence of representing less, and shows up only on tasks that don't require the parts it omits.

To address this, we re-rolled the trajectories with added action noise and re-rendered. We further argue that world models should be trained on both expert and unsuccessful trajectories: the goal of a world model is to model the underlying physics of the world, not just *intent*. We trained every model on this dataset, with the recipe otherwise unchanged.

## Results

By using a temporal discrimination objective, TD-WM recovers a significant portion of state information within the embedding space.

![Object-state probes: TD-WM vs. LeWorldModel]({{ '/static/images/object_state_probe_tdwm_vs_lewm.png' | relative_url }})
*Object-state probes: TD-WM recovers substantially more state information in its embeddings than LeWorldModel.*

We evaluate TD-WM's downstream performance via the cross-entropy method (CEM). Notably, we evaluate at 3× longer goal horizons than [LeWorldModel](https://arxiv.org/abs/2603.19312).

![Long-horizon planning performance]({{ '/static/images/longhorizon_o25_o75.png' | relative_url }})
*Stronger planning performance: under the cross-entropy method, TD-WM plans better at multiple goal horizons.*

## Conclusion

*Coming soon.*

## References

1. Maes, L., Le Lidec, Q., Scieur, D., LeCun, Y., & Balestriero, R. *LeWorldModel: Stable End-to-End Joint-Embedding Predictive Architecture from Pixels.* arXiv:2603.19312, 2026. [[arXiv]](https://arxiv.org/abs/2603.19312)
