# Reverse-ETL Airflow Orchestration

## The Standard Chain

Every reverse-ETL pipeline follows this Airflow pattern:

```
begin → Fivetran (source sync) → dbt job → Hightouch sync(s) → end
```

This ensures:
1. Source data is fresh before dbt runs
2. dbt models are fully built before Hightouch reads them
3. The destination receives the most current data possible

## Reference DAG

```python
from airflow.providers.dbt.cloud.operators.dbt import DbtCloudRunJobOperator
from airflow.providers.dbt.cloud.sensors.dbt import DbtCloudJobRunSensor
from airflow_provider_hightouch.operators.hightouch import HightouchTriggerSyncOperator
from fivetran_provider_async.operators import FivetranOperator

@dag(
    dag_id="elt_<source>",
    schedule="37 2 * * *",
    catchup=False,
    default_args={"retries": 0, "retry_delay": timedelta(minutes=3)},
)
def elt_<source>():
    begin = EmptyOperator(task_id="begin")
    end = EmptyOperator(task_id="end")

    extract_and_load = FivetranOperator(
        task_id="fivetran_task",
        connector_id="<fivetran_connector_id>",
        fivetran_conn_id="fivetran_default",
    )

    with TaskGroup(group_id="transform", prefix_group_id=False,
                   default_args={"dbt_cloud_conn_id": "dbt_cloud"}) as transform:
        run_dbt_job = DbtCloudRunJobOperator(
            task_id="transform_dbt",
            job_id=<dbt_cloud_job_id>,
            wait_for_termination=False,
            additional_run_config={"threads_override": 8},
        )
        wait_for_dbt = DbtCloudJobRunSensor(
            task_id="wait_for_dbt",
            run_id=run_dbt_job.output,
            poke_interval=300,
            timeout=3600,
            retries=2,
        )

    sync_to_destination = HightouchTriggerSyncOperator(
        task_id="run_hightouch_sync",
        sync_id="<hightouch_sync_id>",
    )

    begin >> extract_and_load >> transform >> sync_to_destination >> end
```

## Key Operators

### `FivetranOperator`
Triggers a Fivetran connector sync. `connector_id` is in the Fivetran URL.
- Waits for Fivetran to finish before proceeding

### `DbtCloudRunJobOperator` + `DbtCloudJobRunSensor`
The two-step pattern for dbt jobs:
1. `DbtCloudRunJobOperator` with `wait_for_termination=False` — triggers the job and immediately returns the run ID
2. `DbtCloudJobRunSensor` — polls until the job finishes, with configurable `poke_interval`

Use `additional_run_config={"threads_override": 8}` for parallel thread boost on large jobs.

### `HightouchTriggerSyncOperator`
Triggers a Hightouch sync by its sync ID. Get the sync ID from the Hightouch UI URL.
- Does not wait for the sync to complete by default — check if you need `wait_for_completion=True`

## Adding a New Hightouch Sync Trigger to an Existing DAG

When you add a new dbt model + Hightouch sync that belongs to an existing pipeline:

1. Find the DAG in `airflow/dags/elt/<dag_name>.py`
2. Add a `HightouchTriggerSyncOperator` after the existing `transform` TaskGroup:

```python
sync_new_object = HightouchTriggerSyncOperator(
    task_id="run_hightouch_new_object_sync",
    sync_id="<new_sync_id>",
)

# Update the dependency chain
begin >> extract_and_load >> transform >> [existing_sync, sync_new_object] >> end
```

3. Open a PR in the Airflow repo

## Creating a New DAG

Only create a new DAG if the reverse-ETL pipeline has different upstream dependencies or a different schedule from existing pipelines.

Template for a new reverse-ETL DAG:
- `dag_id`: `retl_<destination>_<domain>` (e.g., `retl_hubspot_customers`)
- `schedule`: align with when the upstream Fivetran sync completes
- `tags`: `["reverse_etl", "<destination>"]`
- `retries: 0` — don't auto-retry reverse-ETL silently (data errors need investigation)

## Credentials & Connection IDs

| Connection | Airflow conn_id |
|---|---|
| dbt Cloud | `dbt_cloud` |
| Fivetran | `fivetran_default` |

Hightouch uses its own provider — no explicit conn_id needed, it uses the installed provider's configured credentials.

## Finding IDs

- **Fivetran connector_id**: in the Fivetran URL: `fivetran.com/dashboard/connectors/<connector_id>`
- **dbt Cloud job_id**: in the dbt Cloud URL after the job is created
- **Hightouch sync_id**: in the Hightouch URL: `app.hightouch.com/<your-workspace>/syncs/<sync_id>`
