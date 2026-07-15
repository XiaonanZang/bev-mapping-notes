---
title: Auto-labeling & Active Learning
---

# Auto-labeling & Active Learning

[← back to overview](index.html)

Every method in the [lineage](index.html) trains on nuScenes, where the HD-map ground truth is
pre-annotated and handed to you. In production that dataset does not exist: a new deployment area has
no labels, and you cannot hand-annotate every road (or every facility) at scale. This page is about
how you generate the labels, using geometric priors and model confidence, so that human effort is
spent only where it is actually needed.

## The three-layer structure of the labeling problem

A useful way to decompose "how do I get labels without annotating from scratch": split the target map
into layers by how expensive each is to label, and attack the cheap ones first.

| Layer | Source | Human effort | What it covers |
|---|---|---|---|
| 1. Geometry | SLAM / accumulated sensor sweeps | none | free occupancy: where surfaces are |
| 2. Structure | a prior map (SD map / floor plan) | none | the structural skeleton: road/lane or wall layout |
| 3. Semantics | active learning | low, targeted | the residual: everything not in the prior |

### Layer 1: geometry comes for free from SLAM

Mobile robots and AVs already solve localization-and-mapping from raw sensors, no labels required.
A spinning LiDAR gives a ring of range measurements per scan; SLAM runs a loop:

1. **Scan matching** (ICP / NDT): align the current scan to the previous one to estimate motion
2. **Map update**: ray-cast each return, marking cells along the ray as free and the return cell as
   occupied, building a probabilistic occupancy grid
3. **Loop closure**: recognize revisited places and correct accumulated drift

The output is an occupancy grid built purely from geometric consistency: no labels, no annotation,
no prior needed.

### Layer 2: a structural prior collapses most of the annotation

Most deployment areas come with a coarse structural map. Indoors it is a CAD floor plan (centimeter-
precision walls, room topology, doorways). Outdoors it is an **SD map** (OpenStreetMap-style road
centerlines, meter-level). Aligning the SLAM occupancy grid to that prior is a **registration
problem**, solvable automatically. After alignment, every occupied cell that matches a prior
structural element is auto-labeled (wall class indoors, road/lane skeleton outdoors) with no human in
the loop. The structural layer of the semantic map becomes free.

### Layer 3: active learning handles only the residual

What remains after layers 1 and 2 is everything not in the prior: furniture and clutter indoors;
signs, markings, temporary obstacles, and construction outdoors. This dynamic/semantic residual is a
far smaller, more tractable annotation target than labeling from scratch. **Active learning**
concentrates human effort here:

- The model's **uncertainty** selects which samples are most worth labeling (uncertainty sampling;
  core-set or diversity-based selection are alternatives)
- A human labels only that selected subset
- Label quality propagates back through retraining, and the loop repeats

The result is a data flywheel: new areas generate new failures, which select new labels, which
improve the model.

## The AV parallel: SD-to-HD map lifting

The three-layer idea has a direct research analogue in AV mapping. AV teams face the same structure:

- **Layer 1**: accumulated LiDAR sweeps from fleet drives → occupancy (free)
- **Layer 2**: OpenStreetMap / SD map → road-network topology as a structural prior (free, globally
  available)
- **Layer 3**: an ML model lifts the SD prior to HD detail (precise lane geometry, markings, signs)

Recent work formalizes layer 3 as **SD-to-HD map lifting**, feeding a coarse prior into a MapTR-style
network:

- **MapEX (2023)**: feeds the OSM road skeleton as an additional input to MapTR; the model attends to
  the prior plus BEV sensor features to predict precise lane boundaries.
- **P-MapNet (2023)**: rasterizes the SD map as an extra input channel, treating the prior as another
  sensor modality.
- **NMP (Neural Map Prior, 2023)**: stores learned map representations indexed by location and
  queries them at inference as a structural prior built from past drives.

The unifying idea: a coarse, freely-available structural prior (SD map) is fused with live sensor
features so the network only has to predict the *residual* detail, not the whole map from scratch.
This is exactly layer 3 of the labeling decomposition, expressed as an architecture.

## Why this matters more than the benchmark

A subtle point that rarely gets discussed: every nuScenes number in this whole lineage assumes
someone already paid the annotation cost. The harder, real problem when entering a new operational
area is that no ground truth exists yet. The auto-labeling and active-learning stack is what makes
scaling to new areas affordable, and it is a harder version of the **ODD-expansion** problem: each new
city (or facility) is a new environment you cannot hand-label, so you generate labels from geometric
priors and model confidence, verify selectively, and iterate.

## Interview framing

> "The literature trains on nuScenes ground truth, but the production problem is that a new area has
> no labels. I think of it in three layers by labeling cost: geometry is free from SLAM occupancy;
> the structural skeleton is free by registering an SD map or floor-plan prior to that occupancy; and
> active learning spends limited human effort only on the semantic residual, selected by model
> uncertainty. The AV research analogue is SD-to-HD lifting, MapEX, P-MapNet, NMP, which feed an SD
> map prior into a MapTR-style network so it only predicts the residual detail. It is the same idea
> as the labeling decomposition, expressed as an architecture."

## Self-check

1. Why can't you rely on the nuScenes-style pre-annotated ground truth in a new deployment area?
2. What does each of the three layers (geometry / structure / semantics) cost in human effort, and where does each come from?
3. How does a structural prior (SD map / floor plan) get turned into free labels?
4. What does active learning select for, and why does that minimize human effort?
5. Name the three SD-to-HD lifting methods and the single idea they share.
6. Why is auto-labeling framed as a harder version of the ODD-expansion problem?
