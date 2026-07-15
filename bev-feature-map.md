---
title: Building the BEV Feature Map
---

# Building the BEV Feature Map

[← back to overview](index.html)

Problem 1 of the [two-problem split](index.html): turn raw sensor data (perspective camera images,
LiDAR points) into a top-down bird's-eye-view feature grid that a mapping head can reason over. Three
methods define the landscape: **LSS** (the original lifting mechanism), **BEVFormer** (attention-based
lifting plus free temporal fusion), and **BEVFusion** (camera + LiDAR fusion). MapTR's encoder is a
descendant of these.

## Why BEV at all

Cameras produce perspective images; planning and mapping live in a metric top-down frame. Projecting
image pixels to BEV requires knowing each pixel's depth, which a monocular camera does not directly
give. The whole "lifting" problem is: how do you place a perspective feature at the right (x, y) in
the ground plane without ground-truth depth.

## LSS (Lift-Splat-Shoot, 2020, NVIDIA)

Not a mapping method: the **BEV lifting backbone** almost everything else builds on.

**Mechanism:**
- ResNet + FPN extracts features for each of the 6 cameras
- A depth head predicts a **discrete depth distribution** per pixel (D bins, e.g. D=41): not one
  depth, but a probability mass over depth candidates
- **Lift**: each pixel's feature is spread into 3D by the outer product of its feature vector with
  its depth distribution, producing D features along the camera ray, weighted by depth probability
- **Splat**: all 6 cameras' lifted features are placed into a shared 3D voxel grid using each
  camera's extrinsic + intrinsic calibration, then collapsed (summed) along height into a 2D BEV map
- **Shoot**: a task head (segmentation, detection) runs on the BEV feature map

**Why collapse 3D to 2D, and why sum**: HD mapping is a ground-plane problem (lane dividers,
boundaries, crossings all live at Z ≈ 0), so keeping the height dimension and running 3D convolutions
wastes memory and compute. Summing each vertical column into one BEV cell reduces memory by a factor
of Z. Summation (not max-pool) is correct because the depth distribution is probabilistic: a pixel's
feature is spread across depth bins, and summing preserves all contributions. LiDAR object detectors
(VoxelNet, PointPillars) keep 3D because height genuinely matters there (a pedestrian at 1.8m differs
from a curb at 0.3m); for camera map prediction, height collapse is the right call.

**The calibration dependency (important)**: the Splat step relies on accurate camera extrinsics
(position + orientation) and intrinsics (focal length, principal point). A 0.5° mounting-angle error
translates to about 26cm of positional error for a feature at 30m depth, a large fraction of the
0.5m Chamfer tolerance on the [nuScenes map benchmark](nuscenes-metrics.html). In production, thermal
expansion, vibration, and impacts cause extrinsic drift, so AV teams run **online recalibration**
(using overlapping camera fields of view and LiDAR cross-checks) to detect and correct drift without
taking the vehicle offline. Calibration accuracy is not metadata: when it degrades, it corrupts every
feature in the BEV grid.

## BEVFormer (2022, Shanghai AI Lab)

Replaces LSS's explicit depth distribution with an **attention-based** lifting, and adds temporal
fusion. Two stacked attention mechanisms:

- **Spatial cross-attention**: a fixed grid of learnable BEV spatial queries (one per BEV cell)
  pulls information from image features. Each query projects 3D reference points (sampled at multiple
  heights above its BEV cell) onto each camera plane using calibration, then attends to image
  features at those projected locations via deformable attention. This is the transformer lifting
  mechanism MapTR's BEV encoder inherited directly.
- **Temporal self-attention**: BEV queries also attend to the *previous* timestep's BEV feature map,
  ego-motion compensated via IMU/odometry. Free temporal fusion at no extra sensor cost.

**How this loosens LSS's rigidity**: LSS's splat is geometrically rigid, so a drifted extrinsic sends
every feature to the wrong BEV cell with no recourse. BEVFormer inherits the calibration dependency
for its reference-point projection, but each query *also* learns small attention offsets around the
projected location, attending slightly away from the geometric projection when that is where the
informative features are. Not a fix for calibration error, but a reduction in brittleness. The whole
lineage is, in part, a gradual softening of the hard geometric constraints the earliest methods
required.

## BEVFusion (2022, two independent papers: MIT and ADLab)

Fuses camera and LiDAR in BEV space:

- **Camera branch**: transformer/LSS-style lifting to BEV
- **LiDAR branch**: pillar pooling (PointPillars-style) to the same BEV grid
- Both projected to the same vehicle-frame BEV grid → channel concatenation + convolutional mixing →
  one unified BEV feature map

Richer than camera-only because LiDAR adds precise geometry (exact depth and structure) while camera
adds semantic richness (color, texture, sign/marking appearance). MapTRv2's fusion backbone is this.

## The stack split (how these compose)

```
BEVFormer  →  BEV encoder (spatial + temporal cross-attention)   →  [any task head]
BEVFusion  →  camera + LiDAR BEV fusion                          →  [any task head]
MapTR      →  BEVFormer-style encoder  +  novel vectorized decoder  →  polylines
MapTRv2    →  BEVFusion-style encoder  +  MapTR decoder             →  73–75% mAP
```

BEVFormer and BEVFusion solve the **encoder** side (build the BEV feature map). MapTR's contribution
is the **decoder** (turn that feature map into vectorized map instances). MapTRv2 plugs BEVFusion's
encoder into MapTR's decoder to get the best of both.

## Fusion strategies (quick reference)

| Strategy | Where fusion happens | Trade-off |
|---|---|---|
| Early fusion | raw sensor level | max info, but tight calibration + sync coupling |
| Mid-level BEV fusion (BEVFusion) | in the shared BEV grid | the common sweet spot: each modality lifted independently, fused once in BEV |
| Late fusion | after per-modality task heads | robust to one modality failing, but loses cross-modal cues |

## Interview framing

> "Building the BEV feature map is the encoder half of the stack. LSS started it with an explicit
> depth distribution and a splat into a voxel grid, which is geometrically rigid and fully dependent
> on calibration. BEVFormer replaced the depth distribution with spatial cross-attention (BEV queries
> project reference points and attend to image features) and added free temporal fusion by attending
> to the ego-motion-compensated previous frame. BEVFusion fuses camera and LiDAR in the shared BEV
> grid. MapTR reuses a BEVFormer-style encoder and contributes the vectorized decoder on top."

## Self-check

1. Why do you need "lifting" at all to go from camera image to BEV?
2. In LSS, why is the depth output a distribution and not a single value, and why sum (not max) on the Z-collapse?
3. Why is calibration accuracy a first-class correctness concern, not metadata?
4. What are BEVFormer's two attention mechanisms, and what does each do?
5. How does learned deformable attention soften the calibration rigidity of LSS?
6. Where does BEVFusion fuse the two modalities, and why is mid-level BEV fusion the common choice?
