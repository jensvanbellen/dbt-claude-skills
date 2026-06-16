# dbt Reverse-ETL Model Pattern

## Directory & Naming

```
models/reverse_etl/consumer_direct/<destination>/<model_name>/
  <model_name>.sql
  _<model_name>.yml         ← underscore prefix, named after model
```

Example:
```
models/reverse_etl/consumer_direct/hubspot/hubspot_subscriptions/
  hubspot_subscriptions.sql
  _hubspot_subscriptions.yml
```

## Config Block

```sql
{{ config(
    materialized = 'table',
    tags = ['job_reverse_etl_customers']
) }}
```

- `materialized = 'table'`: required so Hightouch reads stable, already-computed data
- `materialized = 'view'` only for lower-priority syncs where freshness doesn't matter
- Tag must be a registered `job_reverse_etl_*` tag from `.governance_rules/dbt_accepted_tags.md`
- **Tags always go in the SQL config block** — never in `dbt_project.yml` or model YAML files

## Standard Model Structure

```sql
{{ config(
    materialized = 'table',
    tags = ['job_reverse_etl_customers']
) }}

WITH source_data AS (
    SELECT ...
    FROM {{ ref('upstream_mart_or_intermediate') }}
),

-- ... additional CTEs for enrichment ...

final AS (
    SELECT
        CURRENT_TIMESTAMP AS load_date_time,   -- Always include
        <unique_id_column>,                      -- Must match externalIdMapping in Hightouch
        <all_other_columns>
    FROM joined_data
    WHERE
        -- CRITICAL: Exclude records Hightouch cannot process
        <unique_id_column> IS NOT NULL
        -- Add destination-specific guards (see below)
)

SELECT * FROM final
```

## The Final WHERE Filter — Critical

The WHERE clause in the final CTE is what keeps bad data out of the destination. Missing or wrong filters cause Hightouch errors on every sync run.

**Always filter:**
- Records with NULL on the `externalIdMapping` column (the unique ID)
- Records where the destination contact association lookup would fail
- Records that fail the destination's required field validation

**Comment your filter logic.** These filters evolve over time and the reasons are non-obvious. Future maintainers (or the AI) need to understand why each condition exists before changing it.

## YAML — Contract + Tests

```yaml
version: 2

models:
  - name: hubspot_subscriptions
    config:
      contract:
        enforced: true
    description: '{{ doc("table_hubspot_subscriptions") }}'
    columns:
      - name: load_date_time
        data_type: timestamp_ltz
        description: '{{ doc("col_load_date_time") }}'

      # Critical columns: both constraint (warehouse DDL) AND data_test (dbt)
      - name: customer_number
        data_type: number(38,0)
        description: '{{ doc("col_customer_number") }}'
        data_tests:
          - not_null
        constraints:
          - type: not_null

      # Non-critical columns: data_type declaration only (contract requirement)
      - name: subscription_type_name
        data_type: varchar
        description: '{{ doc("col_subscription_type_name") }}'
```

### Rules for the YAML

- `contract: enforced: true` is required on all reverse-ETL models
- Every column in the SQL must be listed in the YAML with a `data_type`
- Every column in the YAML must exist in the SQL output
- Critical columns (required for destination association or sync to work) get **both** `constraints` and `data_tests`
- Non-critical columns get `data_type` only

### Warehouse Data Types for Contracts

| Use case | data_type |
|---|---|
| IDs, names, statuses, strings | `varchar` |
| Integer IDs | `number(38,0)` |
| Decimals (money) | `number(18,2)` |
| Timestamps (most) | `timestamp_ntz` |
| Timestamps with timezone | `timestamp_ltz` or `timestamp_tz` |
| Dates | `date` |
| Booleans | `boolean` |

## Surrogate Keys

When the mart has no natural primary key, use `dbt_utils.generate_surrogate_key()`:

```sql
{{ dbt_utils.generate_surrogate_key(['subscription_id', 'deal_type', 'created_on']) }} AS deal_id
```

This generates a deterministic hash-based key. Use it as the `externalIdMapping` in the Hightouch sync.

## `load_date_time`

Always include `CURRENT_TIMESTAMP AS load_date_time` as the first column in the SELECT. It records when the row was last synced and is useful for debugging in the destination system.

## Running the Model Locally

```bash
# Build and test
dbt build --select hubspot_subscriptions

# Check row count before/after changes
dbt show --select hubspot_subscriptions --limit 5

# Compile only (catches contract violations without running)
dbt compile --select hubspot_subscriptions
```
