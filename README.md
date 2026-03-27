# Q'thun

Q'thun (pronounced kuh-foon) is a Claude Code plugin that optimizes BigQuery SQL queries by iteratively improving them based on runtime and cost metrics.

## Installation

Add the marketplace and install the plugin:

```
/plugin marketplace add MaxHalford/qthun
/plugin install qthun@qthun
```

To update:

```
/plugin marketplace update qthun
/reload-plugins
```

## Usage

Invoke the skill with:

```
/qthun:bigquery --query <path-to-sql-file> [--budget <usd>] [--iterations <n>]
```

### What it does

1. Runs your original query and records its runtime and cost
2. Analyzes the execution plan to identify bottlenecks
3. Generates an optimized version and executes it
4. Verifies the optimized query returns identical results
5. Keeps the best version and repeats until budget or iterations are exhausted

Progress is tracked in `progress.csv` so optimization can resume across sessions.
