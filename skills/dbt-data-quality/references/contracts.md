# dbt Data Contracts

## When to Use Contracts

Contracts are used for models consumed by **external systems or tools** where a silent schema change would cause downstream failures.

**Currently applied to:** Reverse-ETL models (feeding destinations via Hightouch)

**When to use contracts:**
- Any model consumed by an external system (CRM, ad platform, other APIs)
- Models with very stable, agreed-upon schemas that many teams depend on
- Anything where a column rename or type change would break a sync without warning

**Not needed for:**
- Staging models (internal, schema changes acceptable)
- Intermediate models (internal, consumed only by dbt)
- Standard mart models (unless externally consumed)

## How Contracts Work

A contract is a compile-time promise: "this model will always output these columns with these types." dbt fails the run if the SQL output doesn't match.

```yaml
models:
  - name: hubspot_contact_subscriptions
    config:
      contract:
        enforced: true
    columns:
      - name: contact_id
        data_type: varchar
        description: Destination contact record ID
        constraints:
          - type: not_null
        tests:
          - unique
          - not_null:
              severity: error
      - name: subscription_status
        data_type: varchar
        description: Current subscription status
        tests:
          - not_null:
              severity: error
          - accepted_values:
              values: ['active', 'cancelled', 'paused']
      - name: updated_at
        data_type: timestamp_ntz
        tests:
          - not_null:
              severity: error
```

## Contract Requirements

When `contract: enforced: true` is set:
- **Every column in the YAML must exist in the model SQL** (and vice versa)
- **Every column must have an explicit `data_type`** declared
- The actual data type output by the model must match the declared type
- Any undeclared column in the SQL will cause a compile error

## `constraints` vs `data_tests`

Use **both** on reverse-ETL models (belt-and-suspenders approach):

| | `constraints` | `data_tests` |
|---|---|---|
| **Enforced by** | Warehouse at DDL level | dbt after model runs |
| **When it fails** | At table creation time | After data is inserted |
| **Example** | `constraints: - type: not_null` | `tests: - not_null` |

`constraints` catch structural issues before data is inserted. `data_tests` catch data quality issues after. Both are needed because a column can be structurally nullable (no constraint) but still required to be populated by business rules.

## Common Data Types for Snowflake

| Use case | data_type |
|---|---|
| IDs, names, statuses | `varchar` |
| Integers | `number` |
| Decimals | `number(18,4)` |
| Timestamps | `timestamp_ntz` |
| Dates | `date` |
| Booleans | `boolean` |

## What Happens if the Contract Breaks

If a model with a contract is changed in a way that violates the contract:
- dbt will fail at compile time (before any SQL runs in the warehouse)
- The job fails with a clear error: column missing, type mismatch, or extra undeclared column

This is intentional — it prevents external syncs from receiving silent schema changes.

## Adding a Contract to an Existing Model

1. Add `contract: enforced: true` to the model's YAML config
2. Ensure every column in the SQL is listed in the YAML with a `data_type`
3. Ensure every YAML column exists in the SQL output
4. Run `dbt compile` locally to verify — this catches contract violations without running the model
5. Add `constraints: - type: not_null` for columns that are business-critical
6. Add `severity: error` dbt tests as a second layer
