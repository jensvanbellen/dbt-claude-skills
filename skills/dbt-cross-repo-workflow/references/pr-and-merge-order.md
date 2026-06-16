# PR Linking and Merge Order

## Branch Naming

| Repo | Pattern | Example |
|---|---|---|
| `your_dbt_project` | `feature/<description>` or `fix/<description>` | `feature/marketing-attribution-model` |
| `airflow` | `feature/airflow-<description>` or `feature/dbt-<description>` | `feature/airflow-marketing-attribution` |
| `reverse-etl_hightouch` | `feature/<description>` | `feature/add-subscription-rate-column` |

## Linking PRs

When your change spans multiple repos, always link the related PRs in each description. Example PR description for the Airflow PR:

> **Related PRs:**
> - your_dbt_project: `<link to PR>`
> - (If applicable) reverse-etl_hightouch: `<link to PR>`
>
> The your_dbt_project PR adds the dbt job. This PR adds the Airflow DAG to schedule it.
> **Merge your_dbt_project PR first.**

## Merge Order by Scenario

### New pipeline (dbt job + Airflow DAG)
1. `your_dbt_project` — job must exist in dbt Cloud before Airflow can reference its ID
2. `airflow` — DAG references the dbt Cloud job ID

### Adding a column to a destination sync
1. `your_dbt_project` — column must exist in the warehouse before Hightouch tries to map it
2. `reverse-etl_hightouch` — mapping references the warehouse column

### Removing a column from a sync (reverse order!)
1. `reverse-etl_hightouch` — remove the mapping first so Hightouch stops trying to read the column
2. `your_dbt_project` — then remove the column from the model and YAML

### New reverse-ETL pipeline (full)
1. `your_dbt_project` — mart model must be in the warehouse first
2. `reverse-etl_hightouch` — model + sync YAMLs reference the dbt table
3. `airflow` — DAG references the dbt Cloud job ID

### Schedule change only
1. `airflow` only — nothing else changes

## Working in Both Repos Simultaneously in One AI Session

To give an AI agent access to both repos at once:

1. Make sure both repos are cloned under the same parent directory
2. Tell the agent explicitly:
   > "You'll need to make changes in two repos: `/path/to/your_dbt_project` and `/path/to/airflow`. Please create branches in both before starting."
3. The agent can read and write both directories — it just needs to know the paths
4. Ask it to create both PRs and link them before you review

## Review Process

For changes that span `your_dbt_project` and `airflow`:
- Both PRs need a reviewer from the relevant team
- Don't merge either PR until both are approved

For `reverse-etl_hightouch` changes:
- Always requires Engineering review — these affect live destination data
- Coordinate with the CRM/marketing team if removing or renaming destination fields

## After Merge Checklist

- [ ] Verify dbt job exists in dbt Cloud (for new jobs)
- [ ] Verify Airflow DAG appears in the UI (for new DAGs)
- [ ] Toggle DAG on if it was paused
- [ ] Trigger manually once to confirm end-to-end flow works
- [ ] Check Hightouch sync run history (for reverse-ETL changes)
- [ ] Verify a sample of records in the destination (HubSpot, Google Sheets, etc.)
