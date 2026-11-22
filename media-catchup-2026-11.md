---
title: "Two years of HD map talks and podcasts (Nov 2024 to Nov 2026)"
---

# Two years of HD map talks and podcasts

*November 2024 to November 2026. Video and audio only.*

This is a catch-up: what was said out loud, in talks, keynotes, demos and podcast episodes, about
HD maps for autonomous driving over the last two years. It is organized as one argument rather than
as a list. First, the question everyone is arguing about. Then what each answer implies, and who is
working on it.

> **Verification status, up front.** Everything below was found by search and checked against the
> page, the channel feed or the podcast directory record: the title, date, length and link are real.
> **The contents have not been watched or listened to.** The one-line notes come from titles,
> descriptions and chapter lists. Treat this as a watchlist, not as citations. Dates are upload
> dates; a conference recording often goes up months after the talk.

---

## If you only have an hour

1. [Vanhoucke (Waymo), CVPR 2026 WAD keynote](https://www.youtube.com/watch?v=T8Z1Og8S_Xs), the
   chapter *"Maps are an extremely powerful sensor"* (08:47 to 12:27). The strongest case for the map.
2. [Kendall (Wayve) on Sequoia, "Autonomous Driving 2.0"](https://www.youtube.com/watch?v=8x_O8BeGNTw).
   The strongest case against it.
3. [NVIDIA DRIVE Labs Ep. 38, "Mapless Driving"](https://www.youtube.com/watch?v=u_eZoOS5bFc). Three
   minutes on why HD maps are costly to build and keep.
4. [Gargoum, "AI-Powered Feature Extraction from Mobile LiDAR at Network Scale"](https://www.youtube.com/watch?v=uuRcWoMjC3Q)
   (INTERGEO TV). How lane markings and curbs actually get pulled out of survey LiDAR, at network scale.

---

## 1. The question: does an autonomous vehicle need an HD map?

This was the loudest topic of the two years, and the camps are clear.

**For the map.** Waymo's answer, after 200 million driverless miles, is that the map is a sensor: it
sees past occlusions and around corners, and it doesn't fail in the rain.

- [CVPR'26 WAD keynote, Vincent Vanhoucke, Waymo](https://www.youtube.com/watch?v=T8Z1Og8S_Xs) (2026-09, 24 min). The chapter "Maps are an extremely powerful sensor" runs from 08:47 to 12:27.

**Against the map.** The end-to-end camp argues that the HD map is what kept AV 1.0 stuck city by
city, and that a learned driver should generalize without one.

- [How End-to-End Learning Created Autonomous Driving 2.0, Alex Kendall (Wayve), Sequoia](https://www.youtube.com/watch?v=8x_O8BeGNTw) (2025-11, 42 min). AV 1.0 (HD maps, retrofits, city by city) against end-to-end learning.
- [AV 2.0: How Wayve is Building Scalable, Mapless and End-to-End AI, Driverless Digest](https://www.youtube.com/watch?v=xXzfa6EH4hY) (2025-07, 31 min; also as [audio](https://podcasts.apple.com/us/podcast/av-2-0-how-wayve-is-building-scalable-mapless-and-end/id1811181944?i=1000718191799)).
- [Wayve: Scaling Autonomous Vehicles Without Borders, The Road to Autonomy](https://podcasts.apple.com/us/podcast/wayve-scaling-autonomous-vehicles-without-borders/id1502184916?i=1000713215849) (2025-06, 39 min, audio). Driving in several countries without pre-mapped areas.
- [Mapless Driving: AI Powered Autonomy at Urban Scale, NVIDIA DRIVE Labs Ep. 38](https://www.youtube.com/watch?v=u_eZoOS5bFc) (2025-05, 3 min). Road geometry predicted online in place of a stored map.
- [NVIDIA Hyperion L2++ demo drive, Berlin](https://www.youtube.com/watch?v=V-6OvNwEixM) (2026-10, 25 min). Cameras and radar, no LiDAR, no HD map.
- [Tensor: 10+ Years of End-to-End Self-Driving Without HD Maps](https://www.youtube.com/watch?v=6WHxfm8_Bzc) (2026-05, 5 min).
- [Nuro: HD-Map-Free Autonomy, Mountain View](https://www.youtube.com/watch?v=fX-jiSyevtI) (2026-04, 25 s demo). Posted the same day as Nuro's essay [The Map Debate](https://www.nuro.ai/blog/the-map-debate-do-autonomous-vehicles-need-hd-maps).
- [Imagry's Mapless AI, Eran Ofir, Full Tech Ahead](https://podcasts.apple.com/us/podcast/generative-autonomy-unleashed-imagrys-mapless-ai-revolution/id1831548170?i=1000741840833) (2025-12, 29 min, audio).

**In between.** Two middle positions: build the map from the fleet instead of a survey, or build it
on the vehicle as it drives.

- [How Mobileye REM Expands the ODD for Hands-Free Driving](https://www.youtube.com/watch?v=3wTSorni5zs) (2026-02, 1 min). Crowdsourced road data from millions of production cars.
- [Bosch: Onboard Mapping Generation System, Junko's Talk To Us](https://www.youtube.com/watch?v=YCND7gFv-xI) (2025-11, 40 min). A transformer that generates the map live onboard.
- [Tesla Q3 2026 results call](https://www.youtube.com/watch?v=tFezqXBF-gw) (2026-10, 60 min). [Electrek reports](https://electrek.co/2026/10/22/tesla-building-own-maps-fsd-after-mocking-maps/) Tesla said on this call that it is building its own fleet-fed maps. Second-hand; the call was not watched.

**The map makers' reply.** The map makers' answer is that "mapless" still drives on a map, just a
different one, and they are showing AI drivers running on their maps.

- [TomTom x DeepRoute.ai: AI-powered automated driving in Paris](https://www.youtube.com/watch?v=OChk-kREGTI) (2026-11, 9 min) and [a closer look at the collaboration](https://www.youtube.com/watch?v=CVQGXRELCi4) (5 min). An AI driver on TomTom's Orbis road model map.
- [Why maps matter for autonomous driving, HERE](https://www.youtube.com/watch?v=Q3oFF8ALsk0) (2026-10, 1.5 min).

The rest of this page follows the two ways a map can come into existence: built online by the car,
or built offline from a survey.

---

## 2. If the car builds the map: online HD map research

The research answer to "mapless" is online map construction: predict the vectorized map from the
car's own sensors, optionally helped by a cheap prior. The recorded material here is short paper videos.

- [ArgoTweak: Towards Self-Updating HD Maps through Structured Priors (ICCV 2025)](https://www.youtube.com/watch?v=fW9jHSiSAkU), KTH RPL (5 min). Treats an old map as a prior and corrects it.
- [SDTagNet: Text-Annotated Navigation Maps for Online HD Map Construction (NeurIPS 2025)](https://www.youtube.com/watch?v=OlWSKluE-mc), KIT-MRT (4 min). OpenStreetMap-style navigation maps as the prior.
- [PseudoMapTrainer: Learning Online Mapping without HD Maps (ICCV 2025)](https://www.youtube.com/watch?v=-AgxuYdK_PQ) (4 min). Trains with pseudo-labels instead of HD map labels.
- [MemFusionMap (WACV 2025)](https://www.youtube.com/watch?v=CHxsLP0eZOc) (11 min). Online vectorized map construction with temporal memory.
- [Dense Semantic BEV Map Generation from Sparse LiDAR Point Clouds (IROS 2025)](https://www.youtube.com/watch?v=hv8dzhZIFgk), CityU. LiDAR to a semantic bird's-eye-view map.
- [Seq-BEV: Semantic BEV Map Generation in Full View (T-ITS 2025)](https://www.youtube.com/watch?v=xQCDO6j2jAg), CityU (3 min). The camera version.

**Most of this field's live discussion is not recorded.** The workshops where it happens recur every
year, but none of them posted recordings in the window: KIT's *Bridging the Gap between Map-based and
Map-less Driving* at IEEE IV, ICRA 2026 *Robots Meet Prior Maps*, PPNIV at IROS, TopoCoT at WACV 2026,
and the ISPRS Geospatial Week HD-map workshop. CVPR WAD posts its keynotes and challenge sessions,
not its paper talks. For this side, papers stay the channel.

---

## 3. If a survey builds the map: offline extraction from mobile LiDAR

The other way to get a map is to drive a survey vehicle and extract the features afterwards. This is
the side the vision venues talk about least, and the trade shows talk about most. Most of these talks
come from INTERGEO, the geospatial trade fair, and from surveying podcasts.

**Extracting road features from point clouds.**

- [AI-Powered Feature Extraction from Mobile LiDAR at Network Scale, Suliman Gargoum](https://www.youtube.com/watch?v=uuRcWoMjC3Q), INTERGEO TV (2026-11, 24 min). Every point labelled, then vectorized: lane markings, curbs, guardrails.
- [From Point Clouds to Vectors: AI-Powered Automatic Vectorization, Mario Sessini](https://www.youtube.com/watch?v=q6vSJ1N4f2M), INTERGEO TV (2026-11, 19 min).
- [LIDAR Magazine Podcast #25, Alex Baikovitz (Mach9)](https://lidarmag.com/podcast/) (2025-12, 39 min, audio). AI extraction from point clouds to CAD and GIS. A longer conversation with the same guest: [The Geoholics ep. 260](https://podcasts.apple.com/us/podcast/episode-260-alex-baikovitz-mach9/id1475011444?i=1000725783230) (73 min).
- [The Geoholics ep. 251, Ted Knaak (TopoDOT)](https://podcasts.apple.com/us/podcast/episode-251-topodot-ted-knaak/id1475011444?i=1000706250027) (2025-05, 108 min, audio). Mobile LiDAR feature extraction software. Same speaker on video: [Automated Road Condition Analysis from Mobile LiDAR](https://www.youtube.com/watch?v=YDxmX_7DZyY) (INTERGEO TV, 20 min).
- [Trimble Business Center: AI-Powered Feature Extraction, Positioning Technologies](https://podcasts.apple.com/us/podcast/trimble-business-center-tbc-ai-powered-feature-extraction/id1779487421?i=1000678608319) (2024-11, 13 min, audio). Pre-trained and custom classifiers for point clouds.
- [Smart Point Clouds: What If Every Point Knew What It Was, Florent Poux, MapScaping](https://mapscaping.podbean.com/e/smart-point-clouds-what-if-every-point-knew-what-it-was/) (2026-10, 61 min, audio). Semantic point clouds. Also [LIDAR Magazine #24](https://lidarmag.com/podcast/) (44 min).

**The survey systems.**

- [From Survey Drive to Decision: Mobile Mapping for Infrastructure, Timon Kabelac](https://www.youtube.com/watch?v=GR9iOMiETjs), INTERGEO TV (22 min).
- [Mobile Mapping with a Tesla: Building and Validating a Low-Cost System, Rami Tamimi](https://www.youtube.com/watch?v=V8yDIT_wqFc), INTERGEO TV (18 min).
- [Mobile mapping beyond the car: 360 cameras plus LiDAR (Mosaic), 2D Is Against My Religion](https://podcasts.apple.com/us/podcast/mobile-mapping-beyond-the-car-360-cameras-plus-lidar-mosaic/id1878399775?i=1000800255921) (2026-11, 26 min, audio), and [Mapping With 360 Cameras, MapScaping](https://mapscaping.podbean.com/e/mapping-with-360-cameras/) (42 min).

**Labelling at production scale.**

- [Accelerating HD Mapping for Autonomous Vehicles: an HD map annotation case study, iMerit](https://www.youtube.com/watch?v=Mk92FfEX2Ck) (2026-08, 1.5 min).

---

## 4. Maps from the air, and maps from the crowd

Two cheaper sources than a survey car: imagery from above, and data from vehicles already on the road.

**From the air.**

- [ASCE webinar: AI vs manual digitizing of land use and transportation asset data, Ecopia AI](https://www.youtube.com/watch?v=-S2PPiZkicc) (2026-05, 61 min). Vector road assets extracted from aerial imagery at scale (vendor-sponsored).
- [Mapping What Matters: Pedestrian Asset Data, Ecopia AI](https://www.youtube.com/watch?v=eZMmyJWMoJM) (2025-11, 60 min).
- [Turning Aerial Imagery Into a Searchable World (Vexcel), MapScaping](https://mapscaping.podbean.com/e/turning-aerial-imagery-into-a-searchable-world/) (2026-07, 56 min, audio). 7.5 cm imagery, detection, embeddings.
- [OpenSatMap: a satellite dataset for large-scale map construction (NeurIPS 2024)](https://slideslive.com/39027508) (5 min). Lane lines and road boundaries from 15 to 30 cm satellite imagery.

**From the crowd and the fleet.**

- [Hivemapper, MapScaping](https://mapscaping.podbean.com/e/hivermapper/) (2024-12, 51 min, audio). Street-level mapping from dashcams with token incentives.
- [Mapillary, MapScaping](https://mapscaping.podbean.com/e/mapillary/) (2025-10, 48 min, audio). Street-level imagery at Meta, linked to OpenStreetMap.
- [Telematics Data is Reshaping Our Understanding of Road Networks, MapScaping](https://mapscaping.podbean.com/e/telematics-data-is-reshaping-our-understanding-of-road-networks/) (2025-01, 59 min, audio).

**The map companies on where map making is going.**

- [Full IAA keynote: how maps will change the future of driving, Manuela Locarno Ajayi (TomTom)](https://www.youtube.com/watch?v=QpMmV_LWLAI) (2025-10, 13 min). Same speaker on [real-time lane-level mapping](https://podcasts.apple.com/us/podcast/june-22nd-2026-unifor-ford-contract-talks-begin-tomtoms/id1268090919?i=1000773772932) (Automotive News Daily Drive, 2026-06, audio segment) and [AI in navigation](https://podcasts.apple.com/us/podcast/tomtoms-manuela-locarno-ajayi-ai-in-navigation-is-the/id1137686333?i=1000773594170) (23 min, audio).
- [From Static Maps to Living Systems: How AI Is Changing Global Mapping, Cliff Allison (TomTom) with Matt Forrest](https://www.youtube.com/watch?v=J_9zP3ptdg0) (2026-01, 56 min).
- [Mapping the market: where navigation is heading, Mike Harrell (TomTom), S&P Autology](https://podcasts.apple.com/us/podcast/mapping-the-market-what-direction-are-navigation/id1565815988?i=1000714354420) (2025-06, 29 min, audio).
- [Seeing the Unseen: how intelligent maps power the future of driving (HERE), Highways Voices](https://podcasts.apple.com/us/podcast/seeing-the-unseen-how-intelligent-maps-are-powering/id1551975855?i=1000714377936) (2025-06, 29 min, audio).
- [How Mapbox is Quietly Powering the AI Map Revolution, Kieran McCann, Spatial Stack](https://podcasts.apple.com/us/podcast/how-mapbox-is-quietly-powering-the-ai-map-revolution/id1748871891?i=1000755777661) (2025-10, 41 min, audio).

---

## 5. Open maps: Overture

A separate thread runs alongside all of this: an open, industry-backed base map with stable IDs,
built to be read by machines.

- [GERS, Metadata, and Maps Built for Machines, Amy Rose, MapScaping](https://mapscaping.podbean.com/e/gers-metadata-and-maps-built-for-machines/) (2026-09, 48 min, audio). Persistent feature IDs that merge many sources.
- [Jennings Anderson and Amy Rose on Overture Maps, Software Engineering Radio](https://podcasts.apple.com/us/podcast/jennings-anderson-and-amy-rose-on-overture-maps/id120906714?i=1000736505781) (2025-11, 64 min, audio).
- [Inside Overture Maps, Geographical Thinking (Esri Canada)](https://podcasts.apple.com/us/podcast/inside-overture-maps/id1540357358?i=1000714653509) (2025-06, 40 min, audio).
- [Grounding AI in the Real World, Albi Wiedersberg, Spatial Stack](https://podcasts.apple.com/us/podcast/grounding-ai-in-the-real-world-overture-maps/id1748871891?i=1000802508754) (2026-11, 39 min, audio).

---

## Not sorted yet

Recordings that may touch maps but where the description doesn't say so. Worth a skim, not a sitting.

- CVPR WAD challenge sessions, Argoverse [2026](https://www.youtube.com/watch?v=Hbn-uormOg0) and [2025](https://www.youtube.com/watch?v=FE2Go5AmWn4).
- Keynotes: [Raquel Urtasun (Waabi), CVPR 2026](https://www.youtube.com/watch?v=pfJ2vRsgEJA); Ashok Elluswamy (Tesla) at [CVPR 2026](https://www.youtube.com/watch?v=2V0ddkF1Svo), [ICCV 2025](https://www.youtube.com/watch?v=IRu-cPkpiFk) and [Scaled ML 2026](https://www.youtube.com/watch?v=LFh9GAzHg1c); [Hongyang Li, ICRA 2026 workshop](https://www.youtube.com/watch?v=qaAeP8oVa80); [Waymo's EMMA](https://www.youtube.com/watch?v=iS9YFW28XyM).
- [SeMaNa workshop, IROS 2026 (semantic mapping and navigation, 4 h stream)](https://www.youtube.com/watch?v=MWsrNtEAUxg).
- [DUFOMap: dynamic-aware point-cloud mapping](https://www.youtube.com/watch?v=isDnAVoVD5M), KTH RPL.

---

## How this was put together

- **Window:** 2024-11-21 to 2026-11-21, by upload date.
- **Sources:** conference workshop channels (CVPR WAD, ICRA and IROS workshops), university lab
  channels, the channels of about twenty AV companies and map makers, the INTERGEO TV channel, and
  podcast episodes found by keyword and guest-name search in the Apple Podcasts directory, plus two
  shows scanned in full (MapScaping, LIDAR Magazine Podcast).
- **What came up empty:** Aurora, Applied Intuition, Baidu Apollo, Momenta, Woven by Toyota, Zoox,
  Motional and GM posted no map-related video in the window. Waabi's CVPR 2026 tutorial session on
  mapping and localization has no recording online yet.
- **Limits:** YouTube channel feeds list only the newest 15 uploads, so older items on busy channels
  can be missed. Podcast search is Apple only, and a search for "HD map" returned almost nothing, so
  HD-map-specific episodes are probably under-counted. Chinese-language platforms were not searched.

*Companion to the [BEV Mapping Notes](index.html); the older, pre-2024 talk list is on [Talks](talks.html).*
