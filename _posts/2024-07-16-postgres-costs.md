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

## Prelimiaries

1. Path costs are measured in _arbitrary units_ defined by basic parameters.

   | Parameter              | Description                                                    |
   | ---------------------- | -------------------------------------------------------------- |
   | `seq_page_cost`        | Cost of a sequential page fetch                                |
   | `random_page_cost`     | Cost of a non-sequential page fetch                            |
   | `cpu_tuple_cost`       | Cost of typical CPU time to process a tuple                    |
   | `cpu_index_tuple_cost` | Cost of typical CPU time to process an index tuple             |
   | `cpu_operator_cost`    | Cost of CPU time to execute an operator or function            |
   | `parallel_tuple_cost`  | Cost of CPU time to pass a tuple from worker to leader backend |
   | `parallel_setup_cost`  | Cost of setting up shared memory for parallelism               |

2. PostgreSQL computes two separate costs for each path.

   - `Total Cost` : total estimated cost to fetch _all tuples_
   - `Startup Cost` : cost that is expended before _first tuple_ is fetched

   When a query includes `LIMIT` clause or an `EXISTS(...)` sub-select, it is not necessary to fetch all tuples of the path's result.
   Instead, the cost of fetching a partial result can be estimated by interpolating between the startup cost and the total cost.
   This involves calculating a weighted average based on the proportion of rows expected to be fetched.

   $$
   \small
   \text{Actual Cost} = \text{Startup Cost} +
   (\text{Total Cost} - \text{Startup Cost}) \times \frac{\text{Tuples to Fetch}}{\text{Path's Estimated Number of Result Tuples}}
   $$

   For example, consider a table scan query with a startup cost of 10 and a total cosst of 100. If the query includes `LIMIT 10`, the estimated cost is calculated as follows:

   $$
   \small
   10 + (100 - 10) \times \frac{10}{100} = 10 + (100 - 10) \times 0.10 = 19
   $$

   The `LIMIT` clause is applied as a top-level plan node. Therefore, row counts for base relations and intermediate nodes are calculated without considering the `LIMIT` clause.

3. Other commonly used parameters are:

- `qpqual_cost` : CPU cost of evaluating the `WHERE` clause.
- `tlist_eval_cost` : Cost of evaluating the target list expressions. The target list defines the result of the query. More details about target list can be found [here](https://www.postgresql.org/docs/current/querytree.html).

<!-- For simplication, this post assumes no parallelism is used, that is, only one worker is assigned. -->

If a operator is disabled (i.e., `enable_xxx` is not true), PostgreSQL adds a `disable_cost` (set to 1.0e10) to the startup cost. This effectively prevents the optimizer from selecting the operator.

## Single-Table Query

### [Sequential Scan](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c#L276)

Determines and returns the cost of _scanning a relation sequentially_.

The total cost of a sequential scan is calculated as follows:

$$
\small
\text{total_cost} = \text{startup_cost} + \text{cpu_run_cost} + \text{disk_run_cost}
$$

The startup cost includes the costs incurred before the first tuple is returned. It is calculated as:

$$
\small
\text{startup_cost} = \text{qpqual_startup_cost} + \text{tlist_eval_startup_cost}
$$

The CPU run cost is the cost of processing all tuples. It is calculated as:

$$
\small
\text{cpu_run_cost} = \text{cpu_per_tuple} \times N_{\text{tuples}} + \text{tlist_eval_per_row} \times N_{\text{output_rows}}
$$

If parallelism is used, the CPU cost is divided among all workers:

$$
\small
\text{cpu_run_cost} = \text{cpu_run_cost} \div \text{paraellel_divisor}
$$

The number of output rows is also adjusted for parallel processing:

$$
\small
N_{\text{output_rows}} = N_{\text{output_rows}} \div \text{paraellel_divisor}
$$

The disk run cost is the cost of reading the pages from disk:

$$
\small
\text{disk_run_cost} = \text{spc_seq_page_cost} \times N_{\text{pages}}
$$

### [Sample Scan](<(https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c#L353)>)

Determines and returns the cost of _scanning a relation using sampling_.

The total cost of a sample scan is calculated as follows:

$$
\small
\text{total_cost} = \text{startup_cost} + \text{run_cost}
$$

The startup cost includes the costs incurred before the first tuple is returned. It is calculated as:

$$
\small
\text{startup_cost} = \text{qpqual_startup_cost} + \text{tlist_eval_startup_cost}
$$

The run cost, which includes disk and CPU costs, is calculated as:

$$
\small
\text{run_cost} = \text{spc_page_cost} \times N_{\text{pages}} + \text{cpu_per_tuple} \times N_{\text{tuples}} + \text{tlist_eval_per_row} \times N_{\text{output_rows}}
$$

For disk costs, `spc_page_cost` is set to `spc_random_page_cost` if `NextSampleBlock` is used, indicating random access. Ohterwise, it is set to `spc_seq_page_cost`, indicating sequential access.
This is because the role of function `NextSampleBlock` is to return the block number of the next page to be scanned.
More details about table sampling methods can be found in [here](https://www.postgresql.org/docs/current/tablesample-support-functions.html).

$$ \small N\_{\text{pages}}$$ is the number of pages the sampling method will visit.

### [Gather](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c#L425)

$$
\small
\text{Total Cost} = \text{Startup Cost} + \text{Run Cost}
$$

$$
\small
\text{Run Cost} = \text{Subpath Total Cost} - \text{Subpath Startup Cost} + \text{Parallel Tuple Cost} \times N_{\text{rows}}
$$

### Gather Merge

$$
\small
\text{Total Cost} = \text{Startup Cost} + \text{Run Cost} + \text{Input Total Cost}
$$

$$
\small
\text{Startup Cost} = \text{Comparsion Cost} \times N \times \log N + \text{Parallel Setup Cost} + \text{Input Setup Cost}
$$

$$
\small
\text{Run Cost} = N*{\text{rows}} \times \text{Comparison Cost} \times \log N + \text{CPU Operator cost} \times N*{\text{rows}} + \text{Parallel Tuple Cost} \times N_{\text{rows}} \times 1.05
$$

### Index Scan
