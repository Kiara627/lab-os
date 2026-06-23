# Kiara's dev home — workspace orientation

Working from the workspace root (not just inside a single repo) is intentional — it's where
cross-project coordination happens: multi-repo planning, workspace-wide tooling and rules,
cross-repo logs. Per-repo `CLAUDE.md` cascades when a session works inside a sub-project;
nothing is lost by having the broader view available.

## Project lineage

1. **`job-curator`** — Job Application Curator; first full-stack project under this dev home.
   FastAPI/React/Postgres. Establishes the AI-assisted tooling pattern used in later CAMELS work.

## Active or foundational repos

| Repo | Role | Status |
|---|---|---|
| `job-curator` | AI-assisted full-stack job search app (FastAPI/React/Postgres) | Active — early build |

Nested project repos live at `~/Development/lab-os/<project>/`, each its own git repo,
gitignored from the fork. Plans and backlog track in the fork itself (see Logs and tracking).

## Tooling

- **`lab-os`** — spec-driven-development conventions. This dev home **is** your fork of
  lab-os, so its `.claude/rules/` live here natively; `git pull upstream main` pulls rule
  updates. Treat these as your starting standards — adapt or extend as your project diverges.
  Source-of-truth: github.com/CAMELS-Research-Group/lab-os.
- **`lab-claude-plugins`** — Claude Code plugins installed during setup (e.g. `pr-review-loop`),
  via the plugin marketplace. Source-of-truth: github.com/CAMELS-Research-Group/lab-claude-plugins.
- **`superpowers`** — workflow plugins from the official Claude Code marketplace (brainstorming,
  planning, TDD, subagent-driven execution).

As your project grows, list your own repos and any further tooling here so a session at your
dev home sees the whole workspace.

## Logs and tracking

- **Plans / backlog:** `~/Development/lab-os/_plans/` — tracked by the fork, alongside the
  rules and log
- **Per-repo logs:** `<repo>/project_log.md`
- **Workspace-level decisions** (cross-repo tooling, infra, workspace-wide conventions):
  `~/Development/lab-os/project_log.md`
- **Cost tracking** (inference spend, infra): `~/Development/lab-os/cost-tracking.md`

Entry format defined in your global `~/.claude/CLAUDE.md`.

## Approval gates

Defined in your global `~/.claude/CLAUDE.md`. Cross-cutting items at this level:

- External-facing posts (PRs, issue comments, anything under your name; bot identity OK,
  user identity not)
- Cloud spend above your stated ceiling
- Data exposure risks (raw sensitive or restricted data; derived artifacts need PII review
  per `.claude/rules/02-data-protection.md`)
- Destructive operations on shared state
