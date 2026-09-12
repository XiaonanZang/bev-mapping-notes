---
title: "Offline map generation: extracting a map from the whole survey at once"
---

# Offline map generation — when the model sees the finished map

**Papers:** *LIE: LiDAR-only HD Map Construction with Intensity Enhancement via Online Knowledge
Distillation* · [arXiv 2605.01478](https://arxiv.org/abs/2605.01478) · **IEEE ITSC 2026** ·
*VMA: Divide-and-Conquer Vectorized Map Annotation System for Large-Scale Driving Scene* ·
[arXiv 2304.09807](https://arxiv.org/abs/2304.09807)
**Dataset:** *Toronto-3D* · [arXiv 2003.08284](https://arxiv.org/abs/2003.08284) *(March 2020)*

> **The one idea:** every page in [Part I](index.html) builds a map from what one vehicle can see
> right now. **But if a survey vehicle has already driven the whole city, you can accumulate every
> return into one dense bird's-eye raster first, and extract the map from that.** The perception
> problem nearly evaporates. **What is left — turning a raster into a connected, typed, city-scale
> graph — is the part that still costs money.**

---

## The pipeline, four stages

| # | Stage | Output |
|---|---|---|
| 1 | **Survey** — drive the city, LiDAR + GNSS/INS | a bag of scans |
| 2 | **Pose** — SLAM or map-based registration to one global frame | a consistent trajectory |
| 3 | **Accumulate** — project every return into a BEV grid; rasterize **intensity** | a dense city-scale raster |
| 4 | **Extract** — one learned model over that raster → lanes, boundaries, crossings, arrows | **vectorized map elements** |

**The defining property: aggregation happens *before* the network.** Every frame that will ever
exist is already fused when inference runs. Contrast [StreamMapNet](maptr-refinements.html), which
fuses a few seconds of BEV memory, and [RTMap](rtmap.html), which fuses across traversals *after*
the network.

## Why intensity, and why it is only half the story

This is the paragraph worth keeping.

**Lane paint is retroreflective.** It is engineered to send light back to its source, which is
exactly what a LiDAR measures. In an intensity raster, paint is bright and asphalt is dark —
**the signal is photometric, and the geometry is flat and useless.**

**A curb is the exact opposite.** It is the same material and colour as the road beside it, so
intensity tells you nothing. **What identifies it is a 10–15 cm step in height** — pure geometry.

> **One sensor, two entirely different physical cues.** That is why these systems carry an
> intensity channel *and* a height channel, and why a method that nails lane markings can be
> useless on boundaries. **Ask which cue a paper's headline number is actually resting on.**

## How the raster is actually built

LIE specifies it precisely enough to reimplement, which is rare:

> *"all filtered point clouds in the scene are transformed to a global frame and concatenated to
> increase density and reduce sparsity"*

Then: discretize into Cartesian grid cells, **retain the maximum intensity per cell**, normalize,
and refine with **local maximum filtering and Gaussian smoothing**. Resolution **0.1 m**, with
per-sample tiles cut out by ego pose.

**Three choices there are worth noticing.** *Maximum* per cell, not mean — a single confident bright
return beats an average diluted by grazing-angle misses. *Local-max filtering* thins the paint back
toward a centreline. And **cutting tiles by ego pose is the quiet admission that the model does not
actually consume the whole city** — see the scale problem below.

## The twist: the accumulated map may be a teacher, not an input

**LIE does not deploy the offline map.** It uses it as a **knowledge-distillation teacher**: the
teacher branch fuses student LiDAR features with the corresponding 2D intensity map tile to supply
dense supervision, and **at inference the student sees ordinary single-sweep LiDAR.**

> **That is a third option nobody states plainly.** Aggregate **before inference** (this page),
> aggregate **after inference** ([the next page](framewise-fusion.html)), or **aggregate before
> *training* and throw the map away.** The accumulated map becomes a way of buying dense labels,
> not a runtime artifact.

## The numbers

**LIE**, nuScenes, 60 m × 30 m, mIoU on lane divider / pedestrian crossing / road boundary:

| Method | Modality | nuScenes mIoU |
|---|---|---|
| UniFusion | camera | 53.6 |
| **LIE** | **LiDAR only** | **58.0** |
| LiDAR2Map | camera + LiDAR | 58.1 |

**Argoverse 2:** LIE reaches **50.8 mIoU with only 10% fine-tuning**, against **HDMapNet at 41.8
trained on the full dataset.** Its abstract claims **8.2% higher mIoU than the state-of-the-art
camera-based model** on nuScenes.

**Two things to take from this table.** LiDAR-only draws level with camera+LiDAR fusion — the
intensity channel is carrying real semantic information, not just geometry. **And 58.0 mIoU is not
a solved task.** Anyone claiming offline mapping was finished years ago should be shown this row.

**VMA**, on annotation cost: **~160 minutes to annotate a scene spanning hundreds of metres**, a
reported **52.3% reduction in human effort.** Not zero. **A 52% cut in a job that still takes
2.5 hours per scene is what "automated mapping" currently means.**

## The scale problem, which is the actual open question

VMA's own framing names it: a **divide-and-conquer** scheme is needed to solve the **spatial
extensibility** problem. LIE cuts **tiles by ego pose**. Toronto-3D's baselines are evaluated on
crops.

> **So the map is a map, but the model still sees tiles.** Local geometry is in good shape.
> **Whether the lane you extracted in tile 400 connects to the one in tile 401 is answered by
> stitching logic, not by the network** — and that is where topology errors live. Compare
> [topology extraction](topology-extraction.html): the problem did not go away when the data got
> denser.

## Where you can actually run this today

**Toronto-3D** is the cheapest real entry point. Mobile laser scanning, **~1 km of road, over 78
million points, roughly 1000 points/m² on road surfaces**, from a 32-line sensor. **Eight semantic
classes, and one of them is `road marking`.** Public download; baselines including **RandLA-Net**
and **KPConv** have public code.

**An MLS point cloud is already the accumulated map** — stages 1–3 come pre-done, with labels.

> **And the benchmark reports that road marking is the class the baselines handle worst**, with
> kernel and network settings called out as needing improvement to capture it. **The weakest class
> is the interesting class.**

**What is *not* freely runnable, and it is worth knowing before you plan an evening:** LIE's code
is announced but not released. VMA ships a real checkpoint, but only for **iCurb**, which is
**aerial imagery** — its LiDAR path uses proprietary data.

## Two communities, one problem

This corner is worked on from **two directions that barely cite each other.** The
vision/robotics side produces LIE, MapTR and the online lineage. The **photogrammetry and remote
sensing** side has been extracting road markings from mobile laser scanning for years — Wen et al.
(ISPRS J, 2019) rasterize road-surface intensity into images and train **U-Net** on **3,000
highway/urban images plus 1,000 of an underground garage**, and the line continues into 2025.

**And the topology-correct graph-extraction work grew up somewhere else again: on aerial imagery** —
[iCurb](https://arxiv.org/abs/2103.17118) (RA-L 2021) and
[csBoundary](https://arxiv.org/abs/2111.06020) (RA-L 2022) both take an overhead image and emit a
road-boundary graph, trained on **2,049 NYC aerial images at 5000 × 5000 with RGB + infrared.**

> **Three literatures, one problem, three vocabularies.** If you search only "online HD map
> construction" you will miss two of them.

## Limitations, honestly

- **Wet road and specular returns** wreck intensity; so do worn or repainted markings.
- **Dynamic objects smear** into the accumulation unless filtered.
- **Intensity is not calibrated** across sensors or across time, which is why thresholds do not
  transfer and learned methods are preferred.
- **Pose error becomes blur.** No extractor recovers a lane line that was smeared over 30 cm during
  accumulation — the map inherits the localisation's errors permanently.

---

## Verify before citing

- [ ] LIE's **58.0 / 53.6 / 58.1** table and the exact evaluation range
- [ ] Whether LIE's vectorized results are post-processed from the raster or predicted directly
- [ ] Toronto-3D's **per-class** road-marking scores, not just overall mIoU
- [ ] VMA's **52.3%** — measured against what human baseline?
- [ ] The Wen et al. and *Remote Sensing* 12(9):1379 papers — **cited here from secondary summaries,
      not read** ⚠

*Written 2026-09-12. LIE and the repository details were read at source; items marked ⚠ were not.
Quotes are from the papers; the framing on this page is mine.*
