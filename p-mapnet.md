---
title: "P-MapNet: SD map priors and far-seeing"
---

# P-MapNet — SD and HD map priors

**Paper:** *P-MapNet: Far-seeing Map Constructor Enhanced by both SDMap and HDMap Priors* ·
[arXiv 2403.10521](https://arxiv.org/abs/2403.10521)

> **The one idea:** a cheap, freely available **SD map** — OpenStreetMap road centerlines — is enough
> to extend a mapping model's useful range **far beyond what its sensors can see**, and the gain
> *grows* with range rather than shrinking.

This is the first page in these notes about **priors**. Everything up to
[MapTRv2 and StreamMapNet](maptr-refinements.html) predicts the map from sensors alone. From here on,
the question changes: *what does the vehicle already know before it looks?*

---

## Where the prior comes from

> *"We first obtain the localized SDMap data of the corresponding area from the OSM based on the
> on-board GPS information. Then these SDMap data are transformed to the ego vehicle's coordinate
> system."* — §IV-A

So the pipeline is: **GPS → query OpenStreetMap → transform into the ego frame.** No survey vehicle,
no annotation, no licensing cost. That is the whole appeal.

## The problem that makes it hard: the prior does not line up

> *"SDMap prior inevitably faces spatial misalignment, where the SDMap prior doesn't align precisely
> with the current operational location."* — §IV-A

GPS drifts, OSM geometry is approximate, and the road you are on is not exactly where the map says it
is. **A naïvely concatenated prior is then actively harmful** — you are conditioning on a lie.

**Their fix is a multi-head cross-attention module**, which lets the network *search* for the best
alignment rather than trusting the given one. The BEV feature queries the SD map and decides for
itself which part of the prior corresponds to what it is seeing.

> **Worth sitting with:** the model is not given the prior, it is given a *neighbourhood* of possible
> priors and learns the correspondence. **Weak alignment is treated as a learning problem, not a
> preprocessing problem.**

## The second prior: HD maps as a learned distribution

A separate module uses **an adapted pre-trained masked autoencoder (MAE)** to capture what real HD
maps look like — trained by reconstructing masked HD maps with pixel-wise cross-entropy, then
fine-tuned on actual predictions.

> *"This MAE's input and output comes in different formats: images and segmentation masks."* — §IV-B

The point is not to memorise any specific map. It is to learn that **lanes are parallel, boundaries
are continuous, intersections have structure** — and to use that to clean up a noisy prediction.

---

## The numbers

Tested at three perception ranges: **60×30 m, 120×60 m, 240×60 m.**

| Setting | P-MapNet | Baseline (HDMapNet) | Δ |
|---|---|---|---|
| nuScenes, 240×60 m, camera+LiDAR, mIoU | **49.93** | 36.47 | **+13.46** |
| nuScenes, 240×60 m, mAP | **26.61** | 17.25 | **+9.36** |
| Argoverse 2, 120×60 m, camera-only, mIoU | **45.60** | 38.70 | **+6.90** |

**And the result that matters most:**

> *"Positive margins steadily grow along with the sensing range"* — **+18.73 mIoU at 240×60 m.**

**The gain increases with range.** That is the claim doing the work: 240 m is well past what cameras
resolve reliably, so beyond a certain distance the model is essentially *reading the SD map* and
using sensors to correct it. **The prior is not a small accuracy boost; it is what makes long range
possible at all.**

## Where it fails, in their own words

> *"Not all roads datasets are comprehensively annotated"* — performance degrades on **near-side fork
> roads** when the SD map covers only trunk routes, and results can be **suboptimal at nearby complex
> intersections.** — Appendix A-B2

**The prior helps most exactly where sensors are weakest (far away) and can hurt where sensors are
strongest (close, complex).** That is a clean trade and it should be stated that way.

---

## Why this page exists

Three things follow that matter beyond this paper:

1. **An SD map is not a small HD map.** It gives connectivity and rough geometry — **not** lane
   count, markings, stop lines or traffic-control semantics. It **constrains** the answer; it does
   not contain it.
2. **Cheap priors buy range, which is online mapping's structural weakness.** A per-drive inferred
   map cannot see around a curve; a prior can.
3. **Misalignment is the real engineering problem**, and it points at something bigger — if a single
   vehicle's prior must be searched rather than trusted, then fusing *many* vehicles' observations
   needs a shared frame. That is [RTMap](rtmap.html) and [CSMapping](csmapping.html).

---

## Verify before citing

- [ ] The **+18.73 mIoU at 240×60 m** figure and its exact setting
- [ ] Whether the 49.93 vs 36.47 comparison is camera+LiDAR for **both** rows
- [ ] Venue and peer-review status — read from arXiv, **status not confirmed here**

*Read via arXiv HTML, 2026-08-08. Quotes are from the paper; the framing on this page is mine.*
