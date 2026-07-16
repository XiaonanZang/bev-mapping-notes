---
title: "LSS Lift-Splat, in Code"
---

# LSS Lift-Splat, in Code

[← back to overview](index.html) · [← BEV feature map](bev-feature-map.html)

The [BEV feature map page](bev-feature-map.html) explains *what* LSS does. This page shows the
actual code. The whole camera-to-BEV lift is three ideas, and each is one or two lines of tensor
math. Everything else is bookkeeping.

## The real source

LSS is fully open source: [`nv-tlabs/lift-splat-shoot`](https://github.com/nv-tlabs/lift-splat-shoot),
in `src/models.py`. Our three steps map almost one-to-one onto its methods:

| Step | LSS function | What it does |
|---|---|---|
| 1. Build the ray grid | `create_frustum()` | makes the `(D, H, W, 3)` grid of `(u, v, depth)` |
| 2. Unproject to ego 3D | `get_geometry()` | applies intrinsics⁻¹ + extrinsics |
| 1–2. Lift features | `CamEncode.get_depth_feat()` | softmax the depth, then outer-product with context |
| 3. Splat | `voxel_pooling()` | bin into the BEV grid, sum-pool (via a cumsum trick) |

## The three lines that *are* the method

Strip away the plumbing and this is the whole thing.

**1. The depth distribution + outer-product lift** (LSS's `get_depth_feat`):

```python
depth = depth_logits.softmax(dim=0)                 # (D,H,W) sums to 1 over depth  -> "refuse to pick one depth"
feat  = depth.unsqueeze(1) * context.unsqueeze(0)   # (D,C,H,W)  -> outer product: depth  x  context
```

The multiply broadcasts `(D,1,H,W) * (1,C,H,W) -> (D,C,H,W)`: it smears each pixel's semantic
feature along its ray, weighted by how probable each depth is. That single line is the entire
"soft lift."

**2. The unprojection** (LSS's `get_geometry`):

```python
points  = cat((frustum[...,:2] * frustum[...,2:3], frustum[...,2:3]))  # (u*z, v*z, z)
cam_pts = (Kinv @ points.T).T          # intrinsics^-1  -> 3D camera frame
ego_pts = (rot @ cam_pts.T).T + trans  # extrinsics     -> ego meters
```

The `u*z, v*z, z` trick inverts the pinhole projection (which divides by z): pre-multiply `(u,v)`
by `z`, then `K^-1` recovers the true 3D point. This is where the depth distribution *becomes* a
3D spatial volume.

**3. The splat + height collapse** (LSS's `voxel_pooling`):

```python
ix = ((pts[:,0] - xlo) / xstep).long()   # which BEV x-cell
iy = ((pts[:,1] - ylo) / ystep).long()   # which BEV y-cell   (no z index -> height collapses)
bev.view(nx*ny, C).index_add_(0, flat, f)   # SUM every point's feature into its cell
```

Every frustum point is scattered into its `(x, y)` cell and **added**. Because points are indexed
by `(x, y)` only, all points in a vertical column land in the same cell: that is the "summation
along height," in one line. Real LSS makes this fast with a cumsum trick (`QuickCumsum`), but the
semantics are exactly this `index_add_`.

## The full runnable mini-implementation

Single camera, single batch, tiny feature map for readability. Faithful to the real geometry;
runnable as-is (PyTorch only). Marked pedagogical: the real repo handles 6 cameras, batching, the
cumsum-based pooling, and a ResNet/EfficientNet backbone.

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


# ---------------- STEP 1: create_frustum ----------------
# For each depth bin d and each pixel (u,v), make a point (u, v, depth).
def create_frustum():
    ds = torch.arange(*dbound).view(-1, 1, 1).expand(D, H, W)
    us = torch.linspace(0, W - 1, W).view(1, 1, W).expand(D, H, W)
    vs = torch.linspace(0, H - 1, H).view(1, H, 1).expand(D, H, W)
    return torch.stack((us, vs, ds), dim=-1)            # (D, H, W, 3)


# ---------------- STEP 2: get_geometry ----------------
# Unproject the (u,v,depth) frustum into 3D ego coordinates.
def get_geometry(frustum, intrins, rot, trans):
    points = torch.cat((frustum[..., :2] * frustum[..., 2:3], frustum[..., 2:3]), dim=-1)  # (u*z, v*z, z)
    Kinv = torch.inverse(intrins)
    cam_pts = (Kinv @ points.reshape(-1, 3).T).T.reshape(D, H, W, 3)   # camera frame
    ego_pts = (rot @ cam_pts.reshape(-1, 3).T).T.reshape(D, H, W, 3) + trans
    return ego_pts                                      # (D, H, W, 3) ego meters


# ---------------- STEP 1-2: the OUTER PRODUCT lift ----------------
def lift_features(depth_logits, context):
    depth = depth_logits.softmax(dim=0)                 # (D,H,W) the distribution
    feat = depth.unsqueeze(1) * context.unsqueeze(0)    # (D,C,H,W) depth x context
    return feat.permute(0, 2, 3, 1)                     # (D, H, W, C)


# ---------------- STEP 3: voxel_pooling (the SPLAT) ----------------
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

### Verified output

```
STEP 1  frustum (D,H,W,3):         (41, 8, 22, 3)      D=41 depth bins x 8x22 pixels, each (u,v,depth)
STEP 2  ego points (D,H,W,3):      (41, 8, 22, 3)      same grid, now unprojected to 3D ego meters
STEP 1-2 lifted feat (D,H,W,C):    (41, 8, 22, 64)     the outer product: depth x context
STEP 3  BEV feature map (C,X,Y):   (64, 200, 200)      splatted + summed into a 200x200 BEV grid
```

The `(64, 200, 200)` BEV map is exactly what a task head (segmentation, or a MapTR-style decoder)
runs on. From here the [raster-to-vector page](raster-to-vector.html) picks up.

## Mapping the mini-version back to production LSS

| Mini-version | Production LSS | Difference |
|---|---|---|
| single camera | 6 cameras | loop / batch over the `N` camera dimension |
| `index_add_` sum-pool | `QuickCumsum` | same result, cumsum trick for speed on GPU |
| random `depth_logits`, `context` | ResNet/EfficientNet `CamEncode` | a real backbone predicts `D + C` channels per pixel |
| no supervision | task loss (seg/det) | BEVDepth additionally supervises the depth with LiDAR |

## Self-check

1. Which single line is the "soft depth" lift, and what does the broadcast shape do?
2. Why the `u*z, v*z, z` ordering before applying `K^-1`?
3. In the splat, what makes it a "summation along height" specifically?
4. What does `index_add_` correspond to in the real repo, and why is the real version faster?
5. What two things change to go from this mini-version to BEVDepth?
