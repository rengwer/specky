---
name: specky-execute
description: >
  Execute a planned spec using parallel sub-agents per stage, or apply a targeted fix.
  Use when a user says "execute this spec", "run spec X", "implement issue Y", "start coding
  the plan", "build this feature", "fix the bug in spec X", "patch this stage", or any request
  to implement a spec or apply a surgical fix. Requires a plan to exist (run specky-plan first).
  Spawns independent sub-agents per stage or parallel batch, commits against each stage, updates
  issue status, and runs acceptance tests. This is the primary coding automation skill.
compatibility:
  tools:
    - bash
    - computer
  requires:
    - gh CLI installed and authenticated (`gh auth status`)
---

# Execute — Implement Stages or Apply Fixes

Implement a planned spec by running each stage through dedicated sub-agents, or apply
a targeted fix to an existing implementation. Each sub-agent owns one stage: reads its
task, writes code, runs tests, commits, and reports back. The orchestrator tracks progress,
manages parallelism, and updates the GitHub issue throughout.

Read `../_shared/references/conventions.md` for commit naming and branch conventions.

## Prerequisites

- Spec issue must have an `## Execution Plan` section with embedded plan data (run `specky-plan` first)
- `gh` CLI authenticated

## Modes

- **"execute spec 123"** → Mode A (Full execution)
- **"fix spec 123" / "--fix"** → Mode B (Targeted fix)

---

## Mode A: Full Execution

### Orchestration Overview

```
Orchestrator (this skill)
+-- Reads plan, groups stages by dependency level
+-- For each dependency level (batch):
|   +-- Spawns one sub-agent per stage in the batch (parallel)
|   +-- Each sub-agent: reads task > codes > tests > commits > reports
|   +-- Waits for all in batch before proceeding to next level
+-- Updates issue task-list checkboxes throughout
+-- On completion: runs acceptance test agent
+-- Updates spec issue labels
```

### 1. Load plan and spec context

```bash
gh issue view {id} --json number,title,body,labels
```

Parse the plan YAML from the ` ```specky-plan ` fenced code block in the issue body.
Verify all stages have valid entries. If the plan block is missing, tell user to run `specky-plan`.

Also read the project's `CLAUDE.md` (if present) to understand:
- Testing framework and commands (e.g., `npm test`, `pytest`, `go test`)
- Build commands and verification steps
- Code style and conventions
- Any project-specific instructions

Sub-agents will receive relevant CLAUDE.md instructions for testing and verification.

### 2. Create or switch to feature branch

Derive the spec slug from the issue title (kebab-case, lowercase).

```bash
git checkout -b spec/{slug} 2>/dev/null || git checkout spec/{slug}
```

Add `spec:executing` label:
```bash
gh label create "spec:executing" --color D93F0B 2>/dev/null || true
gh issue edit {id} --add-label "spec:executing"
```

### 3. Group stages by dependency level

Parse the plan's `depends_on` arrays to build execution batches:

- **Batch 0**: stages with no dependencies
- **Batch 1**: stages whose dependencies are all in Batch 0
- etc.

Within each batch, stages marked `parallel: true` can run simultaneously.
Stages marked `parallel: false` run sequentially within the batch.

### 4. Execute each batch

For each batch, **spawn all parallel stages simultaneously** as sub-agents.

#### Sub-Agent Prompt Template

Each sub-agent receives this complete, self-contained prompt:

```
You are a focused implementation agent for one stage of a spec.

## Your Task
Spec ID: {id}
Stage: {n} of {total}
Stage Title: {stage.title}
Stage Description: {stage.description}

## Acceptance Criteria for This Stage
{stage.acceptance_criteria joined as bullet list}

## Full Spec Context
{issue body from gh issue view}

## Codebase Context
Current branch: spec/{slug}
Run `git log --oneline -10` and `git diff main...HEAD --name-only` to understand what's been done.

## Your Responsibilities
1. Read the codebase to understand existing patterns before writing any code
2. Implement ONLY what this stage requires -- do not touch other stages' concerns
3. Write tests as part of this stage (unit + integration as appropriate)
4. Run the tests and confirm they pass before committing
5. Commit with message: `[spec-{id}] stage-{n}: {stage.title}`
6. Report back a structured summary (see format below)

## Project Conventions
{relevant sections from CLAUDE.md — testing commands, build steps, code style}

## Constraints
- Stay within the scope of this stage -- leave other stages' files untouched
- Follow existing code conventions in the repo and CLAUDE.md
- Use the project's testing framework as specified in CLAUDE.md (default to the repo's test runner)
- If you encounter a blocker, report it -- do not guess
- Do NOT update the GitHub issue -- the orchestrator handles that

