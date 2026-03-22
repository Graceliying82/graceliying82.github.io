---
title: "Inside the Map: How Cardiac EP Systems Calculate LAT and Activation"
author: Grace Li
pubDatetime: 2026-03-22T08:30:00Z
postSlug: ep-mapping-lat-explainer
featured: true
draft: false
tags:
  - electrophysiology
  - cardiac-mapping
  - medical-engineering
  - signal-processing
  - 3d-reconstruction
description: "A deep dive into the engineering behind cardiac mapping: from raw catheter signals and sample indices to 3D activation maps and color interpolation."
---

> **Target audience:** Software engineers and technical readers who want to understand the clinical and signal-processing concepts behind 3D cardiac mapping systems.


## Table of Contents

1. [What Is Cardiac Electrophysiology Mapping?](#1-what-is-cardiac-electrophysiology-mapping)
2. [The Hardware: Catheters and the Recording Chain](#2-the-hardware-catheters-and-the-recording-chain)
3. [The Engineering Challenge: Synchronization and the 3D Mesh](#3-the-engineering-challenge-synchronization-and-the-3d-mesh)
4. [The Raw Signal: Electrograms (EGMs)](#4-the-raw-signal-electrograms-egms)
5. [What Is LAT (Local Activation Time)?](#5-what-is-lat-local-activation-time)
6. [How Sample Indices Become LAT Values](#6-how-sample-indices-become-lat-values)
7. [How Many Points Are Collected?](#7-how-many-points-are-collected)
8. [Handling Duplicate Measurements](#8-handling-duplicate-measurements)
9. [From Sparse Points to a Full Surface Map (Interpolation)](#9-from-sparse-points-to-a-full-surface-map-interpolation)
10. [How Colors Are Decided](#10-how-colors-are-decided)
11. [Clinical Interpretation: What the Colors Tell the Doctor](#11-clinical-interpretation-what-the-colors-tell-the-doctor)
12. [Summary Diagram](#12-summary-diagram)
13. [Conclusion](#13-conclusion)
14. [Further Reading](#14-further-reading)

---

## 1. What Is Cardiac Electrophysiology Mapping?

The heart beats because of electrical impulses. In a healthy heart, a signal originates in the **sinoatrial (SA) node**, travels through well-defined pathways, and causes coordinated muscle contraction. In patients with arrhythmias (abnormal heart rhythms), this electrical propagation is disrupted — signals may loop back on themselves, originate from the wrong place, or travel along abnormal accessory pathways.

**Cardiac electrophysiology (EP) mapping** is a clinical procedure in which a cardiologist inserts thin, flexible catheters into the heart through blood vessels and records the electrical activity from inside the heart chambers. The goal is to create a **3D map** of the heart surface annotated with electrical timing data — revealing exactly where and when the abnormal activity originates, so it can be targeted with ablation (tissue destruction via heat or cold).

Modern EP mapping systems — such as [OPAL HDx (formerly Rhythmia)](https://www.bostonscientific.com/us/en/healthcare-professionals/products/mapping-and-navigation-systems/opal-hdx-mapping-system/fp00000293.html) from Boston Scientific, CARTO (Biosense Webster), and EnSite (Abbott) — combine real-time 3D catheter tracking with multi-channel signal recording to build these maps automatically during the procedure.

---

## 2. The Hardware: Catheters and the Recording Chain

### The Mapping Catheter

A mapping catheter is a thin flexible tube (typically 2–4 mm in diameter) with multiple electrodes embedded near its tip. As the cardiologist maneuvers the catheter to touch different regions of the heart wall, each electrode records a local electrical signal.

Types of mapping catheters:
- **Basket catheters** (e.g. [OPAL HDx™](https://www.bostonscientific.com/us/en/healthcare-professionals/products/mapping-and-navigation-systems/opal-hdx-mapping-system/fp00000293.html) formerly Rhythmia): spherical cage with 64 electrodes that expands inside a chamber, enabling high-density mapping with thousands of points in just a few minutes.
- **Multi-electrode catheters** (e.g. PentaRay, HD Grid): designs with 16–20 electrodes (PentaRay: 20 electrodes on 5 splines; HD Grid: 16 electrodes in a 4x4 array) that record over a small area.
- **Point-by-point catheters** (e.g. CARTO SmartTouch): one contact point at a time, moved manually.

### The Reference Catheter

A **reference catheter** (typically placed in the coronary sinus — a fixed anatomical structure in the heart) records continuously throughout the procedure. It provides a stable, repeating electrical signal tied to the cardiac cycle. Every other electrode's activation is measured **relative to** this reference signal.

### The Recording System

All catheter channels are connected to a **recording amplifier** (a multi-channel ADC — analog-to-digital converter). It samples all channels simultaneously at a fixed rate — typically **1000–2000 Hz** (samples per second). Every millisecond (at 1000 Hz), one digital value per channel is written into a continuous rolling buffer.

```
Recording amplifier
  ├── Channel 0: Reference catheter  → [0.1, 0.2, 0.0, -0.3, ...]
  ├── Channel 1: Mapping electrode 1 → [0.0, 0.0, 0.1, -0.8, ...]
  ├── Channel 2: Mapping electrode 2 → [0.0, 0.1, 0.0, -0.2, ...]
  └── ...
      Sample index:                       0    1    2    3    ...
```

The sample index is simply the hardware clock tick counter — it increments automatically with no action required from software.

---

## 3. The Engineering Challenge: Synchronization and the 3D Mesh

Before we dive into the signals, it's important to understand the silent heavy lifting done by the mapping system's engine.

### Hardware Synchronization

The mapping system is a distributed real-time environment. It must synchronize:
1. **Clock cycles** across multiple ADC boards (to ensure 1ms on lead I is the same 1ms on the mapping catheter).
2. **3D Positional Data** from magnetic or impedance-based tracking sensors.
3. **Surface ECG** signals from the patient's chest.

Any "jitter" or latency in the pipeline (e.g., if the 3D position is recorded 50ms after the electrical signal) results in a "noisy" map where points appear to be floating off the wall or misaligned with the electrical activation.

### The 3D Mesh Reconstruction

The "heart" you see on the screen isn't a pre-loaded model; it's a **point-cloud-to-mesh reconstruction** built on the fly. As the catheter moves, the system collects (x, y, z) coordinates. Algorithms like **Ball Pivoting** or **Poisson Surface Reconstruction** are used to wrap a triangular mesh around these points. 

This mesh provides the "canvas" upon which we paint the electrical data.

---

## 4. The Raw Signal: Electrograms (EGMs)

An **electrogram (EGM)** is the raw voltage-vs-time waveform recorded from one electrode inside the heart. It is analogous to a single lead of an ECG, but recorded from inside the heart rather than from the body surface.

### Unipolar vs. Bipolar EGMs

| Type | How recorded | Characteristics |
|---|---|---|
| **Unipolar** | One electrode vs. a far-field reference | Large amplitude; reflects all electrical activity near and far |
| **Bipolar** | Two adjacent electrodes subtracted | Cancels far-field noise; reflects only local activation |

Bipolar EGMs are preferred for mapping because they are more spatially specific.

### The Activation Signature

When a wavefront of electrical activation passes under an electrode, the bipolar EGM shows a characteristic sharp deflection — a rapid downstroke followed by an upstroke. The moment of steepest **negative slope (max −dV/dt)** is universally accepted as the **local activation time** for that electrode.

```
Bipolar EGM signal:

 mV
  |
  |    ___
  |   /   \
  |__/     \___/\___
  |              ↑
  |          max -dV/dt  ← This is when the wavefront passed
  |
  └──────────────────── time (ms)
```

---

## 5. What Is LAT (Local Activation Time)?

**Local Activation Time (LAT)** is the time delay, in milliseconds, between a reference event and the moment a specific region of heart tissue electrically activates.

It answers the question: **"How late does this spot on the heart wall wake up, relative to a known reference event?"**

### Why measure relative time?

The heart beats continuously. Each beat has a slightly different absolute timing. By anchoring all measurements to the **same reference event** (e.g. the reference catheter signal, or the pacing stimulus), LATs from different beats and different catheter positions become directly comparable.

```
Reference fires (e.g. CS catheter): t = 0 ms
  │
  ├── Location A: activates at t = -45 ms → LAT = -45 ms (early)
  ├── Location B: activates at t = 20 ms  → LAT = 20 ms
  └── Location C: activates at t = 100 ms → LAT = 100 ms (late)
```

These LAT values, combined with the 3D position of each electrode, form the raw data for an activation map.

---

## 6. How Sample Indices Become LAT Values

### The annotation

The EP recording system stores activation events as **annotations** — the sample index at which activation was detected on each channel.

```
mapAnnot[electrode_i]       = 2069   # sample index of local activation
referenceAnnot[electrode_i] = 2000   # sample index of reference event (same beat)
```

`mapAnnot = 2069` does **not** mean 2069 milliseconds of absolute time. It means "at the 2069th sample in the recording buffer, electrode i activated."

### The conversion

```
diff_samples = mapAnnot - referenceAnnot         # 2069 - 2000 = 69 samples
diff_seconds = diff_samples / sample_frequency   # 69 / 1000 Hz = 0.069 s
LAT_ms       = diff_seconds × 1000               # 0.069 × 1000 = 69 ms
```

Or in one line:
```
LAT = ((mapAnnot - refAnnot) / fs) * 1000
```

This formula is **sample-rate agnostic**. Whether the system samples at 1000 Hz, 1200 Hz, or 2000 Hz, the formula always produces the correct LAT in milliseconds.

### Beat-to-beat reset

For each new heartbeat, `referenceAnnot` is re-detected on the reference catheter. This means the absolute sample indices keep climbing throughout the procedure (e.g. beat 50 might have `referenceAnnot = 150,000`), but the *difference* between map annotation and reference annotation stays within the physiological range of one cardiac cycle (~0–400 ms for most arrhythmias).

### Sentinel values

EP systems use large out-of-range values (e.g. −10,000 ms) to flag electrodes with no valid annotation — poor contact, no signal detected, or the electrode was outside the window of interest. These must be masked before interpolation.

---

## 7. How Many Points Are Collected?

The number of measurement points is **finite and constrained by the procedure**. There is no automatic scanning — the cardiologist manually navigates the catheter to each location. Coverage depends on:

- How long the procedure takes (typically 2–6 hours)
- The mapping system and catheter type
- Arrhythmia stability (the same rhythm must be sustained throughout)

### Typical point counts by system

| System | Technology | Typical points |
|---|---|---|
| OPAL HDx (BSC) | Basket catheter, automated | 2,000–10,000 |
| CARTO 3 (Biosense Webster) | Point-by-point, magnetic tracking | 100–500 |
| EnSite Velocity (Abbott) | Non-contact or multi-electrode | 500–3,000 |

Meanwhile, the 3D **mesh surface** reconstructed from the same catheter movements may have **10,000–30,000 vertices**. Only a small fraction have real electrode measurements. The rest are interpolated.

---

## 8. Handling Duplicate Measurements

The cardiologist frequently revisits the same anatomical location — either intentionally (to confirm a reading) or because catheter movement is imprecise. How duplicate points are handled varies by system:

### BSC (OPAL HDx) / Abbott (EnSite) - High-Density

- These systems often use a **Voxel-based** approach (e.g., [Voxel-based editing in OPAL HDx](https://www.bostonscientific.com/us/en/healthcare-professionals/products/mapping-and-navigation-systems/opal-hdx-mapping-system/fp00000293.html)).
- Data is collected automatically at high speed (OPAL HDx) or via multi-electrode sweeps (EnSite).
- Points are grouped into **spatial bins (voxels)** on the 3D mesh.
- The system computes a **median or weighted average** LAT per bin to reduce noise.
- Outliers are automatically rejected using quality metrics like cycle length stability and **morphology matching**.

### CARTO (Point-by-Point)

- Traditionally focuses on high-precision, point-by-point data acquisition (e.g., [CARTO 3 System](https://www.jnjmedtech.com/en-US/product/carto-3-system)).
- Accepts points within a configurable spatial tolerance (typically [2–3 mm in clinical practice](https://pubmed.ncbi.nlm.nih.gov/20929532/)).
- Uses the **highest-quality** point for the color map based on a **Smart Index** or stability score.
- Quality criteria: signal amplitude, catheter stability (**Stability+**), contact force, and respiratory phase gating.

### Why this matters

If duplicates were naively averaged, a single noisy reading (e.g. the catheter was briefly off the wall) could corrupt the map in that region. Real systems use quality metrics to weight or exclude readings — good contact force, stable catheter position, and signal amplitude within normal range all increase a point's weight.

---

## 9. From Sparse Points to a Full Surface Map (Interpolation)

After collecting LAT values at a few hundred to a few thousand electrode positions, the system must **fill in the gaps** — assigning LAT values to every vertex on the mesh so the entire surface can be colored.

This is a **spatial interpolation** problem.

The simplest approach. For any unsampled mesh vertex `v`, the estimated LAT is a weighted average of all nearby electrode measurements, where closer electrodes get higher weight:

```
LAT(v) = sum(wi * LATi) / sum(wi)
```

where
```
wi = 1 / (dist(v, electrode_i)^p)
```

- `p = 2` is common (inverse square distance)
- Simple and fast, but can produce "bullseye" artifacts around isolated points

A more sophisticated method. Fits a smooth continuous function through all measurement points:

```
LAT(v) = sum(λi * φ(||v - electrode_i||))
```

Where `φ` is a radial basis function (e.g. thin-plate spline, multiquadric). RBF produces smoother maps and handles irregular sampling better than IDW.

### Geodesic vs. Euclidean Distance

A subtlety: the heart wall is a curved surface. Straight-line (Euclidean) distance through 3D space is not the same as the distance along the surface. High-end systems use **geodesic distance** (shortest path along the mesh surface) for interpolation, which is more physically meaningful — electrical conduction follows the tissue surface, not a straight line through the blood pool.

### Boundary conditions

The mesh typically has **rim vertices** — edges at the top of a chamber where the anatomy was not fully mapped (e.g. the pulmonary vein openings in atrial fibrillation ablation). These are flagged and excluded or handled separately during interpolation.

---

## 10. How Colors Are Decided

Once every mesh vertex has an interpolated LAT value, a **colormap** is applied to produce the final visual.

### The colormap

A colormap is a function that maps a scalar value (LAT in ms) to an RGB color. The standard in EP mapping is:

```
Early activation  →  Red / Magenta   (low LAT, e.g. 0–30 ms)
                  →  Yellow / Green   (mid LAT)
Late activation   →  Purple / Blue    (high LAT, e.g. 100–200 ms)
```

This is sometimes called a **"rainbow"** or **"jet"** colormap, though clinical systems often use proprietary variants.

### The color range (window/level)

The colormap is not applied to the full mathematical range of LAT values — that would waste color resolution on outliers. Instead, the clinician sets:

- **Minimum (early limit):** e.g. 0 ms → maps to red
- **Maximum (late limit):** e.g. 180 ms → maps to purple
- Any LAT below the minimum is clamped to red; above maximum to purple

This is analogous to **windowing** in radiology (CT/MRI window-level adjustment).

### The scalar bar

A color legend (scalar bar) displayed alongside the 3D map shows the LAT scale, so the clinician can read off approximate activation time from color.

### Example

```
LAT range set: 0 ms → 150 ms

  Red       Yellow      Green      Cyan       Blue
   |___________|___________|___________|__________|
  0ms         37ms        75ms       112ms      150ms

A red patch = activates first (likely the origin of the arrhythmia)
A blue patch = activates last (far from the origin)
```

---

## 11. Clinical Interpretation: What the Colors Tell the Doctor

### Activation map (LAT map)

- **Red/early region** = where the electrical wavefront originates (the arrhythmia source or the pacing site)
- **Smooth color gradient** = normal, orderly conduction
- **Abrupt color jumps** = conduction block (the wavefront cannot cross that line directly)
- **A "red island" surrounded by blue** = re-entrant circuit (the wavefront is circling and arriving back at the start)

### Voltage map

A related but different map: instead of timing, it colors the surface by **EGM amplitude** (peak-to-peak voltage):

- **Purple / high voltage** = healthy, thick myocardium
- **Red / low voltage** = scarred or diseased tissue (low amplitude because fewer viable cells)

Scar tissue is often the substrate for arrhythmias. Voltage maps help identify the scar border, where ablation lines are typically drawn.

### Propagation map (animation)

Rather than a static color map, many systems animate the activation: a moving colored band sweeps across the surface in real time, showing the wavefront propagating. This makes re-entrant circuits visually obvious as a continuously rotating wave.

---

## 12. Summary Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    CARDIAC EP MAPPING PIPELINE                   │
│                                                                   │
│  HARDWARE          SIGNAL            ANNOTATION       MAP        │
│                                                                   │
│  Catheter ──▶ ADC ──▶ EGM waveform ──▶ detect       ──▶ LAT    │
│  touches        samples    (voltage       max -dV/dt      value  │
│  heart wall     at         vs time)       per channel    (ms)    │
│                 1000 Hz                   → mapAnnot             │
│                                                                   │
│  Reference ──▶ ADC ──▶ reference  ──▶ detect same  ──▶ ref     │
│  catheter        samples    EGM           event each      value  │
│  (fixed)                                  beat                   │
│                                           → refAnnot             │
│                                                                   │
│                    LAT = (mapAnnot - refAnnot) / fs * 1000       │
│                                                                   │
│  3D position  +  LAT value  =  sparse point cloud on heart       │
│                                                                   │
│  Sparse points ──▶ IDW / RBF interpolation ──▶ LAT at every     │
│                                                   mesh vertex     │
│                                                                   │
│  LAT per vertex ──▶ colormap (early=red, late=blue) ──▶ 3D map  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13. Conclusion

Cardiac EP mapping is a fascinating intersection of **electrophysiology, 3D geometry, and real-time signal processing**. By converting raw analog signals into synchronized digital samples, and then interpolating those samples across a dynamically reconstructed mesh, we provide physicians with a "GPS for the heart."

Understanding the underlying math—from the simple subtraction of sample indices to the complexity of geodesic RBF interpolation—is key to building next-generation mapping systems that are faster, more accurate, and ultimately more effective at curing arrhythmias.

---

## 14. Further Reading

### Clinical / Domain
- Josephson, M.E. *Clinical Cardiac Electrophysiology: Techniques and Interpretations* (5th ed.) — the standard clinical reference
- Zipes & Jalife. *Cardiac Electrophysiology: From Cell to Bedside* (7th ed.) — comprehensive textbook covering mechanisms and mapping
- Calkins H, et al. ["2017 HRS/EHRA/ECAS/APHRS/SOLAECE Expert Consensus Statement on Catheter and Surgical Ablation of Atrial Fibrillation"](https://www.heartrhythmjournal.com/article/S1547-5271(17)30547-8/fulltext) — *Heart Rhythm*, 2017

### Signal Processing
- Steinberg, J.S. & Mittal, S. *Electrophysiology: The Basics* — accessible primer on EP signals
- Bhargava M, et al. "Impact of New Generation of Mapping Systems on Ablation." *Cardiology Clinics*, 2009

### EP Mapping Systems (vendor documentation / white papers)
- [Boston Scientific OPAL HDx™ Mapping System (formerly Rhythmia HDx™)](https://www.bostonscientific.com/us/en/healthcare-professionals/products/mapping-and-navigation-systems/opal-hdx-mapping-system/fp00000293.html)
- [Biosense Webster CARTO 3 System](https://www.biosensewebster.com/our-solutions/mapping-navigation/carto-3-system.html)
- [Abbott EnSite X EP System](https://www.cardiovascular.abbott/us/en/hcp/products/electrophysiology/mapping-and-navigation/ensite-x-ep-system.html)

### Open Data / Research Tools
- [OpenEP Project](https://openep.io) — open-source framework and datasets for EP mapping research
- Coveney S, et al. ["OpenEP: An Open-Source Platform for Electrophysiology Research"](https://www.frontiersin.org/articles/10.3389/fphys.2021.672765/full) — *Frontiers in Physiology*, 2021
- [openep-py on GitHub](https://github.com/openep/openep-py) — Python interface to OpenEP datasets

### Interpolation Methods
- Shepard, D. ["A two-dimensional interpolation function for irregularly-spaced data"](https://dl.acm.org/doi/10.1145/800186.810616) — original IDW paper, ACM 1968
- Buhmann, M.D. *Radial Basis Functions: Theory and Implementations* — Cambridge University Press, 2003

---

*Written as a technical explainer for engineers entering the cardiac EP domain. All clinical decisions should be made by qualified medical professionals.*
