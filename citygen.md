---
title: "CityGen: adapting to a city you have never labelled"
---

# CityGen — a new city, zero labels

**Paper:** *CityGen: Structure-Guided City-Style Synthesis for Cross-City Autonomous Driving* ·
[arXiv 2605.29935](https://arxiv.org/abs/2605.29935) · submitted **28 May 2026**

> **The one idea:** a model trained in Singapore fails in Boston. Instead of labelling Boston,
> **collect unlabelled Boston video, extract its "style," and synthesise labelled Boston-looking
> training data from HD-map structure you already have.**

Every page before this one asks how to build a map. **This one asks why a system that builds maps
well in one city stops working in the next** — which is the question that decides whether any of it
scales.

---

## The problem, stated properly

> *"significant domain shifts in appearance, road topology, and traffic patterns often cause severe
> performance degradation under cross-city deployment"* — Introduction

And the gap the paper actually attacks first:

> *"systematic evaluation frameworks for assessing and improving model transferability across cities
> remain limited"*

**They had to build the benchmark before they could build the method.**

## CityTransfer-Bench — and this may be the more durable contribution

*"The first benchmark specifically designed to evaluate cross-city generalization in autonomous
driving."*

| | |
|---|---|
| Built on | **nuScenes**, 28,130 samples total |
| Train | **three Singapore regions** — One-North, Queenstown, Holland Village *(12,435 samples)* |
| Test | **Boston Seaport** |
| Split | **geographically disjoint** |
| Tasks | perception · segmentation · **decision-making** |
| Baselines | trained-on-everything, and Singapore-only |

> **Compare this with [StreamMapNet](maptr-refinements.html)'s finding that the standard nuScenes
> splits overlap geographically.** Two independent papers arriving at the same complaint: **the
> field's default evaluation was measuring memorisation of places, not generalisation to them.**
> Defining the split *is* defining the problem.

## The method

1. **Structure extraction** — project HD-map elements (lane boundaries, centrelines, 3D boxes) into
   multi-view camera planes and rasterise them. This is the *content* to be preserved.
2. **Style library** — sample **unlabelled** target-city frames, encode them with **InternVL** into
   style descriptors capturing *"materials, illumination, vegetation density, signage, and
   architectural context."*
3. **Diffusion synthesis** — a **DiT** denoises latents conditioned **jointly** on the structural
   control and the style descriptor.
4. **Multi-view attention** — exchange structural cues across cameras so the six views agree.

**Structure from the source city, appearance from the target city, labels for free** — because the
labels came from the HD-map structure you conditioned on.

---

## The numbers — Singapore → Boston

| Task | Baseline (Singapore-only) | CityGen |
|---|---|---|
| Detection (StreamPETR) | mAP 10.18 · NDS 28.05 | **mAP 13.06 · NDS 33.95** |
| Segmentation (CVT) | IoU40 43.47% · IoU50 41.97% | **IoU40 50.84% · IoU50 49.39%** |
| Planning (UniAD) | collision 0.442% · L2 1.045 | **collision 0.350% · L2 0.984** |

**Look at the baseline detection number: 10.18 mAP.** A model trained in Singapore is barely
functional in Boston. **The cross-city gap is not a tuning issue, it is a wall** — and that is the
most important fact on this page.

---

## ⚠ "Zero-label" is not "zero-data"

**CityGen still requires unlabelled video from the target city** to build the style library.

That distinction decides what a deployment actually has to do:

| Step | Still required? |
|---|---|
| Send vehicles to drive the new city | **YES** — you need the footage |
| Have humans annotate that footage | **NO** — this is what is removed |

**So the drive survives; the annotation does not.** Anyone describing this as "no data needed for a
new city" has misread it.

## Limitations, theirs and implied

- **One source–target pair only** (Singapore→Boston). Generalisation to other pairs is untested
- Planning gains are *"more moderate"* than perception and segmentation
- Requires unlabelled target-city video
- No discussion of the compute cost of generative pretraining versus ordinary augmentation

---

## Why this page exists

1. **Cross-city transfer is the scaling constraint**, and a 10.18 mAP baseline makes that concrete.
2. **The benchmark may outlast the method.** Defining what "correct cross-city generalisation" means
   is a harder and more durable contribution than one diffusion pipeline.
3. **It changes what a new-city rollout costs** — from an annotation campaign to a driving campaign.
   Those have very different economics.

---

## Verify before citing

- [ ] Whether "zero-label" excludes **all** target-city labels or only map labels
- [ ] Baseline fairness — is `Baseline_s.g.` trained to convergence?
- [ ] Venue and peer-review status — **preprint as read**
- [ ] Whether CityTransfer-Bench has been released

*Read via arXiv HTML, 2026-08-08. Quotes are from the paper; the framing on this page is mine.*
