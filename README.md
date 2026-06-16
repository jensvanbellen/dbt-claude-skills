# dbt Claude Skills — Tribal Knowledge as an Agent

Your dbt team has unwritten rules. Patterns that took years to learn. Mistakes that caused incidents. Conventions that only exist in Slack threads and one engineer's memory.

This plugin encodes those into Claude Code skills — so every engineer on your team, regardless of experience, gets the same institutional knowledge in their AI context.

## What This Is

A Claude Code skills plugin with five domain skills covering the full lifecycle of dbt work: from model standards to pipeline setup to data quality to reverse-ETL to cross-repo coordination.

Each skill is a structured prompt that Claude loads when it detects you're doing that type of work. The `references/` folders inside each skill contain the actual conventions — think of them as a living CLAUDE.md that's domain-scoped and auto-injected at the right moment.

## Skills

### `dbt-standards`
Enforces naming conventions, config blocks, YAML format, SQL style, PII tagging, and tag governance. Activates automatically when working on any dbt model. It's the one that stops Claude from doing subtly wrong things that pass review but cause incidents later.

**Includes references for:** naming conventions, model config rules, SQL style guide, PII handling

### `dbt-pipeline-setup`
Step-by-step workflow for adding a model to a job, creating new jobs in `dbt_jobs.yml`, and wiring up Airflow scheduling. Covers `dbt build` vs `dbt run` tradeoffs, performance setup for large models, and the tag governance workflow.

**Trigger:** "I want to set up a new pipeline" / "how do I add this to a job?" / "I need to schedule this"

**Includes references for:** jobs config workflow, Airflow orchestration patterns

### `dbt-data-quality`
Guides the correct testing approach by layer, severity decisions (`warn` vs `error`), the dbt vs Monte Carlo division of responsibility, and data contracts for externally-consumed models. Prevents both undertesting and duplicate monitoring.

**Trigger:** "What tests should I add?" / "Should I use warn or error here?" / "How do I add a contract?"

**Includes references for:** testing by layer, dbt vs Monte Carlo, data contracts

### `dbt-reverse-etl`
End-to-end guide for reverse-ETL pipelines: dbt model structure with contracts, Hightouch sync YAML column mappings, Airflow orchestration, and multi-repo coordination. Covers adding/removing columns, renaming fields safely, and debugging sync errors.

**Trigger:** "I want to add a column to the HubSpot sync" / "How do I build a new reverse-ETL model?" / "The Hightouch sync is failing"

**Includes references for:** dbt model pattern, Hightouch sync YAML, Airflow orchestration, multi-repo workflow

### `dbt-cross-repo-workflow`
Guides working across dbt, Airflow, and reverse-ETL repos simultaneously. Covers which repos to touch for each scenario, step-by-step guide to creating a new Airflow DAG, branch naming, PR linking, and merge order (order matters — getting it wrong breaks syncs).

**Trigger:** "I need to create an Airflow DAG" / "This change spans multiple repos" / "What order do I merge these PRs?"

**Includes references for:** common scenarios, Airflow DAG creation, PR and merge order

## Installing

In any Claude Code session, run:

```
/install your-org/dbt-claude-skills
```

Claude Code fetches the repo using your `gh` credentials and symlinks the skills into `~/.claude/skills/`. Skills are available immediately in all projects.

To update after new commits land:

```
/install your-org/dbt-claude-skills
```

Re-running `/install` overwrites the local copies with the latest version.

## Architecture

```
skill = trigger logic + prompt framing
references/ = the actual knowledge base
```

Each skill has two parts:

**`SKILL.md`** — the trigger and framing layer. It tells Claude when to load the skill, what the non-negotiable rules are, and which reference files to read for deeper guidance. Keep this concise — it's always in context.

**`references/*.md`** — the detailed knowledge base. These are read on-demand when Claude hits that type of work. They can be long and specific — patterns, code templates, decision trees, checklists.

The skill activates, reads the relevant references, and applies them. The result is an agent that behaves like your most experienced engineer was in the room.

## Adapting for Your Team

The `references/` folders are where your team's actual conventions live. Replace them with yours.

**What to customize:**

| File | What to replace |
|---|---|
| `dbt-standards/references/naming-conventions.md` | Your layer naming convention, YAML file prefixes, field naming rules |
| `dbt-standards/references/model-config-rules.md` | Your tag governance rules, schema conventions, config block requirements |
| `dbt-standards/references/sql-style-guide.md` | Your SQL style rules — or keep these as-is, they're fairly universal |
| `dbt-standards/references/pii-and-security.md` | Your PII tagging mechanism, masking policies, profile safety rules |
| `dbt-pipeline-setup/references/jobs-config-workflow.md` | Your `dbt_jobs.yml` structure, CI job steps, tag governance files |
| `dbt-pipeline-setup/references/airflow-orchestration.md` | Your DAG patterns, connection IDs, dbt Cloud project/account IDs |
| `dbt-data-quality/references/testing-by-layer.md` | Your test requirements per layer — adjust severity defaults to match your team's bar |
| `dbt-data-quality/references/dbt-vs-monte-carlo.md` | If you use a different observability tool, replace Monte Carlo references |
| `dbt-data-quality/references/contracts.md` | Which models require contracts in your project |
| `dbt-reverse-etl/references/dbt-model-pattern.md` | Your reverse-ETL model directory structure, config requirements |
| `dbt-reverse-etl/references/hightouch-sync-yaml.md` | Your Hightouch workspace, sync patterns, destination-specific config |
| `dbt-reverse-etl/references/airflow-orchestration.md` | Your ELT chain patterns, reverse-ETL DAG naming |
| `dbt-cross-repo-workflow/references/common-scenarios.md` | Adjust repo names and paths to match your setup |
| `dbt-cross-repo-workflow/references/airflow-dag-creation.md` | Your existing DAG schedule slots, Airflow UI location |
| `dbt-cross-repo-workflow/references/pr-and-merge-order.md` | Your review process, team ownership rules |

**What to keep:**
- The skill trigger descriptions in `SKILL.md` (adjust wording to match your team's vocabulary)
- The overall structure — it maps well to how most dbt teams work
- The SQL style guide — most of it is universal Snowflake/dbt best practice

**How to maintain it:**
When Claude gets something wrong, update the relevant reference file and open a PR. The skills are documentation that happens to be read by an AI. The same people who maintain your CLAUDE.md should maintain these.

## How These Skills Relate to Official Skills

```
Official: dbt:using-dbt-for-analytics-engineering  ← generic dbt execution
Custom:   dbt-standards                             ← your rules on top

Official: mc-agent-toolkit:prevent                  ← MC context before edits
Custom:   dbt-data-quality                          ← your testing patterns + contracts

Official: mc-agent-toolkit:monitor-creation         ← generic MC monitor creation
Custom:   dbt-data-quality                          ← when/why to create monitors
```

Don't replace the official skills — use both together. The official skills handle generic dbt knowledge; these handle your team's specific conventions.

## Contributing

When you find a pattern Claude gets wrong, update the relevant reference file and open a PR. The skills are living documentation.

## Versioning

This repo uses semantic versioning. The current version is reflected in `tile.json`.

| Change | Version bump |
|---|---|
| New skill added | `minor` |
| Existing skill updated (rules, references) | `patch` |
| Skill removed or renamed | `major` |
