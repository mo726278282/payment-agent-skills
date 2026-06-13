---
name: code-simplification
description: Simplify code for clarity without changing behavior. Use when refactoring, reviewing code with accumulated complexity, or when code works but is hard to read.
---

# Code Simplification

## Overview

Reduce complexity while preserving exact behavior. The goal is not fewer lines — it's code that is easier to read, understand, modify, and debug.

The core test: **"Would a new team member understand this code?"**

## Simplification Axes

### 1. Eliminate Duplication

```python
# Before — two bank handlers each copy-paste 400 lines of infrastructure
class ChannelAHandler:
    async def _check_login_failed_attempts(self, phone): ...
    async def _record_login_failed_attempt(self, phone): ...
    # ... 400 lines of boilerplate

class ChannelBHandler:
    async def _check_login_failed_attempts(self, phone): ...  # identical
    async def _record_login_failed_attempt(self, phone): ...  # identical
    # ... 400 lines copied

# After — extract to BaseHandler, subclasses focus on business logic
class ChannelAHandler(BaseChannelHandler):
    async def pre_login(self, data): ...  # business logic only
```

### 2. Reduce Nesting

```python
# Before
if user:
    if user.status == 1:
        if user.balance > amount:
            result = process()
            return result
return None

# After
if not user:
    return None
if user.status != 1:
    return None
if user.balance <= amount:
    return None
return process()
```

### 3. Name for Intent

```python
# Before
d = get_data()
r = do_stuff(d)
if r[0]:
    p(r[1])

# After
payment_data = fetch_payment(payment_id)
result = verify_unfreeze_conditions(payment_data)
if result.can_unfreeze:
    execute_unfreeze(result)
```

### 4. One Function, One Job

```python
# Before — one function does three things
async def handle_timeout(order):
    is_online = check_collector(order.payment_id)
    bill = await probe_bill(order.payment_id)
    if is_online and bill:
        unfreeze(order)
    else:
        cancel(order)

# After — separated responsibilities
async def handle_timeout(order):
    can_unfreeze = await verify_unfreeze_conditions(order)
    execute_decision(order, can_unfreeze)
```

## Principles

- **Test protection first** — ensure all tests pass before simplifying
- **One change at a time** — don't refactor multiple files simultaneously
- **Run tests immediately** — confirm behavior unchanged after each change
- **Don't change behavior** — only structure, not functionality
- **Don't change tests** — if tests need changing, you changed behavior
