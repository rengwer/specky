# Specky — Spec-Driven Development Skills

This repository contains Claude Code skills for spec-driven development automation using GitHub Issues.

## Skills

The `specky` skill automates the full development lifecycle. Sub-skills are in `specky/`:

| Sub-Skill | Purpose |
|---|---|
| `specky-spec` | Create, fetch, or fill out spec issues (structured interview) |
| `specky-plan` | Research context and decompose spec into executable stages |
| `specky-execute` | Implement stages via parallel sub-agents, or apply targeted fixes |
| `specky-pr` | Create PR linked to issue, finalize after merge |
| `specky-bugs` | List, prioritize, and triage bug issues |

## Key Conventions

See [specky/_shared/references/conventions.md](specky/_shared/references/conventions.md) for full details.

- **Branch format:** `spec/{slug}` (kebab-case of spec title)
- **Commit format:** `[spec-{id}] stage-{n}: {description}`
- **Fix commit format:** `[spec-{id}] fix: {description}`
- **PR title:** `Spec #{id}: {issue title}`
- **Plan storage:** embedded in issue body as ` ```specky-plan ` fenced code block

## Prerequisites

Requires the `gh` CLI installed and authenticated:

```bash
gh auth status
```

## Zero Local State

All state lives in GitHub Issues. No local cache directory is needed.
Sub-agents read the project's `CLAUDE.md` for testing commands and conventions.
