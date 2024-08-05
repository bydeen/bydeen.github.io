---
title: "Understanding Cost Estimation in PostgreSQL"
excerpt_separator: "<!--more-->"
categories:
  - PostgreSQL
tags:
  - PostgreSQL
  - Query Optimization
  - Cost Estimation
use_math: true
---

The foundational concepts for this analysis were derived from [The Internals of PostgreSQL](https://www.interdb.jp/pg/), which provides the basic equation for cost estimation. The detailed code analysis and organization of functions presented in this post are based on my examination of [costsize.c](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c).

This post assumes no parallelism is used, that is, only one worker is assigned.

## Single-Table Query

### Sequential Scan

$ \small
\text{Total Cost} = \text{Startup Cost} + \text{CPU Run Cost} + \text{Disk Run Cost}
$

$ \small
\text{CPU Run Cost} = \text{CPU Cost per Tuple} \times N*{\text{tuples}} + \text{Output Columns Evaluation Cost per Row} \times N*{\text{rows}}
$

$ \small
\text{Disk Run Cost} = \text{Sequential Page Cost} \times N\_{\text{pages}}
$

### Sample Scan

Work in progress :construction:...

### Gather

$ \small
\text{Total Cost} = \text{Startup Cost} + \text{Run Cost}
$

$ \small
\text{Run Cost} = \text{Subpath Total Cost} - \text{Subpath Startup Cost} + \text{Parallel Tuple Cost} \times N\_{\text{rows}}
$

### Gather Merge

$ \small
\text{Total Cost} = \text{Startup Cost} + \text{Run Cost} + \text{Input Total Cost}
$

$ \small
\text{Startup Cost} = \text{Comparsion Cost} \times N \times \log N + \text{Parallel Setup Cost} + \text{Input Setup Cost}
$

$ \small
\text{Run Cost} = N*{\text{rows}} \times \text{Comparison Cost} \times \log N + \text{CPU Operator cost} \times N*{\text{rows}} + \text{Parallel Tuple Cost} \times N\_{\text{rows}} \times 1.05
$

### Index Scan
