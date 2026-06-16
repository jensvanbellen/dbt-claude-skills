---
name: dbt-cross-repo-workflow
description: Guides working across the data repos simultaneously — your_dbt_project, airflow, and reverse-etl_hightouch. Use when a task spans multiple repos, such as adding a new pipeline end-to-end, modifying a reverse-ETL sync, or setting up a new Airflow DAG for a dbt job.
allowed-tools: "Read, Write, Edit, Glob, Grep, Bash(git *)"
user-invocable: true
metadata:
  author: your-org
---

# Cross-Repo Workflow

Many data tasks span more than one repo. This skill keeps you from making changes in the wrong order, forgetting to update a linked file, or creating a hard-to-debug inconsistency between repos.

## The Three Repos

| Repo | What lives here |
|---|---|
| `your_dbt_project` | Model SQL, YAML, jobs config, governance rules |
| `airflow` | Scheduling DAGs — when and in what order things run |
| `reverse-etl_hightouch` | Hightouch model definitions and sync column mappings |

All three repos should be cloned into the same parent folder. This lets you reference all of them in a single AI session.

## Reference Guides

| Guide | When to read |
|---|---|
| [references/common-scenarios.md](references/common-scenarios.md) | Which repos to touch for common tasks (new pipeline, new column, schedule change) |
| [references/airflow-dag-creation.md](references/airflow-dag-creation.md) | Step-by-step for creating a new Airflow DAG from scratch |
| [references/pr-and-merge-order.md](references/pr-and-merge-order.md) | Branch naming, PR linking, and merge order rules |

## How to Set Up an AI Session for Cross-Repo Work

When asking an AI agent (Claude, Windsurf, etc.) to make changes across multiple repos:

1. **Tell the agent explicitly which repos are in scope**:
   > "I need changes in both `/path/to/your_dbt_project` and `/path/to/airflow`"

2. **Give it the exact path** of the repo it's NOT currently working in — agents default to the current working directory and won't look elsewhere unless told

3. **Ask it to create branches in both repos** before making any changes:
   > "Create a branch `feature/my-pipeline` in both repos before starting"

4. **Ask for linked PRs**: once both PRs exist, ask the agent to add the other PR's link to each PR description

5. **Confirm merge order** before merging either PR — order matters (see `pr-and-merge-order.md`)

## Quick Decision: Which Repos Does My Task Touch?

| Task | Repos |
|---|---|
| Build a new model (no scheduling) | `your_dbt_project` only |
| Build a new model + add it to an existing job + that job already has a DAG | `your_dbt_project` only |
| Build a new model that needs its own new dbt job + scheduling | `your_dbt_project` + `airflow` |
| Add a column to a destination sync | `your_dbt_project` + `reverse-etl_hightouch` |
| Add a brand-new reverse-ETL pipeline | `your_dbt_project` + `reverse-etl_hightouch` + `airflow` |
| Change schedule of an existing pipeline | `airflow` only |
| Change a dbt job's steps | `your_dbt_project` (jobs/dbt_jobs.yml) only |

## STOP Conditions

Before starting cross-repo work:
- Confirm all relevant repos are cloned locally
- Confirm you know which repos need changes (use the table above)
- Confirm you know the merge order before opening PRs
