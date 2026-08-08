---
title: BEV Mapping Notes
---

# BEV Mapping Notes

Study notes on **online HD map generation** for autonomous driving: how a vehicle builds a
local, vectorized HD map from its own sensors in real time, without relying on a pre-surveyed
map. The notes follow one connected lineage of methods on the shared nuScenes benchmark, pairing
the intuition with the architecture and the interview framing for each.

Written to be read on a phone in short sittings, then recalled cold.

The notes come in two parts. **Part I** follows how a vehicle builds a map from its own sensors.
**Part II** covers what has to happen next: map priors, crowdsourcing across a fleet, and moving to
a city you have never labelled.

## Why online HD mapping

An autonomous vehicle must always know where it is and where it is allowed to go. HD maps supply
both: lane boundaries, centerlines, intersection topology, road markings, and sign positions,
stored as precise vectorized structures the planner queries directly ("give me the centerline of
this lane", "which lanes can I legally enter from here?").

For a decade the industry built these maps offline: survey vehicles drove every road, engineers
hand-annotated the data, and the finished map was streamed to vehicles. It works, but it is
artisanal, cannot scale to every road, and goes stale when the road changes. **Online HD map
generation** removes that dependency: the vehicle constructs the local map as it drives, covering
only the area immediately ahead. Build the map as you go.

## The two-problem split

Every method in this lineage solves one or both of two separable problems:

1. **Build a BEV feature map** from raw sensors (perspective camera images, LiDAR points) into a
   top-down bird's-eye-view grid the mapping task can reason over.
2. **Predict map elements** from that BEV feature map: lane dividers, boundaries, crossings, as
   usable structured output.

LSS, BEVFormer, and BEVFusion solve problem 1. HDMapNet and MapTR solve problem 2. Keeping this
split in mind is the fastest way to place any new paper in the landscape.

## Part I roadmap

| # | Page | What it covers |
|---|---|---|
| 1 | **Overview & the lineage** (this page) | the problem, the two-problem split, the character story |
| 2 | [nuScenes & map metrics](nuscenes-metrics.html) | the dataset, sensor suite, HD map layers, mAP + Chamfer distance |
| 3 | [Building the BEV feature map](bev-feature-map.html) | LSS lift-splat (with runnable code), BEVFormer spatial/temporal attention, BEVFusion fusion |
| 4 | [Raster to vector: HDMapNet → DETR → MapTR](raster-to-vector.html) | the paradigm shift + MapTR's hierarchical queries, permutation-equivalent matching, Hungarian |
| 5 | [Refinements & temporal: MapTRv2 + StreamMapNet](maptr-refinements.html) | aux seg head, uniform sampling, streaming BEV memory |
| 6 | [Topology & lane-graph extraction](topology-extraction.html) | BEV pixels to a navigable lane graph, intersections, HD map formats |
| 7 | [Auto-labeling & active learning](auto-labeling.html) | solving the ground-truth problem: geometry priors, active learning, SD-to-HD lifting |

## The lineage, as a story

- **LSS (2020)** is the BEV lifting backbone. It turns perspective camera pixels into a BEV
  feature map by predicting a per-pixel depth distribution and "splatting" features along the
  camera ray. Not a mapping method itself, but the technical foundation almost everything else
  stands on.
- **HDMapNet (2022)** is the first to frame online HD mapping as a direct learning problem on
  nuScenes. It predicts per-pixel segmentation masks, then recovers vectors with hand-crafted
  post-processing. It set the benchmark, and exposed the core limitation: a pixel mask does not
  say which pixels form one lane instance, in what order, or how lanes connect.
- **DETR (2020)** is the idea donor from object detection: a query-based transformer decoder plus
  Hungarian matching that emits a *set* of structured predictions end-to-end, with no anchors and
  no post-processing.
- **MapTR (2022)** is the main character. It transplants DETR's set-prediction idea into mapping
  and outputs vectorized instances directly (typed, ordered polylines), eliminating the brittle
  post-processing stack. Its innovations: a hierarchical instance-plus-point query design and
  permutation-equivalent matching for ambiguous point orderings.
- **MapTRv2 (2023)** refines training (an auxiliary dense segmentation head, uniform point
  sampling) and adds fusion, pushing accuracy well past v1.
- **StreamMapNet (2023)** adds temporal depth: an ego-motion-compensated streaming BEV memory
  that accumulates evidence across frames, filling in occluded or briefly-seen elements.

## The lineage at a glance

| Method | Year | Role | Architecture summary | nuScenes mAP |
|---|---|---|---|---|
| LSS | 2020 | BEV lifting | depth dist. → outer product → voxel collapse → BEV | ~72% IoU (drivable area); no map mAP |
| HDMapNet | 2022 | map (raster) | LSS BEV → seg heads → post-processing | ~25–55% IoU per class (not mAP) |
| DETR | 2020 | detection (idea donor) | CNN → transformer enc/dec → Hungarian matching | ~42 AP on COCO (not nuScenes) |
| MapTR | 2022 | map (vector) | hierarchical queries + permutation-equiv. matching | ~36–50% mAP (camera-only) |
| MapTRv2 | 2023 | map (vector) | + aux seg head + uniform point sampling | ~61–65% mAP (cam); ~73–75% (fusion) |
| StreamMapNet | 2023 | map (vector) | + ego-motion-compensated BEV memory (temporal) | ~65–70% mAP (camera-only) |

## Part I in one line

BEV segmentation (LSS + HDMapNet) solved the learning problem but produced the wrong output
format. MapTR solved the format problem with end-to-end vectorized set prediction borrowed from
DETR. MapTRv2 and StreamMapNet refined the training and added temporal depth, all measured on the
same nuScenes map benchmark.

---

# Part II: priors, crowdsourcing, and the new city

Part I answers one question. How does a vehicle build a map from its own sensors?

By StreamMapNet that question is largely settled. A different set of problems is left standing.
These are the ones that decide whether any of it reaches a public road.

## Part II roadmap

| # | Page | The one idea |
|---|---|---|
| 8 | [P-MapNet: SD map priors and far-seeing](p-mapnet.html) | a free OpenStreetMap road graph extends useful range far past the sensors. **The gain grows with distance** |
| 9 | [PriorDrive: unifying every kind of prior](priordrive.html) | SD maps, stale vendor HD maps and past drives, all encoded as vectors in **one representation** |
| 10 | [RTMap: crowdsourcing a map across many vehicles](rtmap.html) | many traversals fused into a **self-evolving map**, with per-vertex uncertainty and change detection |
| 11 | [CSMapping: a generative prior over what maps look like](csmapping.html) | crowdsourced noise cleaned by a **diffusion prior** over real map structure |
| 12 | [CityGen: adapting to a city you have never labelled](citygen.html) | a Singapore model scores **10.18 mAP** in Boston. Unlabelled video plus synthesis closes part of the gap |

## The three problems

**1. Range.** An online map cannot see around a curve. A prior can. *Pages 8 and 9.*

**2. Persistence.** A per-drive map is thrown away. Nothing accumulates. Nothing can be audited
before a vehicle relies on it. Crowdsourcing is how a map gets built and stays alive.
*Pages 10 and 11.*

**3. The new city.** Everything above still collapses when the deployment moves. *Page 12.*

## One requirement keeps reappearing

Every paper in Part II needs a shared coordinate frame. Each solves it differently.

| Paper | How it handles the frame |
|---|---|
| P-MapNet | searches for the alignment of a weakly aligned prior |
| PriorDrive | injects 1 m of Gaussian noise to model localization error |
| RTMap | fuses traversals with respect to a base frame |
| CSMapping | runs a factor graph over submaps for global consistency |

Four papers. Four methods. One requirement: **same place, same frame.**

Nobody writes a paper about this. It moves no benchmark number. It is still the thing that decides
whether many vehicles can contribute to one map.

## Simulated history is not crowdsourcing

PriorDrive builds its "historical" priors by running models over nuScenes. nuScenes was collected by
a single vehicle. The repeated passes are imitated, not real.

RTMap and CSMapping are the genuinely multi-vehicle work. **The difference between them is the
entire engineering problem.**

## Part II in one line

Sensors alone give a short-range map that is thrown away every drive and fails in a new city.
Priors buy range. Crowdsourcing buys persistence. Synthesis buys the new city. All three need a
shared frame, and none of them removes the need to prove the map is right.

---

*These notes are written to be read on a phone in fragmented time, then recalled from a blank page.*
