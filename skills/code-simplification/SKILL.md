---
name: code-simplification
description: 代码简化——减复杂度不改行为。重构、审查发现冗余、代码能跑但难读时使用。
---

# 代码简化

## 概述

简化代码——减少复杂度但保持行为不变。目标不是更少行，而是更容易读、理解、修改和调试。

核心测试：**"一个新加入的同事能看懂这段代码吗？"**

## 简化方向

### 1. 消除重复

```python
# Before — 两个银行 Handler 各自 copy-paste 了 400 行
class PaytmBank:
    async def _check_login_failed_attempts(self, phone): ...
    async def _record_login_failed_attempt(self, phone): ...
    # ... 400 行基础设施代码

class AirtelBank:
    async def _check_login_failed_attempts(self, phone): ...  # 完全一样
    async def _record_login_failed_attempt(self, phone): ...  # 完全一样
    # ... 400 行复制粘贴

# After — 提取到 BaseBankHandler，子类只需实现业务逻辑
class PaytmBank(BaseBankHandler):
    async def pre_login_http(self, data): ...  # 只写业务代码
```

### 2. 降低嵌套

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

### 3. 命名表意

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

### 4. 函数做一件事

```python
# Before — 一个函数做三件事
async def handle_timeout(order):
    # 1. 检查采集端在线
    is_online = check_collector(order.payment_id)
    # 2. 爬取账单
    bill = await grab_bill(order.payment_id)
    # 3. 决定解冻/取消
    if is_online and bill:
        unfreeze(order)
    else:
        cancel(order)

# After — 职责分离
async def handle_timeout(order):
    can_unfreeze = await verify_unfreeze_conditions(order)
    execute_decision(order, can_unfreeze)

async def verify_unfreeze_conditions(order):
    collector_ok = check_collector(order.payment_id)
    bill_ok = await probe_bill(order.payment_id)
    return collector_ok and bill_ok
```

## 简化原则

- **先有测试保护** — 简化前确保测试全绿
- **每次只改一处** — 不要同时重构多个文件
- **改完立刻跑测试** — 确认行为不变
- **不改行为** — 只改结构，不改功能
- **不改测试** — 如果测试需要改，说明你改了行为
