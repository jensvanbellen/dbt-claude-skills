---
name: dbt-data-quality
description: Guides the correct data quality approach at your company — which dbt tests to apply at each layer, when to use warn vs error severity, how to implement dbt contracts on reverse-ETL models, and how to divide responsibilities between dbt tests and Monte Carlo monitoring. Use when adding tests, setting up monitoring, or reviewing quality coverage for a model.
allowed-tools: "Read, Write, Edit, Glob, Grep"
user-invocable: true
metadata:
  author: your-org
---

# dbt Data Quality

Data quality uses **two complementary tools** with distinct responsibilities:

| | dbt Tests | Monte Carlo |
|---|---|---|
| **When it runs** | Only during `dbt build` or `dbt test` | Continuously, always-on |
| **What it watches** | Output of staging, intermediate, and mart SQL | Raw source tables + reporting marts |
| **Type of check** | Explicit assertions you write | Statistical anomaly detection |
| **Example** | `ticket_id` is never null after `int_typeform_csat_tickets` runs | Fivetran loaded 40% fewer rows than usual |

**Do not think of them as alternatives.** Use both.

## Reference Guides

| Guide | When to read |
|---|---|
| [references/testing-by-layer.md](references/testing-by-layer.md) | Deciding which tests to write for a staging, intermediate, or mart model |
| [references/dbt-vs-monte-carlo.md](references/dbt-vs-monte-carlo.md) | Deciding whether to add a dbt test, a Monte Carlo monitor, or both |
| [references/contracts.md](references/contracts.md) | Setting up data contracts on reverse-ETL or externally-consumed models |

## Quick Decision Tree

**New staging model?**
→ Add `unique` + `not_null` on PK. Add `accepted_values` for status/type columns. Add `relationships` for foreign keys. Monte Carlo already covers the source.

**New intermediate model?**
→ Validate business logic with `expression_is_true`. Enforce grain with `unique_combination_of_columns`. Use `severity: warn` for known data gaps, `error` for blocking ones.

**New mart model?**
→ Basic dbt tests. `severity: error` on any field that feeds external systems or stakeholder reports and must never be null. Inform data governance committee for catalog ownership assignment. Notify Monte Carlo is now monitoring this mart.

**New reverse-ETL model (feeds an external destination)?**
→ Add a data contract (`contract: enforced: true`). Use both `constraints` (warehouse DDL-level) AND `data_tests` (dbt-level). `severity: error` on all critical fields.

## Severity Rules

| Condition | Severity | Reason |
|---|---|---|
| `subscription_status_name IS NULL` | `warn` | Known data gap — track it but don't block |
| `customer_id IS NULL` (in external feed) | `error` | Can't sync without this — block the job |
| Foreign key references deleted record | `warn` | Historical data issue — known, not fixable |
| Primary key is not unique | `error` | Grain is broken — stop downstream processing |
| Amount column is negative on a revenue table | `error` | Silent bad data reaching finance dashboards |

**Rule of thumb:**
- `error` = a downstream system or stakeholder would receive bad/incomplete data
- `warn` = a known data quality issue you're tracking but can't fix right now

## STOP Conditions

**Before adding a test, ask:**
- Does a Monte Carlo monitor already cover this? (e.g., freshness, row count anomaly on a mart)
- Is this test asserting something that can actually happen, or am I testing a database constraint?
- For `severity: error` — are you sure this should block the entire job if it fails?
