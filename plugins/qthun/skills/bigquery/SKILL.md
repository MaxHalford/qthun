---
name: bigquery
description: Optimize BigQuery SQL query through iterative improvement
---

# Query tuning

You are a program who's mission is to optimize a BigQuery SQL query. Here are the steps you should follow:

1. Parse the input parameters.
2. If `progress.csv` does not exist, this means this is the first step of the optimization process. In this case:
  - Create a BigQuery session for the optimization run.
  - Run the original query and record its runtime and cost in `progress.csv` as the first entry.
  - Save the query's execution plan to `<query_name>.plan.json`.
  - Compute and save identifying statistics to `<query_name>.stats.json` by querying the results destination table.
3. Open the `progress.csv` file to:
  - Determine what improvements have already been made to the query.
  - Determine how much budget is available.
  - Determine how many optimization steps are remaining.
4. If the budget is exhausted or the maximum number of steps has been reached, stop.
5. Otherwise, analyze the current query and its execution plan to identify potential optimizations.
6. Generate a new optimized query based on the analysis, and execute it.
7. Check that the optimized query returns the same results as the original query by comparing their identifying statistics (computed from the results destination table). If they don't match, discard the optimized query and keep the previous best version.
8. Extract the runtime and cost of the optimized query, and compare it to the previous best version. If the optimized query is better, keep it as the new best version. If not, discard it and keep the previous best version.
9. Save state:
  - Append runtime and cost in `progress.csv`.
  - Save the optimized query's execution plan to `<query_name>_step<step_number>.plan.json`.
10. Repeat from step 3.

## Parameters

```
--query <path>: Path to a .sql file containing the query to optimize. The file name (without extension) is used as the query name for job IDs and artifact files.
--budget <float>: The maximum amount (in USD) to spend on query execution during the optimization process. Default is 100 USD.
--iterations <int>: The maximum number of optimization steps to perform. Default is 10.
```

## Progress

Optimization progress is tracked in a CSV file called `progress.csv`. Each row in the file represents a single optimization step, and contains the following columns:

- `step`: The step number of the optimization process.
- `runtime`: The runtime of the optimized query in this step, measured in seconds.
- `cost`: The cost of the optimized query in this step, estimated in USD.
- `description`: A brief description of the optimization that was performed in this step.
- `query`: The SQL query after the optimization step was applied.

## BigQuery client

All queries are run as **asynchronous jobs** using `bq query --nosync`, and results are written to **session-scoped temp tables** using `CREATE TEMP TABLE ... AS`. This avoids "response too large" errors (which occur even with `--nosync` for large result sets) and gives us control over job IDs and budget limits.

All queries within an optimization run share a **BigQuery session**. Results stored as temp tables can be queried directly for computing statistics without re-running the original query.

### Session management

Create a session at the start of an optimization run:

```sh
SESSION_ID=$(bq query --nouse_legacy_sql --format=json --create_session \
  --location=<DATA_LOCATION> \
  --label=qthun:session \
  'SELECT 1' | grep 'In session:' | awk '{print $3}')
```

The `--location` must match the location of the dataset being queried (e.g., `EU`, `US`).

Pass the session ID to all subsequent queries using `--session_id`:

```sh
--session_id=$SESSION_ID
```

Benefits of using a session:
- Each query's results are stored in a session-scoped destination table that can be queried for stats.
- Temp tables created in one step are accessible in later steps.
- Session state (e.g., variables, temp tables) persists across queries.

### Running a query

Derive the query name from the file path (e.g., `process_steps_impact.sql` → `process_steps_impact`). Use it to build predictable job IDs.

Wrap the query in `CREATE TEMP TABLE ... AS` to store results in a named session temp table. This avoids "response too large" errors and makes results easily queryable for stats.

```sh
QUERY_NAME="process_steps_impact"
STEP=0

bq query \
  --nouse_legacy_sql \
  --nosync \
  --job_id="qthun_${QUERY_NAME}_step${STEP}" \
  --label=qthun:${QUERY_NAME} \
  --maximum_bytes_billed=<remaining_budget_in_bytes> \
  --session_id=$SESSION_ID \
  "CREATE OR REPLACE TEMP TABLE qthun_step${STEP} AS $(cat ${QUERY_NAME}.sql)"
```

Then poll for completion:

```sh
bq wait "qthun_${QUERY_NAME}_step${STEP}"
```

#### Flag reference

| Flag | Purpose |
|---|---|
| `--nosync` | Submit asynchronously, return immediately. Avoids "response too large" errors. |
| `--job_id` | Predictable ID: `qthun_<query_name>_step<N>`. Makes jobs easy to reference later. |
| `--label` | Tag as `qthun:<query_name>` for cost tracking in billing exports. |
| `--maximum_bytes_billed` | Fail the query if it would scan more than this many bytes. Use to enforce the remaining budget. Convert USD to bytes: `budget_usd / 6.25 * (1024^4)`. |
| `--session_id` | Attach to the session. Results are stored in session-scoped destination tables. |

### Retrieving cost and runtime

After the job completes, use `bq show` to get execution statistics:

```sh
bq show --format=json -j "qthun_${QUERY_NAME}_step${STEP}"
```

Extract the relevant fields from the JSON output:

- **Runtime**: `statistics.endTime - statistics.startTime` (both in milliseconds since epoch). Compute the difference and convert to seconds.
- **Bytes processed** (cost proxy): `statistics.query.totalBytesBilled`. BigQuery on-demand pricing is based on bytes billed. To estimate cost in USD: `totalBytesBilled / (1024^4) * 6.25` ($6.25 per TiB).

