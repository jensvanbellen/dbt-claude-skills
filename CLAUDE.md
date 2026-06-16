# dbt Claude Skills — AI Agent Instructions

## What This Is

A set of **Claude Code skills** that encode dbt conventions, pipeline workflows, and data quality standards into reusable agent instructions. Skills are SKILL.md files that Claude Code loads as context when invoked — they tell the AI what rules to follow, what to check, and what to refuse, without requiring the engineer to repeat themselves each session.

This repo is designed to be **forked and customized**. Replace `your_dbt_project`, `your-org`, and placeholder references with your actual project names. The structure and patterns transfer; the specifics are yours to fill in.

## Skills in This Repo

| Skill | Purpose |
|---|---|
| `dbt-standards` | Naming conventions, config block rules, YAML format, SQL style, PII governance |
| `dbt-pipeline-setup` | End-to-end pipeline scaffolding: staging model → dbt job → Airflow DAG |
| `dbt-data-quality` | Data test strategy, Monte Carlo vs dbt tests, contracts, testing by layer |
| `dbt-reverse-etl` | Reverse ETL model pattern (Hightouch), multi-repo workflow, Airflow orchestration |
| `dbt-cross-repo-workflow` | Cross-repo change management: PR ordering, dbt + Airflow dependency chains |

Skills are **non-invocable by default** (`user-invocable: false`) — they are loaded by other skills or by CLAUDE.md `@` imports, not typed directly by the user.

## Repository Structure

```
skills/
├── dbt-standards/
│   ├── SKILL.md              # Core rules — loaded by all other skills
│   └── references/           # Detailed guides linked from SKILL.md
│       ├── naming-conventions.md
│       ├── model-config-rules.md
│       ├── sql-style-guide.md
│       └── pii-and-security.md
├── dbt-pipeline-setup/
│   ├── SKILL.md
│   └── references/
│       ├── airflow-orchestration.md
│       └── jobs-config-workflow.md
├── dbt-data-quality/
│   ├── SKILL.md
│   └── references/
│       ├── testing-by-layer.md
│       ├── dbt-vs-monte-carlo.md
│       └── contracts.md
├── dbt-reverse-etl/
│   ├── SKILL.md
│   └── references/
│       ├── dbt-model-pattern.md
│       ├── hightouch-sync-yaml.md
│       ├── airflow-orchestration.md
│       └── multi-repo-workflow.md
└── dbt-cross-repo-workflow/
    ├── SKILL.md
    └── references/
        ├── pr-and-merge-order.md
        ├── airflow-dag-creation.md
        └── common-scenarios.md

tile.json   # Claude Code skill registry metadata
```

## How to Install in Your dbt Project

### Option A: Import via CLAUDE.md

In your dbt project's `CLAUDE.md`, add `@` imports pointing to the skill files:

```markdown
@path/to/dbt-claude-skills/skills/dbt-standards/SKILL.md
@path/to/dbt-claude-skills/skills/dbt-pipeline-setup/SKILL.md
```

Claude Code loads these files into every session automatically.

### Option B: Publish as a tile and install via CLI

```bash
# From this repo root
claude mcp add-tile tile.json   # registers the skill tile
```

Then reference skills from your project's CLAUDE.md by name.

## Customizing for Your Project

Every skill file contains placeholder strings that must be replaced before use. Search and replace:

| Placeholder | Replace with |
|---|---|
| `your_dbt_project` | Your actual dbt project name |
| `your-org` | Your org name (GitHub, dbt Cloud) |
| `airflow` | Your Airflow environment name if different |
| `reverse-etl_hightouch` | Your Hightouch workspace or reverse ETL tool |

Reference files under `references/` may contain additional project-specific examples — review them all before publishing to your team.

## Design Principles

**Skills enforce, not suggest.** Every SKILL.md includes explicit STOP conditions — cases where the agent must halt and ask the engineer rather than guessing. Ambiguous conventions produce inconsistent code at scale; hard stops prevent drift.

**Reference files over long prompts.** Detailed rules live in `references/*.md` and are linked from the SKILL.md. This keeps the primary skill concise (faster to load, cheaper in context) while giving the agent a clear "go read this" path when it hits specific situations.

**Layered loading.** `dbt-standards` is the foundation — all other skills depend on it being present. Never invoke a workflow skill without also loading `dbt-standards`.

## Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md` with the standard frontmatter:
   ```yaml
   ---
   name: skill-name
   description: One sentence — what this skill does and when to load it.
   allowed-tools: "Read, Write, Edit, Glob, Grep"
   user-invocable: false
   metadata:
     author: your-org
   ---
   ```
2. Add reference guides under `skills/<skill-name>/references/` for anything too long for the SKILL.md.
3. Register it in `tile.json` under `skills`.
4. Import it from CLAUDE.md in your dbt project.
