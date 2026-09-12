---
title: "Talks: mapping for autonomous driving, on video"
---

# Talks — the same material, for the commute

A short, curated list of recorded talks covering the ground in these notes. **Video is the format
that works when reading does not** — a queue, a bus, a walk.

> **⚠ VERIFICATION STATUS, STATED UP FRONT.** These were found by search. **YouTube pages do not
> render for automated fetching, so titles and channels are as reported by search and the CONTENTS
> ARE UNVERIFIED.** Treat this as a watchlist, not as citations. **Nothing on this page should be
> quoted in an argument until it has actually been watched.**

---

## For Part III — making the map offline

**[[10] Mapping — Waabi CVPR 2024 Tutorial on Self-Driving Cars](https://www.youtube.com/watch?v=KcQT3c4nlaw)**
**Start here.** A tutorial module devoted to mapping, from the group whose lineage produced much of
the offline LiDAR mapping literature. A tutorial rather than a paper talk, so it should give the
shape of the whole problem rather than one method.

**[Map-based Localization for Autonomous Driving — Cyrill Stachniss (ECCV 2020 Workshops)](https://www.youtube.com/watch?v=NB63C8L8CRM)**
**This is stage 2 of the offline pipeline** — the pose that everything downstream inherits. If the
accumulation in [offline map generation](offline-map-generation.html) is blurred, this is the talk
about why.

**[Photogrammetry I & II — Cyrill Stachniss, Uni Bonn 2021 (playlist)](https://www.youtube.com/playlist?list=PLgnQpQtFTOGRYjqjdZxTEQPZuFHQa7O7Y)**
and **[Introduction to SLAM from a Photogrammetric Perspective](https://www.youtube.com/watch?v=NW8hUcbjuIQ)**
**The other community, lecturing.** Road-marking extraction from mobile laser scanning has a long
life in photogrammetry and remote sensing that the vision literature cites lightly. **This is where
the vocabulary of that side gets taught** — registration, point clouds, coordinate frames. A course,
not a talk, so treat it as a reference rather than a sitting.

**[[CVPR'23 WAD] Keynote — Chen Wu, Waymo](https://www.youtube.com/watch?v=fXsbI2VkHgc)**
The operator's view from the company that maps a city before it drives in it.

## For Part III — frame-wise extraction and fusion

**[Offboard Perception for Autonomous Driving — Charles Qi, Waymo (3D-DLAD @ IV 2021)](https://www.youtube.com/watch?v=hyTMyZgWcFY)**
**The single most on-point talk found.** Qi is the first author of *Offboard 3D Object Detection
from Point Cloud Sequences* (CVPR 2021), which is the clearest statement of the idea underneath
[framewise fusion](framewise-fusion.html): **offline may use the entire sequence and unlimited
compute, and that is a different problem, not merely an easier one.** It is about objects rather
than map elements — **the transfer to mapping is the interesting part, and it is not made for you.**

**[[CVPR'23 WAD] Keynote — Ashok Elluswamy, Tesla](https://www.youtube.com/watch?v=6x-Xb_uT7ts)**
**The counter-position, and worth watching precisely because it argues against these notes.** The
camera-only, map-light end of the field. If the offline-mapping argument cannot survive this talk,
it is not a strong argument.

**[[CVPR'22 WAD] Keynote — Yuning Chai, Cruise](https://www.youtube.com/watch?v=EJ0hgOc0XLA)**
A third operator's framing, from a different mapped-and-sensor-rich deployment.

## Cross-cutting — benchmarks and what "correct" means

**[[CVPR'23 WAD] Keynote — Andreas Geiger, Universität Tübingen](https://www.youtube.com/watch?v=x_42Fji1Z2M)**
Geiger is behind KITTI. **Benchmark design is its own subject** — what a metric rewards decides what
the field builds, which is the thread running under [nuScenes & map metrics](nuscenes-metrics.html)
and under the geographic-split problem noted in [frame-wise fusion](framewise-fusion.html).

**[[CVPR'25] Keynote — Xianming Liu, XPeng](https://www.youtube.com/watch?v=nqYA7KvceDw)**
Scaling autonomous driving with large foundation models — the direction that would, if it lands,
change what any of this costs.

## The standing source

**[WAD at CVPR — channel](https://www.youtube.com/c/wadatcvpr)**
The Workshop on Autonomous Driving posts its keynotes every year. **One channel, most of the
industry's public technical talks, and it refreshes annually** — the cheapest subscription in this
field.

---

## How to use this page

1. **One talk per commute. Do not batch.**
2. **Watch the counter-position early** — the Tesla keynote — so the argument in these notes gets
   tested rather than reinforced.
3. **If a talk contains a number worth keeping, find the paper and check it there.** A slide is not
   a citation.

*Compiled 2026-09-12. Links found by search; **contents not verified** — see the note at the top.*