### Reading query results

Since results are stored in named temp tables (`qthun_step0`, `qthun_step1`, etc.), you can query them directly by name within the session:

```sh
# Sample rows
bq query --nouse_legacy_sql --format=json \
  --session_id=$SESSION_ID \
  "SELECT * FROM qthun_step${STEP} LIMIT 10"

# Compute stats
bq query --nouse_legacy_sql --format=json \
  --session_id=$SESSION_ID \
  "SELECT COUNT(*) as row_count, ... FROM qthun_step${STEP}"
```

### Retrieving query execution plan

The execution plan is included in the job metadata returned by `bq show`. From the JSON output of `bq show --format=json -j <job_id>`, extract the `statistics.query.queryPlan` field. This is an array of query stages, each containing:

- `name`: Stage name.
- `id`: Stage ID.
- `inputStages`: IDs of stages that feed into this one.
- `startMs` / `endMs`: Timing for the stage.
- `waitMsAvg` / `waitMsMax`: Time spent waiting for slots.
- `readMsAvg` / `readMsMax`: Time spent reading input.
- `computeMsAvg` / `computeMsMax`: Time spent in computation.
- `writeMsAvg` / `writeMsMax`: Time spent writing output.
- `recordsRead` / `recordsWritten`: Row counts in and out.
- `shuffleOutputBytes` / `shuffleOutputBytesSpilled`: Data shuffled between stages (spill indicates memory pressure).
- `steps`: Array of sub-operations within the stage (e.g., `READ`, `COMPUTE`, `WRITE`, `AGGREGATE`), each with a `substeps` description.

Save the full `queryPlan` array to `<query_name>.plan.json` (or `<query_name>_step<N>.plan.json` for optimization steps). Use the plan to identify bottlenecks such as large shuffles, data skew, spills to disk, or stages with disproportionately high compute time.

## Identifying statistics

If we edit a query to optimize it, it's important to not change its behavior. To ensure this, we compute identifying statistics by querying the **session-scoped destination table** of each step's results (see "Reading query results" above). This should include:

- Row count
- Schema (column names and types)
- Sample of data (e.g. first 10 rows)
- Column sketches
  - For numeric columns: min, max, mean, stddev
  - For categorical columns: top 10 most common values and their counts

These statistics are to be saved in a `<query_name>.stats.json` file. After each optimization step, we should compare the statistics of the optimized query to the original query to ensure they match. If they don't match, this indicates that the optimization has changed the behavior of the query, which is not acceptable.

## Query tuning techniques

### WITH clause deduplication

WITH clauses in BigQuery are **not materialized** — they act like macros and are inlined everywhere they're referenced. If a CTE is referenced multiple times, this causes duplicate execution of the same stages. When a multiply-referenced CTE is expensive, replace it with a `CREATE TEMP TABLE` to materialize it once.

### WHERE clause ordering

BigQuery may not reorder WHERE expressions. Place the **most selective** condition first in AND clauses to enable short-circuiting. For OR clauses, place the **least selective** first. Prefer simple equalities on BOOL/INT/FLOAT/DATE columns before expensive operations like LIKE or REGEXP. Replace `REGEXP_CONTAINS` with `LIKE` when a simple pattern suffices.

### Filter by native column types

Avoid casting columns in WHERE clauses (e.g., `CAST(date_col AS STRING)`). Filtering on native types enables partition and cluster pruning at the storage level, rather than requiring slot workers to do the filtering.

### Reduce data before JOINs

Perform aggregations or filtering in subqueries **before** joining, so that the join operates on fewer rows. This reduces shuffle volume. Especially important when join keys are non-unique on both sides, which causes row explosion.

### Join ordering by table size

Place the **largest table first** in the FROM clause, followed by the smallest, then by decreasing size. BigQuery's optimizer only auto-reorders under specific conditions, so manual ordering is recommended.

### Filter both sides of JOINs

Push WHERE filters to apply to both sides of a join, not just one. Check the execution plan to confirm filters are applied as early as possible. If not, use subqueries to pre-filter each table before joining.

### Avoid functions on JOIN columns

Wrapping join columns in functions (e.g., `TRIM()`, `LOWER()`) prevents BigQuery from performing join optimizations like snowflake join optimization. Clean data during ingestion/ELT instead. This alone can yield 90%+ improvement in slot time.

### Replace self-joins with window functions

Self-joins used for row-dependent relationships (e.g., comparing a row to its previous row) can square the output size. Replace with window functions like `LAG()`, `LEAD()`, `ROW_NUMBER()`, etc.

### Aggregation placement

Aggregate as **late** and seldom as possible, since aggregation is costly. **Exception**: if early aggregation drastically reduces the row count before a join, aggregate early. This only works when both sides of the join are already at the same granularity (one row per join key value).

### Clustering keys in join conditions

If tables are clustered by a shared key (e.g., `account_slug`), consider adding that key to join conditions. This can enable co-located joins and reduce shuffles — but test both with and without, as the extra column in the join/shuffle key can increase bytes billed and runtime in some cases.

When adding clustering keys to joins, **place the clustering key first** in the `ON` clause (e.g., `ON a.account_slug = b.account_slug AND a.id = b.id`). In testing, this ordering was ~10% faster than placing it last, likely because BigQuery uses the first condition to guide partition/cluster pruning.

Note: BigQuery may already leverage clustering keys at the scan level even without them in join conditions. Adding them to joins is not always beneficial — always compare runtime and bytes billed before and after.

### Other

Feel free to come up with your own optimization techniques, given the execution plan and statistics.
