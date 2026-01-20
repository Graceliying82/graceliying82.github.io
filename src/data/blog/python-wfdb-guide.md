---
title: "Introduction to WFDB: Working with PhysioNet Data"
author: Grace Li
pubDatetime: 2026-01-20T07:10:00Z
postSlug: python-wfdb-guide
featured: false
draft: false
tags:
  - Python
  - Data Science
  - Electrophysiology
description: A beginner's guide to using the WFDB library to download, read, and plot ECG data from PhysioNet.
---

## Introduction

The **WFDB** (Waveform Database) package is the standard Python tool for interacting with PhysioNet, the world's largest repository of medical research data. If you are working with ECG, EEG, or other physiological signals, mastering WFDB is essential.

## Installation

First, ensure you have the library installed:

```bash
pip install wfdb
```

## downloading Data

The `dl_database` function allows you to download records directly from PhysioNet.

```python
import wfdb
import os

# Download 5 records from the PTB Diagnostic Database (ptbdb)
# to a local 'data' directory.
wfdb.dl_database('ptbdb', 'data', limit=5, overwrite=False)
```

## Reading Records

Once downloaded, you can read the data using `rdsamp` (read sample). This function returns two items:
1.  **Signals**: A numpy array containing the raw signal data.
2.  **Fields**: A dictionary containing metadata (frequency, units, comments, etc.).

```python
# Read record 'patient001/s0010_re' from the 'data' folder
signals, fields = wfdb.rdsamp('data/patient001/s0010_re')

print(f"Sampling Frequency: {fields['fs']} Hz")
print(f"Signal Shape: {signals.shape}")
print(f"Comments: {fields['comments']}")
```

## Plotting Signals

WFDB has a built-in plotting helper, but you can also use standard Matplotlib.

### Using `wfdb.plot_wfdb`

```python
record = wfdb.rdrecord('data/patient001/s0010_re')
wfdb.plot_wfdb(record=record, title='Record s0010_re from PTBDB')
```

### Using Matplotlib (Manual Control)

For more custom visualizations (like in DeepPulse), you can plot the numpy array directly:

```python
import matplotlib.pyplot as plt

# Plot the first lead (Lead I)
plt.figure(figsize=(10, 4))
plt.plot(signals[:, 0]) # All time points, 1st channel
plt.title("Lead I ECG")
plt.xlabel("Samples")
plt.ylabel("Voltage (mV)")
plt.show()
```

## Summary

| Function | Purpose |
| :--- | :--- |
| `wfdb.dl_database` | Downloads datasets from PhysioNet |
| `wfdb.rdsamp` | Reads signal data into numpy arrays |
| `wfdb.rdrecord` | Reads data into a WFDB Record object |
| `wfdb.plot_wfdb` | Quickly visualizes the signals |

This library is the backbone of applications like DeepPulse, enabling meaningful analysis of cardiac data.
