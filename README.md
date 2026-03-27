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

## Examples

<details>
  <summary><h3>BigQuery ~ 115s → 78s (redundant join + pre-aggregation)</h3></summary>

```sql
-- Before
SELECT
    s.tenant_id,
    t.title AS tag_name,
    t.value AS tag_value,
    s.step_level,
    c.name AS component,
    m.category AS material,
    s.step_name,
    s.country,
    s.facility,
    o.report_year,
    COUNT(DISTINCT s.entity_id) AS n_entities,
    SUM(o.units * mm.center) AS impact,
    SUM(o.units * (mm.upper - mm.lower)) AS uncertainty,
    SUM(o.units) AS total_units
FROM analytics.steps AS s
LEFT JOIN analytics.metrics AS mm ON (s.step_id = mm.step_id)
LEFT JOIN analytics.entities AS e ON (s.entity_id = e.entity_id)
LEFT JOIN analytics.components AS c ON (s.component_id = c.component_id)
LEFT JOIN analytics.materials AS m ON (s.material_id = m.material_id)
INNER JOIN analytics.orders AS o ON (e.entity_id = o.entity_id)
LEFT JOIN analytics.tags AS t ON (e.entity_id = t.entity_id)
WHERE mm.scope = 'STEP'
AND mm.indicator = 'PRIMARY'
GROUP BY ALL
```

- Q'thun spots out that joining with `analytics.entities` is not necessary, which saves ~15s
- It then applies a heuristic, which is to pre-aggregate figures before joining with `analytics.tags`

```sql
-- Optimized: removed redundant entities join, LEFT->INNER JOIN on metrics,
-- pre-aggregate before tags join to reduce fan-out (115.7s -> 77.9s)
WITH core AS (
    SELECT
        s.tenant_id,
        s.entity_id,
        s.step_level,
        c.name AS component,
        m.category AS material,
        s.step_name,
        s.country,
        s.facility,
        o.report_year,
        SUM(o.units * mm.center) AS impact,
        SUM(o.units * (mm.upper - mm.lower)) AS uncertainty,
        SUM(o.units) AS total_units
    FROM analytics.steps AS s
    INNER JOIN analytics.metrics AS mm ON (s.step_id = mm.step_id)
    LEFT JOIN analytics.components AS c ON (s.component_id = c.component_id)
    LEFT JOIN analytics.materials AS m ON (s.material_id = m.material_id)
    INNER JOIN analytics.orders AS o ON (s.entity_id = o.entity_id)
    WHERE mm.scope = 'STEP'
    AND mm.indicator = 'PRIMARY'
    GROUP BY ALL
)
SELECT
    core.tenant_id,
    t.title AS tag_name,
    t.value AS tag_value,
    core.step_level,
    core.component,
    core.material,
    core.step_name,
    core.country,
    core.facility,
    core.report_year,
    COUNT(DISTINCT core.entity_id) AS n_entities,
    SUM(core.impact) AS impact,
    SUM(core.uncertainty) AS uncertainty,
    SUM(core.total_units) AS total_units
FROM core
LEFT JOIN analytics.tags AS t ON (core.entity_id = t.entity_id)
GROUP BY ALL
```

</details>
