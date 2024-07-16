---
title: "Understanding Cost Estimation in PostgreSQL"
excerpt_separator: "<!--more-->"
categories:
  - PostgreSQL
tags:
  - PostgreSQL
  - Query Optimization
  - Cost Estimation
---

This post assumes no parallelism is used, that is, only one worker is assigned.

## Single-Table Query

### Sequential Scan

$
\text{Total Cost} = \text{Startup Cost} + \text{CPU Run Cost} + \text{Disk Run Cost}
$

$
\text{CPU Run Cost} = \text{CPU Cost per Tuple} \times N_{\text{tuples}} + \text{Output Columns Evaluation Cost per Row} \times N_{\text{rows}}
$

$
\text{Disk Run Cost} = \text{Sequential Page Cost} \times N_{\text{pages}}
$

### Sample Scan

Work in progress :construction:...

### Gather

$
\text{Total Cost} = \text{Startup Cost} + \text{Run Cost}
$

$
\text{Run Cost} = \text{Subpath Total Cost} - \text{Subpath Startup Cost} + \text{Parallel Tuple Cost} \times N_{\text{rows}}
$

### Gather Merge

$
\text{Total Cost} = \text{Startup Cost} + \text{Run Cost} + \text{Input Total Cost}
$

$
\text{Startup Cost} = \text{Comparsion Cost} \times N \times \log N + \text{Parallel Setup Cost} + \text{Input Setup Cost}
$

$
\text{Run Cost} = N_{\text{rows}} \times \text{Comparison Cost} \times \log N + \text{CPU Operator cost} \times N_{\text{rows}} + \text{Parallel Tuple Cost} \times N_{\text{rows}} \times 1.05
$

### Index Scan

### References

The baseline for this analysis was derived from [The Internals of PostgreSQL](https://www.interdb.jp/pg/).
