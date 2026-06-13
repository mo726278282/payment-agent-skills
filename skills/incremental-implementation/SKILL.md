---
name: incremental-implementation
description: Deliver changes incrementally — one slice at a time, test, verify, then expand. Use for multi-file changes, feature builds, and refactoring.
---

# Incremental Implementation

## Overview

Build in thin vertical slices — implement one piece, test it, verify it, then expand. Avoid implementing an entire feature in one pass. Each increment should leave the system in a working, testable state.

## When to Use

- Implementing any multi-file change
- Building a new feature from a task breakdown
- Refactoring existing code
- **Any time you're tempted to write more than ~100 lines before testing**

## Core Principles

### Small Commits

```
  Implement → Test → Verify → Commit
  └─────── one increment ───────┘
                                → Implement → Test → Verify → Commit
                                  └─────── next increment ────────┘
```

After each step: `git commit`. If something breaks, revert one commit — not a massive block.

### Keep the System Working

Never leave the system in a broken state. After each step:
```bash
pytest tests/ -q  # all green
```

### One Concern Per Step

One commit = one logical change. Don't change the Handler and time_out in one commit.

## Typical Increment Order for Channel Onboarding

```
Commit 1: Database — INSERT channel_type record
Commit 2: Handler skeleton (pre_login)
Commit 3: Handler OTP flow (send_otp + verify_otp)
Commit 4: Handler account flow (fetch_accounts + set_active)
Commit 5: Controller dispatch registration
Commit 6: Collector implementation
Commit 7: Data extractor
Commit 8: Verification pipeline mappings
Commit 9: End-to-end validation
```

## Anti-Patterns

- ❌ "Write everything first, then test" — 10 bugs, no idea which change broke what
- ❌ "This commit has 3 unrelated changes" — can't revert independently
- ❌ "I'll commit after I finish refactoring" — wrong direction halfway, no rollback point