## Report Format (output this when done)
STATUS: success | blocked | partial
STAGE: {n}
FILES_CHANGED: [list of files]
TESTS_WRITTEN: [list of test files/names]
TESTS_PASSED: true | false
COMMIT_HASH: {hash}
NOTES: {any important context for downstream stages}
BLOCKER: {if STATUS=blocked, describe what's needed}
```

### 5. Process sub-agent reports

As each sub-agent completes, parse its report:

**On success:**
- Check off the task-list item in the issue body for this stage
- Comment: `gh issue comment {id} --body "Stage {n} complete. Commit: {hash}. Files: {files_changed}"`
- Update the plan YAML in the issue body's ` ```specky-plan ` block, setting stage `status` to `complete`

**On blocked:**
- Comment describing the blocker
- Pause execution of dependent stages
- Surface the blocker to the user with options:
  - **"Resolve now"** — user provides guidance, fix is applied inline (see Mode B below)
  - **"Skip stage"** — mark skipped, note in summary
  - **"Abort"** — stop execution, leave branch in current state

**On partial:**
- Treat as blocked — do not mark complete or advance dependents

### 6. Advance through batches

Only start the next batch when all stages in the current batch are `complete` or `skipped`.

Keep the user informed between batches:
```
Batch 1 complete: [Stage 1: DB migration, Stage 2: repo layer]
Starting Batch 2: [Stage 3: service logic]
```

### 7. Run acceptance tests

After all stages complete, spawn an **acceptance test sub-agent**:

```
You are a test audit agent for spec {id}.

## Original Spec
{issue body from gh issue view}

## What Was Built
{list of all FILES_CHANGED across all stage reports}

## Your Task
1. Review the acceptance criteria against the implemented code
2. Write end-to-end or integration tests that directly validate each criterion
3. Name test file: tests/spec-{id}-acceptance.{ext}
4. Run the tests and confirm they pass
5. Commit: [spec-{id}] acceptance tests

## Report
STATUS: passed | failed | partial
CRITERIA_COVERED: {n}/{total}
TEST_FILE: {path}
FAILING_CRITERIA: {list any not covered}
```

### 8. Update issue and finalize

```bash
gh label create "spec:in-review" --color FBCA04 2>/dev/null || true
gh issue edit {id} --add-label "spec:in-review"
gh issue comment {id} --body "Execution complete. All stages done. Ready for PR."
```

If acceptance tests flag gaps, surface them and offer to fix inline (Mode B).

### 9. Summarize for user

```
Spec #{id} execution complete.

Stages:  6/6 complete
Tests:   Acceptance criteria 4/4 covered
Branch:  spec/user-auth-jwt
Next:    Run `specky-pr` to open a pull request
```

---

## Mode B: Targeted Fix

Use when a specific stage needs a surgical correction, or a bug needs fixing against a spec.

### 1. Identify the fix target

Determine if this is:
- **A gap from acceptance tests**: specific criterion not met
- **A blocked stage**: resolve the blocker and implement
- **A bug issue**: fetch via `gh issue view {bug_id}`, link to parent spec
- **User-described issue**: understand scope before touching code

### 2. Scope the fix

Before writing any code, explicitly state what will and will not change:
```
Will change: src/auth/jwt.middleware.ts -- add token expiry validation
Will NOT change: any other auth files, routes, tests unrelated to expiry
```

Get user confirmation if scope seems larger than expected.

### 3. Implement the fix

Make the targeted change. Follow existing patterns in the codebase.

Run affected tests:
```bash
# Run only tests related to changed files
npx jest --testPathPattern="jwt|auth" --no-coverage
```

If tests fail, fix them as part of this change.

### 4. Commit

```bash
git commit -m "[spec-{id}] fix: {short description}"
```

For bug issues:
```bash
git commit -m "[spec-{id}][bug-{bug-id}] fix: {description}"
```

### 5. Update GitHub issue

```bash
gh issue comment {id} --body "Fix applied: {description}. Commit: {hash}. Files: {list}"
```

If this resolves a linked bug issue:
```bash
gh issue close {bug-id} --comment "Resolved by fix in spec #{id}. Commit: {hash}"
```

### 6. Confirm to user

Report what changed, what was tested, and the commit hash.
Suggest `specky-pr` if a PR is not yet open.

## Flags

| Flag | Default | Behavior |
|---|---|---|
| `--stages` | all | Comma-separated stage numbers to run (e.g., `2,3`) |
| `--no-parallel` | false | Force sequential execution |
| `--skip-acceptance-tests` | false | Skip acceptance test generation |
| `--audit` | false | Run completion audit sub-agent after execution |
| `--dry-run` | false | Show execution plan without running |
| `--force` | false | Re-run completed stages |
| `--fix` | false | Run in targeted fix mode (Mode B) |
| `--bug-id` | none | Bug issue number to link fix against |
| `--spec-id` | required | Parent spec issue number (required for --fix mode) |

## Error Recovery

If execution is interrupted mid-spec:
1. The plan state is in the issue body (` ```specky-plan ` block) — check stage statuses there
2. Re-run with `--stages {n}` to resume from a specific stage
3. Completed stages (status: `complete`) are not re-run unless `--force` is set
