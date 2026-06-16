# dbt Tests vs Monte Carlo: Division of Responsibility

## The Core Principle

These tools are **complementary**, not overlapping. Using both creates defense-in-depth.

| | dbt Tests | Monte Carlo |
|---|---|---|
| **Trigger** | Runs only when `dbt build` or `dbt test` executes | Runs continuously, 24/7 |
| **Authored by** | You, as explicit assertions | Auto-learned from data patterns + custom rules |
| **Scope** | SQL output from your models | Raw source tables AND reporting mart tables |
| **Example catch** | "After running `int_typeform_csat`, `ticket_id` is never null" | "Fivetran loaded 40% fewer rows at 3am" |

## What Monte Carlo Covers (Don't Duplicate in dbt)

- **Row count anomalies** on source tables (e.g., Fivetran suddenly loads 50% fewer rows)
- **Freshness monitoring** on source tables (e.g., a source hasn't been ingested in 6 hours)
- **Schema changes** on sources (e.g., a column was dropped or renamed in the raw data)
- **Field-level anomalies** on reporting marts (e.g., avg order value dropped by 80%)
- **Null rate changes** on mart columns over time

These are statistical — you don't know the threshold upfront. Monte Carlo learns it.

## What dbt Tests Cover (Don't Replace with Monte Carlo)

- **Primary key integrity**: `unique` + `not_null` — these are facts about your model's design
- **Referential integrity**: `relationships` — FK validity isn't statistical, it's a hard rule
- **Accepted values**: valid status codes — these come from your business requirements
- **Business logic correctness**: `end_date > start_date`, `amount >= 0` — explicit rules
- **Grain enforcement**: `unique_combination_of_columns` — your model's intended grain

These are assertions you author because you know the rule. Monte Carlo can't infer them.

## Where Each Tool Applies by Layer

| Layer | dbt Tests | Monte Carlo |
|---|---|---|
| Sources (raw) | None (1-to-1 with staging, tests live in staging) | Default monitors (freshness, row count, schema) |
| Staging | PK tests, accepted_values, relationships | Monitors inherited from source |
| Intermediate | Business logic, grain enforcement | Not typically monitored |
| Marts | Basic tests + error severity on critical fields | Custom field-level monitors |
| Reverse-ETL marts | Contracts + constraints + error tests | Custom monitors |

## When a New Mart Goes to Production

When you create a new mart:
1. Monte Carlo auto-discovers it and applies **default monitors** (freshness, row count, schema)
2. For business-critical marts: add **custom Monte Carlo monitors** via the `mc-agent-toolkit:monitor-creation` skill
3. Assign ownership in your data catalog — run this through the data governance committee
4. Set descriptions and owners in the catalog after the model is stable

## Adding Custom Monte Carlo Monitors

For new marts, use the `mc-agent-toolkit:monitor-creation` skill to create monitors-as-code. Typical monitors to add:
- Row count threshold (e.g., "always at least 10,000 rows")
- Field-level null rate on critical columns
- Custom SQL monitor for business-specific rules Monte Carlo can't infer

These get version-controlled and can be improved by AI over time.

## The Alert Routing Rule

When an alert fires:

| Alert Source | First Look |
|---|---|
| Monte Carlo on source table | Check Fivetran sync status |
| Monte Carlo on mart | Check if upstream dbt job ran successfully |
| dbt test failure in CI | Check your model SQL logic |
| dbt test failure in production | Check if source data changed (new values, null influx) |

The `mc-agent-toolkit:prevent` skill surfaces Monte Carlo context automatically before you touch a model — use it to understand the blast radius before making changes.
