---
name: planning-and-task-breakdown
description: Break work into ordered, verifiable tasks. Use when you have a spec and need to decompose it into implementable units, or when a task feels too large to start.
---

# Planning and Task Breakdown

## Overview

Decompose work into small, verifiable tasks with explicit acceptance criteria. Each task should be small enough to implement, test, and verify in a single focused session.

## When to Use

- You have a spec and need to break it into implementable units
- A task feels too large or vague to start
- Work needs to be parallelized across multiple agents
- Scoping or estimating work

## Principles

### Small Tasks

Each task:
- Touches one file or 2-3 closely related files
- Is one clear change, not "refactor authentication module"
- Is independently testable, not dependent on other unfinished tasks
- Has clear completion criteria

### Ordered Dependencies

Label dependencies between tasks:
```
Task 1: Create channel_type table record        ← no dependency
Task 2: Implement Channel Handler               ← depends on 1
Task 3: Register dispatch in controller         ← depends on 2
Task 4: Implement Collector                     ← depends on 1
Task 5: Add data extractor                      ← depends on 2
Task 6: Update verification pipeline mappings   ← depends on 1
```

### Acceptance Criteria

Every task lists "what done looks like":
```
Task 2: Implement Channel Handler
  ✅ pre_login() passes HTTP API test
  ✅ send_otp() successfully delivers OTP
  ✅ verify_otp() validates correctly
  ✅ fetch_accounts() returns correct account list
```

### Parallel Markers

Tag tasks that can run in parallel:
```
Task 4 (Collector) ∥ Task 2 (Handler)   ← parallelizable
Task 5 (Extractor) ∥ Task 6 (Mappings)  ← parallelizable
```

## Output Format

```markdown
## Task Breakdown: [Feature Name]

### Prerequisites
- [ ] Confirm channel API documentation
- [ ] Obtain test credentials

### Tasks
- [ ] 1. Database setup (channel_type table)
- [ ] 2. Channel Handler implementation ∥
- [ ] 3. Controller dispatch registration
- [ ] 4. Collector implementation ∥
- [ ] 5. Data extractor
- [ ] 6. Verification pipeline mappings
- [ ] 7. End-to-end validation

### Estimate
- Total tasks: 7
- Parallelizable: 2 groups
- Estimated time: 4-6 hours
```

For payment channel onboarding, use the `payment-channel-onboard` skill as the standard task template.
