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
    # last axis = the 3 COORDINATES (u, v, depth), NOT rgb. it's a coordinate grid, no features yet.
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
    depth = depth_logits.softmax(dim=0)                 # (D,H,W) depth distribution, sums to 1 over D
    # per-pixel outer product  d (len D)  x  c (len C)  ->  a D x C matrix at EACH pixel:
    # each depth bin gets a full copy of the feature c, scaled by that bin's probability.
    feat = depth.unsqueeze(1) * context.unsqueeze(0)    # (D,1,H,W)*(1,C,H,W) -> (D,C,H,W)
    return feat.permute(0, 2, 3, 1)                     # (D, H, W, C)


# STEP 3: voxel_pooling -- the SPLAT. this is a group-by-key SUM: key=(x,y) cell, value=C-vector.
def voxel_pooling(ego_pts, feat):
    xlo, xstep, nx = make_grid(xbound)
    ylo, ystep, ny = make_grid(ybound)
    pts = ego_pts.reshape(-1, 3)                # N=D*H*W points, each an (x,y,z)   <- the "entries"
    f = feat.reshape(-1, C)                     # each point's C-vector             <- the "value"
    ix = ((pts[:, 0] - xlo) / xstep).long()     # integer x-cell
    iy = ((pts[:, 1] - ylo) / ystep).long()     # integer y-cell  (z is never used -> height collapses)
    keep = (ix >= 0) & (ix < nx) & (iy >= 0) & (iy < ny)   # drop points outside the grid
    ix, iy, f = ix[keep], iy[keep], f[keep]
    bev = torch.zeros(nx, ny, C)
    flat = ix * ny + iy                         # one integer per point = the (x,y) "dict key"
    # group-by-key sum: add each point's feature into its cell's row. differentiable scatter-add;
    # real LSS uses QuickCumsum (sort-by-key + cumsum + boundary-diff) for the same result, faster.
    bev.view(nx * ny, C).index_add_(0, flat, f)
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

### BEVFormer, in code

The mental shift from LSS: LSS **pushes** features out (forward, depth-based). BEVFormer **pulls**
them in (backward). You start from a grid of learnable **BEV queries**, one per ground cell, and
each query goes and *fetches* the features it needs: from the camera image (spatial cross-attention)
and from the past (temporal self-attention). No depth estimation anywhere.

```
BEV queries Q:  a grid of H_bev x W_bev learnable C-vectors, one per ground cell (x, y)
```

**One encoder layer (pseudo-code).** BEVFormer stacks L of these. Order matters: temporal first,
then spatial, then FFN.

```
BEVFORMER_LAYER(Q, image_feats, prev_bev, ego_motion):

    # 1. TEMPORAL SELF-ATTENTION: each query pulls from the previous frame (BEV -> BEV)
    prev_aligned = warp(prev_bev, ego_motion)      # move t-1 BEV into the current ego frame
    Q = norm(Q + temporal_self_attn(Q, ref=own_cell, value=prev_aligned))

    # 2. SPATIAL CROSS-ATTENTION: each query pulls from the camera images (BEV -> image)
    for each query q at ground cell (x, y):
        pillar = [(x,y,z_1), ..., (x,y,z_k)]        # sample several HEIGHTS along the pillar
        for each 3D point in pillar:
            for each camera it projects into (hit-test via K, R, t):
                uv = project(point, camera)          # 3D -> image pixel = the reference point
                q += deformable_attn(q, ref=uv, value=image_feat[camera])
    Q = norm(Q)

    # 3. feed-forward
    Q = norm(Q + FFN(Q))
    return Q
```

**The shared engine: deformable attention.** Both attentions are the same operation with a
different `value`. It departs from vanilla attention in a way worth stating precisely: there is
**no K and no QKᵀ dot product**. The attention weights are predicted **directly from the query** by
a linear layer; the value is **sampled** at a reference point plus learned offsets.

