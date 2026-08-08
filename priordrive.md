---
title: "PriorDrive: unifying every kind of prior"
---

# PriorDrive — one encoder for every prior

**Paper:** *PriorDrive: Enhancing Online HD Mapping with Unified Vector Priors* ·
[arXiv 2409.05352](https://arxiv.org/abs/2409.05352) · Sep 2024, revised to **v4** Nov 2025

> **The one idea:** priors arrive in incompatible formats — an OSM road graph, a stale vendor HD map,
> a map this vehicle built last week. **Encode them all as vectors in one representation, and the
> model stops caring where the prior came from.**

[P-MapNet](p-mapnet.html) uses one prior type at a time. PriorDrive asks the next question: **what if
you have several, of different ages and qualities?**

---

## Three priors, one encoder

| Prior | Source | What it contributes |
|---|---|---|
| **SD map** | OpenStreetMap | *"crucial long-distance centerline skeletons"* |
| **Outdated HD map** | vendors, following MapEX | precise geometry that may be **wrong now** |
| **Online historical local map** | earlier passes through the same area | what was seen before, under different occlusion |

**Every prior gets Gaussian noise added, σ = 1 metre**, deliberately — to model localisation
uncertainty rather than pretend it away.

> **Do not skip that detail.** A paper that simulates its priors still has to inject a metre of
> positional noise for the setup to be realistic. **Alignment is not a footnote in this field; it is
> a modelled quantity.**

## The two pieces

**Unified Vector Encoder (UVE).** Encodes any prior as vectors with point attributes — position,
direction, type — then applies **intra-vector attention** (within one polyline) and **inter-vector
attention** (across polylines), with masking. Best configuration: M=2, N=2, dimension 64.

**Hybrid Prior Representation (HPQuery).** How the encoded prior meets the detection queries, at both
instance and point level. Three options were tried — addition, replacement, concatenation.
**Concatenation won by +5.2 mAP.**

---

## The numbers

| Setting | PriorDrive | Comparison |
|---|---|---|
| nuScenes, MapTRv2, 24 epochs | **75.8 mAP** | 61.5 baseline (**+14.3**) |
| nuScenes, MapTRv2, 110 epochs | **80.5 mAP** | 76.0 (InteractionMap, prior SOTA) |
| Argoverse 2 | **72.8 mAP** | 71.6 (InteractionMap) |
| OpenLane-V2, topology | **OLS 46.2 / 48.5** | 44.1 (TopoLogic) |

**+14.3 mAP at 24 epochs is a large number.** Read it carefully though: much of it is *"a prior lets
you converge faster."* The gap narrows at 110 epochs (80.5 vs 76.0). **Priors buy accuracy and they
also buy training budget**, and those are different claims.

**Three benchmarks including OpenLane-V2 topology** is worth noting on its own — topology is where
sensor-only online mapping is weakest, and it is what a planner actually consumes.

---

## ⚠ The thing to be precise about, because it is easy to overstate

**PriorDrive does *not* do multi-vehicle fleet aggregation.** I assumed it did before reading it.

> *"certain road sections are captured through multiple drives. Therefore, we use each baseline model
> to infer the entire dataset, thereby generating online local prior maps at different trips"*
> — Implementation Details

nuScenes was **collected by a single vehicle.** The "historical" priors are **synthesised** by running
baseline models over the dataset to imitate repeated passes. **It is a simulation of fleet history,
not fleet history.**

**That does not weaken the paper** — it is a clean way to study prior fusion without a fleet. **But
it means the actual crowdsourcing problem — many vehicles, one frame, conflicting observations —
is somewhere else.** See [RTMap](rtmap.html) and [CSMapping](csmapping.html).

---

## Why this page exists

1. **"Prior" is not one thing.** Age, source and format vary, and a system that handles only one is
   brittle in exactly the way deployment punishes.
2. **The unification is the contribution**, not any single prior. That is an engineering insight more
   than a modelling one, and it generalises.
3. **Simulated history ≠ crowdsourcing.** Knowing where that line falls is the difference between
   describing this field accurately and repeating a press release.

---

## Verify before citing

- [ ] The **+14.3 mAP** figure and its epoch count — the 24-vs-110 distinction matters
- [ ] That the **σ = 1 m** noise applies to all three prior types
- [ ] The simulated-history point above, from Implementation Details — **this is the one I got wrong
      first time**

*Read via arXiv HTML, 2026-08-08. Quotes are from the paper; the framing on this page is mine.*
