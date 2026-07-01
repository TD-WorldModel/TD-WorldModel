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

# ---- Correspondence line shown under the affiliations. ----
correspondence: "hmolin, wgpeng [at] stanford.edu"

# ---- Status shown in place of the link buttons. ----
# When the paper is ready, delete `status` and add a `links:` list instead, e.g.
# links:
#   - text: Paper
#     url: "#"
#     icon: fas fa-file-pdf
# status: Paper in progress   # (moved into the teaser caption below)

# ---- Teaser (optional). Drop a file in static/images/ and point here. ----
# Change `teaser_width` to resize the teaser figure (any CSS width, e.g. 80% or 640px).
teaser: /static/images/main_figure.png
teaser_width: 100%
teaser_caption: "Temporal Discriminative training objective for JEPA world models. Paper in progress."

# ---- Citation (optional). Paper in progress; add a BibTeX entry when ready. ----
# bibtex: |
#   @article{...}
---

<!--
  FIGURE SIZE: each in-text figure sets its own width via {:style="width:NN%"}
  right after the image. Change that percentage (or use px, e.g. 480px) to resize.
  The teaser figure at the top is sized by `teaser_width` in the front matter above.
-->

## Motivating Issue

The standard recipe for JEPA-style world models [\[1\]](#ref-1) is by now familiar: an encoder, a next-latent predictor co-trained against it, and an regularizer to keep the encoder from collapsing to a trivial representation. Planning is L2 cost-to-go in latent space. LeWorldModel [\[2\]](#ref-2) is a canonical instance with SIGReg as its anti-collapse regularizer. SIGReg constrains the geometry of the latent distribution by pushing embeddings toward an isotropic Gaussian, which is an elegant solution to trivial collapse. Yet when we tested it, it planned poorly, and the representation turned out to be the cause.

**SIGReg prevents collapse, but does not enforce information retention**. While it successfully avoids trivial collapse, it does not incentivize the encoder to retain physical information. This is exactly what we observe when we train and evaluate on OGBench's Scene environment [\[3\]](#ref-3).

![Example start and end of trajectory in OGBench Scene]({{ '/static/images/ogbench_scene.png' | relative_url }}){:style="width:50%"}
*Example start and end of trajectory in OGBench Scene.*

The highly dynamic cube's $$(x,y,z)$$ position and 7-DOF arm were weakly represented within embeddings. This is in contrast to the slower moving objects (drawer, window, buttons) that were cleanly recoverable. From this, we hypothesized that although anti-collapse is necessary, it is not sufficient to produce informative latents.

![Probing a SIGReg-trained encoder on OGBench Scene]({{ '/static/images/repr_probe_sigreg_step23000.png' | relative_url }}){:style="width:70%"}
*Held-out $$R^2$$ from probing a SIGReg-trained encoder: the cube and 7-DOF arm are only weakly recoverable, while the drawer, window, and buttons decode almost perfectly.*

Motivated by these results, we trace the problem to the training objective. Next-embedding MSE is minimized by making the future predictable, and the surest way to achieve that is to stop representing the parts of it that are hard to anticipate. The small, fast-moving, control-relevant variables (the arm's precise pose, the cube) are exactly what a regressor would rather forget, because forgetting them lowers the loss. Clearly, the training objective needs to change.

## Temporal Discriminative World Modeling

Broadly speaking, a good world model should encode agent-relevant dynamics and simulate future states conditioned on candidate actions. The appeal of a JEPA-style latent world model is its ability to learn abstractions of the world without having to represent irrelevant details of the scenery. In order to do so, the objective must specify what types of information should be retained.

In light of this, we propose a Temporal Discriminative training objective. We leverage the natural temporal structure of action-labeled video to contrast the current state against the future, pushing the encoder to pick out what separates the two. Further, the action condition of the predictor biases the model towards learning agent-relevant dynamics.

With our Temporal Discriminative World Model (TDWM), we are able to train stably and avoid trivial collapse with only one training objective. The intrinsic anti-collapse from temporal discrimination is qualitatively different from variance-based regularization: our latent spaces have an effective rank of around 40, while SIGReg expands to around 150. For reference, the environment contains a total of 24 degrees of freedom.

### The objective

Following the basic JEPA recipe, we co-train an encoder $$f_\theta$$ and a predictor $$g_\phi$$. The encoder embeds each frame $$o_i$$ of a clip, $$z_i = f_\theta(o_i)$$, and from an 3-frame history the predictor predicts the embedding $$k$$ steps ahead using the intermediate actions,

$$\hat z_{t+k} = g_\phi\big(z_{t-2:t},\; a_{t:t+k-1},\; k\big).$$

Both encoder and predictor outputs are norm-constrained, so only their direction is free. Writing $$c_{ij} = \text{cossim}(\hat{z}_i, z_j)$$ for the cosine similarity between a prediction with target frame $$i$$ and an encoded frame $$j$$ of the same clip, we regress every such cosine similarity onto a target similarity that decays with the temporal gap $$\lvert i-j\rvert$$:

$$\mathcal{L}=\mathbb{E}_{i,j\sim \text{clip}}\left[\left(c_{ij}-e^{-|i-j|/\sigma}\right)^2\right].$$

At $$i=j$$ this reduces to a next-embedding prediction objective in a norm-constrained space, while the partial-similarity targets at $$i \neq j$$ push temporally distant frames toward orthogonality, making collapse strictly suboptimal. Cosine similarity is a natural measure here as it allows us to specify a degree of similarity rather than explicit target latent.

Importantly, with exponential attenuation these target similarities can all be satisfied at once. Write $$\tau_{ij} = e^{-\lvert i-j\rvert/\sigma}$$ for the target similarity between frames $$i$$ and $$j$$, so the objective regresses each realized cosine $$c_{ij}$$ onto $$\tau_{ij}$$. For frames ordered $$i \le j \le k$$,

$$\tau_{ik} = e^{-|i-k|/\sigma} = e^{-|i-j|/\sigma}\cdot e^{-|j-k|/\sigma} = \tau_{ij}\cdot \tau_{jk},$$

so a long-gap target is just the product of the short-gap targets. The pairwise targets therefore reinforce one another rather than conflict, so a single embedding arrangement can satisfy all of them at once. More formal analysis to come when we find a competent mathematician.

### Justifying Temporal Distance

![Example start and end of trajectory in OGBench Scene]({{ '/static/images/ogbench_scene.png' | relative_url }}){:style="width:50%"}
*Example start and end of trajectory in OGBench Scene.*

By temporal distance, we mean how far two states are separated in time rather than in physical space. When performing latent planning via the Cross-Entropy Method (CEM) [\[4\]](#ref-4), this is a more suitable measure of distance to goal completion than LeWorldModel's embedding distance.

As in the figure above, the arm is already in its goal state while the drawer needs to be opened. With a latent space like LeWorldModel's, the start and goal are already low cost as most things are already correctly positioned. Closing the drawer requires moving the arm out of position which temporarily increases the cost. In TDWM's latent space, states are ordered by their distance in time which means that moving the arm towards the drawer smoothly decreases cost as this action decreases the time until the goal is achieved.

### Optimizations

Noting that 97% of training FLOPs are allocated to the encoder and that we sample long contiguous blocks of frames, we employ multi-horizon prediction to increase gradient quality substantially whilst modestly increasing the compute cost. In addition, we find that using a fused Muon-AdamW optimizer substantially accelerates convergence.

### Data

![Normalized predictor fidelity vs. action perturbation magnitude]({{ '/static/images/predictor_fidelity_ood.png' | relative_url }}){:style="width:70%"}
*Normalized predictor fidelity as a function of out-of-distribution action perturbation magnitude ($$\mathrm{RMS}(\Delta a / \sigma_a)$$). Trained on expert trajectories only, TDWM degrades sharply on OOD actions; training on the noised dataset largely closes the gap. We report perturbation size as the RMS of the per-dimension z-scored action deviation so that every action dimension contributes on equal, unit-free footing*

Interestingly, when training only on expert data, LeWM outperforms TDWM on planning. Probing both predictors with off-policy (noised) actions demonstrates why: TDWM's prediction accuracy degrades sharply on actions outside of the expert distribution (we return to this in [The off-manifold tradeoff](#the-off-manifold-tradeoff)). We hypothesize that LeWM's robustness can be attributed to it encoding less of the actual scene. In this way, it has a simpler extrapolation task. Further, we note that in the expert trajectories, action correlate linearly with state to an $$R^2$$ of 0.71. TDWM may therefore be modeling agent intent rather than the underlying physics of the environment.

To address this, we supplement the dataset by re-rolling the trajectories with added action noise and re-rendering. This brings action correlation with state down to an $$R^2$$ of 0.31. We argue that world models should be trained on data including both expert and unsuccessful trajectories, as this forces the world model to understand the underlying physics of the world. Importantly, this kind of data is readily available. Recent embodied datasets retain failed trajectories at meaningful scale: DROID [\[5\]](#ref-5) releases roughly 16k failed teleoperation attempts alongside its 76k successes; RoboMIND [\[6\]](#ref-6) contributes a further 5k real-world failure demonstrations, each labeled with its cause; and AgiBot World [\[7\]](#ref-7) preserves failed demonstrations within its 1M-trajectory corpus. Alongside these public datasets, as embodied intelligence becomes deployed at scale, failed trajectories will be abundant.

We trained every model on this dataset with the recipe otherwise unchanged. Both the TDWM and LeWM variants are trained for 0.8 exaFLOPs and the presented results come from the checkpoint with the highest O25 CEM score.

## Results

We evaluate TDWM along three axes: what its latents encode (probing), whether anti-collapse emerges without explicit regularization (SIGReg loss), and how it plans (CEM success rate at long goal horizons). All comparisons are against LeWM trained on the same noised dataset and we stick to pure 1-step auto-regressive prediction for a fair comparison.

### TDWM recovers dynamic variables

![Held-out R² for object-state probes]({{ '/static/images/object_state_probe_tdwm_vs_lewm.png' | relative_url }}){:style="width:70%"}
*Held-out $$R^2$$ for object-state probes on the final embedding. TDWM and LeWM are matched on the static scene elements, but TDWM recovers roughly twice as much information about the agent-controlled variables (cube and arm).*

The probe results separate cleanly into two regimes. On the static scene elements (drawer, window, and buttons), both models score between 0.91 and 0.97 and are effectively tied. On the dynamic variables, the gap is large: TDWM reaches $$R^2 = 0.56$$ on cube position versus LeWM's 0.27 (2.1×), and 0.69 versus 0.39 on the 7-DOF arm (1.8×).

This is the pattern the asymmetric-information argument in [Temporal Discriminative World Modeling](#temporal-discriminative-world-modeling) predicts. Information about static geometry is easily predictable and thus survives mere anti-collapse pressure. The state of the arm and cube is more difficult to predict but the most reliable discriminant between temporally adjacent frames, pushing TDWM to represent them more cleanly.

### Anti-collapse emerges without regularization

TDWM is trained with no SIGReg term, yet its final SIGReg loss falls from ~30 to 2.4. LeWorldModel achieves a SIGReg loss of 0.6 with direct optimization. TDWM therefore closes roughly 92% of the init-to-LeWM gap on a quantity it never sees. Clearly, explicit regularization is not necessary for avoiding collapse. The discriminative objective enforces enough decorrelation through its orthogonality targets to keep the representation healthy on its own.

### Stronger long-horizon planning

![CEM success rate at goal offsets O25, O50, and O75]({{ '/static/images/longhorizon_o25_o75.png' | relative_url }}){:style="width:65%"}
*CEM success rate at goal offsets O25, O50, and O75. Note that O75 is 3× the horizon evaluated in [\[2\]](#ref-2). Error bars are standard error across seeds.*

TDWM outperforms LeWM at every horizon: 63% vs. 50% at O25, 38% vs. 28% at O50, and 15% vs. 4% at O75. The absolute margin is broadly stable, but the relative gap widens with horizon. Long-horizon planning is the regime in which a missing cube position compounds worst, and it is also where the representational advantage pays out most.

### The off-manifold tradeoff

![CEM planning-component ablations at O25]({{ '/static/images/o25_ablation.png' | relative_url }}){:style="width:100%"}
*Planning success at O25 across CEM ablations: LeWM (50), TDWM expert + noise (63), TDWM expert with a perfect (MuJoCo) predictor (66), and a perfect encoder with a perfect predictor (77, the absolute upper bound).*

As discussed in [Data](#data), the richer representation degrades predictor robustness out of distribution. To better understand where the models fail, we perform ablations on each component of CEM planning. As shown in the figure above, taking the expert-trained encoder and using ground-truth MuJoCo environment as the predictor module, we achieve a CEM score of 66 (isolating the perfect-predictor upper bound). By training on noised action trajectories, the trained predictor almost entirely closes this gap up to 63. As context, performing CEM directly in the simulator's representation space with its ground-truth predictor, we achieve a maximal CEM score of 77, an absolute upper bound for performance.

### Limitations

This is preliminary work and should be read as such. Our evaluation is on a single environment (the Scene manipulation task). We are actively expanding to more environments and evaluating scalability of these ideas.

## Conclusion

The standard recipe for latent world models has a quiet misspecification: making the future predictable and retaining information about it pull in opposite directions whenever a variable is hard to anticipate. Anti-collapse stops the encoder from collapsing, but does not stop it being uninformative.

TDWM replaces that objective. Discriminating between temporally adjacent states forces a focus on dynamics, and the predictor's action conditioning points the gradient at agent relevant dynamics. The result is a representation that selectively preserves the agent-controlled variables (cube position, arm pose) and outperforms previous world models at long horizon planning.

## References

1. <a id="ref-1"></a>LeCun, Y. *A Path Towards Autonomous Machine Intelligence.* OpenReview preprint (version 0.9.2), 2022. [[OpenReview]](https://openreview.net/pdf?id=BZ5a1r-kVsf)
2. <a id="ref-2"></a>Maes, L., Le Lidec, Q., Scieur, D., LeCun, Y., & Balestriero, R. *LeWorldModel: Stable End-to-End Joint-Embedding Predictive Architecture from Pixels.* arXiv:2603.19312, 2026. [[arXiv]](https://arxiv.org/abs/2603.19312)
3. <a id="ref-3"></a>Park, S., Frans, K., Eysenbach, B., & Levine, S. *OGBench: Benchmarking Offline Goal-Conditioned RL.* International Conference on Learning Representations (ICLR), 2025.
4. <a id="ref-4"></a>de Boer, P.-T., Kroese, D. P., Mannor, S., & Rubinstein, R. Y. *A Tutorial on the Cross-Entropy Method.* Annals of Operations Research, 134(1):19-67, 2005.
5. <a id="ref-5"></a>Khazatsky, A., Pertsch, K., Nair, S., et al. *DROID: A Large-Scale In-The-Wild Robot Manipulation Dataset.* arXiv:2403.12945, 2025. [[arXiv]](https://arxiv.org/abs/2403.12945)
6. <a id="ref-6"></a>Wu, K., Hou, C., Liu, J., et al. *RoboMIND: Benchmark on Multi-embodiment Intelligence Normative Data for Robot Manipulation.* Robotics: Science and Systems XXI (RSS), 2025. [[DOI]](https://doi.org/10.15607/RSS.2025.XXI.152)
7. <a id="ref-7"></a>AgiBot-World Contributors, et al. *AgiBot World Colosseo: A Large-scale Manipulation Platform for Scalable and Intelligent Embodied Systems.* arXiv:2503.06669, 2025. [[arXiv]](https://arxiv.org/abs/2503.06669)
