# Validation Report: Drone-Based Glaciogenic Cloud Seeding — Kenai Peninsula, Alaska

**Source:** https://www.rainmaker.com/blog/alaska-validation-report
**Author:** Rainmaker Research Team (Rainmaker Technology Corporation, El Segundo, CA)
**Published:** August 24, 2026
**Saved:** 2026-08-25
**Tags:** science, technology, fundamentals, research

> Primary source — technical validation report, not a press release. Uses peer-reviewed methodology (QPE ensemble approach from Friedrich et al. 2020, physical validation framework from SNOWIE 2017 campaign). Rainmaker Technology Corporation is a drone-based cloud seeding startup. Report covers operations August 22–23, 2026.

---

## TL;DR
Rainmaker successfully validated drone-based glaciogenic cloud seeding in the Kenai Peninsula, Alaska. Seven coordinated drone missions released silver iodide (AgI) flares, producing seven identifiable radar seeding signatures on the PAHG NEXRAD. Estimated net new precipitation: 45–65 acre-feet (mean: 57.6 acre-feet) over a 249.53 km² affected area, using a QPE ensemble method validated against the SNOWIE research campaign methodology. This is a significant proof-of-concept for autonomous drone-based precipitation enhancement — a technology already deployed across the American West and Colorado River Basin.

---

## What Cloud Seeding Is

**Glaciogenic cloud seeding** uses silver iodide (AgI) as an ice-nucleating agent. AgI particles mimic the crystal structure of ice and cause supercooled water droplets in clouds to freeze around them — creating ice particles that grow and eventually fall as precipitation.

**Why drones:** Traditional cloud seeding uses fixed ground generators or manned aircraft. Drones enable precise altitude targeting of cloud layers, can respond rapidly to atmospheric windows, and offer significant cost advantages at scale.

**Why it works:** Glaciogenic seeding is the best-studied and most extensively validated form of precipitation enhancement. It has been deployed across the Colorado River Basin, throughout the American West, and internationally. The physical mechanism is well-understood; the challenge is operational validation (proving that observed precipitation is caused by seeding, not natural variability).

---

## The Operation

**Dates:** August 22–23, 2026
**Location:** Kenai Peninsula, Alaska
**Duration:** 3 hours, 06:18–09:54 UTC (Aug 23)
**Drones:** EL-151 and EL-153 (Rainmaker "Elijah" platform)
**AgI released:** ~374.3 g total (19.7 g per flare × 19 flares)
**Missions:** 7 coordinated seeding missions

**Atmospheric conditions:**
- Strengthening low pressure in Gulf of Alaska bringing northeasterly flow
- Freezing level: ~1,874 m MSL
- -5°C isotherm: ~2,813 m | -15°C isotherm: ~4,756 m (AgI activation range: -5 to -15°C)
- Nearly saturated atmosphere throughout column
- Mixed-phase conditions confirmed (both ice and supercooled liquid water present)
- Seeding altitudes: 3.57–4.27 km MSL
- All release temperatures within the -5 to -15°C AgI activation window

---

## Validation Methodology

**The challenge:** Cloud seeding operations occur in evolving natural clouds, not in isolation. Attributing observed precipitation to seeding (vs. natural variability) is inherently difficult.

**Rainmaker's approach** builds on the 2017 SNOWIE (Seeded and Natural Orographic Wintertime clouds — the Idaho Experiment) campaign, which first demonstrated physical tracing of seeded precipitation via radar.

**Requirements for attribution (seeding signature must satisfy all):**
1. Appears within the physically plausible downwind corridor from the AgI release point
2. Exhibits physically consistent motion (advects with ambient wind, not terrain-locked)
3. Shows coherent vertical development consistent with seeded ice growth and sedimentation
4. Has appropriate lifecycle (growth and persistence across successive radar volumes)
5. Is **repeatable** across separate releases with similar timing, motion, and morphology

**QPE (Quantitative Precipitation Estimation):**
- Composite reflectivity from lowest unblocked PAHG NEXRAD radar gate
- 27-member Z-S ensemble (Z = aS^b where a=100–500, b=2.0–2.2) to capture uncertainty in ice crystal characteristics
- 5 dBZ subtracted from all gates to isolate seeding enhancement above natural background
- Result: ensemble distribution of precipitation volumes, not a single estimate

