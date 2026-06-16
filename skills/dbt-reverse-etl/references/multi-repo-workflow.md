# Multi-Repo Workflow for Reverse-ETL Changes

Most reverse-ETL changes span **two repos simultaneously**:
- `your_dbt_project` — model SQL + YAML
- `reverse-etl_hightouch` — sync column mappings

Some changes also touch a third repo:
- `airflow` — if the schedule or pipeline chain changes

## The Coordination Pattern

### Adding a new column to an existing sync

**your_dbt_project PR:**
- Add column to model SQL
- Add `data_type` to YAML (contract requirement)
- Add `constraints` + `data_tests` if it's a critical column

**reverse-etl_hightouch PR:**
- Add new mapping entry in the sync YAML
- `from:` = UPPER_CASE warehouse column name
- `to:` = destination field API name

**Link the PRs:** Mention the Hightouch PR in the dbt PR description and vice versa.
**Merge order:** Merge dbt first → then Hightouch. The dbt model needs to exist before Hightouch reads it.

### Removing a column from a sync

**Before opening any PR:**
1. Check if the destination uses the field actively (workflows, lists, reports)
2. Align with the relevant team if the field has external usage

**your_dbt_project PR:**
- Remove column from SQL
- Remove from YAML

**reverse-etl_hightouch PR:**
- Remove the mapping entry

**Merge order:** Merge Hightouch first → then dbt. Remove the mapping before the column disappears from the warehouse.

### Renaming a column

This is the most dangerous change — a rename breaks the sync immediately if not coordinated.

**Sequence:**
1. In the dbt model SQL: add the new column name, keep the old name temporarily as an alias
2. In YAML: add the new column with `data_type`, keep the old one
3. In Hightouch: update the mapping `from:` to the new UPPER_CASE name
4. Merge dbt PR → verify sync still works with both columns available
5. In a follow-up dbt PR: remove the old column alias and YAML entry

This two-step approach prevents a gap where the Hightouch mapping points to a column that no longer exists.

### Adding a new reverse-ETL model (full pipeline)

Step-by-step across all repos:

1. **your_dbt_project** — build model + YAML (with contract)
2. **Hightouch UI** — create a new model pointing to the dbt table, create the sync definition
3. **reverse-etl_hightouch** — create the sync YAML file
4. **airflow** — add a `HightouchTriggerSyncOperator` to the relevant DAG (or create a new DAG)

**Merge order:**
1. Merge `your_dbt_project` PR first (the table must exist in the warehouse)
2. Configure in Hightouch UI (needs the model to be discoverable)
3. Merge `reverse-etl_hightouch` PR
4. Merge `airflow` PR last

## Branch Naming

- `your_dbt_project`: `feature/reverse-etl-<description>`
- `reverse-etl_hightouch`: `feature/<description>`
- `airflow`: `feature/retl-<description>`

## Checking Sync Health After Deployment

After merging:
1. Trigger the Airflow DAG manually (or wait for scheduled run)
2. Monitor the Hightouch sync status in the UI
3. Check for errors in the Hightouch sync run history
4. Spot-check a sample of records in the destination to confirm values look correct

Common errors to watch for:
- `contact not found` — the destination contact lookup failed (check `associationMappings` filter logic)
- `required field missing` — a non-null destination field received a null value (check dbt WHERE filter)
- `invalid type` — data type mismatch between warehouse and destination field (check `data_type` in YAML)

If errors appear: pause the Hightouch sync first, then investigate. Don't let a broken sync run repeatedly and flood the destination with errors.
