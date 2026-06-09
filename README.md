# lab-rules

Cross-repo conventions for `WatsonWBlair`'s lab repos.

## Working with the lab (Claude Code setup)

Two repos bootstrap a lab Claude Code environment:

1. **Conventions** (this repo) — clone and junction it so Cowork sees the lab rules (see [How repos consume it](#how-repos-consume-it) below).
2. **Tooling** — the lab's Claude Code plugins live in [`lab-claude-plugins`](https://github.com/WatsonWBlair/lab-claude-plugins). Add the marketplace and install what you need:

   ```
   /plugin marketplace add WatsonWBlair/lab-claude-plugins
   /plugin install pr-review-loop@lab-claude-plugins
   ```

## What's here

- `.claude/rules/` — markdown files defining lab-wide conventions. Consumed by Cowork locally (via a junction at `Development\.claude\rules\`) and by the PR-review GitHub Action at review time.

## How repos consume it

**Locally (Cowork)**: clone this repo to `C:\Users\watso\Development\lab-rules\`. Create a Windows junction so Cowork sees it at the Development root:

```powershell
cmd /c mklink /J "C:\Users\watso\Development\.claude\rules" "C:\Users\watso\Development\lab-rules\.claude\rules"
```

No admin required.

**In CI (PR reviewer)**: each lab repo's `.github/workflows/pr-review.yml` checks this repo out alongside the PR repo:

```yaml
- uses: actions/checkout@v4
  with: { path: pr-repo }
- uses: actions/checkout@v4
  with:
    repository: WatsonWBlair/lab-rules
    path: lab-rules
```

The reviewer then concatenates `lab-rules/.claude/rules/*.md` + `pr-repo/.claude/rules/*.md` into its prompt context.

## Override semantics

Per-repo rules extend or override lab rules. Specific wins over general. A per-repo `02-data-protection.md` listing the repo's specific gated datasets supplements the lab-wide PII checklist; a per-repo rule contradicting a lab rule applies only in that repo.

## Scope discipline

This repo holds **hard rules** — commit format, PR template usage, data-protection invariants, security, approval gates. Soft conventions (code style, library preferences) stay per-repo. Audit periodically to keep this lean — every file here is loaded into every Cowork session and every PR review.