---

## Results

### Seven Confirmed Seeding Signatures (A–G)

| Signature | Mission | Flares | Release UTC | Detection UTC | Duration | Max extent |
|-----------|---------|--------|-------------|---------------|----------|------------|
| A | 1 | 4 | 06:38 | 07:18 (+40 min) | 56 min / 9 vols | 16.4 km² |
| B | 2 | 4 | 07:07 | 07:39 (+33 min) | 98 min / 15 vols | 41.2 km² |
| C | 3 | 3 | 07:37 | 08:07 (+31 min) | 84 min / 13 vols | 39.8 km² |
| D | 4 | 2 | 08:20 | 08:42 (+23 min) | 91 min / 14 vols | 35.4 km² |
| E | 5 | 2 | 08:43 | 09:03 (+20 min) | 77 min / 12 vols | 30.4 km² |
| F | 6 | 2 | 09:07 | 09:24 (+17 min) | 63 min / 10 vols | 16.9 km² |
| G | 7 | 2 | 09:29 | 09:59 (+30 min) | 42 min / 7 vols | 9.4 km² |

Detection lag decreased from 40 min (A) to 17 min (F) — attributed to the earlier signatures being embedded in heavier natural precipitation, making initial identification more difficult.

Signatures D, E, F were detected at all four available radar tilts (0.48°–1.8°), reaching tops above 4.2 km MSL — the greatest vertical extent, correlating with their higher-altitude, colder-temperature releases.

### Quantitative Precipitation Estimation

| Metric | Value |
|--------|-------|
| Total mean QPE | **57.6 acre-feet** (71,072 m³) |
| 5th–95th percentile range | 41.7–89.4 acre-feet |
| 25th / 50th / 75th percentile | 45.7 / 53.0 / 64.8 acre-feet |
| Affected area | 249.53 km² |
| Core area (>1.0 mm precipitation) | Subset of the above |

**Signatures B and C** together accounted for >50% of total precipitation, each contributing >14 acre-feet. Both persisted >1 hour.

---

## Significance

**Why 57.6 acre-feet matters:** One acre-foot is ~1,233 cubic meters, enough water for two average American households for a year. 57.6 acre-feet = ~71,000 cubic meters = ~71 million liters of additional precipitation from a 3-hour operation with ~374 g of AgI.

**The technology maturity argument:** Glaciogenic cloud seeding is not experimental — it's deployed operationally across the American West, including the Colorado River Basin and for Great Salt Lake restoration. This report validates that drone delivery of AgI (as opposed to ground generators or manned aircraft) is capable of producing demonstrable precipitation enhancement with measurable results.

**The scalability question:** Seven missions, two drones, 3 hours. The platform is clearly designed for autonomous multi-drone coordination. The question is: at what scale does this become a meaningful water supply tool vs. a supplemental enhancement?

---

## Methodological Caveats

The QPE uncertainty range (41.7–89.4 acre-feet) is significant — more than 2× between 5th and 95th percentile. The uncertainty sources include:
- Choice of Z-S relationship (captured in the 27-member ensemble)
- Radar calibration and beam geometry
- Vertical hydrometeor evolution (sublimation/evaporation below radar beam)
- Polygon placement accuracy
- Possible contamination from weak natural precipitation

The 5 dBZ background subtraction mitigates some contamination but introduces its own assumptions. The QPE represents frozen precipitation as measured ~1,800 m MSL and above — what reaches the surface is inferred, not directly measured.

---

## Questions & Gaps
- What is the energy cost and economic cost of the operation? 374 g of AgI plus drone operation costs vs. 57.6 acre-feet of water — is this cost-effective relative to other water supply methods?
- The report covers a single 3-hour Intensive Observation Period (IOP). What's the operational cadence for a full season deployment? How often can suitable atmospheric windows be exploited?
- The validation methodology is designed for individual signature identification. How does QPE scale when many signatures overlap or merge — does the ensemble approach handle this accurately?
- Rainmaker plans "refinement of its cloud seeding platform in the upcoming winter season" — what specifically is being refined? The drone platform, the AgI dispersion mechanism, the atmospheric targeting, or the validation methodology?

## Related Notes
- This note stands alone in the research library — no prior notes on climate technology, weather modification, or atmospheric science. First entry in `science/research/` or similar.
