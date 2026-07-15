---
title: Topology & Lane-Graph Extraction
---

# Topology & Lane-Graph Extraction

[← back to overview](index.html)

The single hardest part of online HD mapping, and the one a planner actually depends on. A semantic
BEV map tells you *which pixels are lane markings*. A planner needs something stronger: an ordered,
connected **lane graph**, where each lane is a directed path and the graph encodes which lane leads
to which through merges, splits, and intersections. This page is about closing that gap.

## The core problem: pixels do not equal topology

A semantic segmentation output is a raster: per-pixel class labels (lane divider, boundary,
crossing). It says nothing about:

- **Instance**: which pixels belong to *the same* lane, versus a neighboring one.
- **Order**: in what direction the lane runs (the point sequence, start to end).
- **Connectivity**: which lanes flow into which at a junction, i.e. the graph edges.

All three must be recovered before the map is navigable. There are two families of approach: the
classical post-processing pipeline, and the learned end-to-end decoder.

## Approach 1: classical centerline extraction (post-processing)

The pre-MapTR route (used by HDMapNet). Given a binary/semantic mask, recover vectors with
hand-crafted image processing:

1. **Skeletonization (morphological thinning)**: iteratively erode a thick pixel region down to a
   1-pixel-wide medial axis, the centerline. Algorithms: Zhang-Suen thinning, medial-axis
   transform. Converts a blob of "lane divider" pixels into a thin curve.
2. **Graph construction from the skeleton**: treat skeleton pixels as nodes, adjacency as edges.
   Find endpoints (degree-1 pixels) and junctions (degree ≥ 3 pixels) to segment the skeleton into
   individual polylines.
3. **Vectorization**: fit/simplify each polyline (e.g. Douglas-Peucker) into an ordered sequence of
   (x, y) points.

**Why it is brittle**: skeletonization is fragile exactly where topology is hardest. At merges and
intersections the medial axis produces spurious branches, gaps, and short spurs. A noisy mask (the
common case at range or under occlusion) yields a broken skeleton, and no amount of heuristics
reliably reconstructs the true connectivity. The pixel metric (IoU) hides this: a mask can score
well per-pixel while its recovered graph is topologically wrong.

## Approach 2: learned end-to-end vectorization (MapTR)

MapTR removes the post-processing stack entirely. Instead of predicting pixels and then recovering
structure, it predicts the structure directly, as a *set of typed, ordered polylines*, via a
DETR-style transformer decoder. See the [raster-to-vector page](raster-to-vector.html) for the full
decoder. The key points for topology:

- Each **instance query** emits one complete map element (one lane divider, one boundary), so
  *instance separation is solved by construction*, not recovered afterward.
- Each instance expands into **K point queries** that regress an *ordered* point sequence, so
  *ordering is native*, not inferred from a skeleton.
- **Permutation-equivalent matching** handles the forward/backward ambiguity of a polyline during
  training so the ordered output stays stable.

What MapTR outputs is a set of ordered polylines in BEV coordinates. That is most of a lane graph,
but not the connectivity edges yet: MapTR gives clean *geometry per element*, and connectivity is
either derived geometrically (below) or predicted by a topology-aware successor.

## From polylines to a navigable graph

Once you have ordered, typed polylines, the lane graph is built in steps:

1. **Centerlines from dividers**: a drivable lane is the region between two adjacent lane dividers
   (or a divider and a road boundary). Interpolate the centerline as the midline between adjacent
   divider polylines. Some methods predict centerlines directly instead.
2. **Nodes and edges**: represent each lane centerline segment as a graph node; an edge connects
   node A to node B if the end of A is spatially continuous with the start of B (endpoint proximity
   plus heading agreement).
3. **Intersection modeling**: at a junction, multiple incoming lanes connect to multiple outgoing
   lanes. The connections are not purely geometric (you cannot always infer a left turn from
   geometry alone), which is why modern work predicts the connectivity explicitly.

### Topology-aware successors (worth naming)

- **TopoNet / lane-topology methods**: predict lane centerlines *and* a lane-lane adjacency matrix
  (which lane connects to which) as an explicit graph-prediction head, rather than deriving edges by
  geometry. This directly targets the connectivity the planner needs at intersections.
- **LaneGAP, MapTRv2 centerline variants**: fold centerline/graph representations into the MapTR
  paradigm so the output is closer to a ready lane graph.

## HD map formats: the output target

The lane graph is stored in a standard vectorized HD-map format, both of which encode geometry
*and* topology:

- **Lanelet2**: the map is a set of *lanelets*, each a small drivable atom bounded by a left and
  right linestring, plus a routing graph of lanelet-to-lanelet connections and attached traffic
  rules (speed, right-of-way). Open-source, widely used in research and Autoware.
- **OpenDRIVE (.xodr)**: an industry standard describing roads, lanes, and junctions with an
  explicit successor/predecessor lane linkage model. Common in simulation (CARLA) and OEM
  toolchains.

Both are graph-plus-geometry: the point-sequence gives the shape, the successor/predecessor links
give the topology. That is exactly what a MapTR-style vectorized output feeds into.

## Handling occlusion

Real topology is often partially observed: a lane is occluded by a vehicle, or runs past the
sensor's range. Two levers:

- **Temporal accumulation** (see [StreamMapNet](maptr-refinements.html)): fuse BEV features across
  frames so a briefly-visible segment is filled in from earlier observations.
- **Learned completion / priors**: the decoder can hallucinate the most likely continuation of a
  partially observed lane, or a structural prior (an SD map, see
  [auto-labeling](auto-labeling.html)) can supply the coarse topology to be refined.

## Interview framing

> "A semantic BEV map gives you pixels; a planner needs an ordered, connected lane graph. Classically
> you recover that with skeletonization and graph heuristics, but that pipeline breaks exactly at
> merges and intersections, and it is not trainable. MapTR's contribution is to predict the
> structure end-to-end: each instance query is one map element, each point query traces its ordered
> geometry, so instance separation and ordering come for free. Connectivity, the actual lane-to-lane
> edges at a junction, is the remaining piece; geometry gets you part way, and topology-aware methods
> predict the adjacency explicitly. The final graph lands in Lanelet2 or OpenDRIVE, which encode both
> the geometry and the successor/predecessor links."

## Self-check

1. What three things does a lane graph need that a semantic mask does not provide?
2. Why does classical skeletonization break at intersections?
3. How does MapTR's query design solve instance separation and ordering by construction?
4. Where does connectivity (lane-to-lane edges) still have to come from?
5. What do Lanelet2 and OpenDRIVE encode beyond geometry?
6. Two ways to handle occluded or out-of-range lane segments?
