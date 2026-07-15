---
title: "Refinements & Temporal: MapTRv2 + StreamMapNet"
---

# Refinements & Temporal: MapTRv2 + StreamMapNet

[← back to overview](index.html)

MapTR set the vectorized paradigm ([raster-to-vector](raster-to-vector.html)). Two successors push
it further without changing the core idea: **MapTRv2** refines training and adds fusion, and
**StreamMapNet** adds temporal depth. Both are measured on the same [nuScenes benchmark](nuscenes-metrics.html).

## MapTRv2 (2023, HUST VL Lab)

A training and architecture refinement of MapTR, not a new paradigm. It targets two practical
weaknesses of v1: slow BEV-feature convergence, and suboptimal point distribution along polylines.

### Auxiliary dense prediction head

A pixel-level segmentation head runs **jointly with** the vectorized decoder during training. The
segmentation loss forces the BEV feature map to learn dense, spatially precise representations early,
so the vectorized decoder benefits from better-initialized features. At inference the segmentation
head is **discarded**; only the vectorized decoder runs.

This is the interesting design idea worth naming: a *dense auxiliary task used only during training*
to shape the shared feature map, then thrown away. It gives the decoder a warmer start without any
inference cost.

### Uniform point sampling

v1 placed the K points at fixed intervals in *parameter space*. v2 resamples them to be evenly
distributed along the actual **arc length** of the polyline. This matters for long, curved elements
(lane dividers on bends), where fixed-interval sampling clusters points in straight sections and
leaves gaps through curves, exactly where geometric precision matters most.

### More element types

Stop lines and road dividers are added to the prediction set.

### Performance (nuScenes)

- Camera-only: ~61–65% mAP (a large gain over MapTR-small's ~50%)
- Camera + LiDAR fusion: ~73–75% mAP
- State of the art on the nuScenes online-map benchmark at publication

MapTRv2's fusion backbone is [BEVFusion](bev-feature-map.html)-style; the encoder-plus-decoder
composition is "BEVFusion encoder + MapTR decoder".

## StreamMapNet (2023)

Extends the MapTR paradigm with one targeted addition: **temporal fusion**. MapTR is single-frame: it
predicts the local map from the current observation only. In practice map elements are often
partially occluded, at the edge of the sensor field of view, or only briefly visible. Accumulating
evidence across consecutive frames produces more complete and consistent predictions.

### Streaming BEV memory

- Same MapTR-style BEV feature extraction and vectorized decoder
- Added: a **recurrent BEV memory buffer** that stores BEV features from previous frames,
  **ego-motion compensated** into the current frame's coordinate system (using IMU/odometry to warp
  past features to the current ego pose)
- The current-frame BEV features and the warped historical memory are fused (concatenation +
  convolution, or cross-attention) before entering the decoder
- The decoder therefore sees a temporally aggregated BEV representation, improving recall on
  partially observed elements

**Pipeline:** current-frame cameras → BEV features → fuse with ego-motion-warped historical memory →
MapTR decoder → vectorized map.

### The ego-motion compensation detail

The reason you cannot just concatenate the previous BEV map: the vehicle has moved, so the previous
frame's grid is in a different coordinate frame. You **warp** the stored features into the current
ego pose using odometry before fusing, so a lane segment seen 3 frames ago lands at the correct
current (x, y). This is the same idea as BEVFormer's temporal self-attention, applied to map
prediction with an explicit memory.

### Performance (nuScenes, camera-only)

~65–70% mAP, outperforming single-frame MapTR by a substantial margin, particularly on long lane
dividers where temporal accumulation fills in occluded segments.

## How they relate

| | Adds over MapTR | Mechanism | Gains most on |
|---|---|---|---|
| MapTRv2 | training refinement + fusion | aux seg head, uniform sampling, LiDAR fusion | overall accuracy, curved elements |
| StreamMapNet | temporal fusion | ego-motion-compensated streaming BEV memory | occluded / briefly-seen elements |

They are complementary: one improves the per-frame feature quality, the other adds cross-frame
evidence. Both keep MapTR's hierarchical-query vectorized decoder unchanged.

## Interview framing

> "MapTRv2 is a training refinement: an auxiliary dense segmentation head that shapes the BEV features
> during training and is dropped at inference, plus uniform arc-length point sampling for curved
> elements, plus fusion. It roughly doubles the useful accuracy over MapTR. StreamMapNet adds the
> temporal axis MapTR lacks: a streaming BEV memory, ego-motion compensated into the current frame,
> so evidence accumulates across frames and fills in occluded lane segments. Neither changes the core
> vectorized decoder; they improve the feature map and add time."

## Self-check

1. What does MapTRv2's auxiliary segmentation head do, and why is it discarded at inference?
2. Why does uniform arc-length point sampling matter on curves specifically?
3. What single capability does StreamMapNet add that single-frame MapTR lacks?
4. Why must the historical BEV memory be ego-motion compensated before fusing?
5. On which elements does temporal accumulation help most, and why?
