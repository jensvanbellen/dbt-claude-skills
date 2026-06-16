# Common Cross-Repo Scenarios

## Scenario 1: New dbt Model → New Job → New Airflow DAG

**Repos:** `your_dbt_project` + `airflow`

### In `your_dbt_project`:
1. Build the model SQL + YAML (use `dbt-standards`)
2. Add the approved tag in the config block
3. Add the tag to `.governance_rules/dbt_accepted_tags.md`
4. Add the tag to `.pre-commit-config.yaml`
5. Add a new job entry in `jobs/dbt_jobs.yml`
6. Open PR

### In `airflow` (after dbt PR is merged and job is live in dbt Cloud):
1. Create `dags/dbt/dbt_reporting_<domain>.py` using the pure-dbt template
2. Look up the dbt Cloud job ID (from the URL after merge)
3. Pick a cron schedule that doesn't conflict with existing heavy jobs
4. Open PR

**Merge order:** `your_dbt_project` first → get the job ID → then `airflow`

---

## Scenario 2: New Model Added to an Existing Job

**Repos:** `your_dbt_project` only

1. Build the model SQL + YAML
2. Add the tag that matches the existing job (check `jobs/dbt_jobs.yml` for the job's `--select tag:...`)
3. No Airflow changes needed — the existing DAG already schedules that job

---

## Scenario 3: Add Column to Destination Sync

**Repos:** `your_dbt_project` + `reverse-etl_hightouch`

### In `your_dbt_project`:
1. Add the column to the model SQL
2. Add `data_type` for the new column in the YAML (required by contract)
3. Add `constraints` + `data_tests` if the column is critical

### In `reverse-etl_hightouch`:
1. Add a mapping entry in `syncs/<sync-file>.yaml`:
   ```yaml
   - to: <destination_property_api_name>
     from: MY_NEW_COLUMN_UPPER_CASE
     type: standard
   ```
2. Open PR

**Merge order:** `your_dbt_project` first (column must exist in warehouse) → then `reverse-etl_hightouch`

---

## Scenario 4: New Reverse-ETL Pipeline (end-to-end)

**Repos:** `your_dbt_project` + `reverse-etl_hightouch` + `airflow`

### In `your_dbt_project`:
1. Build the reverse-ETL mart model with contract (see `dbt-reverse-etl` skill)
2. Tag with the appropriate `job_reverse_etl_*` tag
3. Open PR

### In `reverse-etl_hightouch`:
1. Create a model file in `models/<name>.yaml`
2. Create a sync file in `syncs/<name>-to-<destination>.yaml` with `schedule: type: manual`
3. Open PR

### In `airflow`:
1. Find the relevant existing ELT DAG or create a new one
2. Add a `HightouchTriggerSyncOperator` with the new sync ID
3. Open PR

**Merge order:** `your_dbt_project` → `reverse-etl_hightouch` → `airflow`

---

## Scenario 5: Schedule Change Only

**Repos:** `airflow` only

1. Find the DAG file in `dags/dbt/` or `dags/elt/`
2. Change the `schedule` cron expression
3. Open PR — no dbt changes needed

---

## Scenario 6: Change dbt Job Steps

**Repos:** `your_dbt_project` only

1. Edit `jobs/dbt_jobs.yml`
2. Open PR — Airflow still triggers the same job ID, so no Airflow changes

---

## What Needs to Change in Airflow vs dbt_jobs.yml?

| Change | Where it lives |
|---|---|
| When the job runs (schedule/timing) | `airflow` repo |
| What dbt models/tags the job runs | `your_dbt_project/jobs/dbt_jobs.yml` |
| Upstream dependencies (Fivetran before dbt) | `airflow` repo |
| Downstream triggers (Hightouch after dbt) | `airflow` repo |
| Job threads, timeout | `airflow` repo (`additional_run_config`, `timeout`) |
