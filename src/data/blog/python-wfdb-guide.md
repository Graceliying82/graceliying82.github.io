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

## Exploring Different Databases

PhysioNet hosts dozens of databases. While we used `ptbdb` (PTB Diagnostic ECG Database) above, you can access many others by changing the **slug**:

*   **`mitdb`**: MIT-BIH Arrhythmia Database (The "Hello World" of ECG).
*   **`bidmc`**: BIDMC Congestive Heart Failure Database.
*   **`incartdb`**: St. Petersburg INCART 12-lead Arrhythmia Database.

To find more, check the [PhysioNet Index](https://physionet.org/about/database/).

## What Does the Data Look Like?

When you call `rdsamp`, the `fields` dictionary gives you crucial context. Here is what you can expect:

```python
# signals, fields = wfdb.rdsamp(...)
print(fields)

# Output Example:
{
    'fs': 1000,                  # Sampling frequency (Hz)
    'sig_len': 38400,            # Total number of samples
    'n_sig': 12,                 # Number of signals (leads)
    'base_date': None,
    'base_time': None,
    'units': ['mV', 'mV', ...],  # Units for each channel
    'sig_name': ['i', 'ii', 'iii', 'avr', 'avl', 'avf', 'v1', 'v2', 'v3', 'v4', 'v5', 'v6'],
    'comments': [
        'age: 69', 
        'sex: male', 
        'clinical diagnosis: myocardial infarction'
    ]
}
```

## Pagination and Handling Large Datasets

Some databases contain thousands of records. Downloading everything at once is slow and fills up your disk. Instead, you can "paginate" by fetching the record list first and downloading in chunks.

```python
# 1. Get the list of ALL records keys
all_records = wfdb.get_record_list('ptbdb') # e.g., ['patient001/s0010_re', ...]

print(f"Total records found: {len(all_records)}")

# 2. Define your "page" size
page_size = 5
page_number = 1 # 2nd page (0-indexed)

start_idx = page_number * page_size
end_idx = start_idx + page_size

# 3. Slice the list
target_records = all_records[start_idx:end_idx]

# 4. Download ONLY those specific records
wfdb.dl_database('ptbdb', 'data', records=target_records, overwrite=False)

print(f"Downloaded records {start_idx} to {end_idx}")
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
