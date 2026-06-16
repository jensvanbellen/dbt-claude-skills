# Creating a New Airflow DAG

This guide walks through creating a new Airflow DAG for a dbt job. Read the full `dbt-pipeline-setup` airflow reference for the operator patterns — this guide focuses on the creation process itself.

## Before You Start

You need the dbt Cloud job ID. This only exists after the `jobs/dbt_jobs.yml` PR is merged:
1. Merge the `your_dbt_project` PR with the new job entry in `jobs/dbt_jobs.yml`
2. Open dbt Cloud and navigate to your project's jobs list
3. Find the new job and copy its ID from the URL

## Step 1 — Choose the Right DAG Type

**Use `dags/dbt/dbt_reporting_<domain>.py`** if:
- Your job only runs dbt models
- No Fivetran source refresh needed before it
- No Hightouch sync after it

**Use `dags/elt/elt_<source>.py`** if:
- Your job needs a Fivetran sync to run first (fresh source data)
- Your job triggers a Hightouch sync after it
- You need Monte Carlo alerting callbacks

Most analytics-owned jobs are the **dbt type**. ELT chain jobs are usually owned by engineering.

## Step 2 — Pick a Schedule Slot

All times are UTC. Avoid creating a new job at the same time as an existing heavy one (warehouse contention). Check the existing schedules in your Airflow DAGs list.

Pick a slot that gives upstream jobs time to finish first (if your job depends on the daily reporting job, schedule it after midnight + the daily job's typical runtime).

## Step 3 — Create the DAG File

File goes in `dags/dbt/dbt_reporting_<domain>.py`.

Use this exact template — it matches all existing DAGs:

```python
from datetime import datetime
from datetime import timedelta

from airflow.decorators import dag
from airflow.providers.dbt.cloud.operators.dbt import DbtCloudRunJobOperator
from airflow.providers.dbt.cloud.sensors.dbt import DbtCloudJobRunSensor
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.utils.task_group import TaskGroup


"""
### <Domain> daily refresh.
#### Purpose
<One or two sentences describing what this job does and why.>
#### Notes
- Trigger dbt job <job_id>.
"""


@dag(
    dag_id="dbt_reporting_<domain>",
    start_date=datetime(2022, 2, 9),
    schedule="<HH MM> * * *",
    catchup=False,
    default_args={"retries": 1, "retry_delay": timedelta(minutes=3)},
    doc_md=__doc__,
    tags=["dbt", "<domain>", "daily"],
)
def dbt_reporting_<domain>():
    begin = EmptyOperator(task_id="begin")
    end = EmptyOperator(task_id="end")

    with TaskGroup(
        group_id="transform_dbt_reporting_<domain>",
        prefix_group_id=False,
        default_args={"dbt_cloud_conn_id": "dbt_cloud"},
    ) as transform_dbt_reporting_<domain>:
        run_dbt_reporting_<domain> = DbtCloudRunJobOperator(
            task_id="run_dbt_reporting_<domain>",
            job_id=<job_id>,  # https://cloud.getdbt.com/deploy/<account_id>/projects/<project_id>/jobs/<job_id>
            wait_for_termination=False,
            additional_run_config={"threads_override": 8},
            trigger_rule="all_done",
            retries=0,
        )

        wait_for_run_dbt_reporting_<domain> = DbtCloudJobRunSensor(  # noqa: F841
            task_id="wait_for_transform_<domain>",
            run_id=run_dbt_reporting_<domain>.output,
            poke_interval=120,
            timeout=3600,
            retries=2,
            retry_delay=timedelta(minutes=1),
        )
        run_dbt_reporting_<domain> >> wait_for_run_dbt_reporting_<domain>

    begin >> transform_dbt_reporting_<domain> >> end  # noqa: W503


dag = dbt_reporting_<domain>()
```

**Fill in:**
- `<domain>` → snake_case name (e.g., `marketing_attribution`)
- `<job_id>` → the dbt Cloud job ID from step 1
- `<HH MM>` → the UTC schedule (e.g., `08 4` for 04:08 UTC)
- `<domain>` tag → the domain tag matching the dbt job
- docstring → clear description of the job's purpose

**Timeout guidance:**
- Fast jobs (<5 min): `poke_interval=60`, `timeout=1800`
- Normal jobs (5-15 min): `poke_interval=120`, `timeout=3600`
- Slow jobs (>15 min): `poke_interval=300`, `timeout=7200`

## Step 4 — Open the PR

```bash
git checkout -b feature/airflow-dbt-<domain>
git add dags/dbt/dbt_reporting_<domain>.py
git commit -m "Add Airflow DAG for dbt_reporting_<domain>"
git push -u origin feature/airflow-dbt-<domain>
```

In the PR description, link the `your_dbt_project` PR that created the job:
> "Related: your_dbt_project PR #XYZ — adds the dbt job this DAG schedules"

## Step 5 — Verify After Merge

1. Find the DAG in the Airflow UI
2. Toggle it ON if it was paused
3. Trigger it manually once to confirm the job runs successfully
4. Check the dbt Cloud run was triggered and completed

## When You Need Two TaskGroups

Use two TaskGroups if the job has a sequential structure — for example: transform → then run tests. Only add complexity when there's a real dependency between the two jobs. One TaskGroup is sufficient for most analytics jobs.
