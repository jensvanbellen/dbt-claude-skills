# Naming Conventions

## Models

| Layer | Convention | Example |
|---|---|---|
| Staging | `stg_<source>_<entity>` | `stg_hubspot_contacts` |
| Intermediate | `int_<description>` | `int_repairs_actions` |
| Marts | `<descriptive_name>` (no prefix) | `exchange_rates` |
| Snapshots | `snp_<entity>_snapshot` | `snp_hubspot_waiting_list` |
| Seeds | `<description>` | `ref_product_categories` |
| Macros | `<verb_or_noun>_<description>` | `apply_pii_tags` |
| Tests | `test_<condition>` | `test_not_null_percentage` |

## YAML Files

| Layer | Convention | Example |
|---|---|---|
| Sources | `_src_<source>.yml` | `_src_hubspot.yml` |
| Staging | `_stg_<source>.yml` | `_stg_hubspot.yml` |
| Intermediate | `_int_<source>.yml` | `_int_stripe.yml` |
| Marts | `_mrt_<domain>.yml` | `_mrt_marketing.yml` |

**Always prefix YAML and MD files with `_`.** This keeps them at the top of the directory, above SQL files.

One YAML per directory (normal dirs). Source directories need **both** `_src_<source>.yml` and `_stg_<source>.yml`.

## Fields

- All lowercase, `snake_case`
- Primary key: `_id` suffix → `customer_id`
- Foreign key: `_id` suffix → `subscription_id`
- Timestamps: `_at` suffix → `created_at`
- Dates: `_date` or `_on` suffix → `contract_start_date`
- Booleans: `is_` prefix → `is_deleted`
- Metadata: `load_date_time`

**Never use reserved SQL keywords as column names** (e.g., `year`, `date`, `value`, `name`).

## Schemas

- Lowercase, source-related: `stripe` → `analytics_stripe` in the warehouse
- The `analytics_` prefix is applied automatically via `profiles.yml`
- Models in the `analytics_reporting` schema don't need `schema =` in config
- All other schemas **must** declare `schema = '<source>'` in the config block

## Documentation References

- Table descriptions: `{{ doc("table_<name>") }}` in the YAML, defined in `<name>.md`
- Column descriptions: `{{ doc("col_<name>") }}` in the YAML, defined in `<name>.md`
- MD files get the same `_` prefix and live alongside their YAML
- **Every model must have a `description` using a `{{ doc("table_<name>") }}` block** — never use inline strings
- **Every column must have a `description` using a `{{ doc("col_<name>") }}` block** — never use inline strings
- If no matching `table_<name>` doc block exists in `docs/`, add one to the most relevant `.md` file in `docs/` before writing the YAML
- If no matching `col_<name>` doc block exists in `docs/`, add one to the most relevant `.md` file in `docs/` before writing the YAML
- For shared/general columns, add to `docs/general_columns.md`; for domain-specific ones, add to the relevant domain file

## YAML Ordering

- **Models within a YAML file must be listed alphabetically** by model name
- **Columns within each model must follow the SQL SELECT output order** (not alphabetical — mirrors the query for easy cross-referencing)

## Directories

| dbt Layer | Directory | Orientation |
|---|---|---|
| Staging | `models/staging/<source>/` | Source-oriented |
| Intermediate | `models/intermediate/<source>/` | Source-oriented (WIP, not always in place) |
| Marts | `models/marts/<domain>/` | Business domain-oriented |

**Staging is source-oriented, marts are domain-oriented.** This is intentional — do not mix them.
