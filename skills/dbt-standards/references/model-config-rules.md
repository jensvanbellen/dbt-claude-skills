# Model Config Rules

Every SQL model must have a config block at the top. Minimum: the materialization.

## Minimum Config Block

```sql
{{ config(
    materialized = 'view'
) }}
```

## Full Config Block (All Options)

```sql
{{ config(
    materialized = 'incremental',
    schema = 'typeform',          -- Required if NOT in the default reporting schema
    tags = ['job_typeform'],      -- Required: at least one approved tag
    unique_key = 'response_id',   -- Required for incremental models
    event_time = 'submitted_at'   -- Required for models with 5+ min runtimes
) }}
```

## Rules by Field

### `materialized`
Always set. Default by layer:
- Staging → `'view'`
- Intermediate → `'view'`
- Marts → `'table'` (or `'incremental'` for high-volume append data)

### `schema`
- **Omit** for models that belong in the default reporting schema (dbt adds prefix automatically)
- **Required** for all other schemas: `schema = 'typeform'` → becomes `analytics_typeform` in the warehouse
- The `analytics_` prefix comes from `profiles.yml`, not the model config

### `tags`
- **Required** on every model
- Tags go in the **config block**, not in the YAML (except sources — those get tags in YAML)
- Every tag must exist in `.governance_rules/dbt_accepted_tags.md`
- If no existing tag fits, you must add a new approved tag to that file AND to `ci_cd/.pre-commit-config.yaml`

```sql
-- Correct: tags in config block
{{ config(tags = ['job_churned']) }}

-- Wrong: tags only in YAML for non-source models
```

#### Tag axes — these are orthogonal

Approved tags fall into separate axes (see `dbt_accepted_tags.md`): `domain_*`, `data_product_*`, `job_*`, plus legacy. A model normally carries **one tag per axis** — e.g. `['domain_hr', 'data_product_hr_insights', 'job_hr']`. `domain_*` and `data_product_*` are unlimited; the rule below applies only to the `job_*` axis.

#### One owning pipeline `job_*` tag per model

A model must have **exactly one *owning* pipeline `job_*` tag** — the job responsible for its scheduled build.

- `job_daily` is a **cadence** tag, not a pipeline tag. It is the one exception: it may coexist with a pipeline `job_*` tag.
- A second model needing to run inside another job should be pulled in via **dependency selection** (`tag:job_hr+`), **not** by adding a second pipeline `job_*` tag.

**Why:** a model carrying two pipeline `job_*` tags gets built by both jobs → redundant compute and concurrent writes to the same target table (race). It also forces the daily job's hand-maintained `--exclude tag:job_x ...` list to grow with every new job, or the model double-builds.

```sql
-- Correct: one owning pipeline job tag (+ orthogonal domain/data_product)
{{ config(tags = ['domain_hr', 'data_product_hr_insights', 'job_hr']) }}

-- Correct: cadence tag may coexist with one pipeline tag
{{ config(tags = ['job_daily', 'job_sap']) }}

-- Wrong: two pipeline job tags → double-build / race
{{ config(tags = ['job_hr', 'job_planning']) }}
```

**Sanctioned exception:** reverse-ETL fan-out, where one model legitimately feeds two sync jobs (e.g. `['job_reverse_etl_customers', 'job_reverse_etl_marketing_leads']`). Only use multiple pipeline tags when the jobs do **not** overlap in schedule (no concurrent write to the same table). When unsure, confirm with the data team before codifying.

### `unique_key`
- **Required** for all incremental models
- Use a single column when possible: `unique_key = 'response_id'`
- Use a composite key when no single column is unique:

```sql
{{ config(unique_key = ['received_on', 'subscription_asset_id']) }}
```

### `event_time`
- Use for models with 5+ minute runtime
- Must be a date/timestamp column
- Enables the `--sample` flag in CI: `dbt build --sample="7 days"`
- Required for the `ci_prd` job to work correctly on big models

```sql
{{ config(event_time = 'submitted_at') }}
```

### `cluster_by`
- Only for large `table` or `incremental` models
- Only when the table is consistently filtered or joined on that column downstream
- Date columns are the most common cluster key
- When using multiple keys, put the **most selective** column first

```sql
{{ config(
    materialized = 'table',
    cluster_by = ['created_at', 'country_code']
) }}
```

### `pre_hook` / `post_hook`
- Use a **post-hook to apply PII tags** after running a model
- See the `apply_pii_tags` macro for the pattern
- Other hooks: trigger temp tables (pre-hook) or cluster rebuilds (post-hook)

## Development-Only WHERE Clauses

For big models, add a dev filter to limit data during local testing:

```sql
{% if target.name is not none and 'dev' in target.name %}
    WHERE created_at >= DATEADD('year', -1, CURRENT_DATE())  -- noqa: LT02
{% endif %}
```

- Only triggers on dbt profiles with `dev` in the name (e.g., `dev-non-pii`)
- Use `-- noqa: LT02` when SQLFluff conflicts with Jinja if-statements
- **Always run with a dev profile** — never run big local builds against the PII prod target

## Incremental Model WHERE Clause

Incremental filter goes inside the model body, not the config:

```sql
{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

- If source data arrives late (outside the incremental window), it won't be picked up
- Logic changes don't fix historical rows — use `dbt build --full-refresh` via the _Run a Specific Model in PRD_ job

## YAML Config Block (Staging Only — for Tags on Sources)

For **source** tags, apply via the YAML `config` block, not in a SQL model:

```yaml
sources:
  - name: hubspot
    config:
      tags: ['job_hubspot_legacy']
```
