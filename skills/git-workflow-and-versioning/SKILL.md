---
name: git-workflow-and-versioning
description: Git workflow practices. Use when making any code change — committing, branching, resolving conflicts, or organizing parallel work.
---

# Git Workflow and Versioning

## Overview

Git is your safety net. Commits are save points, branches are sandboxes, and history is documentation.

## Core Principles

### Trunk-Based Development (Recommended)

```
main always deployable
  ├── fix/payment-timeout    (1-2 days)
  ├── feat/paytm-ios         (1-3 days)
  └── feat/new-channel-navi  (1 day)
```

- `main` always deployable
- Branches survive no longer than 3 days
- Small, frequent merges

### Commit Discipline

One commit = one logical change:
```
✅ "Add Paytm iOS device profile generation"
✅ "Fix UPI extraction for Navi bank"
❌ "Fix Navi UPI + update time_out + refactor BaseHandler"
```

Commit message format:
```
<type>: <short description>

<detailed description (optional)>

Ref: ISSUE-123
```

Types: `feat` / `fix` / `refactor` / `test` / `docs` / `chore`

### Branch Naming

```
fix/<description>      — bug fixes
feat/<description>     — new features
refactor/<description> — refactoring
```

### Merge After Review

```bash
# Create branch from main
git checkout main
git pull origin main
git checkout -b feat/new-bank

# Develop and commit
git add -A
git commit -m "feat: add new bank handler"

# Push and create MR
git push origin feat/new-bank
# Open MR in GitLab UI targeting main
```

## GitLab MR via API

```bash
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  --data "source_branch=feat/new-bank&target_branch=main&title=feat: add new bank" \
  "https://gitlab.example.com/api/v4/projects/<project>/merge_requests"
```

## Notes

- GitLab may be self-hosted (internal network), not github.com
- Store tokens in project config, not in code
