---
title: "RTMap: crowdsourcing a map across many vehicles"
---

# RTMap — many traversals, one map

**Paper:** *RTMap: Real-Time Recursive Mapping with Change Detection and Localization* ·
[arXiv 2507.00980](https://arxiv.org/abs/2507.00980)

> **The one idea:** stop treating the online map as a per-drive throwaway. **Crowdsource it across
> many traversals into a "self-evolutional memory"** — and carry an explicit *uncertainty* for every
> vertex so later passes know how much to trust earlier ones.

This is the page where online mapping stops being a perception problem and becomes an
**infrastructure** problem.

---

## Three operations, and the split is the design

| Step | What runs | Where |
|---|---|---|
| **Perceive** | map elements, **uncertainties**, correspondences, ego pose | onboard |
| **Localize** | pose optimisation *against the crowdsourced prior map* | onboard |
| **Fuse** | merge this traversal into the shared map | **cloud** |

**Note the middle row.** Localisation is not a separate module that happens to run nearby — the
vehicle localises **with respect to the crowdsourced map it is also contributing to.** Mapping and
localisation are the same loop.

## How many vehicles become one map

Observations from multiple traversals arrive *"with their relative pose w.r.t. a **base frame**"*,
and the cloud solves a weighted least-squares problem:

- each observation is pulled toward the current estimate, **weighted by its own uncertainty**
- the previous map estimate is itself a term, so the map has memory
- **union-find** tracks vertex identity across traversals
- **Hungarian voting** preserves topology rather than just geometry

> **This is the answer to "how do distributed observations become one map."** You need (a) a shared
> frame, (b) per-observation uncertainty, (c) an identity mechanism so the same lane boundary seen
> six times is one object and not six. **All three are non-trivial and none is a perception
> problem.**

## Uncertainty is modelled, not hand-waved

Every vertex outputs **Laplace** location parameters (μ, σ) per coordinate, trained with a negative
log-likelihood loss. Uncertainties compose across traversals as mixtures.

**And it pays for itself:** on straight-road crowdsourcing, **57.6% mAP with uncertainty vs 50.7%
without.** Nearly seven points, from knowing what you do not know.

## Change detection, and it is elegant

Three query types:

| Query | Means |
|---|---|
| `Q_map` | reliable correspondence with the prior map |
| `Q_fake` | element in the prior that **is not there any more** |
| `Q_new` | newly observed element |

> *"category confidence associated with `Q_fake` will be significantly lower"* than `Q_map` — §3.3

So **change detection falls out of the confidence scores**, with no separate change-detection
pipeline. The world changed if the model stops believing in something it was told to expect.

---

## The numbers

| | Result |
|---|---|
| Crowdsourcing after 3 cycles (TbV), straight | **57.6% mAP** *(50.7% without uncertainty)* |
| Crowdsourcing after 3 cycles, turning | 55.4% mAP |
| Localisation (nuScenes) | lateral error **0.121 m**, yaw **0.368°** |
| Change detection (TbV) | 48.9% changed / 66.0% unchanged, **57.4% mean** vs baseline 54.1% |

**"After 3 cycles" is the interesting phrase.** The map improves with traversals — which is the
economic argument for crowdsourcing stated as a number.

**And note change detection is hard**: 48.9% on changed elements. **Detecting what changed is
substantially harder than mapping what is there**, which is exactly why stale maps are dangerous.

## Limitations, theirs

- Validation does not cover **unstructured terrain** or the most complex urban environments
- **No explicit occlusion detection** — a parked truck and a removed lane line can look alike
- **Z-axis uncertainty is fixed**, because it is not well observed

---

## Why this page exists

1. **This is the missing half of the practical stack.** Priors ([P-MapNet](p-mapnet.html),
   [PriorDrive](priordrive.html)) tell a vehicle what to expect. **Crowdsourcing is how that prior
   gets built and kept alive** — and unlike PriorDrive's simulated history, this is genuinely
   multi-vehicle.
2. **It confirms that a shared coordinate frame is load-bearing, not incidental.** *"Relative pose
   w.r.t. a base frame"* is the enabling condition for fusion. Without it there is nothing to
   average.
3. **Uncertainty is the currency of aggregation.** Seven mAP points say that an observation without a
   confidence is worth much less than one with.

---

## Verify before citing

- [ ] The **57.6% vs 50.7%** ablation and what "3 cycles" means precisely
- [ ] Whether TbV localisation numbers are comparable to the nuScenes ones quoted
- [ ] The base-frame construction — **is the frame given, or estimated?** The distinction matters and
      I have not confirmed it

*Read via arXiv HTML, 2026-08-08. Quotes are from the paper; the framing on this page is mine.*
