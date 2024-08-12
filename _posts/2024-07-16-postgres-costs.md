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

## [Sequential Scan](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c#L276)

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

`cpu_per_tuple` is the sum of `cpu_tuple_cost` and `qpqual_cost` per tuple.

If parallelism is used, the CPU cost and the number of output row is divided among all workers:

$$
\small
\text{cpu_run_cost} = \frac{\text{cpu_run_cost}}{\text{parallel_divisor}} \text{, } N_{\text{output_rows}} = \frac{N_{\text{output_rows}}}{\text{parallel_divisor}}
$$

The disk run cost is the cost of reading the pages from disk:

$$
\small
\text{disk_run_cost} = \text{spc_seq_page_cost} \times N_{\text{pages}}
$$

## [Sample Scan](<(https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c#L353)>)

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

## [Gather or Gather Merge](https://www.postgresql.org/docs/current/how-parallel-query-works.html)

When the optimizer determines that parallel query is the fastest execution strategy for a particular query, it will create a query plan that includes a `Gather` or `Gather Merge` node.

### [Gather](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c#L425)

Determines and returns the cost of _gather_ path.

The total cost of a gather is calculated as follows:

$$
\small
\text{total_cost} = \text{startup_cost} + \text{run_cost}
$$

The startup cost includes the costs incurred before the first tuple is returned. It is calculated as:

$$
\small
\text{startup_cost} = \text{subpath_startup_cost} + \text{parallel_setup_cost}
$$

The run cost includes the cost of processing all tuples and the additional cost of parallel tuple communication. It is calculated as:

$$
\small
\text{run_cost} = \text{subpath_total_cost} - \text{subpath_startup_cost} + \text{parallel_tuple_cost} \times N_{\text{rows}}
$$

### [Gather Merge](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c#L464)

Determines and returns the cost of _gather merge_ path. A `Gather Merge` merges several pre-sorted input streams using a heap that at any given instant holds the next tuple from each stream. If there are $ \small N$ streams, about $ \small N \times \log_2 N$ tuple comparisons are needed to construct the heap at startup, and for each output tuple, about $ \small \log_2 N$ comparisons are needed to replace the top heap entry with the next tuple from the same stream.

The total cost of a gather merge is calculated as follows:

$$
\small
\text{total_cost} = \text{startup_cost} + \text{run_cost} + \text{input_total_cost}
$$

The startup cost includes the costs of heap creation and parallel setup cost. It is calculated as:

$$
\small
\text{startup_cost} = \text{comparison_cost} \times N \times \log_2 N + \text{parallel_setup_cost}
$$

$ \small N$ is the number of workers plus one to account for the leader.

The run cost includes the cost of maintaining the heap per tuple, a small cost for heap management, and the communication cost. It is calculated as:

$$
\small
\text{run_cost} = \text{rows} \times \text{comparison_cost} \times \log_2 N + \text{cpu_operator_cost} \times \text{rows} + \text{parallel_tuple_cost} \times N_{\text{rows}} * 1.05
$$

Extra 5% added to the communication cost because `Gather Merge` requires blocking until a tuple is available from every worker.

## [Index Scan](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c#L530)

Determines and returns the cost of _scanning a relation using an index_.

In cost estimation of index scans, the `amcostestimate` function is used. This function estimates the processing cost for scanning the index, as well as the selectivity of the index. It returns several values: `indexStartupCost`, `indexTotalCost`, `indexSelectivity`, `indexCorrelation`, and `index_pages`.

The total cost of a index scan is calculated as follows:

$$
\small
\text{total_cost} = \text{startup_cost} + \text{run_cost}
$$

The startup cost includes the initial costs of scanning the index, evaluating the `WHERE` clause, and processing the target list expressions. It is calculated as:

$$
\small
\text{startup_cost} = \text{indexStartupCost} + \text{qpqual_startup_cost} + \text{tlist_eval_startup_cost}
$$

The run cost includes both disk I/O costs and CPU costs.

First, to estimate the number of main-table pages fetched and compute the I/O cost, PostgreSQL set `max_IO_cost` and `min_IO_cost`.

