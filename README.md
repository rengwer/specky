# Specky — Spec-Driven Development Skills

Heavily influenced by: [GSD](https://github.com/gsd-build/get-shit-done)

Claude Code skills that automate the full feature development lifecycle — from GitHub issue to merged PR — using AI sub-agents.

---

## Table of Contents

- [How to Install](#how-to-install)
- [How to Use](#how-to-use)
- [Lifecycle](#lifecycle)
- [Conventions](#conventions)
- [Skill Reference](#skill-reference)
- [Parallelization](#parallelization)

---

## How to Install

### Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- [`gh` CLI](https://cli.github.com/) installed and authenticated (`gh auth login`)

### 1. Add skills to your project

Choose one of the three methods below.

#### Option A: Git Subtree (recommended)

Pulls the skill files directly into your repo — no symlinks, no extra setup for collaborators.

```bash
cd /path/to/your-project

# Add the remote (one time)
git remote add specky https://github.com/your-org/claude-skills.git

# Pull into your project
git subtree add --prefix=.claude/skills/specky specky main --squash
```

**To update:**
```bash
git subtree pull --prefix=.claude/skills/specky specky main --squash
```

#### Option B: Git Submodule

Keeps the skill repo as a separate checkout inside your project. Collaborators must run `git submodule init && git submodule update` after cloning.

```bash
cd /path/to/your-project
git submodule add https://github.com/your-org/claude-skills.git .claude/skills/specky
```

**To update:**
```bash
git submodule update --remote .claude/skills/specky
```

#### Option C: Symlink

Clone once, then symlink into every project. Pull to update all projects at once.

```bash
# Clone the skill repo (one time)
git clone https://github.com/your-org/claude-skills.git ~/.claude-skills

# Link into your project
mkdir -p /path/to/your-project/.claude/skills
ln -s ~/.claude-skills/specky /path/to/your-project/.claude/skills/specky
```

**Windows (requires admin or Developer Mode):**
```cmd
git clone https://github.com/your-org/claude-skills.git %USERPROFILE%\.claude-skills
mklink /D C:\path\to\your-project\.claude\skills\specky %USERPROFILE%\.claude-skills\specky
```

**To update:** `git -C ~/.claude-skills pull` — all linked projects get the latest skills immediately.

### 2. Authenticate with GitHub

```bash
gh auth login
gh auth status  # verify
```

That's it. No cache directory, no `.gitignore` changes needed. All state lives in GitHub Issues.

---

## How to Use

Invoke skills naturally in Claude Code. Claude detects your intent and routes to the correct sub-skill automatically.

**Examples:**

```
create a spec for user authentication
get my specs
plan spec 123
execute spec 123
fix the login bug in spec 123
open a PR for spec 123
show me critical bugs
triage bug 200
```

You can also invoke sub-skills directly:

```
/specky-spec
/specky-plan 123
/specky-execute 123
/specky-pr 123
/specky-bugs
```

### Chaining

Skills return enough context to chain directly into the next step:

```
> create a spec for user authentication
Created spec #123: User Authentication
Branch: spec/user-authentication

Ready to plan? Run: specky-plan 123

> plan spec 123
...
```

---

## Lifecycle

### Feature Development

```
specky-spec       Create a new issue, or fetch existing ones.
    |             Conducts structured interview for acceptance criteria,
    |             constraints, dependencies, and scope.
    |             Returns spec ID and branch name for chaining.
    v
specky-plan       Searches codebase and related issues for context.
    |             Asks targeted clarifying questions.
    |             Decomposes spec into stages with dependency graph.
    |             Appends task-list + plan data to issue body.
    v
specky-execute    Reads plan from issue body.
    |             Reads project CLAUDE.md for test/build conventions.
    |             Spawns parallel sub-agents per stage batch.
    |             Each agent: reads task > codes > tests > commits > reports.
    |             Orchestrator tracks progress, handles blockers.
    |             Runs acceptance tests on completion.
    |             Applies targeted fixes inline when gaps are found.
    v
specky-pr         Opens PR linked to issue (Closes #id).
                  After merge: optionally deletes feature branch.
```

### Bug Triage & Fix

```
specky-bugs       Fetches and prioritizes bug issues by severity.
    |             Triages: reproduces, identifies root cause, links to spec.
    v
specky-execute    Applies targeted fix (--fix mode).
    |             Commits with [spec-{id}][bug-{id}] prefix.
    v
specky-pr         Opens PR, auto-closes bug on merge.
```

### GitHub Issue Label Lifecycle

Labels accumulate as a spec progresses:

| Label | Meaning |
|---|---|
| `spec` | Spec issue (always present) |
| `spec:planned` | Execution plan created, task-list appended |
| `spec:executing` | Sub-agents actively implementing |
| `spec:in-review` | PR open, awaiting merge |
| `bug` | Bug report |
| `bug:triaged` | Root cause identified |

Issue closure happens automatically when the PR merges (via `Closes #id` in the PR body).

### Zero Local State

All state lives in GitHub Issues:

- Spec details, acceptance criteria, research notes → **issue body sections**
- Execution plan with stages and dependencies → **hidden HTML comment in issue body** (`<!-- specky-plan ... specky-plan -->`)
- Stage completion status → **task-list checkboxes + plan JSON updates**

No local cache, no `.specky-cache/`, no `.gitignore` changes. Works across machines and team members.

### CLAUDE.md Integration

Sub-agents automatically read the project's `CLAUDE.md` to discover:

- **Test commands** — `npm test`, `pytest`, `go test`, etc.
- **Build commands** — `npm run build`, `cargo build`, etc.
- **Code conventions** — linting, formatting, style rules
- **Project-specific instructions** — anything else in CLAUDE.md

This means specky adapts to your project's tooling without any configuration. If your CLAUDE.md says "run `pnpm test` for tests", that's what sub-agents will use.

---

## Conventions

### Branch Naming

```
spec/{slug}
```

- `{slug}` is a kebab-case slug derived from the spec title

**Examples:**
```
spec/user-auth-jwt
spec/payment-gateway-integration
```

### Commit Messages

```
[spec-{id}] stage-{n}: {description}
```
Example: `[spec-123] stage-2: add JWT middleware`

For fixes:
```
[spec-{id}] fix: {description}
[spec-{id}][bug-{bug-id}] fix: {description}
```

### PR Title

```
Spec #{id}: {issue title}
```

---

## Skill Reference

### `specky-spec`

Create, fetch, or fill out spec issues. Three modes:

- **Create** — Makes a new GitHub issue labeled `spec`, then conducts a structured interview. Returns the spec ID and branch name so you can chain into `specky-plan`.
- **Fetch** — Lists specs by label, milestone, assignee, or search keywords
- **Interview** — Walks through problem/goal, acceptance criteria, scope, constraints, and dependencies one question at a time. Previews the composed body before updating the issue.

The interview is the highest-value part of the system. It ensures specs have measurable acceptance criteria before any code is written.

**Flags:**

| Flag | Default | Behavior |
|---|---|---|
| `--skip-interview` | false | Create/fetch without interview |
| `--criteria-only` | false | Only interview for acceptance criteria |
| `--labels` | `spec` | Additional labels |
| `--milestone` | none | Assign to milestone |
| `--assignee` | none | Filter by assignee (`@me`) |

### `specky-plan`

Research context and decompose a spec into executable stages.

1. Searches codebase and related issues for architectural context
2. Asks targeted clarifying questions for ambiguous or missing details
3. Appends research findings to the issue body
4. Decomposes into stages following natural seams (schema, API, logic, UI, tests)
5. Appends task-list + embedded plan JSON to issue body
6. No local files created — everything is in the issue

**Flags:**

| Flag | Default | Behavior |
|---|---|---|
| `--skip-research` | false | Skip research, go straight to decomposition |
| `--max-stages` | 10 | Warn if plan exceeds this |
| `--no-parallel` | false | Treat all stages as sequential |

### `specky-execute`

The primary implementation skill. Two modes:

**Full execution** — Reads plan from issue body, reads project CLAUDE.md for testing/build conventions, then spawns one sub-agent per stage grouped by dependency batch. Each agent reads its task, writes code, runs tests (using the project's test framework), commits, and reports back. The orchestrator manages parallelism, updates issue checkboxes, and handles blockers. Runs acceptance tests on completion.

**Targeted fix** (`--fix`) — Scopes a surgical fix to specific files, implements it, runs affected tests, and commits with traceability. Used for audit gaps, blocked stages, and bug fixes.

**Flags:**

| Flag | Default | Behavior |
|---|---|---|
| `--stages 2,3` | all | Run only specified stages |
| `--no-parallel` | false | Force sequential execution |
| `--skip-acceptance-tests` | false | Skip acceptance test generation |
| `--audit` | false | Run completion audit sub-agent |
| `--dry-run` | false | Show plan without running |
| `--force` | false | Re-run completed stages |
| `--fix` | false | Targeted fix mode |
| `--bug-id` | none | Link fix to a bug issue |
| `--spec-id` | required | Parent spec (required for --fix) |

### `specky-pr`

Create a PR or finalize after merge. Two modes:

**Create PR** — Pushes branch, generates PR description from spec context and stage breakdown, creates PR with `Closes #{id}` to auto-close the issue on merge.

**Finalize** (`--finalize`) — Optionally deletes the feature branch after PR is merged.

**Flags:**

| Flag | Default | Behavior |
|---|---|---|
| `--target` | `main` | Target branch |
| `--draft` | false | Open as draft PR |
| `--reviewers` | none | Comma-separated reviewers |
| `--finalize` | false | Post-merge cleanup |
| `--delete-branch` | false | Delete branch after finalize |

### `specky-bugs`

Fetch, prioritize, and triage bugs. Two modes:

**List** — Fetches bug issues filtered by severity, assignee, milestone, or component. Displays a prioritized table.

**Triage** — Investigates a specific bug: confirms repro steps, searches affected code, identifies root cause with file paths, links to parent spec, estimates impact, and documents findings in the issue.

**Flags:**

| Flag | Default | Behavior |
|---|---|---|
| `--severity` | all | Filter by severity label |
| `--assigned-to-me` | false | Only my bugs |
| `--milestone` | none | Filter by milestone |
| `--skip-code-search` | false | Skip code exploration |
| `--skip-link` | false | Don't link to parent spec |
| `--auto-link` | false | Link without confirmation |

---

## Parallelization

`specky-execute` uses a **dependency-batch model** to maximize parallel execution while respecting stage ordering.

### How it works

1. **Plan parsing** — The orchestrator reads the plan JSON from the issue body's HTML comment and builds a dependency graph from each stage's `depends_on` array.

2. **Batch grouping:**
   - **Batch 0** — stages with no dependencies (run immediately)
   - **Batch 1** — stages whose dependencies are all in Batch 0
   - **Batch N** — stages whose dependencies are all complete

3. **Within each batch** — all stages marked `parallel: true` are spawned as simultaneous sub-agents. Stages marked `parallel: false` run sequentially within the batch.

4. **Between batches** — the orchestrator waits for every stage in the current batch to reach `complete` (or `skipped` with user approval) before starting the next batch.

### Example

Given a plan with 5 stages:

```
Stage 1: DB migration      (no deps, parallel: true)
Stage 2: Repo layer        (no deps, parallel: true)
Stage 3: Service logic     (depends_on: [1, 2])
Stage 4: API endpoints     (depends_on: [3])
Stage 5: Frontend          (depends_on: [3], parallel: true)
```

Execution order:

```
Batch 0:  Stage 1 + Stage 2  (run simultaneously)
Batch 1:  Stage 3            (waits for 1 and 2)
Batch 2:  Stage 4 + Stage 5  (run simultaneously)
```

### Sub-agent isolation

Each sub-agent receives a self-contained prompt with:
- Its stage title, description, and acceptance criteria
- Full spec context from the GitHub issue
- Relevant project conventions from CLAUDE.md (testing, build, style)
- Instructions to scope changes to its stage only

Sub-agents do **not** update GitHub issues directly — the orchestrator handles all status updates after parsing each agent's structured report.

### Handling blockers

If a sub-agent reports `STATUS: blocked`, the orchestrator:
- Comments on the issue describing the blocker
- Pauses all downstream dependent stages
- Surfaces the blocker to the user with three options: **Resolve now** (apply inline fix), **Skip stage**, or **Abort**

To resume after a blocker is resolved:

```
execute spec 123 --stages 3
```

Completed stages are not re-run unless `--force` is passed.
