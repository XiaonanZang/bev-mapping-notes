---
title: nuScenes & Map Metrics
---

# nuScenes & Map Metrics

[← back to overview](index.html)

The shared proving ground. Every method in this lineage trains and reports on **nuScenes**, so
understanding the dataset and its map-estimation metric matters both for the interview and for
reproducing any of these papers.

## nuScenes: the dataset

Released in 2019 by nuTonomy (later Aptiv, now Motional). The first large-scale public AV dataset to
combine a full surround sensor suite with annotated HD map layers, exactly the combination needed to
study online map generation.

- **Scenes**: 1,000 driving scenes of 20 seconds each, across Boston and Singapore (geographic and
  weather diversity).
- **Scale**: 1.4M camera images, 390K LiDAR sweeps, 23 object classes with 3D boxes.

### Sensor suite (per vehicle)

- **6 cameras**: full 360° surround (front, front-left, front-right, back, back-left, back-right)
- **1 LiDAR**: top-mounted, 32-beam, full 360° sweep at 20 Hz
- **5 RADAR** units, plus IMU and GPS
- Full extrinsic and intrinsic calibration provided (which is why calibration is a first-class
  concern in the [BEV lifting](bev-feature-map.html) methods)

### HD map layers (the part this lineage predicts against)

nuScenes ships a vector map per location with annotated layers: `lane_divider`, `road_boundary`,
`ped_crossing`, `drivable_area`, `road_segment`, `stop_line`, and more. These are the ground truth all
online map methods predict.

The three **canonical evaluation classes** for map estimation are: **lane divider**, **pedestrian
crossing**, **road boundary**.

### How it is used

The `nuscenes-devkit` (Python) loads scenes, renders sensor data, and exposes a map API for querying
HD map elements in a spatial range. The standard online-mapping setup queries the local map within a
**30m × 60m ego-centric BEV window** and uses it as prediction ground truth. Free to download after
registration.

## The metric: mAP with Chamfer distance

Map estimation is scored with **mean Average Precision (mAP)**, computed per map-element class over
three distance thresholds and averaged.

### Chamfer distance (the matching criterion)

A predicted map element is a polyline; so is the ground truth. They have variable length and no
point-to-point correspondence, so you cannot use a simple coordinate error. **Chamfer distance**
solves this: it is the average of nearest-point distances in *both* directions (every predicted point
to its nearest GT point, and every GT point to its nearest predicted point). This handles
variable-length polylines without requiring aligned points.

A predicted element counts as a true positive if its Chamfer distance to a GT element is below a
threshold.

### The three thresholds

mAP is averaged over Chamfer thresholds of **0.5m, 1.0m, and 1.5m**. Reporting across three
tolerances rewards both loose and tight geometric accuracy, so a method cannot look good by only
roughly matching shape.

### Why mAP, not IoU

Early work (HDMapNet) reported **IoU**, a pixel-level metric. Later work switched to **mAP**, an
instance-level metric, because IoU cannot tell you whether the pixels form the right *instances* with
the right *geometry*. This is the same raster-versus-vector distinction that motivates the whole
[raster-to-vector](raster-to-vector.html) shift. One consequence: IoU numbers (HDMapNet) and mAP
numbers (MapTR onward) are not directly comparable.

## Benchmark numbers at a glance

| Method | Metric | Score (camera-only unless noted) |
|---|---|---|
| LSS | IoU (drivable area) | ~72% (no map mAP; predates the benchmark) |
| HDMapNet | IoU per class | ~25% divider / ~55% crossing / ~45% boundary |
| MapTR | mAP | ~36–50% (by model size) |
| MapTRv2 | mAP | ~61–65% (camera); ~73–75% (fusion) |
| StreamMapNet | mAP | ~65–70% |

## Interview framing

> "Everything in the online-mapping lineage is benchmarked on nuScenes: 6 cameras plus a 32-beam
> LiDAR, with annotated vector map layers as ground truth in a 30x60m ego window. Map quality is mAP
> over Chamfer distance at 0.5/1.0/1.5m thresholds. Chamfer is used because predicted and GT polylines
> have no point correspondence, so you score nearest-point distance both ways. The field moved from
> IoU to mAP precisely because pixel IoU cannot measure whether you recovered the right instances and
> geometry."

## Self-check

1. What sensors does a nuScenes vehicle carry, and why is calibration provided?
2. What are the three canonical map-evaluation classes?
3. Why is Chamfer distance used instead of a direct point-to-point error?
4. What do the three thresholds (0.5/1.0/1.5m) accomplish?
5. Why did the benchmark move from IoU to mAP, and why can't you compare the two directly?
