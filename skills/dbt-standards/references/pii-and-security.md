# PII Handling & Security

## How PII Masking Works

The warehouse holds PII data. Protect it via a two-layer mechanism:

1. **dbt meta tags** on columns in YAML declare which columns are PII
2. A **post-hook** (`apply_pii_tags` macro) runs after each model and applies warehouse masking policies to those columns

The actual masking SQL and policies are defined in a separate infrastructure/warehouse repo — don't modify them here.

## Applying PII Tags

In the model's YAML file, add `config.meta.pii_tag` to any PII column. Also add the catalog classification so the column is discoverable in the data catalog:

```yaml
models:
  - name: stg_hubspot_contacts
    columns:
      - name: email
        description: "{{ doc('col_email') }}"
        config:
          meta:
            pii_tag: pii_email
            atlan:
              classificationNames:
                - PII

      - name: first_name
        description: "{{ doc('col_first_name') }}"
        config:
          meta:
            pii_tag: pii_string
            atlan:
              classificationNames:
                - PII

      - name: date_of_birth
        description: "{{ doc('col_date_of_birth') }}"
        config:
          meta:
            pii_tag: pii_temporal
            atlan:
              classificationNames:
                - PII
```

The global post-hook in `dbt_project.yml` reads the `pii_tag` values and applies warehouse column-level masking tags automatically — **do not add `post_hook` to individual model configs**.

### Which tag to use

| `pii_tag` value | Use for | Masked as |
|---|---|---|
| `pii_email` | Email addresses | `********@****.***` |
| `pii_string` | Names, phone numbers, addresses, demographics | `********` |
| `pii_temporal` | Date of birth, sensitive dates | `0001-01-01` |
| `pii_numeric` | Age, GPS coordinates | `0` |
| `pii_salary` | Salaries, pay grades | `********` |
| `pii_trackers_string` | Subscription IDs, asset IDs, barcodes | `********` |

The full tag reference with examples is in `macros/apply_pii_tags.sql` in the dbt project repo.

The global `dbt_project.yml` post-hook calls `apply_pii_tags()` on every model — no need to add it per model.

## Profile Safety

**Always make sure your default dbt profile is a non-PII dev profile.**

In `~/.dbt/profiles.yml`:
- The `target:` key sets your default profile
- Default must be a profile with `dev` in the name (e.g., `dev-non-pii`)
- Use `dbt run --target dev-non-pii` explicitly when uncertain

If you run a model against the PII production target during development, you may expose real PII in development schemas.

## Warehouse MCP Warning

When using the warehouse MCP (e.g., in Windsurf or Claude Code), be mindful:
- Queries run directly against the warehouse and may return PII data
- Avoid querying production schemas with PII unless strictly necessary
- Don't paste PII data into prompts or share it in AI context

## Development-Only WHERE Clauses & PII

Development WHERE clauses limit the date range of data loaded in dev:

```sql
{% if target.name is not none and 'dev' in target.name %}
    WHERE created_at >= DATEADD('year', -1, CURRENT_DATE())
{% endif %}
```

This reduces PII exposure in development schemas by limiting data volume. Combine with a non-PII profile for full protection.

## Reverse-ETL & PII

Reverse-ETL models (feeding external destinations via Hightouch) must use dbt contracts:
- All columns declared in YAML with explicit `data_type`
- `contract: enforced: true` on the model

This prevents accidental PII column additions silently flowing to external systems. See `dbt-data-quality` skill for full contract rules.
