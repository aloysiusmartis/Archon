---
description: Gather Azure DevOps PR context and prepare scope artifact for review agents (AzDO equivalent of archon-pr-review-scope)
argument-hint: (none - reads from workflow artifacts)
---

# AzDO PR Review Scope

**Input**: $ARGUMENTS
**Workflow ID**: $WORKFLOW_ID

---

## Your Mission

Verify the Azure DevOps PR is in a reviewable state, gather all context needed for the parallel
review agents, and write `$ARTIFACTS_DIR/review/scope.md`.

**IMPORTANT**: This is an Azure DevOps repo. Do NOT use `gh` commands. Use `az repos pr` and `git`
instead. When review agents later call `gh pr diff {number}`, they will get an error — they should
fall back to `git diff $BASE_BRANCH...HEAD` which always works.

---

## Phase 1: IDENTIFY — Determine PR

### 1.1 Get PR ID

```bash
if [ -f "$ARTIFACTS_DIR/.pr-id" ]; then
  PR_ID=$(cat "$ARTIFACTS_DIR/.pr-id" | tr -d '\n ')
fi

# From arguments if not in artifacts
if [ -z "$PR_ID" ] && [ -n "$ARGUMENTS" ]; then
  PR_ID=$(echo "$ARGUMENTS" | grep -oE '[0-9]+' | head -1)
fi

# Detect from current branch
if [ -z "$PR_ID" ]; then
  CURRENT_BRANCH=$(git branch --show-current)
  PR_ID=$(az repos pr list --source-branch "$CURRENT_BRANCH" --status active --output json \
    | jq -r '.[0].pullRequestId // empty' 2>/dev/null || echo "")
fi

echo "PR ID: $PR_ID"
```

If no PR ID found, write a minimal scope with git diff only and proceed.

### 1.2 Fetch PR Details

```bash
az repos pr show --id "$PR_ID" --output json
```

Extract and note:
- `pullRequestId` — PR number
- `title` — PR title
- `targetRefName` — base branch (strip `refs/heads/` prefix)
- `sourceRefName` — head branch (strip `refs/heads/` prefix)
- `status` — must be `active`
- `isDraft` — draft status
- `createdBy.displayName` — author
- `url` — PR URL

Write PR ID to `.pr-number` for review agent compatibility:
```bash
echo "$PR_ID" > "$ARTIFACTS_DIR/.pr-number"
```

### 1.3 Verify Reviewability

The PR is reviewable if `status == "active"`. Draft PRs are fine to review.

---

## Phase 2: GATHER — Collect Diff and Files

### 2.1 Get the Diff

```bash
BASE_REF=$(az repos pr show --id "$PR_ID" --query "targetRefName" -o tsv | sed 's|refs/heads/||')
git fetch origin "$BASE_REF" 2>/dev/null || true
git diff "origin/$BASE_REF...HEAD" --stat
```

Get the full diff (truncate if >50KB for token efficiency):
```bash
git diff "origin/$BASE_REF...HEAD" | head -c 50000
```

### 2.2 Get Changed Files

```bash
git diff "origin/$BASE_REF...HEAD" --name-only
```

Categorise by type:
- **Source files** (`.ts`, `.py`, `.go`, `.rs`, etc.) — subject to code review
- **Test files** (`*.test.*`, `*.spec.*`, `*_test.*`) — subject to test coverage review
- **Config files** (`.yaml`, `.json`, `.toml`, `.*rc`) — scope-limited
- **Docs** (`*.md`, `docs/`) — subject to docs impact review
- **Assets** — out of scope

### 2.3 Count Additions/Deletions

```bash
git diff "origin/$BASE_REF...HEAD" --shortstat
```

---

## Phase 3: CONTEXT — Read Workflow Artifacts

Check for prior workflow context:

```bash
# Implementation report from this workflow run
cat "$ARTIFACTS_DIR/implementation.md" 2>/dev/null || echo "(not found)"

# Investigation or plan
cat "$ARTIFACTS_DIR/investigation.md" 2>/dev/null || cat "$ARTIFACTS_DIR/plan.md" 2>/dev/null || echo "(not found)"

# Validation results
cat "$ARTIFACTS_DIR/validation.md" 2>/dev/null || echo "(not found)"
```

---

## Phase 4: RULES — Read CLAUDE.md

```bash
cat CLAUDE.md 2>/dev/null | head -200
```

Extract key rules reviewers must enforce (type safety, error handling, naming, etc.).

---

## Phase 5: WRITE — Scope Artifact

Create the directory and write the scope:

```bash
mkdir -p "$ARTIFACTS_DIR/review"
```

Write `$ARTIFACTS_DIR/review/scope.md` with this exact structure:

```markdown
# PR Review Scope

**PR**: !{pr_id} — {title}
**Author**: {author}
**Base**: {base_branch} ← {head_branch}
**Status**: {status} (draft: {is_draft})
**URL**: {url}

---

## ⚠️ Azure DevOps Context

This is an Azure DevOps PR. `gh pr diff` is NOT available.
To get the diff, use: `git diff origin/{base_branch}...HEAD`
The diff is pre-embedded in the "Full Diff" section below.

---

## Changed Files

### Source Files (Review Required)
{list}

### Test Files
{list}

### Config / Docs / Assets (Scope-Limited)
{list}

**Stats**: +{additions} / -{deletions} across {n} files

---

## NOT Building (Scope Limits)

{List anything explicitly out of scope based on the investigation/plan artifact, or "None specified."}

---

## CLAUDE.md Rules to Enforce

{Key rules extracted from CLAUDE.md — type safety, naming, patterns, etc.}

---

## Workflow Context

### Investigation / Plan
{Summary or "(not available)"}

### Implementation
{Summary or "(not available)"}

### Validation
{Summary or "(not available)"}

---

## Full Diff

```diff
{git diff output — up to 50KB}
```
```

---

## Phase 6: OUTPUT — Summary

Print:
```
PR Review Scope prepared:
- PR: !{id} ({title})
- Base: {base_branch} ← {head_branch}
- Changed files: {n} source, {n} test, {n} other
- Scope artifact: $ARTIFACTS_DIR/review/scope.md

Note: az repos pr list and git diff used (no gh CLI — Azure DevOps repo)
```

---

## Success Criteria

- `$ARTIFACTS_DIR/review/scope.md` written with full diff embedded
- `$ARTIFACTS_DIR/.pr-number` written for review agent compatibility
- AzDO context note present so review agents know to use `git diff` not `gh pr diff`
