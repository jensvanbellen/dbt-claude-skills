# Airflow Orchestration

## Airflow is the Schedule Source of Truth

dbt Cloud knows *what* to run. Airflow decides *when* and *in what order*.

- Airflow lives in a **separate repo** from `your_dbt_project`
- All schedule changes must go in the Airflow repo, not in `jobs/dbt_jobs.yml`
- dbt Cloud job cron fields should remain empty/null — Airflow owns timing

## Two Types of dbt DAGs

### Type 1 — Pure dbt DAG (`dags/dbt/`)

For jobs that **only run dbt** — no Fivetran source refresh, no Hightouch sync. Most analytics jobs are this type.

**File**: `dbt_reporting_<domain>.py`
**Location**: `dags/dbt/`

```python
from datetime import datetime, timedelta

from airflow.decorators import dag
from airflow.providers.dbt.cloud.operators.dbt import DbtCloudRunJobOperator
from airflow.providers.dbt.cloud.sensors.dbt import DbtCloudJobRunSensor
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.utils.task_group import TaskGroup


"""
### <Domain> daily refresh.
#### Purpose
<What this job does — one or two sentences.>
#### Notes
- Trigger dbt job <job_id>.
"""


@dag(
    dag_id="dbt_reporting_<domain>",
    start_date=datetime(2022, 2, 9),      # Standard start date for all DAGs
    schedule="33 4 * * *",                # UTC time — pick a slot that avoids busy hours
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

### Type 2 — ELT Chain (`dags/elt/`)

For pipelines that **chain source refresh → dbt → optional downstream sync** (Fivetran → dbt → Hightouch). Used for ELT pipelines that depend on fresh data from external sources.

These also use the `mcd_callbacks` pattern for Monte Carlo alerting in production.

**File**: `elt_<source_or_domain>.py`
**Location**: `dags/elt/`

```python
from datetime import datetime, timedelta

from airflow.decorators import dag
from airflow.models import Variable
from airflow.providers.dbt.cloud.operators.dbt import DbtCloudRunJobOperator
from airflow.providers.dbt.cloud.sensors.dbt import DbtCloudJobRunSensor
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.utils.task_group import TaskGroup
from airflow_mcd.callbacks import mcd_callbacks
from airflow_provider_hightouch.operators.hightouch import HightouchTriggerSyncOperator
from fivetran_provider_async.operators import FivetranOperator


"""
### <Source> ELT pipeline.
#### Purpose
<What this pipeline does.>
#### Process
1. Trigger Fivetran <source> connector.
2. Run dbt Cloud transformations.
3. Sync data to <destination> via Hightouch.
"""


env = Variable.get("env")
dag_callbacks = mcd_callbacks.dag_callbacks if env == "prod" else {}
task_callbacks = mcd_callbacks.task_callbacks if env == "prod" else {}


@dag(
    dag_id="elt_<source>",
    start_date=datetime(2022, 2, 9),
    schedule="37 2 * * *",
    catchup=False,
    default_args={
        "retries": 2,
        "retry_delay": timedelta(minutes=3),
        "owner": "data-engineering",
        "email_on_failure": True,
    },
    doc_md=__doc__,
    tags=["elt", "<domain>"],
    **dag_callbacks,
)
def elt_<source>():
    begin = EmptyOperator(task_id="begin")
    end = EmptyOperator(task_id="end")

    extract_and_load = FivetranOperator(
        task_id="fivetran_task",
        connector_id="<fivetran_connector_id>",
        fivetran_conn_id="fivetran_default",
    )

    with TaskGroup(
        group_id="transform",
        prefix_group_id=False,
        default_args={"dbt_cloud_conn_id": "dbt_cloud"},
    ) as transform:
        run_dbt_job = DbtCloudRunJobOperator(
            task_id="transform_<source>",
            job_id=<job_id>,  # https://cloud.getdbt.com/deploy/<account_id>/projects/<project_id>/jobs/<job_id>
            wait_for_termination=False,
            additional_run_config={"threads_override": 8},
            trigger_rule="all_success",
            retries=0,
            **task_callbacks,
        )
        wait_for_dbt = DbtCloudJobRunSensor(
            task_id="wait_for_dw_transformations",
            run_id=run_dbt_job.output,
            poke_interval=60,
            timeout=3600,
            retries=2,
            retry_delay=timedelta(minutes=1),
            **task_callbacks,
        )
        run_dbt_job >> wait_for_dbt

    # Hightouch sync (only if pipeline has a downstream sync)
    sync_to_destination = HightouchTriggerSyncOperator(
        task_id="run_hightouch_sync",
        sync_id="<hightouch_sync_id>",
        **task_callbacks,
    )

    begin >> extract_and_load >> transform >> sync_to_destination >> end


dag = elt_<source>()
```

**If there's no Hightouch sync**, drop that operator and end with `>> transform >> end`.
**If there's no Fivetran source**, this is a pure dbt DAG — use Type 1 instead.

## Key Operator Details

### `DbtCloudRunJobOperator`
- `wait_for_termination=False` — don't block, immediately hand off to the sensor
- `additional_run_config={"threads_override": 8}` — standard for all dbt jobs
- `trigger_rule="all_done"` — run even if prior tasks warned/failed (standard for dbt DAGs)
- `trigger_rule="all_success"` — only run if prior tasks succeeded (standard for ELT chains)
- `retries=0` — retry logic lives at the sensor level, not the trigger

### `DbtCloudJobRunSensor`
- `poke_interval=120` — poll every 2 min for fast jobs, `300` for slow jobs (>10 min)
- `timeout=3600` — 1 hour standard; use `10800` for very long jobs
- `retries=2` + `retry_delay=timedelta(minutes=1)` — always include

### `mcd_callbacks`
Only in ELT chain DAGs. Connects Airflow task events to Monte Carlo for alerting:
```python
env = Variable.get("env")
dag_callbacks = mcd_callbacks.dag_callbacks if env == "prod" else {}
task_callbacks = mcd_callbacks.task_callbacks if env == "prod" else {}
```
Spread into `@dag(...)` as `**dag_callbacks` and into each operator as `**task_callbacks`.

## DAG Naming Conventions

| Type | Pattern | Example |
|---|---|---|
| Pure dbt | `dbt_reporting_<domain>` | `dbt_reporting_churned` |
| ELT chain | `elt_<source>` | `elt_hubspot`, `elt_stripe` |

## DAG Tags

```python
tags=["dbt", "<domain>", "daily"]    # for dbt DAGs
tags=["elt", "<domain>"]             # for ELT chains
```

## Scheduling — Pick a Slot

All times are **UTC**. Spread jobs to avoid warehouse contention:
- Pre-midnight: for the main daily reporting job
- Early AM: for other domain jobs
- Align with upstream Fivetran completion for ELT chains

## Finding a dbt Cloud Job ID

After merging the `jobs/dbt_jobs.yml` PR:
1. Open dbt Cloud and navigate to your project's jobs
2. Find the new job
3. The job ID is in the URL — add it as a comment in the DAG file

## Connection IDs

| Service | Airflow conn_id |
|---|---|
| dbt Cloud | `dbt_cloud` |
| Fivetran | `fivetran_default` |

Hightouch uses its own provider — no explicit conn_id needed.
