---
title: "Mastering Activation Mapping: Choosing the Window of Interest"
author: Grace Li
pubDatetime: 2026-01-18T14:30:00Z
postSlug: activation-mapping-window-of-interest
featured: true
draft: false
tags:
  - Electrophysiology
  - Activation Mapping
  - Atrial Flutter
  - Cardiology
description: A technical guide on selecting the correct Window of Interest (WOI) for activation mapping in complex cases like atrial flutter and slow conduction.
---

## Introduction

Activation mapping is a cornerstone of modern electrophysiology, essential for diagnosing and treating arrhythmias like atrial flutter. However, the quality of an activation map is only as good as the parameters set to create it. One of the most critical—and often misunderstood—parameters is the **Window of Interest (WOI)**.

Recently, I came across an amazing tutorial that broke down this concept with intuitive clarity.

<br>
<iframe width="100%" height="315" src="https://www.youtube.com/embed/CoRDSVjHQVU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
<br>

In this post, I want to synthesize those insights into a technical reference for choosing the WOI, particularly for complex cases involving slow conduction.

## What is the Window of Interest (WOI)?

The WOI dictates which electrograms (EGMs) the mapping system analyzes relative to a stable reference. It effectively tells the system: *"Look for local activation only within this specific time range."*

Setting this incorrectly can lead to:
- **Cycle skipping:** Assigning activation to the wrong beat.
- **Fragmented maps:** Failure to visualize the full reentrant circuit.
- **"Early-Meets-Late" artifacts:** Misinterpreting propagation lines.

## Strategy for Setting the WOI

### 1. Establish a Stable Reference
Before touching the window, ensure you have a stable reference, typically a coronary sinus (CS) catheter. The variation in tachycardia cycle length (TCL) should be **<10%** to ensure map consistency.

### 2. Determine Tachycardia Cycle Length (TCL)
Measure the TCL accurately. Your WOI duration should generally cover **90-95% of the TCL**.
*   *Example:* If TCL = 240ms, a WOI of ~220-230ms is appropriate.
*   *Why not 100%?* To avoid "cross-talk" or sensing the next beat's far-field signal at the edges of the window.

### 3. Positioning the Window
The standard approach is a **50/50 split** around the reference electrogram:
*   **Window Start:** -50% of TCL
*   **Window End:** +50% of TCL

However, for **Atrial Flutter**, especially macro-reentrant circuits, you may need to adjust this to ensure the entire circuit is captured.

## Special Case: Slow Conduction & Atrial Flutter

In reentrant tachycardias, the critical isthmus often exhibits **slow conduction**. 

If the WOI is too narrow or centered incorrectly, the electrograms in the slow conduction zone might fall *outside* the window. This results in a map that looks like a focal breakout rather than a continuous loop.

**Best Practice:**
1.  **Observe the P-wave:** In typical flutter, setting the WOI to encompass the P-wave or end of it can help capture determining isthmus conduction.
2.  **Look for Mid-Diastolic Potentials:** In the critical isthmus (e.g., CTI for typical flutter), look for fractionated, low-amplitude signals. Ensure your WOI includes these potentials.
3.  **Adjust Guidelines:**
    - If you see a "collision" or line of block where you expect conduction, try **shifting the WOI**.
    - Moving the window forward or backward by 10-20ms can sometimes "connect" the early and late points, revealing the true reentrant path.

## Conclusion

Precise window setting isn't just a technicality; it's the lens through which we visualize the arrhythmia mechanism. By carefully choosing the WOI—especially in cases of slow conduction—we can differentiate between focal mechanisms and macro-reentry, leading to more effective ablation strategies.

---
*Reference: Insights adapted from recent community tutorials on activation mapping.*
