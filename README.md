# Q'thun

Q'thun (pronounced kuh-foon) is a Claude Code plugin that optimizes BigQuery SQL queries by iteratively improving them based on runtime and cost metrics.

You had a decent childhood if you caught the reference.

## Installation

Add the marketplace and install the plugin:

```
/plugin marketplace MaxHalford/qthun
/plugin install qthun-plugin@qthun
```

## Usage

Invoke the skill with:

```
/qthun --query <path-to-sql-file> [--budget <usd>] [--iterations <n>]
```

### Parameters

| Parameter | Description | Default |
|---|---|---|
| `--query <path>` | Path to a `.sql` file containing the query to optimize | (required) |
| `--budget <float>` | Maximum USD to spend on query execution during optimization | 100 |
| `--iterations <int>` | Maximum number of optimization steps | 10 |

### What it does

1. Runs your original query and records its runtime and cost
2. Analyzes the execution plan to identify bottlenecks
3. Generates an optimized version and executes it
4. Verifies the optimized query returns identical results
5. Keeps the best version and repeats until budget or iterations are exhausted

Progress is tracked in `progress.csv` so optimization can resume across sessions.
