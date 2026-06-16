---
name: dbt-standards
description: Enforces your team's dbt conventions — naming, config blocks, YAML format, SQL style, PII tagging, and tag governance. Use when building or modifying any dbt model in the your_dbt_project project.
allowed-tools: "Read, Write, Edit, Glob, Grep"
user-invocable: false
metadata:
  author: your-org
---

# dbt Standards

These rules are **non-negotiable** for all work in the `your_dbt_project` project. They layer on top of the official `using-dbt-for-analytics-engineering` skill — use that skill for generic dbt execution, and enforce everything in this skill on top of it.

> Violations here have caused real incidents. Don't skip the rules even for "quick" changes.

## Reference Guides

Read the relevant reference when you hit that type of work:

| Guide | When to read |
|---|---|
| [references/naming-conventions.md](references/naming-conventions.md) | Creating any new model, YAML, field, or macro |
| [references/model-config-rules.md](references/model-config-rules.md) | Adding or editing a model's config block |
| [references/sql-style-guide.md](references/sql-style-guide.md) | Writing or reviewing any SQL |
| [references/pii-and-security.md](references/pii-and-security.md) | Working with any column that may contain personal data |

## Quick Checklist — Run Before Every Commit

- [ ] Model name follows the correct layer prefix convention
- [ ] Config block present with at minimum `materialized`
- [ ] Tag added from approved list (`.governance_rules/dbt_accepted_tags.md`)
- [ ] Exactly one owning pipeline `job_*` tag (`job_daily` cadence tag may coexist; see `references/model-config-rules.md`)
- [ ] YAML file created/updated with model `description` using `{{ doc("table_name") }}`
- [ ] YAML file created/updated with column descriptions using `{{ doc("col_name") }}`
- [ ] Columns in YAML match SQL SELECT output order
- [ ] Models in YAML are listed alphabetically
- [ ] YAML filename uses `_src_`/`_stg_`/`_int_`/`_mrt_` prefix
- [ ] No raw table names — only `{{ ref() }}` and `{{ source() }}`
- [ ] No `SELECT *` except `SELECT * FROM final`
- [ ] No commented-out code left behind
- [ ] Empty line at end of every file
- [ ] `pre-commit install` was run in this repo checkout

## Layer Materializations (Never Deviate Without Reason)

| Layer | Default Materialization |
|---|---|
| Staging | `view` |
| Intermediate | `view` |
| Marts | `table` (or `incremental` for high-volume) |
| Snapshots | Uses dbt snapshot mechanism — not a regular model |

## STOP Conditions

**Do not proceed if:**
- A model has no config block
- A model has no tag from the approved list
- A model has two pipeline `job_*` tags (excluding the `job_daily` cadence tag) without a sanctioned reverse-ETL fan-out reason
- A model references a raw table name instead of `{{ ref() }}`
- A YAML file is missing for a new model