```python
class DeformAttn(nn.Module):
    def __init__(self, C, n_pts):
        super().__init__()
        self.offset = nn.Linear(C, n_pts * 2)     # query -> WHERE to sample (offsets from the ref point)
        self.attn   = nn.Linear(C, n_pts)         # query -> HOW MUCH to weight each sample (no K!)
        self.n_pts  = n_pts

    def forward(self, query, ref_pts, value):
        # query:(Nq,C)  ref_pts:(Nq,2) in [-1,1]  value:(C,Hv,Wv)
        offsets = self.offset(query).view(-1, self.n_pts, 2) * 0.1   # (Nq, n_pts, 2)
        weights = self.attn(query).softmax(-1)                       # (Nq, n_pts) weights straight from Q
        samples = ref_pts[:, None, :] + offsets                      # look NEAR the reference point
        grid    = samples.view(1, -1, self.n_pts, 2)
        sampled = F.grid_sample(value[None], grid, align_corners=True)  # bilinear sample the value map
        sampled = sampled[0].permute(1, 2, 0)                        # (Nq, n_pts, C)
        return (sampled * weights[..., None]).sum(1)                  # (Nq, C) weighted sum
```

Why swap out QKᵀ: vanilla attention over a 200x200 BEV grid attending to full image/history maps is
`O((HW)^2)`, intractable. Deformable makes it `O(HW * n_pts)` with `n_pts` around 4-8. The `offset`
net is also what gives the soft tolerance to calibration error: the query can sample slightly away
from the projected reference point.

**Deformable self-attention vs conventional self-attention, side by side.** The clearest way to see
what deformable attention drops. Conventional first:

```python
class SelfAttnHead(nn.Module):            # the vanilla head
    def __init__(self, C):
        self.q = nn.Linear(C, C)
        self.k = nn.Linear(C, C)          # <-- K exists
        self.v = nn.Linear(C, C)
        self.proj = nn.Linear(C, C)

    def forward(self, x):                 # x: (N, C) tokens
        Q, K, V = self.q(x), self.k(x), self.v(x)   # all three from the SAME x  ("self")
        scores = Q @ K.T / sqrt(C)        # (N, N)  <-- ALL-PAIRS dot product, O(N^2)
        A = scores.softmax(dim=-1)        # (N, N)  softmax over ALL N keys
        out = A @ V                       # (N, C)  weighted sum over all tokens
        return self.proj(out)
```

Deformable:

```python
class DeformSelfAttnHead(nn.Module):
    def __init__(self, C, n_pts):
        self.v = nn.Linear(C, C)              # V still exists
        self.offset = nn.Linear(C, n_pts * 2) # query -> WHERE to sample   (REPLACES K)
        self.attn = nn.Linear(C, n_pts)       # query -> HOW MUCH to weight (REPLACES Q@K.T)
        self.proj = nn.Linear(C, C)
        self.n_pts = n_pts

    def forward(self, x, ref_pts, value_map): # x:(N,C)  ref_pts:(N,2) in[-1,1]  value_map:(C,H,W)
        offsets = self.offset(x).view(-1, self.n_pts, 2) * 0.1  # (N, n_pts, 2)  WHERE
        weights = self.attn(x).softmax(dim=-1)                  # (N, n_pts)  HOW MUCH (over n_pts, NOT N)
        samples = ref_pts[:, None, :] + offsets                 # (N, n_pts, 2)  near the reference point
        vmap = self.v(value_map.permute(1, 2, 0)).permute(2, 0, 1)   # value-projected feature map
        sampled = F.grid_sample(vmap[None], samples.view(1, -1, self.n_pts, 2), align_corners=True)
        sampled = sampled[0].permute(1, 2, 0)                   # (N, n_pts, C)
        out = (sampled * weights[..., None]).sum(1)             # (N, C)  weighted sum over n_pts
        return self.proj(out)
```

Line for line:

| | Conventional | Deformable |
|---|---|---|
| Q projection | `q(x)` | `x` drives offsets/weights directly |
| K projection | `k(x)` present | **gone** |
| V projection | `v(x)` | `v(value_map)` |
| Where weights come from | `Q @ Kᵀ` (compare query to every key) | `Linear(x)` (predict directly) |
| Softmax over | `N` keys (all tokens) | `n_pts` sample points (4-8) |
| What it attends to | **all** N positions (global) | a few **sampled** points near a reference (local) |
| Needs a reference point? | no (positional encoding) | **yes** (a spatial location to sample around) |
| Cost | `O(N^2)` | `O(N * n_pts)` |

Verified cost on a 50x50 grid (N=2500):

```
CONVENTIONAL:  score matrix (N,N) = (2500, 2500) = 6,250,000 entries
DEFORMABLE:    weights   (N,n_pts) = (2500, 4)    =    10,000 entries   -> 625x fewer
```

On the real 200x200 BEV (N=40,000), conventional self-attention is ~1.6 billion pairwise scores per
layer per head; deformable is 40,000 x 4 = 160,000. That factor is why BEVFormer can afford temporal
attention over a full BEV grid every frame.

