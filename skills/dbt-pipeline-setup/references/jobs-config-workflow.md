# dbt Jobs Config Workflow

## How Jobs Are Configured

All dbt Cloud jobs live in **`jobs/dbt_jobs.yml`** in the `your_dbt_project` repo. No manual changes in the dbt Cloud UI — everything is code.

To change a job:
1. Edit `jobs/dbt_jobs.yml` in the repo
2. Open a PR
3. Merge → change is live in dbt Cloud automatically (via GitHub Actions)

## Tag Governance

Every model tag must be registered before use:
- Approved tags list: `.governance_rules/dbt_accepted_tags.md`
- Pre-commit enforcement: `.pre-commit-config.yaml` (at the repo root)

If you create a new tag without adding it to both files, the pre-commit hook will block your PR.

**One owning pipeline `job_*` tag per model.** Tag a model with the single `job_*` of the job that owns its scheduled build. `job_daily` is a cadence tag and may coexist. Other jobs that need the model should select it by dependency (`tag:job_hr+`), not a second pipeline tag — two pipeline `job_*` tags cause double-builds and force the daily job's `--exclude` list to grow. See `dbt-standards/references/model-config-rules.md`.

## jobs/dbt_jobs.yml Structure

```yaml
jobs:
  - name: job_typeform           # matches the tag used in models
    description: "Runs all Typeform models"
    environment: production
    execute_steps:
      - dbt build --select tag:job_typeform
    triggers:
      # Scheduling is handled by Airflow — leave empty or set to manual
      schedule:
        cron: null
```

Key fields:
- `name`: must match the tag name convention (`job_<source>` or `job_<domain>`)
- `execute_steps`: prefer `dbt build` with tag selection
- Do **not** set a cron here — scheduling lives in Airflow

## Adding a New Job — Full Checklist

1. Decide on the tag name (e.g., `job_churned`)
2. Add the tag to `.governance_rules/dbt_accepted_tags.md`
3. Add the tag to the accepted tags list in `.pre-commit-config.yaml`
4. Add the tag to the model's config block: `tags = ['job_churned']`
5. Add the new job entry in `jobs/dbt_jobs.yml`
6. Open a PR with all four changes together
7. After merge: verify the job appears in dbt Cloud
8. Then set up the Airflow DAG (see `airflow-orchestration.md`)

## CI Job (`ci_prd`)

The CI job runs on every PR automatically. Its steps:

```shell
dbt clone --select state:unmodified
dbt clone --select state:modified+, config.materialized:incremental, state:old
dbt seed
dbt build --select state:modified+ --full-refresh --sample="7 days" --fail-fast
dbt snapshot --select state:modified+
```

The `--sample="7 days"` flag uses the `event_time` config to limit data scanned. If your new model has >5 min runtime, add `event_time` to its config block to make CI efficient.

## Selecting Models in Jobs

Preferred selection methods:
- By tag: `dbt build --select tag:job_typeform` (recommended — decoupled)
- By path: `dbt build --select models/marts/finance/` (acceptable for domain jobs)
- By model name: `dbt build --select my_model+` (for targeted runs only)

Avoid `dbt run -s +my_model` in production jobs — too fragile as DAG evolves.

## Running a Full Refresh in Production

For incremental models that need a full history rebuild:
- Use the dedicated _Run a Specific Model in PRD_ job in dbt Cloud
- Command: `dbt build --select <model_name> --full-refresh`
- Don't create a separate job for this — it's handled by the shared utility job
