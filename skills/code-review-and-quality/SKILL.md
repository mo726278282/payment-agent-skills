---
name: code-review-and-quality
description: Multi-axis code review with quality gates. Use before merging any change — self-review, agent output review, or peer review.
---

# Code Review and Quality

## Overview

5-axis code review with quality gates. Every change gets reviewed before merge — no exceptions.

**The approval standard:** Approve a change when it definitely improves overall code health, even if it isn't perfect. Don't block because it isn't exactly how you'd write it.

## 5 Axes

### 1. Correctness — Does it do the right thing?

- Logic correct? Edge cases covered?
- Null handling? Exception paths?
- Concurrency-safe? Transaction boundaries correct?

### 2. Readability — Can someone understand this in 6 months?

- Names clearly express intent?
- Complex logic commented?
- Function length reasonable (< 50 lines)?

### 3. Architecture — Is it in the right place?

- Follows existing patterns?
- New dependencies justified?
- Correct layer (handler / service / model)?

### 4. Security — Any vulnerabilities?

- SQL injection (parameterized queries)?
- Sensitive info in logs (passwords, tokens)?
- Input validation sufficient?

### 5. Performance — Will it slow things down?

- N+1 queries?
- Unnecessary large data loads?
- Missing indices?

## Payment System Specific Checks

### Dual-Path Sync
If you changed the TIMEOUT path, the HOLD_RETRY path must be synced. This is the most common bug source.

### Channel Handler Duplication
Does the new handler reuse BaseHandler, or copy-paste 400 lines of infrastructure?

### Logging
New critical operations must have corresponding logs in `[TAG][STEP]` format.

### Mapping Tables
When adding a channel, are both `channel_id_to_name` and `channel_name_to_source` updated?

## Review Report Template

```markdown
## Review: [Change Title]

### Summary
[One-line overview]

### Findings
| # | Severity | Axis | Issue |
|---|----------|------|-------|
| 1 | 🔴 High | Correctness | ... |
| 2 | 🟡 Medium | Architecture | ... |
| 3 | 🟢 Low | Readability | ... |

### Recommendations
- [Specific suggestions]

### Verdict
✅ Approve (after suggestions) / ❌ Needs rework
```