The takeaway: conventional attention **discovers** where to look by comparing the query to every key
(`QKᵀ`); deformable **predicts** where to look with a linear layer and samples the value there. It
trades a content-based global lookup for a learned, sparse, spatial one. For BEV that is exactly
right, the evidence for a cell is spatially near its own location (previous BEV) or near its
projected pixel (image), so a learned local sample is all you need.

**Spatial cross-attention (BEV query -> image features).** The value is the **image feature maps
from the camera backbone**; that different domain is what makes it *cross*.

```python
def spatial_cross_attn(bev_q, bev_xy, img_feat, deform, project_fn):
    ref_img = project_fn(bev_xy)              # BEV cell (x,y) -> its (u,v) pixel in the image, in [-1,1]
    return deform(bev_q, ref_img, img_feat)   # deformable-attend at that projected location
```

The whole content here is `project_fn`: the `K, R, t` projection from a ground cell to an image
pixel. Note that one BEV cell fans out to **many `(u,v)`**: `k` pillar heights each project to a
pixel, in each of the 1-2 cameras the point hits. BEVFormer averages over those valid (hit)
reference points. (This is BEVFormer hedging the unknown **height**, the same way LSS hedges the
unknown **depth** with a distribution: cover the missing 3rd axis by sampling several candidates.)

**Temporal self-attention (BEV query -> previous BEV features).** Same engine; the value is the
**warped previous BEV**, so it stays inside the BEV domain, which is what makes it *self*.

```python
def temporal_self_attn(bev_q, bev_grid_xy, prev_bev, deform, warp_fn):
    prev_aligned = warp_fn(prev_bev)          # ego-motion compensate the t-1 BEV into the current frame
    ref_self = bev_grid_xy                    # each query attends around ITS OWN cell in the prev BEV
    return deform(bev_q, ref_self, prev_aligned)
```

The key line is `warp_fn`: the vehicle moved between `t-1` and `t`, so the previous BEV grid is in a
different coordinate frame. You warp it by the ego motion so a lane segment seen last frame lands at
the correct *current* `(x,y)`, then each query looks up its own cell in that aligned history. (Real
BEVFormer attends to both current queries and the warped previous BEV; the previous-frame half is
the new idea.)

**The value source is the whole story of self vs cross:**

| | Query (offsets + weights from) | Value (features sampled) | domain |
|---|---|---|---|
| Spatial **cross**-attn | current BEV query | **image** backbone features | BEV → image → *cross* |
| Temporal **self**-attn | current BEV query | previous **BEV** features (warped) | BEV → BEV → *self* |

The query is always the same BEV cell (it drives *where* to look and *how much* to weight). Only the
value source changes, and its domain is what names the attention.

**Runnable mini (verified output):** single camera, simplified projection/warp, tiny grid.

```python
import torch, torch.nn as nn, torch.nn.functional as F
# ... DeformAttn, spatial_cross_attn, temporal_self_attn as above ...

Nq = H_bev * W_bev                          # 50*50 = 2500 BEV cells
bev_q   = torch.randn(Nq, C)                # learnable BEV queries, flattened
deform  = DeformAttn(C, n_pts)
img_feat = torch.randn(C, Hf, Wf)           # from the image backbone   (the CROSS value)
prev_bev = torch.randn(C, H_bev, W_bev)     # BEV feature map at t-1     (the SELF value)

tsa     = temporal_self_attn(bev_q, bev_grid_xy, prev_bev, deform, warp_fn)   # pull from past
sca     = spatial_cross_attn(tsa, bev_world_xy, img_feat, deform, project_fn) # pull from image
bev_out = sca.view(H_bev, W_bev, C).permute(2, 0, 1)
```

```
BEV queries (Nq, C):         (2500, 32)      2500 = 50x50 BEV cells, each a 32-vector
after temporal self-attn:    (2500, 32)      pulled from warped previous BEV
after spatial cross-attn:    (2500, 32)      pulled from image features
BEV feature map (C,H,W):     (32, 50, 50)    reshape back to the grid -> ready for a task head
```

Same output shape as LSS's BEV map (`C x H x W`), reached by *pulling* instead of *pushing*.

**The one-line contrast:** LSS decides where each pixel goes (forward, depth-based); BEVFormer lets
each BEV cell decide which pixels to look at (backward, attention-based), and gets temporal fusion
for free by also looking at the warped previous BEV.

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
