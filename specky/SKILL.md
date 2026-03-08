---
name: specky
description: >
  Spec-driven development lifecycle automation using GitHub Issues and Claude Code.
  Use this skill whenever a user mentions specs, issues, feature development
  lifecycle, executing a spec, planning tasks, committing against a spec, bugs,
  or any combination of GitHub Issues + development automation. Always invoke this
  skill when the user is working with GitHub Issues and wants AI to drive any part
  of the dev lifecycle.
compatibility:
  tools:
    - bash
    - computer
  requires:
    - gh CLI installed and authenticated (`gh auth status`)
---

# Spec-Driven Development

This skill automates the full development lifecycle — from GitHub issue to merged PR —
using AI sub-agents.

## Architecture Overview

```
specky-spec  -->  specky-plan  -->  specky-execute  -->  specky-pr
                                         ^
specky-bugs  ----  (fix mode) -----------+
```

Each sub-skill is self-contained and can be invoked independently.

## Sub-Skills

Read the relevant SKILL.md before invoking any sub-skill:

| Sub-Skill | File | Purpose |
|---|---|---|
| `specky-spec` | `specky-spec/SKILL.md` | Create, fetch, or fill out spec issues |
| `specky-plan` | `specky-plan/SKILL.md` | Research context and decompose spec into stages |
| `specky-execute` | `specky-execute/SKILL.md` | Implement stages via sub-agents, or apply targeted fixes |
| `specky-pr` | `specky-pr/SKILL.md` | Create PR linked to issue, finalize after merge |
| `specky-bugs` | `specky-bugs/SKILL.md` | List, prioritize, and triage bug issues |

## Shared Conventions

Read `_shared/references/conventions.md` for:
- Issue label lifecycle
- Plan storage format (embedded in issue body)
- Branch and commit message formats
- gh CLI command reference

## Prerequisites

The `gh` CLI must be installed and authenticated:

```bash
gh auth status
```

If not authenticated, run `gh auth login` first.

## Zero Local State

All state lives in GitHub Issues. The execution plan is stored as a fenced code block
(` ```specky-plan `) in the issue body, so no local cache directory is needed. This means
specky works across machines and team members without syncing local files.

## CLAUDE.md Integration

Sub-agents automatically read the project's `CLAUDE.md` for testing commands, build steps,
and code conventions. No additional configuration is needed — specky adapts to whatever
tooling the project uses.
