---
title: "Raster to Vector: HDMapNet → DETR → MapTR"
---

# Raster to Vector: HDMapNet → DETR → MapTR

[← back to overview](index.html)

The central plot of the whole lineage: the shift from predicting *pixels* (a rasterized map that
needs brittle post-processing) to predicting *vectorized instances* directly (typed, ordered
polylines a planner can read). This page walks HDMapNet (the raster baseline), DETR (the idea it
borrowed), and MapTR (the method that fused them), then covers MapTR's key concepts in depth.

## HDMapNet: the raster baseline and its limitation

HDMapNet (2022, Tsinghua MARS Lab) was the first to frame online HD mapping as a direct learning
problem on nuScenes.

**Pipeline:**
- LSS-style BEV lifting from cameras, plus optional pillar-based LiDAR features
- Camera and LiDAR features fused in BEV space
- A convolutional BEV encoder
- Three binary segmentation heads, one per class (lane divider, ped crossing, road boundary),
  predicted per BEV pixel
- **Hand-crafted post-processing**: morphological thinning to centerlines, then vectorization
  heuristics to turn raster centerlines into polylines

**Performance (nuScenes, IoU per class, cam+LiDAR):** lane divider ~25%, ped crossing ~55%, road
boundary ~45%.

**The limitation that motivates everything after it**: IoU is a pixel metric, and it hides the real
problem. A mask that labels pixels "lane divider" correctly still does not say which pixels form the
*same* lane, in what *order* it runs, or how lanes *connect*. That structure has to be recovered by
the post-processing stack, and that stack is brittle at intersections, lossy, and not trainable. The
output format is wrong for the consumer.

## DETR: the idea donor

DETR (2020, FAIR) is an object detector on COCO. It is in this story only for the architectural idea
MapTR transplanted.

**Core:**
- CNN backbone → flattened spatial tokens → transformer encoder (self-attention over tokens)
- Transformer decoder: **N learnable query embeddings** cross-attend to encoder output; self-
  attention between queries lets them coordinate (so two queries do not claim the same object)
- Two heads per query: class label + bounding box (x, y, w, h)
- **Hungarian matching** at training time: assign each ground-truth object to exactly one query via
  minimum-cost bipartite matching; unmatched queries predict "no object"; loss flows only through
  matched pairs

**The key idea: detection as direct set prediction.** No anchors, no non-maximum suppression, no
hand-crafted post-processing. The model emits a *set* of structured predictions end-to-end.

**What MapTR borrowed**: the query-based decoder and Hungarian matching. What it had to invent: the
geometry (bounding boxes → polylines) and the ordering-ambiguity solution (unique boxes → ambiguous
point orderings).

## MapTR: end-to-end vectorized mapping

MapTR (2022, HUST VL Lab, NeurIPS 2022) replaced the rasterized paradigm with end-to-end vectorized
instance prediction. It emits typed, ordered polylines directly, deleting the post-processing stack.

**Architecture:**
- Backbone (ResNet / VoVNet) extracts per-camera image features
- BEV lifting (GKT or deformable cross-attention; see [BEV feature map](bev-feature-map.html))
  produces a shared BEV feature map
- **Hierarchical transformer decoder** (the heart of MapTR)
- Heads: a classification head per instance, a point-regression head per point

**Pipeline:** 6 cameras → image features → BEV feature map → hierarchical decoder → set of
(class, ordered polyline) instances, directly usable.

**Performance (nuScenes mAP, camera-only, 3 classes):** MapTR-nano ~36.9%, MapTR-tiny ~43.7%,
MapTR-small ~50.3% (Chamfer thresholds 0.5/1.0/1.5m, averaged).

---

## Key concept 1: hierarchical query design

Two levels of query, which cleanly separate "find the element" from "trace its geometry":

- **N instance queries**, one per map element, capture global identity: *what and where* this
  element is.
- Each instance query expands into **K point queries** (e.g. K=20), one per point along the
  element's geometry. Conditioned on the instance, each point query attends locally to BEV features
  to regress its own (x, y).
- **Self-attention across all queries** (across instances and across points) lets neighboring
  elements resolve their spatial relationships.

This is why instance separation and point ordering come out natively, rather than being recovered
from a pixel mask.

## Key concept 2: deformable attention

Each point query does not attend to the whole BEV grid. It attends to a small, learned set of
sampled BEV locations, with reference points initialized from the instance query's predicted center.

- **Much cheaper** than full cross-attention: cost is proportional to a fixed number of sample
  points, not the BEV grid size, so it scales to large grids.
- **Learned offsets**: the query samples slightly *away* from the geometric reference when that is
  where the informative features are. This same mechanism gives a soft, learned tolerance to small
  calibration errors (see the calibration story on the [BEV feature map](bev-feature-map.html) page).

## Key concept 3: permutation-equivalent modeling (the signature innovation)

**Problem**: a polyline with K points has 2 equivalent orderings (forward and backward). A closed
polygon has 2K equivalents (K rotations × 2 directions). A naive L1 loss on point coordinates would
penalize a geometrically perfect prediction that happens to be ordered the other way, which
destabilizes training.

**Solution**: during training, for each predicted element, enumerate all valid equivalent orderings
of the ground truth and backpropagate only through the **minimum-cost** ordering. The model is
rewarded for correct geometry regardless of which equivalent direction it chose.

This is what makes vectorized training stable without sacrificing geometric precision. It is MapTR's
answer to the problem DETR never had (a box has no ordering ambiguity; a polyline does).

## Key concept 4: Hungarian matching (instance level)

- N queries, M ground-truth elements (M ≤ N)
- Hungarian algorithm finds the minimum-cost **bijective** assignment of GT elements to queries
- Unmatched queries predict "no object"
- Loss flows only through matched pairs

This gives **set prediction without positional bias**: the model is not forced to emit elements in
any fixed order or grid position. Borrowed directly from DETR.

## Training loss

| Component | Purpose |
|---|---|
| Classification loss (focal) | class prediction per instance query |
| Point regression loss (L1) | (x, y) coordinates, applied to the best-matching permutation |
| Instance-level Hungarian matching | assigns GT elements to queries |

Two nested matchings: **instance-level Hungarian** (which query owns which element) wrapping
**point-level permutation-equivalent** matching (which ordering to score each matched element by).

## The one-line arc

HDMapNet learned the map but emitted pixels, forcing a brittle post-processing stack. DETR showed
how to emit a *set* of structured objects end-to-end with queries and Hungarian matching. MapTR
transplanted that into mapping, added a hierarchical instance-plus-point query and permutation-
equivalent matching for polyline geometry, and got a directly usable vectorized map.

## Self-check

1. Why is a per-pixel IoU metric misleading for map quality?
2. What exactly did MapTR borrow from DETR, and what did it have to invent?
3. What do instance queries versus point queries each represent?
4. Why does point ordering create a loss problem, and how does permutation-equivalent matching fix it?
5. Why deformable attention instead of full cross-attention?
6. Describe the two nested matchings in MapTR's training.
