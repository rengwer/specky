---
name: specky-bugs
description: >
  Fetch, prioritize, and triage bug issues from GitHub. Use when a user says "show me bugs",
  "list open bugs", "triage bug X", "investigate this bug", "root cause issue Y", "what bugs
  are assigned to me", "show critical bugs", or any request to retrieve, prioritize, or
  investigate bug issues. Combines listing and triage into one flow.
compatibility:
  tools:
    - bash
    - computer
  requires:
    - gh CLI installed and authenticated (`gh auth status`)
---

# Bugs — List, Prioritize, and Triage

Fetch bug issues from GitHub, display them prioritized by severity, and triage
a selected bug (reproduce, root cause, link to spec, document findings).

Read `../_shared/references/conventions.md` for label standards.

## Prerequisites

- `gh` CLI authenticated (`gh auth status`)

## Modes

- **"show bugs" / "list bugs"** → Mode A (Fetch + Display)
- **"triage bug 200"** → Mode B (Investigate)

---

## Mode A: Fetch and Prioritize Bugs

### 1. Determine fetch strategy

Ask the user (if not already clear from context):
- **All open bugs** (default)
- **Assigned to me**
- **By severity label**: `severity:critical`, `severity:high`, `severity:medium`, `severity:low`
- **By component label**
- **By milestone**

### 2. Fetch bug issues

```bash
# All open bugs
gh issue list --label "bug" --state open --json number,title,body,labels,assignees,milestone

# Assigned to me
gh issue list --label "bug" --assignee "@me" --state open --json number,title,body,labels,assignees,milestone

# By severity
gh issue list --label "bug" --label "severity:critical" --state open --json number,title,body,labels,assignees,milestone
```

### 3. Enrich each bug

For each fetched bug, parse the issue body for:
- Repro steps (if present)
- Linked parent spec (scan for `#N` references)
- Severity and priority from labels

### 4. Present prioritized results

Show a summary table sorted by severity:

```
ID     | Title                          | Severity | Linked Spec | Assignee
-------|--------------------------------|----------|-------------|--------
200    | Login fails on SSO timeout     | critical | #123        | @alice
201    | Export produces empty CSV       | high     | --          | --
202    | Tooltip misaligned on mobile   | low      | --          | @bob
```

Suggest: "Pick a bug to triage, or run `specky-execute` with `--fix` to go straight to fixing."

---

## Mode B: Triage a Bug

### 1. Load the bug

```bash
gh issue view {id} --json number,title,body,labels,assignees,comments
```

Display summary:
```
Bug #{id}: {title}
Severity:    {severity label}
Repro Steps: present / missing
Linked Spec: #{spec_id} / none
```

### 2. Confirm or gather repro steps

If repro steps are missing or vague:
> "Can you describe the exact steps to reproduce this, including any inputs, environment, or user state required?"

If repro steps exist:
> "Do these repro steps still apply, or has the context changed?"

### 3. Search for affected code

Use the bug title, description, and repro context to locate likely affected files.
Search the codebase for relevant keywords, error messages, and component names.

### 4. Link to parent spec

Search for the spec this bug traces back to:
```bash
gh issue list --label "spec" --search "{keyword from bug}" --json number,title
```

If found, confirm with user before linking:
> "This bug looks related to spec #{spec_id} ({title}). Should I link them?"

If confirmed:
```bash
gh issue view {id} --json body -q .body
gh issue edit {id} --body "{existing body}\n\n## Linked Spec\nRelated to #{spec_id}"
```

### 5. Determine root cause

State the root cause clearly with file paths and line numbers:

```
Root Cause: The JWT middleware does not refresh the token on expiry — it only validates
on initial request. On timeout, the session silently fails instead of prompting re-auth.

Affected file(s): src/auth/jwt.middleware.ts (line 42), src/session/session.service.ts (line 87)
```

If root cause cannot be determined statically, note what's needed (logs, test case, env details).

### 6. Estimate impact

Assess:
- **Scope**: how many users / workflows affected?
- **Workaround**: is there a known workaround?
- **Regression risk**: does fixing this risk breaking related functionality?

### 7. Update bug issue

Append `## Triage Notes` to the issue body:

```bash
gh issue view {id} --json body -q .body
gh issue edit {id} --body "{existing body}\n\n## Triage Notes\n### Root Cause\n{root_cause}\n\n### Affected Files\n{files}\n\n### Impact\n{impact}\n\n### Repro Steps (confirmed)\n{steps}"
```

Add label and comment:
```bash
gh label create "bug:triaged" --color C2E0C6 2>/dev/null || true
gh issue edit {id} --add-label "bug:triaged"
gh issue comment {id} --body "Triaged. Root cause: {summary}. Linked spec: #{spec_id}. Ready for fix."
```

### 8. Confirm and suggest next step

```
Bug #{id} triaged.

Root Cause:  {one-line summary}
Affected:    {file list}
Linked Spec: #{spec_id} / none (orphan)
Impact:      {scope summary}

Next: Run `specky-execute --fix --spec-id {spec_id} --bug-id {id}` to apply the fix.
```

## Flags

| Flag | Default | Behavior |
|---|---|---|
| `--severity` | all | Filter: critical, high, medium, low |
| `--assigned-to-me` | false | Only bugs assigned to current user |
| `--milestone` | none | Filter by milestone name |
| `--skip-code-search` | false | Skip code exploration during triage |
| `--skip-link` | false | Don't attempt to find and link a parent spec |
| `--auto-link` | false | Link parent spec without asking for confirmation |
