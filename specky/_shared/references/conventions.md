# Shared Spec Conventions

## Issue Label Lifecycle

Labels are additive — a spec accumulates labels as it progresses.

| Label | Meaning |
|---|---|
| `spec` | This is a spec issue (always present) |
| `spec:planned` | Execution plan created, task-list appended to issue body |
| `spec:executing` | Sub-agents actively working |
| `spec:in-review` | PR open, awaiting merge |
| `bug` | Bug report (handled by bugs/execute fix flow) |
| `bug:triaged` | Bug triaged, root cause identified |

Issue closure (via PR merge with `Closes #id`) is the completion signal.

## Issue Status Mapping

| Phase | GitHub State | Labels |
|---|---|---|
| Created | `open` | `spec` |
| Planned | `open` | `spec`, `spec:planned` |
| Executing | `open` | `spec`, `spec:executing` |
| In Review | `open` | `spec`, `spec:in-review` |
| Done | `closed` | *(auto-closed by PR merge)* |

## Plan Storage

The execution plan is stored directly in the GitHub issue body in two forms:

1. **Human-readable task-list** — checkboxes in the `## Execution Plan` section
2. **Machine-readable YAML** — fenced code block with `specky-plan` info string:

~~~
```specky-plan
spec_id: 123
planned_at: "2026-03-08"
stages:
  - stage: 1
    title: Database migration
    description: Create users table with JWT columns
    parallel: false
    depends_on: []
    acceptance_criteria:
      - Table exists with correct schema
      - Migrations are reversible
    status: pending
```
~~~

Stage `status` values: `pending`, `complete`, `blocked`, `skipped`

The fenced block is visible to collaborators and parseable by `specky-execute`.
No local cache is needed — GitHub is the sole source of truth.

## Branch and Commit Naming

- **Branch:** `spec/{slug}` where `{slug}` is a kebab-case slug derived from the spec title (e.g., `spec/user-auth-jwt`, `spec/payment-gateway-integration`)
- **Commit:** `[spec-{id}] stage-{n}: {description}` (e.g., `[spec-123] stage-2: add JWT middleware`)
- **Fix commit:** `[spec-{id}] fix: {description}` or `[spec-{id}][bug-{bug-id}] fix: {description}`
- **PR title:** `Spec #{id}: {issue title}`

## CLAUDE.md Awareness

Sub-agents read the project's `CLAUDE.md` (if present) to pick up:
- Testing framework and commands
- Build and verification steps
- Code style and conventions
- Project-specific instructions

This means specky automatically adapts to each project's tooling without configuration.

## gh CLI Commands by Sub-Skill

| Sub-Skill | Key Commands |
|---|---|
| specky-spec | `gh issue create`, `gh issue list`, `gh issue view`, `gh issue edit`, `gh label create` |
| specky-plan | `gh issue view`, `gh issue edit`, `gh issue list --search`, `gh search code`, `gh label create` |
| specky-execute | `gh issue view`, `gh issue edit`, `gh issue comment`, `gh label create` |
| specky-pr | `gh pr create`, `gh pr list`, `gh issue edit`, `gh issue comment`, `gh label create` |
| specky-bugs | `gh issue list --label bug`, `gh issue view`, `gh issue edit`, `gh issue comment`, `gh label create` |
