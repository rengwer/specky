---
name: specky-spec
description: >
  Create, fetch, or fill out spec issues in GitHub. Use when a user says "create a spec",
  "get my specs", "show my issues", "fill out this spec", "add acceptance criteria",
  "new spec for X", "fetch specs", or any request to create, retrieve, or enrich GitHub
  issues for spec-driven development. This is always the first step in the lifecycle.
compatibility:
  tools:
    - bash
    - computer
  requires:
    - gh CLI installed and authenticated (`gh auth status`)
---

# Spec — Create, Fetch, or Fill

Create new spec issues, fetch existing ones, or conduct a structured interview to
fill out spec details. This is the entry point for all spec-driven development.

Read `../_shared/references/conventions.md` for label standards.

## Prerequisites

- `gh` CLI authenticated (`gh auth status`)

## Modes

Determine the user's intent and route accordingly:

- **"create a spec"** → Mode A (Create + Interview)
- **"get my specs" / "show issues"** → Mode B (Fetch + Display)
- **"fill out spec 123" / "add criteria to 123"** → Mode C (Interview only)

---

## Mode A: Create a New Spec

### 1. Gather minimum fields

> "What's the title of this spec?"

> "Any labels beyond `spec`? (optional)"

### 2. Ensure labels exist

```bash
gh label create spec --description "Spec issue" --color 0E8A16 2>/dev/null || true
```

### 3. Create the issue

```bash
gh issue create --title "{title}" --label "spec" --body ""
```

Capture the issue number and generate the spec slug (kebab-case of title).

Confirm:
```
Created spec #{id}: {title}
Branch: spec/{slug}
```

**Return the spec ID and slug** so the user can chain directly into planning:
```
Ready to plan? Run: specky-plan {id}
```

### 4. Proceed to Interview (below)

Automatically run the interview flow for the newly created issue.

---

## Mode B: Fetch Existing Specs

### 1. Determine fetch strategy

Ask the user (if not already clear from context):
- **By label** (default `spec`)
- **By search keywords**
- **By milestone**
- **By assignee** (`@me` for current user)

### 2. Fetch issues

```bash
# By label (default)
gh issue list --label "spec" --state open --json number,title,body,labels,assignees,milestone

# By search
gh issue list --search "{keywords}" --state open --json number,title,body,labels,assignees,milestone

# Assigned to me
gh issue list --assignee "@me" --label "spec" --state open --json number,title,body,labels,assignees,milestone

# Specific issue
gh issue view {id} --json number,title,body,labels,assignees,milestone
```

### 3. Parse linked issues

Scan each issue body for `#N` references to identify linked parent/child issues.

### 4. Present results

Show a summary table:

```
ID     | Title                        | Labels            | Status
-------|------------------------------|-------------------|--------
123    | Add JWT authentication       | spec              | open
124    | Payment gateway integration  | spec, spec:planned| open
```

Suggest next steps: `specky-plan` or interview if details are thin.

---

## Mode C: Interview (Fill Spec Details)

This is the highest-value flow. It ensures specs have real acceptance criteria before
any code is written.

### 1. Load the current issue

```bash
gh issue view {id} --json number,title,body,labels,assignees
```

Display what's filled vs. missing:
```
Issue #{id}: {title}
---
Description:         filled / missing
Acceptance Criteria: filled / missing
Labels:              {labels}
Assignees:           {assignees}
```

### 2. Conduct the interview

Ask questions **one at a time**, conversationally. Skip sections already well-populated.
Adapt based on answers.

#### 2a. Problem / Goal
If description is thin or missing:
> "What problem does this spec solve, or what is the goal? Describe it as if explaining to a new team member."

#### 2b. Acceptance Criteria
If acceptance criteria are missing:
> "What does 'done' look like? List the conditions that must be true for this to be considered complete."

Push for specifics if vague:
> "Can you make that measurable? For example: 'User can log in with SSO within 2 seconds' rather than 'SSO works'."

#### 2c. Out of Scope
> "Is there anything that might seem related but is explicitly NOT part of this spec?"

#### 2d. Technical Constraints
> "Are there any technical constraints, required technologies, or architectural decisions this spec must follow?"

#### 2e. Dependencies
> "Does this depend on any other issues, external systems, or decisions not yet made?"

Search for related issues if the user names something:
```bash
gh issue list --search "{dependency keyword}" --json number,title
```

#### 2f. Notes for Reviewer
> "Anything else a developer picking this up should know?"

### 3. Preview before writing

Show the composed spec body:

```
## Description
{composed description}

## Acceptance Criteria
- {criterion 1}
- {criterion 2}

## Out of Scope
- {item}

## Technical Constraints
- {constraint}

## Dependencies
- #{linked_id}: {title}
```

Ask: "Does this look right before I update the issue?"

### 4. Update GitHub issue

```bash
gh issue edit {id} --body "{composed body}"
gh issue edit {id} --add-label "spec"
```

### 5. Confirm and suggest next steps

```
Spec #{id}: {title}
Branch: spec/{slug}

Ready to plan? Run: specky-plan {id}
```

Always return the spec ID, title, and branch name so the user can chain into the next step.

## Flags

| Flag | Default | Behavior |
|---|---|---|
| `--skip-interview` | false | Create/fetch without running interview |
| `--criteria-only` | false | Only interview for acceptance criteria |
| `--labels` | `spec` | Additional comma-separated labels |
| `--milestone` | none | GitHub milestone to assign |
| `--assignee` | none | Filter by assignee (`@me` for current user) |
