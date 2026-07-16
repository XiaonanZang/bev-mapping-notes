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

### LSS lift-splat, in code

The whole camera-to-BEV lift is three ideas, and each is one or two lines of tensor math. LSS is
fully open source ([`nv-tlabs/lift-splat-shoot`](https://github.com/nv-tlabs/lift-splat-shoot),
`src/models.py`).

**Step-by-step workflow (pseudo-code).** The abstract shape before the real code:

```
LIFT-SPLAT-SHOOT :  N camera images  ->  one BEV feature map

# ---- one-time setup: the ray grid ----
frustum = create_frustum()
    for each depth bin d in [d_min .. d_max]:          # D bins
        for each feature-map pixel (u, v):             # H x W
            store the point (u, v, d)                  # -> (D, H, W, 3): a ray of D points per pixel

# ---- per frame ----
for each camera n in 1..N:

    # STEP 2  GEOMETRY: where does each frustum point land in 3D?
    ego_pts[n] = extrinsic[n] @ ( intrinsic[n]^-1 @ frustum )    # (D,H,W,3) in ego meters

    # STEP 1  LIFT: attach a feature to each point, weighted by soft depth
    depth_logits, context = backbone(image[n])         # (D,H,W) , (C,H,W)
    depth   = softmax(depth_logits, over D)            # depth distribution, sums to 1
    feat[n] = depth  (outer product)  context          # (D,H,W,C)

# STEP 3  SPLAT: drop every point into the BEV grid, SUM features per cell
bev = zeros(C, X, Y)
for every lifted point p across all cameras:
    (x, y) = ego_pts[p][:2]                            # ignore z  -> height collapses
    if (x, y) inside grid:
        bev[:, cell_of(x, y)] += feat[p]               # summation along height

# STEP 4  SHOOT
output = task_head(bev)                                # segmentation / detection / MapTR decoder
```

Two things to notice in the shape of the loop: **geometry is frame-independent** (the frustum and
its unprojection depend only on calibration, so `create_frustum` runs once), while the **features
are per-frame** (the backbone re-runs each frame). And the splat is the only place the `N` cameras
*merge*: they all sum into the same shared BEV grid.

The three steps above map almost one-to-one onto LSS's methods:

| Step | LSS function | What it does |
|---|---|---|
| Build the ray grid | `create_frustum()` | makes the `(D, H, W, 3)` grid of `(u, v, depth)` |
| Unproject to ego 3D | `get_geometry()` | applies intrinsics⁻¹ + extrinsics |
| Lift features | `CamEncode.get_depth_feat()` | softmax the depth, then outer-product with context |
| Splat | `voxel_pooling()` | bin into the BEV grid, sum-pool (via a cumsum trick) |

**The three lines that *are* the method.** Strip away the plumbing and this is the whole thing.

*1. The depth distribution + outer-product lift* (`get_depth_feat`):

```python
depth = depth_logits.softmax(dim=0)                 # (D,H,W) sums to 1 over depth  -> "refuse to pick one depth"
feat  = depth.unsqueeze(1) * context.unsqueeze(0)   # (D,C,H,W)  -> outer product: depth  x  context
```

The multiply broadcasts `(D,1,H,W) * (1,C,H,W) -> (D,C,H,W)`: it smears each pixel's semantic
feature along its ray, weighted by how probable each depth is. That single line is the entire
"soft lift".

*2. The unprojection* (`get_geometry`):

```python
points  = cat((frustum[...,:2] * frustum[...,2:3], frustum[...,2:3]))  # (u*z, v*z, z)
cam_pts = (Kinv @ points.T).T          # intrinsics^-1  -> 3D camera frame
ego_pts = (rot @ cam_pts.T).T + trans  # extrinsics     -> ego meters
```

The `u*z, v*z, z` trick inverts the pinhole projection (which divides by z): pre-multiply `(u,v)`
by `z`, then `K^-1` recovers the true 3D point. This is where the depth distribution *becomes* a
3D spatial volume.

*3. The splat + height collapse* (`voxel_pooling`):

```python
ix = ((pts[:,0] - xlo) / xstep).long()   # which BEV x-cell
iy = ((pts[:,1] - ylo) / ystep).long()   # which BEV y-cell   (no z index -> height collapses)
bev.view(nx*ny, C).index_add_(0, flat, f)   # SUM every point's feature into its cell
```

Every frustum point is scattered into its `(x, y)` cell and **added**. Because points are indexed
by `(x, y)` only, all points in a vertical column land in the same cell: that is the "summation
along height", in one line. Real LSS makes this fast with a cumsum trick (`QuickCumsum`), but the
semantics are exactly this `index_add_`.

**The full runnable mini-implementation.** Single camera, single batch, tiny feature map for
readability. Faithful to the real geometry; runnable as-is (PyTorch only). The real repo handles
6 cameras, batching, cumsum-based pooling, and a ResNet/EfficientNet backbone.

```python
import torch

# ----- config -----
H, W = 8, 22            # feature-map size (tiny for readability; real ~ 8x22 after backbone)
C = 64                  # context feature channels
D = 41                  # number of depth bins
dbound = (4.0, 45.0, 1.0)     # depth 4m..45m, 1m steps -> D=41
xbound = (-50.0, 50.0, 0.5)   # BEV grid x, 0.5m cells
ybound = (-50.0, 50.0, 0.5)   # BEV grid y


def make_grid(bound):
    lo, hi, step = bound
    n = int((hi - lo) / step)
    return lo, step, n


# STEP 1: create_frustum -- for each depth bin d and pixel (u,v), make a point (u, v, depth)
def create_frustum():
    ds = torch.arange(*dbound).view(-1, 1, 1).expand(D, H, W)
    us = torch.linspace(0, W - 1, W).view(1, 1, W).expand(D, H, W)
    vs = torch.linspace(0, H - 1, H).view(1, H, 1).expand(D, H, W)
    return torch.stack((us, vs, ds), dim=-1)            # (D, H, W, 3)


# STEP 2: get_geometry -- unproject the (u,v,depth) frustum into 3D ego coordinates
def get_geometry(frustum, intrins, rot, trans):
    points = torch.cat((frustum[..., :2] * frustum[..., 2:3], frustum[..., 2:3]), dim=-1)  # (u*z, v*z, z)
    Kinv = torch.inverse(intrins)
    cam_pts = (Kinv @ points.reshape(-1, 3).T).T.reshape(D, H, W, 3)   # camera frame
    ego_pts = (rot @ cam_pts.reshape(-1, 3).T).T.reshape(D, H, W, 3) + trans
    return ego_pts                                      # (D, H, W, 3) ego meters


# STEP 1-2: the OUTER PRODUCT lift
def lift_features(depth_logits, context):
    depth = depth_logits.softmax(dim=0)                 # (D,H,W) the distribution
    feat = depth.unsqueeze(1) * context.unsqueeze(0)    # (D,C,H,W) depth x context
    return feat.permute(0, 2, 3, 1)                     # (D, H, W, C)


# STEP 3: voxel_pooling -- the SPLAT (bin by x,y; sum-pool; height collapses)
def voxel_pooling(ego_pts, feat):
    xlo, xstep, nx = make_grid(xbound)
    ylo, ystep, ny = make_grid(ybound)
    pts = ego_pts.reshape(-1, 3)
    f = feat.reshape(-1, C)
    ix = ((pts[:, 0] - xlo) / xstep).long()
    iy = ((pts[:, 1] - ylo) / ystep).long()
    keep = (ix >= 0) & (ix < nx) & (iy >= 0) & (iy < ny)   # drop points outside the grid
    ix, iy, f = ix[keep], iy[keep], f[keep]
    bev = torch.zeros(nx, ny, C)
    flat = ix * ny + iy
    bev.view(nx * ny, C).index_add_(0, flat, f)         # sum-pool = summation along height
    return bev.permute(2, 0, 1)                          # (C, X, Y)


if __name__ == "__main__":
    torch.manual_seed(0)
    intrins = torch.tensor([[120.0, 0.0, W / 2],
                            [0.0, 120.0, H / 2],
                            [0.0, 0.0, 1.0]])
    rot = torch.eye(3)                                  # camera aligned with ego (simplified)
    trans = torch.tensor([0.0, 0.0, 1.5])              # camera 1.5m above ego origin

    frustum = create_frustum()
    print("STEP 1  frustum (D,H,W,3):        ", tuple(frustum.shape))
    ego_pts = get_geometry(frustum, intrins, rot, trans)
    print("STEP 2  ego points (D,H,W,3):     ", tuple(ego_pts.shape))
    depth_logits = torch.randn(D, H, W)                 # backbone would produce these
    context = torch.randn(C, H, W)
    feat = lift_features(depth_logits, context)
    print("STEP 1-2 lifted feat (D,H,W,C):   ", tuple(feat.shape))
    bev = voxel_pooling(ego_pts, feat)
    print("STEP 3  BEV feature map (C,X,Y):  ", tuple(bev.shape))
```

Verified output:

```
STEP 1  frustum (D,H,W,3):         (41, 8, 22, 3)      D=41 depth bins x 8x22 pixels, each (u,v,depth)
STEP 2  ego points (D,H,W,3):      (41, 8, 22, 3)      same grid, now unprojected to 3D ego meters
STEP 1-2 lifted feat (D,H,W,C):    (41, 8, 22, 64)     the outer product: depth x context
STEP 3  BEV feature map (C,X,Y):   (64, 200, 200)      splatted + summed into a 200x200 BEV grid
```

The `(64, 200, 200)` BEV map is exactly what a task head (segmentation, or a MapTR-style decoder)
runs on. From here the [raster-to-vector page](raster-to-vector.html) picks up.

**Mini-version → production LSS:**

| Mini-version | Production LSS | Difference |
|---|---|---|
| single camera | 6 cameras | loop / batch over the `N` camera dimension |
| `index_add_` sum-pool | `QuickCumsum` | same result, cumsum trick for speed on GPU |
| random `depth_logits`, `context` | ResNet/EfficientNet `CamEncode` | a real backbone predicts `D + C` channels per pixel |
| no supervision | task loss (seg/det) | BEVDepth additionally supervises the depth with LiDAR |

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
