---
name: specky-pr
description: >
  Open a pull request for a spec and handle post-merge cleanup. Use when a user says "commit this spec",
  "open a PR for spec X", "create a pull request", "push the spec branch", "submit for review",
  "the PR merged, clean up", "complete spec X", "close out this spec", or any request to create
  a PR linked to a spec issue or finalize after merge.
compatibility:
  tools:
    - bash
    - computer
  requires:
    - gh CLI installed and authenticated (`gh auth status`)
---

# PR — Create Pull Request and Finalize

Create a PR linked to a spec issue, or finalize after merge (clean up branch).

Read `../_shared/references/conventions.md` for branch naming and PR title conventions.

## Prerequisites

- Feature branch `spec/{slug}` exists with commits
- `gh` CLI authenticated

## Modes

- **"open a PR" / "commit spec"** → Mode A (Create PR)
- **"PR merged, clean up" / "complete spec"** → Mode B (Finalize)

---

## Mode A: Create Pull Request

### 1. Verify branch state

```bash
git status
git log main..HEAD --oneline
```

Confirm there are commits ahead of main and no uncommitted changes.
If there are uncommitted changes, warn the user.

### 2. Push branch

```bash
git push -u origin spec/{slug}
```

### 3. Load spec context

```bash
gh issue view {id} --json number,title,body,labels
```

Parse the `## Execution Plan` section from the issue body for stage details.

### 4. Generate PR description

Build the PR body from spec context:

```markdown
## Summary
{spec description -- first 2-3 sentences}

## Spec Reference
Closes #{id} -- {title}

## Changes by Stage
{for each stage in plan: "- Stage N: {title}"}

## Acceptance Criteria
{spec.acceptance_criteria as checklist}
- [ ] {criterion 1}
- [ ] {criterion 2}

## Test Coverage
{acceptance test file path and criteria coverage, if available}

## Notes
{any blockers skipped, open questions, or reviewer guidance}
```

### 5. Create PR

```bash
gh pr create \
  --title "Spec #{id}: {spec title}" \
  --body "{generated description}" \
  --base main
```

The `Closes #{id}` in the PR body automatically closes the issue on merge.

### 6. Update issue labels

```bash
gh label create "spec:in-review" --color FBCA04 2>/dev/null || true
gh issue edit {id} --add-label "spec:in-review"
gh issue comment {id} --body "PR opened: {pr_url}"
```

### 7. Confirm to user

```
PR opened: {pr_url}
Issue #{id} labeled: spec:in-review

Next: merge the PR. The issue will close automatically via "Closes #{id}".
Run `specky-pr --finalize {id}` after merge to clean up branch.
```

---

## Mode B: Finalize After Merge

### 1. Verify PR is merged

```bash
gh pr list --state merged --search "Spec #{id}" --json number,title,state,mergedAt
```

If not merged, inform the user and stop.

### 2. Optionally delete feature branch

If user requests it:
```bash
git branch -d spec/{slug}
git push origin --delete spec/{slug}
```

### 3. Confirm to user

```
Spec #{id} finalized.

PR:      Merged
Issue:   Closed (auto-closed by PR)
```

## Flags

| Flag | Default | Behavior |
|---|---|---|
| `--target` | `main` | Target branch for PR |
| `--draft` | false | Open as draft PR |
| `--reviewers` | none | Comma-separated reviewer GitHub usernames |
| `--finalize` | false | Run post-merge cleanup instead of creating PR |
| `--delete-branch` | false | Delete feature branch after finalize |
