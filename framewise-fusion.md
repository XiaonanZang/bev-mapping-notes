---
title: "Frame-wise extraction and fusion: the same model, offline and online"
---

# Frame-wise extraction, then fusion — where the aggregation sits

**Papers:** *GlobalMapNet: An Online Framework for Vectorized Global HD Map Construction* ·
[arXiv 2409.10063](https://arxiv.org/abs/2409.10063) *(Sept 2024)* ·
*Online Temporal Fusion for Vectorized Map Construction in Mapless Autonomous Driving* ·
[arXiv 2409.00593](https://arxiv.org/abs/2409.00593) · **IEEE RA-L 10(4):3948–3955, April 2025**
**Dataset:** *Ithaca365* · [arXiv 2208.01166](https://arxiv.org/abs/2208.01166) · CVPR 2022

> **The one idea:** run the network on one frame at a time, then fuse the *finished detections* in
> world coordinates. **Offline and online stop being different systems and become different window
> sizes** — all frames, or the last few. **And the accuracy question moves off perception and onto
> the pose and the data association.**

---

## Four stages

| # | Stage | What it produces |
|---|---|---|
| 1 | **Extract** — the network runs on one scan or a short window | per-frame detections, ego frame |
| 2 | **Transform** — lift detections into a shared frame using the pose | detections in world coordinates |
| 3 | **Associate** — decide which detection in frame *k* is the same object as in frame *k+n* | correspondences |
| 4 | **Fuse** — merge the associated evidence into one geometry, with confidence | the map |

**The network never sees more than one frame's sensor data.** Aggregation happens afterwards, in
geometry and statistics.

## The distinction that decides whether you understand this page

There is a large body of work that *looks* like this and is not. Getting this wrong is the fastest
way to misread the field.

| | What is fused | Where | Examples |
|---|---|---|---|
| **Temporal feature fusion** | **BEV features or decoder queries**, inside the network, before any output exists | end-to-end, trained, opaque | [StreamMapNet](maptr-refinements.html), query-propagation methods |
| **Output-level fusion** | **finished, typed, vectorized detections**, outside the network | explicit, inspectable | **this page**, [RTMap](rtmap.html), [CSMapping](csmapping.html) |

**Both are "using multiple frames." They fail differently.** Feature fusion is trained jointly and
you cannot ask it why. **Output fusion leaves you an auditable artifact: how many independent frames
saw this lane boundary, and did they agree?** For anything that has to be signed off before a
vehicle relies on it, that question is not cosmetic.

> **And note the near-miss.** [RTMap](rtmap.html) and [CSMapping](csmapping.html) fuse **across
> traversals** — different days, different vehicles. **This page fuses across frames within one
> traversal.** Same machinery, easier regime: the pose is better and the scene is genuinely
> identical. **The independence assumption is correspondingly weaker** — a hundred frames of the
> same lane line from one pass is not a hundred independent measurements.

## GlobalMapNet — keep the global map, and feed it back

The architecture is two parts:

- **GlobalMapBuilder** produces local vectorized predictions and merges them into a **global map
  held in permanent storage**, updated continuously as the vehicle drives.
- **GlobalMapFusion** feeds that accumulated historical map **back** in as a prior, so the next
  local prediction is consistent with what has already been mapped.

**That closed loop is the whole point.** The map is not a write-only log of predictions — it becomes
a prior on the next prediction, which is [P-MapNet](p-mapnet.html) and
[PriorDrive](priordrive.html)'s idea with the prior built from your own history instead of from
OpenStreetMap.

## The RA-L system — the same shape, built for the planner

The April 2025 RA-L system is the cleanest statement of all four stages:

> *"the system efficiently fuses all historical road marking detections from an off-the-shelf
> network into a semantic voxel map, which is implemented using a hashing-based strategy to exploit
> the sparsity of road elements. Then reliable voxels are found by examining the fused information
> and incrementally clustered into an instance-level representation of road markings. Finally, the
> system incorporates domain knowledge to estimate the geometric and topological structures of
> roads, which can be directly consumed by the planning and control (PnC) module."*

**Read the phrase "off-the-shelf network."** The perception model is a replaceable component. **The
contribution is stages 2–4**, which is the argument of this page stated by someone building a
product.

**And it reports the result that matters:**

> *"the output of our system is more consistent and accurate than the network output by a large
> margin"*

**A hashing-based sparse voxel map is a good engineering detail to keep.** Road markings occupy a
vanishing fraction of a city's volume; a dense grid is almost all zeros.

## The experiment you can actually run

This is unusually cheap, and it is why this page names GlobalMapNet first.

**Its repository publishes checkpoints for both `globalmapnet_*` *and* `streammapnet_*`, on both
nuScenes and Argoverse 2.** StreamMapNet is the per-frame/temporal-feature baseline; GlobalMapNet
adds the global fusion on top.

```
bash tools/dist_test.sh <config> <checkpoint> 1 --eval
```

**The trailing `1` is the GPU count.** So a controlled comparison — *does aggregating after the
network beat the network's own output?* — is an **evaluation-only run on a single GPU, from one
codebase, with no training.** Most claims in this area cost a week of compute to check. This one
does not.

> **Note which evaluation split.** The configs use the **geographically disjoint resplit** rather
> than the original nuScenes split, following StreamMapNet's finding that the standard splits
> overlap geographically between train and val. **Numbers on the two splits are not comparable**,
> and this is the single benchmark detail most worth getting right in this subfield.

## The dataset built for the assumption

The premise of stage 2 is *assume a good localization prior.* **Ithaca365** is built to supply
exactly that: a **15 km route driven repeatedly** across weather, lighting and traffic, with
**high-precision GPS/INS specifically to establish correspondence across routes.**

It lets you vary the thing everything here rests on — **run the same route twice and ask how much of
the fusion's gain survives degraded pose.**

## What nobody seems to report

**Almost no paper in this area ablates pose error**, which is odd, because stage 2 is where the
pose enters and stages 3–4 inherit whatever it got wrong.

The closest thing to an acknowledgement is indirect: [PriorDrive](priordrive.html) injects
**1 m of Gaussian noise** into its priors specifically to model localisation uncertainty — an
admission that clean poses are unrealistic, made in a paper that is not about pose at all.

> **The open question, stated plainly: in a fused map, how much of the reported gain comes from the
> model and how much from a pose prior that would not exist outside a well-surveyed benchmark?**
> **I have not found a paper that answers this**, and I have not yet read the RA-L paper's body,
> where the answer may well be. ⚠ **Treat the question as open, not as a finding.**

## Limitations

- **Association is the least-documented stage** and the most likely source of silent error. Two
  lane lines 30 cm apart, or one lane line seen at two different pose estimates, look similar.
- **Correlated errors do not average out.** A systematic bias in the per-frame detector survives
  fusion intact, however many frames you stack.
- **Change is invisible within a single traversal.** Fusing across traversals reintroduces it, and
  with it the change-detection problem [RTMap](rtmap.html) handles.

---

## Verify before citing

- [ ] **Does the RA-L paper ablate pose error?** Its body is unread here ⚠ — this is the open
      question above
- [ ] GlobalMapNet's reported numbers, and on **which split**
- [ ] How association is actually implemented in each system — Hungarian, Chamfer threshold, voxel
      voting, learned
- [ ] Whether Ithaca365's GPS/INS accuracy is sufficient to serve as ground-truth pose or only as a
      strong prior

*Written 2026-09-12. Abstracts and repository documentation were read at source; paper bodies marked
⚠ were not. Quotes are from the papers; the framing on this page is mine.*
