# Hightouch Sync YAML

The `reverse-etl_hightouch` repo has two separate directories:
- `models/` — defines the warehouse data sources Hightouch reads from
- `syncs/` — defines how that data maps to destinations

## Repo Structure

```
reverse-etl_hightouch/
  models/            ← model definition files
  syncs/             ← sync configuration files
  manifest-<your-company>-prod-<id>.yaml  ← READ ONLY (managed by Hightouch UI)
  aliases-<your-company>-prod-<id>.yaml
```

**The manifest file is READ ONLY.** It lists all sources and destinations and is auto-managed by Hightouch. Never edit it manually — changes are ignored.

## Models: Connecting Hightouch to dbt

A model file links Hightouch to a dbt table or a raw warehouse query.

### dbt model reference (preferred)
```yaml
# models/hubspot-subscriptions.yaml
type: dbt_model
modelId: model.your_dbt_project.hubspot_subscriptions
primaryKey: SUBSCRIPTION_ID
```

### Raw SQL model (for non-dbt sources)
```yaml
# models/active-customers.yaml
type: raw_sql
source: <your-warehouse-connection-name>
primaryKey: CUSTOMER_NUMBER
sql: |
  SELECT
    customer_number,
    email,
    ...
  FROM analytics_reporting.customers
  WHERE is_active = TRUE
```

When creating a new reverse-ETL pipeline from a dbt model, create a model file of type `dbt_model`. You only need `raw_sql` if the data doesn't flow through dbt.

## Sync Configuration

Sync files live in `syncs/` and reference the model + destination.

### HubSpot Sync (upsert to custom object)

```yaml
# syncs/hubspot-subscriptions-to-hubspot.yaml
model: hubspot-subscriptions        # matches the model filename (without .yaml)
destination: hubspot
description: Sync for Subscriptions object. Is triggered through an Airflow DAG.
config:
  mode: upsert
  type: object
  object: your_company_subscriptions   # HubSpot object API name
  mappings:
    - to: subscription_status       # HubSpot property internal name
      from: SUBSCRIPTION_STATUS_NAME  # Warehouse column — always UPPER_CASE
      type: standard
    # ... additional mappings ...
  externalIdMapping:
    to: subscription_id             # HubSpot field for the unique record ID
    from: SUBSCRIPTION_ID           # Warehouse column — UPPER_CASE
    type: standard
  associationMappings:
    - to: contact_to_your_company_subscriptions
      type: reference
      lookup:
        by: customer_number
        from: CUSTOMER_NUMBER
        byType: NUMBER
        object: 0-1
        objectLabel: your_company_subscriptions
  rateLimit:
    rateLimit: 10
    rateLimitTime: second
  configVersion: 2
  shouldTruncate: true
  associationMode: append
schedule:
  type: manual                      # ALWAYS manual — Airflow triggers
schedulePaused: false
```

### HubSpot Sync (upsert to standard object — contacts)

```yaml
config:
  mode: upsert
  type: object
  object: contacts                  # Standard HubSpot object
  mappings:
    - to: firstname
      from: FIRST_NAME
      type: standard
    - to: address                   # Template field (composite values)
      type: template
      template: "{{ row['STREET'] }} {{ row['HOUSE_NUMBER'] }}"
  externalIdMapping:
    to: customer_number             # HubSpot property used as the unique ID
    from: CUSTOMER_NUMBER
    type: standard
  associationMappings: []           # No associations for contacts sync
```

### Google Ads (Customer Match audience)

```yaml
config:
  type: CustomerMatchUserList
  mappings:
    - to: hashedEmail
      from: EMAIL
      type: standard
    - to: hashedPhoneNumber
      from: PHONE_NUMBER
      type: standard
    - to: addressInfo.hashedFirstName
      from: FIRST_NAME
      type: standard
  accountId: '<google_ads_account_id>'
  cleanPhone: true
  userListType: CONTACT_INFO
  adUserDataConsent: GRANTED
  customSegmentName: Active_customers_daily_sync
  adPersonalizationConsent: GRANTED
schedule:
  type: interval
  schedule:
    interval:
      unit: day
      quantity: 1
  startDate: '2023-06-24T02:00:00.000Z'
```
Note: Google Ads + Facebook syncs are NOT triggered manually. They use their own interval schedule.

### Google Sheets (mirror/full replace)

```yaml
config:
  mode: mirror
  mappings: []
  sheetName: Subscription Types
  spreadsheetId: <google_sheets_id>
  sheetSelection: select
  autoSyncColumns: true
schedule:
  type: manual
```

## Key Field Rules

### `from:` column names
Always **UPPER_CASE** — warehouse column names are case-sensitive in Hightouch and it normalizes them to uppercase.

### `schedule: type: manual`
Required for **all HubSpot and Google Sheets syncs**. Airflow is the trigger for these. Ad platform syncs (Google Ads, Facebook) have their own `interval` schedules.

### `externalIdMapping`
The unique key Hightouch uses to determine whether to insert or update (upsert). Must correspond to a `unique` column in the dbt model. Always use `UPPER_CASE` for `from:`.

### `associationMappings`
Links custom HubSpot objects to contacts. If you're syncing a custom object, you need this to associate records with HubSpot contacts. Not needed for direct contacts syncs.

## Adding a Column Mapping to an Existing Sync

1. Find the HubSpot property API name (HubSpot UI → Settings → Properties → copy "Internal name")
2. Add to `config.mappings`:
```yaml
    - to: my_new_hubspot_property
      from: MY_NEW_COLUMN         # UPPER_CASE warehouse column name
      type: standard
```
3. Open a PR in `reverse-etl_hightouch` alongside the dbt PR

## Removing a Column Mapping

1. Check whether the destination uses this field actively (workflows, lists, reports)
2. If yes — align with the CRM/marketing team first
3. Remove from `config.mappings`
4. Never remove `externalIdMapping` or `associationMappings`

## Hightouch CLI (`ht`)

Hightouch has no MCP — use the `ht` CLI for inspection and testing:
```bash
ht syncs describe <sync_id>           # Inspect sync config
ht syncs runs list syncs/<sync-file>  # Check recent run history
ht inspect syncs/<sync-file> -f yaml  # Full sync YAML
```

## Finding Sync and Model IDs

- **Sync ID**: in Hightouch URL → `app.hightouch.com/<your-workspace>/syncs/<sync_id>`
- **Model ID for dbt**: `model.your_dbt_project.<model_name>` (standard dbt model reference)
- **Available destinations**: listed in the manifest YAML file
