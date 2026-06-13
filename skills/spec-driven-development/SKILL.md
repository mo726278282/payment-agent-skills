---
name: spec-driven-development
description: Create a spec before writing code. Use when starting a new project, feature, or significant change where requirements are unclear.
---

# Spec-Driven Development

## Overview

Write a structured specification before writing any code. The spec is the shared source of truth — it defines what we're building, why, and how we'll know it's done. Code without a spec is guessing.

## When to Use

- Starting a new project or feature
- Requirements are ambiguous or incomplete
- The change touches multiple files or modules
- You're about to make an architectural decision
- The task would take more than 30 minutes to implement

**Skip for:** single-line fixes, config changes, pure refactoring.

## Spec Structure

```markdown
# Spec: [Feature Name]

## Goal
One sentence describing what this feature does.

## Background
Why is this needed? What problem does it solve?

## Scope
### In Scope
- Specific deliverables

### Out of Scope
- Explicitly excluded items

## Design
### Data Flow
[Architecture diagram or data flow description]

### API
[Endpoint definitions]

### Database Changes
[Tables / fields to modify]

## Task Breakdown
1. [Task list]
...

## Acceptance Criteria
- [ ] Verifiable signs that the feature works
- [ ] Edge cases covered
- [ ] Tests pass

## Risks
- Potential issues and mitigations
```

## Example: Adding a Payment Channel

```markdown
# Spec: Add XYZ Bank Payment Channel

## Goal
Support XYZ Bank as a new payment channel.

## Background
XYZ Bank is an emerging digital bank with UPI account query API.
Authentication: OTP + OAuth2 token.

## Scope
### In Scope
- Channel Handler (pre_login / send_otp / verify_otp / fetch_accounts)
- Background Collector
- Data Extractor
- Verification pipeline mappings

### Out of Scope
- SMS notification parsing (XYZ has no SMS alerts)
- WebView login (API-only integration)

## API
- Account query: POST https://api.xyzbank.com/v1/accounts/query
- OTP send: POST https://api.xyzbank.com/v1/auth/otp/send
- OAuth2: POST https://api.xyzbank.com/v1/auth/token

## Database
- INSERT INTO channel_type (name, status) VALUES ('XYZBANK', 1)

## Tasks
1. DB: channel_type table record
2. Channel Handler: XYZBank class
3. Controller dispatch
4. Collector: jobs/xyzbank/xyzbank.py
5. Data Extractor
6. Verification pipeline mappings

## Acceptance
- [ ] POST /api/v1/login/pre_login channel_name=xyzbank returns session
- [ ] Collector can login and fetch accounts
- [ ] Callback correctly updates stored state
- [ ] Full trace visible in log system
```

After writing the spec, use the `planning-and-task-breakdown` skill to decompose it into implementable tasks.