- `max_IO_cost`: Uncorrelated Index Ordering with Table Ordering

  To compute $\small N_\text{pages_fetched}$, [approximation method by Mackert and Lohman](https://dl.acm.org/doi/pdf/10.1145/68012.68016) is used, which is defined in [index_pages_fetched] function(https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c#L859). If it's an index-only scan, PostgreSQL use the measured fraction of the entire heap that is all-visible. Therefore, $\small N_\text{pages_fetched}$ is $\small N_\text{pages_fetched} = \lceil N_\text{pages_fetched} \times (1.0 - \text{allvisfrac}) \rceil$. Then, `spc_random_page_cost` is charged per page fetched.

  $$
  \small
  \text{max_IO_cost} = N_\text{pages_fetched} \times \text{spc_random_page_cost}
  $$

  If `loop_count` > 1:

  $$
  \small
  \text{max_IO_cost} = \frac{N_\text{pages_fetched} \times \text{spc_random_page_cost}}{\text{loop_count}}
  $$

- `min_IO_cost`: Exactly Correlated Index Ordering with Table Ordering:

  $\small N_\text{pages_fetched}$ is exactly $\small \text{indexSelectivity} \times \text{table_size}$. If it's an index-only scan, PostgreSQL use the measured fraction of the entire heap that is all-visible. Therefore, $\small N_\text{pages_fetched}$ is $\small N_\text{pages_fetched} = \lceil N_\text{pages_fetched} \times (1.0 - \text{allvisfrac}) \rceil$.

  In the case of `loop_count` > 1, PostgreSQL assumes all the fetches are random.

  $$
  \small
  \text{min_IO_cost} = \frac{N_\text{pages_fetched} \times \text{spc_random_page_cost}}{\text{loop_count}}
  $$

  Else, PostgreSQL considers three cases:

  - If `pages_fetched` > 1:

    $$
    \small
    \text{min_IO_cost} = \text{spc_random_page_cost} + (N_\text{pages_fetched} - 1) \times \text{spc_seq_page_cost}
    $$

  - If `pages_fetched` = 1:

    $$
    \small
    \text{min_IO_cost} = \text{spc_random_page_cost}
    $$

  - If `pages_fetched` < 0:

    $$
    \small
    \text{min_IO_cost} = 0
    $$

`loop_count` is the number of repetitions of the indexscan to factor into estimates of caching behavior.

Then, PostgreSQL interpolates based on estimated index order correlation.

$$
\small
\text{disk_run_cost} = \text{max_IO_cost} + \text{indexCorrelation}^2 \times (\text{min_IO_cost} - \text{max_IO_cost})
$$

Second, CPU run cost is calculated as:

$$
\small
\text{cpu_run_cost} = \text{cpu_per_tuple} \times N_{\text{tuples}} + \text{tlist_eval_per_row} \times N_{\text{output_rows}}
$$

If parallelism is used, the CPU cost and the number of output row is divided among all workers:

$$
\small
\text{cpu_run_cost} = \frac{\text{cpu_run_cost}}{\text{parallel_divisor}}
$$

Finally, run cost is calculated as:

$$
\small
\text{run_cost} = \text{indexTotalCost} - \text{indexStartupCost} + \text{disk_run_cost} + \text{cpu_run_cost}
$$

## [Bitmap Heap Scan](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c#L998)

Determines and returns the cost of _scanning a relation using a bitmap index-then-heap plan_.

In cost estimation of bitmap heap scans, the `compute_bitmap_pages` function is used. This function computes the number of pages fetched from heap in bitmap heap scan. It returns several values: `indexTotalCost`, `tuples_fetched`, and `pages_fetched`.

The total cost of a bitmap heap scan is calculated as follows:

$$
\small
\text{total_cost} = \text{startup_cost} + \text{run_cost}
$$

The startup cost includes the cost of obtaining the bitmap, evaluating the `WHERE` clause, and processing the target list expressions. It is calculated as:

$$
\small
\text{startup_cost} = \text{indexTotalCost} + \text{qpqual_startup_cost} + \text{tlist_eval_startup_cost}
$$

## [Nested Loop Join]()

## [Merge Join]()

## [Hash Join]()
