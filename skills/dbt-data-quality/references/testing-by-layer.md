# Testing by Layer

## Staging Layer

Staging models have a 1-to-1 relation with source tables. Source tables are already monitored by Monte Carlo (row counts, freshness, schema changes). So staging tests focus on the **shape and referential integrity** of the cleaned data, not anomaly detection.

### Required Tests

```yaml
models:
  - name: stg_hubspot_contacts
    columns:
      - name: contact_id
        tests:
          - unique
          - not_null
      - name: subscription_id
        tests:
          - relationships:
              to: ref('stg_hubspot_subscriptions')
              field: subscription_id
      - name: contact_type
        tests:
          - accepted_values:
              values: ['customer', 'lead', 'partner']
```

- `unique` + `not_null` on the **primary key** of every staging model
- `accepted_values` on status/type/category columns with known allowed values
- `relationships` to validate foreign key integrity against other staging models

### Not needed at staging
- Business logic tests (those belong in intermediate)
- Freshness checks (Monte Carlo covers sources)
- Row count checks (Monte Carlo covers sources)

---

## Intermediate Layer

Intermediate models contain complex transformations. Tests here validate **business logic correctness and grain**.

### Standard Tests

```yaml
models:
  - name: int_typeform_csat_tickets
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - ticket_id
            - response_date
    columns:
      - name: ticket_id
        tests:
          - not_null
      - name: end_date
        tests:
          - dbt_utils.expression_is_true:
              expression: "end_date > start_date"
              severity: error
      - name: subscription_status_name
        tests:
          - not_null:
              severity: warn  # Known data gap
```

- `dbt_utils.unique_combination_of_columns` when no single column is the grain key
- `dbt_utils.expression_is_true` for business rules (date ordering, amount ranges, etc.)
- `not_null` on columns used in downstream JOINs — a null here silently drops rows downstream
- `severity: warn` for known data quality issues that don't block
- `severity: error` for business logic that must hold (date ordering, mandatory relationships)

### Key judgment call: warn vs error
If the test failing means downstream data is **silently wrong** (stakeholders get bad numbers without knowing), use `error`. If the test failing means **known incomplete data** that you're tracking, use `warn`.

---

## Marts Layer

Mart tests protect **stakeholder-facing data**. Focus on what would cause visible damage to reports or external systems.

### Standard Tests

```yaml
models:
  - name: hubspot_contact_subscriptions
    config:
      contract:
        enforced: true          # Reverse-ETL models only
    columns:
      - name: contact_id
        data_type: varchar      # Required if contract is enforced
        tests:
          - unique
          - not_null:
              severity: error   # Can't sync to destination without this
        constraints:
          - type: not_null      # Warehouse DDL-level (reverse-ETL only)
      - name: subscription_status
        data_type: varchar
        tests:
          - not_null:
              severity: error
          - accepted_values:
              values: ['active', 'cancelled', 'paused']
```

- Basic `unique` + `not_null` on primary keys
- `severity: error` on fields that **cannot reach stakeholders or external systems as null**
- dbt contracts + warehouse `constraints` for reverse-ETL models (see `contracts.md`)

### What NOT to test at marts

- Don't duplicate Monte Carlo anomaly detection (row counts, freshness) with dbt tests
- Don't add `expression_is_true` for business rules already tested upstream in intermediate
- Don't test things that are guaranteed by upstream tests (if `int_model` already guarantees non-null `customer_id`, the mart referencing it doesn't need to re-test it unless it transforms the column)

---

## Test Coverage Summary

| Test Type | Staging | Intermediate | Marts |
|---|---|---|---|
| `unique` on PK | Required | If applicable | Required |
| `not_null` on PK | Required | Required | Required |
| `accepted_values` | For type/status cols | If applicable | If applicable |
| `relationships` (FK) | Required | Optional | Not needed |
| `unique_combination_of_columns` | If composite key | For grain enforcement | If applicable |
| `expression_is_true` | Not needed | Business rules | Sparingly |
| `severity: error` | Default | For blocking logic | For external/stakeholder data |
| `severity: warn` | Rarely | For known data gaps | For tracked gaps |
| dbt contracts | Not needed | Not needed | Reverse-ETL only |
