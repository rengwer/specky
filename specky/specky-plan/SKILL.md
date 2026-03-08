---
name: specky-plan
description: >
  Research and plan a spec issue into executable stages. Use when a user says "plan this spec",
  "break down issue X", "research spec Y", "what stages does this need", "decompose this feature",
  "create a plan for this issue", or any request to analyze a spec and produce an execution plan.
  Combines research (codebase context, related issues, clarifying questions) with stage decomposition
  into a single flow. Produces a task-list and embedded plan data in the issue body. No local cache needed.
compatibility:
  tools:
    - bash
    - computer
  requires:
    - gh CLI installed and authenticated (`gh auth status`)
---

# Plan — Research and Decompose a Spec

Research a spec's context (codebase, related issues, industry standards) and decompose it
into ordered execution stages with dependency graph and parallelization hints.

Read `../_shared/references/conventions.md` for stage schema and conventions.

## Prerequisites

- Spec issue must exist in GitHub (run `specky-spec` first)
- `gh` CLI authenticated

## Steps

### 1. Load the spec from GitHub

```bash
gh issue view {id} --json number,title,body,labels,assignees
```

If the issue body lacks acceptance criteria, suggest running `specky-spec` interview first.

### 2. Research phase

#### 2a. Search for related issues and code

```bash
gh issue list --search "{spec domain keywords}" --json number,title,labels
gh search code "{spec domain keywords}" --repo {owner/repo}
```

Explore the codebase for relevant patterns, existing implementations, and architectural context.

#### 2b. Identify and resolve open questions

Read the spec carefully. Flag anything that is:
- **Ambiguous** — could be interpreted multiple ways
- **Missing** — no detail where detail is needed
- **Contradictory** — conflicts with codebase or other specs

For each gap, ask the user **one targeted question at a time**. Resolve conversationally.

#### 2c. Append research findings to issue

Update the issue body with a `## Research Notes` section containing key findings:

```bash
gh issue view {id} --json body -q .body
gh issue edit {id} --body "{existing body}\n\n## Research Notes\n{summary of findings}"
```

### 3. Decompose into stages

Analyze the spec's acceptance criteria, description, and research findings to produce stages.

**Stage decomposition principles:**
- Each stage should be independently committable (no half-baked state)
- Map to natural seams: schema changes, API layer, business logic, UI, tests, docs
- A stage that touches one file or system boundary is the right granularity
- Tests for a stage are part of that stage, not separate at the end
- If a stage has no dependencies on another, mark it `parallel: true`

**Example stage sequence for a REST API feature:**
1. Database migration (parallel: false)
2. Repository/data layer (depends_on: [1])
3. Service/business logic (depends_on: [2])
4. API controller + routes (depends_on: [3])
5. Integration tests (depends_on: [4])
6. Documentation update (depends_on: [4], parallel: true with 5)

### 4. Append execution plan to issue body

Append both a human-readable task-list AND a machine-readable plan block to the issue body.
The plan data is stored as a fenced YAML code block with `specky-plan` info string.

```bash
gh issue view {id} --json body -q .body
gh issue edit {id} --body "{existing body}\n\n## Execution Plan\n\n- [ ] Stage 1: {title}\n- [ ] Stage 2: {title}\n...\n\n\`\`\`specky-plan\n{plan YAML}\n\`\`\`"
```

The embedded plan YAML follows this schema:

```yaml
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

Stage `status` values: `pending`, `complete`, `blocked`, `skipped`

The fenced block is visible to collaborators and parseable by `specky-execute`.
No local cache is needed — GitHub is the sole source of truth.

### 5. Update issue labels

```bash
gh label create "spec:planned" --color 0052CC 2>/dev/null || true
gh issue edit {id} --add-label "spec:planned"
```

### 6. Present plan for approval

Show the plan table:

```
| Stage | Title                | Parallel | Depends On |
|-------|----------------------|----------|------------|
| 1     | Database migration   | No       | --         |
| 2     | Repository layer     | No       | Stage 1    |
| 3     | Service logic        | No       | Stage 2    |
| 4     | API endpoints        | No       | Stage 3    |
| 5     | Integration tests    | Yes      | Stage 4    |
| 6     | Documentation        | Yes      | Stage 4    |
```

Ask: "Does this plan look right? Run `specky-execute` when ready."

## Flags

| Flag | Default | Behavior |
|---|---|---|
| `--skip-research` | false | Skip research phase, go straight to decomposition |
| `--max-stages` | 10 | Warn if decomposition exceeds this |
| `--no-parallel` | false | Treat all stages as sequential |
