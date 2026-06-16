# SQL Style Guide

## Formatting

- SQL **keywords**: `UPPER CASE` (SELECT, FROM, WHERE, JOIN, GROUP BY, etc.)
- Column names: `snake_case` (lowercase with underscores)
- Always leave a **new empty line at the end of every file**
- Use **CTEs** over subqueries — always

```sql
-- Correct
WITH source AS (
    SELECT * FROM {{ source('hubspot', 'contacts') }}
),

renamed AS (
    SELECT
        id AS contact_id,
        created_at
    FROM source
)

SELECT * FROM renamed

-- Wrong: subquery
SELECT *
FROM (
    SELECT id AS contact_id FROM {{ source('hubspot', 'contacts') }}
) AS renamed
```

## References

- **Always** use `{{ ref('model_name') }}` for dbt models — never raw schema/table names
- **Always** use `{{ source('source_name', 'table_name') }}` for raw sources
- Raw table names break lineage and CI

## SELECT

- **No `SELECT *`** except `SELECT * FROM final` at the end of a CTE chain
- Explicit column selection forces intentionality and avoids scanning unnecessary data in the warehouse
- Pulling wide upstream models carelessly scans everything — be explicit

```sql
-- Correct
SELECT
    contact_id,
    email,
    created_at
FROM contacts

-- Wrong
SELECT * FROM contacts

-- Correct (only exception)
SELECT * FROM final
```

## GROUP BY

- **Never** `GROUP BY 1, 2` — use explicit column names or `GROUP BY ALL`

```sql
-- Correct
GROUP BY contact_id, created_date

-- Also correct (Snowflake-specific, acceptable)
GROUP BY ALL

-- Wrong
GROUP BY 1, 2
```

## Aliases

- Use **clear, descriptive aliases**
- **No aliases** when there are no JOINs — unnecessary noise

```sql
-- Correct: aliases with joins
SELECT
    c.contact_id,
    s.subscription_status
FROM contacts AS c
LEFT JOIN subscriptions AS s ON c.contact_id = s.contact_id

-- Correct: no aliases without joins
SELECT
    contact_id,
    email
FROM contacts
```

## Comments

- Add **meaningful comments**: explain the *why* and *how*, not the *what*
- Column names and SQL are the *what* — comments add the context behind decisions
- No commented-out code: delete it. History lives in GitHub.

```sql
-- Correct: explains why
-- Use the most recent status since status_name reflects current state, not historical
QUALIFY ROW_NUMBER() OVER (PARTITION BY contact_id ORDER BY updated_at DESC) = 1

-- Wrong: explains what (obvious)
-- Select contact_id and email
SELECT contact_id, email
```

## Deduplication

- `SELECT DISTINCT` is a **red flag** — it usually masks a JOIN producing duplicates
- Fix the JOIN logic first; if deduplication is intentional, use `ROW_NUMBER() + QUALIFY`

```sql
-- Correct: controlled deduplication
SELECT *
FROM my_table
QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) = 1

-- Avoid unless you know exactly why
SELECT DISTINCT id, email FROM contacts
```

## JOINs

- **Avoid OR conditions in JOINs** — the warehouse can't use partition pruning on them
- Replace with `COALESCE` so the join key is always a single equality check

```sql
-- Wrong: OR join
JOIN other ON a.id = other.id OR a.fallback_id = other.id

-- Correct: COALESCE join
JOIN other ON COALESCE(a.id, a.fallback_id) = other.id
```

- **CROSS JOIN is forbidden** — always use an explicit `ON` condition to control which rows are combined
- If you need every date paired with every availability record, use a range join instead:

```sql
-- Wrong: CROSS JOIN
FROM calendar AS cal
CROSS JOIN availability AS avail

-- Correct: range join — only pairs rows where the date falls within the validity window
FROM availability AS avail
INNER JOIN calendar AS cal
    ON cal.date >= avail.period_start
        AND cal.date < avail.period_end
```

- If you genuinely cannot avoid a CROSS JOIN, you need explicit sign-off — document the exact expected row count in a comment and get a second pair of eyes

## Filters

- **Push filters early** — apply WHERE clauses inside the first CTE, not at the end
- Early filters reduce data scanned across all subsequent CTEs

```sql
-- Correct: filter early
WITH filtered_contacts AS (
    SELECT contact_id, email
    FROM {{ source('hubspot', 'contacts') }}
    WHERE is_deleted = FALSE
),
...

-- Wrong: filter at the end
WITH all_contacts AS (
    SELECT contact_id, email
    FROM {{ source('hubspot', 'contacts') }}
),
...
SELECT * FROM all_contacts WHERE is_deleted = FALSE
```

## Macros

- If you write the same logic twice, extract it into a macro
- Reference the macro everywhere after that

## ANSI SQL

- Prefer ANSI SQL over warehouse-specific syntax where both work
- Use `<>` not `!=` for not-equals (ANSI standard)

## Performance Red Flags — Stop and Reconsider

| Pattern | Problem | Fix |
|---|---|---|
| `SELECT *` on wide upstream | Forces full column scan | Be explicit |
| `SELECT DISTINCT` | Hides bad JOIN logic | Fix the JOIN, use ROW_NUMBER |
| `CROSS JOIN` | Silently explodes row count | Forbidden — use explicit `JOIN ... ON` or a range join |
| OR in JOIN condition | No partition pruning | Use COALESCE |
| CTE referenced many times | Re-executes every time | Extract into an upstream model |
| Filters at end of query | Scans all data first | Push filters to first CTE |

## The CTE Rule

A CTE is **not** a cached result in most warehouses — it re-executes every time it's referenced downstream. If a heavy CTE (joining large tables, wide aggregations) is referenced more than once, extract it into its own upstream intermediate model.
