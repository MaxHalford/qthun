# Q'thun

Q'thun (pronounced kuh-foon) is a Claude Code plugin that optimizes BigQuery SQL queries by iteratively improving them based on runtime and cost metrics.

What it does:

1. Runs your original query and records its runtime and cost
2. Analyzes the query + execution plan to identify bottlenecks
3. Suggests an optimization version and executes it
4. Verifies the optimized query returns identical results
5. Keeps the best version and repeats until budget or iterations are exhausted

Progress is tracked in `progress.csv` so optimization can resume across sessions.

## Installation

Add the marketplace and install the plugin:

```sh
/plugin marketplace add MaxHalford/qthun
/plugin install qthun@qthun
```

To update:

```sh
/plugin marketplace update qthun
/reload-plugins
```

You can also [configure auto-updates](https://code.claude.com/docs/en/discover-plugins#configure-auto-updates).

## Usage

Invoke the skill from Claude Code with:

```sh
/qthun:bigquery --query <path-to-sql-file> [--budget <usd>] [--iterations <n>]
```

## Success stories

<details>
  <summary><h3>BigQuery ~ 115s -> 78s (redundant join + pre-aggregation)</h3></summary>
  Quelque chose d'assez discret pour passer inaperçu.
</details>
