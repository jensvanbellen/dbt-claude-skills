---
name: dbt-reverse-etl
description: Guides building and modifying reverse-ETL pipelines at your company — dbt model structure, data contracts, Hightouch sync YAML, and Airflow orchestration. Use when adding or changing a reverse-ETL model that syncs data to HubSpot (or another destination) via Hightouch.
allowed-tools: "Read, Write, Edit, Glob, Grep, Bash(git *)"
user-invocable: true
metadata:
  author: your-org
---

# Reverse-ETL

Reverse-ETL pushes data **from your warehouse → dbt mart → Hightouch → destination** (e.g., HubSpot). Changes here affect live destination data and customer-facing workflows — treat them with extra caution.

**The full stack for a reverse-ETL change spans two repos:**
- `your_dbt_project` — the mart model that shapes the data
- `reverse-etl_hightouch` — the Hightouch sync YAML that maps columns to destination fields

Changes that span both repos need **coordinated PRs** (see [references/multi-repo-workflow.md](references/multi-repo-workflow.md)).

## Reference Guides

| Guide | When to read |
|---|---|
| [references/dbt-model-pattern.md](references/dbt-model-pattern.md) | Building or modifying the dbt mart model |
| [references/hightouch-sync-yaml.md](references/hightouch-sync-yaml.md) | Adding/changing column mappings in the Hightouch sync |
| [references/airflow-orchestration.md](references/airflow-orchestration.md) | Wiring up or changing the Airflow DAG |
| [references/multi-repo-workflow.md](references/multi-repo-workflow.md) | Coordinating changes across dbt + Hightouch repos |

## Step-by-Step Workflow

### Step 1 — Check blast radius before touching anything

Run the `mc-agent-toolkit:prevent` skill first. It surfaces:
- Active Monte Carlo alerts on the model
- Downstream dependencies in the DAG
- How many destination records will be affected

**Do not modify the model until you understand the blast radius.**

### Step 2 — Modify the dbt model

Read [references/dbt-model-pattern.md](references/dbt-model-pattern.md) for the full pattern.

Key rules:
- Model lives in `models/reverse_etl/consumer_direct/<destination>/<model_name>/`
- Materialized as `table` (usually) — data must be stable for Hightouch to read
- Tag with `job_reverse_etl_*` tag in the **SQL config block** — never in `dbt_project.yml` or YAML (check `.governance_rules/dbt_accepted_tags.md`)
- Contract must be `enforced: true` — every column needs `data_type` in YAML
- Include `CURRENT_TIMESTAMP AS load_date_time` in every reverse-ETL model
- The final WHERE filter is critical — bad filter logic = bad data in the destination

### Step 3 — Update the YAML (contract first, then tests)

After changing the SQL:
1. Update the YAML with any added/removed columns — the contract will fail build if out of sync
2. Every new column needs a `data_type` declaration
3. Add `constraints: - type: not_null` for columns required by the destination
4. Add `data_tests: - not_null` as the dbt-side check (belt-and-suspenders)
5. Run `dbt compile` locally to catch contract violations before running

### Step 4 — Update the Hightouch sync YAML

Read [references/hightouch-sync-yaml.md](references/hightouch-sync-yaml.md) for the mapping format.

If you added a column to the dbt model and want it synced:
- Add a new entry to `config.mappings` in the sync YAML
- `from:` uses the **UPPER_CASE** warehouse column name
- `to:` is the destination field API name (check destination property settings)

### Step 5 — Test end-to-end

1. Run the dbt model with `dbt build --select <model_name>`
2. Check that the contract passes, tests pass, row count looks right
3. Trigger the Hightouch sync manually (via the Hightouch UI or CLI)
4. Verify a sample of records in the destination look correct

### Step 6 — Airflow (only if scheduling changes)

The Airflow DAG pattern is: `Fivetran → DbtCloudRunJobOperator → HightouchTriggerSyncOperator`

Only update Airflow if you're:
- Changing the job schedule
- Adding a new sync trigger after a dbt job
- Changing upstream dependencies

See [references/airflow-orchestration.md](references/airflow-orchestration.md).

## Hard Rules

- **Never remove a column from a contract model** without first checking if Hightouch is mapping it — a deleted column breaks the sync immediately at next run
- **Never rename a column** without updating the Hightouch sync mapping in the same PR
- **Always include a final WHERE filter** to exclude records Hightouch can't process (null ID, no destination contact association, etc.)
- **No PII columns** in reverse-ETL models unless explicitly required by the destination — check with the team first
- **`schedule: type: manual` in all Hightouch sync YAMLs** — Airflow is the trigger, never Hightouch's built-in scheduler
