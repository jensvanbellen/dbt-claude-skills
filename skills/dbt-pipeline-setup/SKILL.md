---
name: dbt-pipeline-setup
description: Guides the end-to-end workflow for setting up a new dbt pipeline at your company — from model creation to dbt_jobs.yml configuration to Airflow scheduling. Use when a user wants to add a new model to a job, create a new dbt job, or wire up a pipeline with scheduling.
allowed-tools: "Read, Write, Edit, Glob, Grep, Bash(git *)"
user-invocable: true
metadata:
  author: your-org
---

# dbt Pipeline Setup

Setting up a new pipeline spans three systems: `your_dbt_project` (models + jobs config), and `airflow` (scheduling). This skill walks you through every decision.

## Reference Guides

| Guide | When to read |
|---|---|
| [references/jobs-config-workflow.md](references/jobs-config-workflow.md) | Configuring dbt_jobs.yml and tags |
| [references/airflow-orchestration.md](references/airflow-orchestration.md) | Airflow scheduling and cross-dependency chains |

## Step-by-Step Pipeline Workflow

### Step 1 — Build the Model + YAML

Use the `dbt:using-dbt-for-analytics-engineering` skill for the actual model building. Come back here once the model SQL is working.

Apply all `dbt-standards` rules:
- Correct naming prefix for the layer
- Config block with `materialized`, `schema` (if not the default reporting schema), `unique_key` (if incremental), `event_time` (if 5+ min runtime)
- YAML with column descriptions

### Step 2 — Add a Tag

Every model needs at least one approved tag. Tags connect models to jobs.

1. Check `.governance_rules/dbt_accepted_tags.md` in the repo for the approved list
2. Pick the tag that fits the job this model belongs to (e.g., `job_typeform`, `job_churned`)
3. Add to the config block: `tags = ['job_typeform']`
4. If no existing tag fits → you're creating a new job (see Step 3b)

### Step 3a — Existing Job: Add to It

If your model fits an existing job's scope, you're done after tagging. The job already selects by that tag.

Verify by checking `dbt_jobs.yml` — look for the job entry and its `--select` or tag filter.

### Step 3b — New Job: Create It

Read [references/jobs-config-workflow.md](references/jobs-config-workflow.md) for the full format.

Steps to create a new job:
1. Add the new tag to `.governance_rules/dbt_accepted_tags.md`
2. Add the new tag to the pre-commit config in `ci_cd/.pre-commit-config.yaml`
3. Add a new job entry in `dbt_jobs.yml`
4. Open a PR — **the job goes live in dbt Cloud only after merging**

### Step 4 — Choose `dbt build` vs `dbt run`

| Command | Use when |
|---|---|
| `dbt build` | Always preferred — runs models + tests + seeds + snapshots in correct order |
| `dbt run` | Only for jobs where you explicitly don't want tests to block downstream models |
| `dbt test` | Standalone test runs after data is already refreshed |

**Note on `dbt build`:** A test failure skips all downstream models in the same job. Models in other jobs run normally. Use `severity: warn` for known non-blocking issues so they don't cascade.

### Step 5 — Airflow Scheduling

Read [references/airflow-orchestration.md](references/airflow-orchestration.md) for full details.

Key rules:
- Airflow is the **single source of schedule truth** — dbt_jobs.yml does not control when jobs run
- If your job depends on an upstream source refresh (Fivetran), schedule Airflow so Fivetran runs first
- If your job feeds a downstream sync (Hightouch), chain that after the dbt job
- Schedule changes go in the Airflow repo (separate from `your_dbt_project`)

### Step 6 — Performance Setup for Large Models

For models with 5+ minute runtimes:

1. Add `event_time = '<timestamp_column>'` to the config block
2. Add a dev-only WHERE clause using the same timestamp
3. These enable `--sample="7 days"` in CI runs to avoid scanning full history

For marts that will be queried heavily on a date range:
- Consider `cluster_by = ['<date_column>']` on `table`/`incremental` models
- Only worthwhile for genuinely large tables (millions of rows+)

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Tag not in approved list | Pre-commit blocks CI | Add to `.governance_rules/dbt_accepted_tags.md` first |
| Forgetting `event_time` on large model | CI takes forever on `--sample` | Add it upfront for models >5 min |
| Scheduling in dbt_jobs.yml instead of Airflow | Job runs at wrong time, no dependency awareness | Schedule in Airflow only |
| Not chaining Fivetran → dbt → Hightouch in Airflow | Stale data syncs to destination | Align Airflow schedule with upstream freshness |
| Creating a new tag without updating pre-commit | Pre-commit blocks the PR | Always update both files together |
