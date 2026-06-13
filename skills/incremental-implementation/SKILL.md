---
name: incremental-implementation
description: 增量实现——一次改一点，测过再继续。适用于多文件变更、大功能开发、重构。
---

# 增量实现

## 概述

垂直切片式构建——实现一小片，测试，验证，然后扩展。避免一次性实现整个功能。每个增量都应使系统处于可工作、可测试的状态。

## 何时使用

- 实现任何跨文件的变更
- 从任务拆解构建新功能
- 重构现有代码
- **任何你想一次写超过 ~100 行代码的时候**

## 核心原则

### 小步提交

```
  实现 → 测试 → 验证 → 提交
  └────── 一个增量 ──────┘
                          → 实现 → 测试 → 验证 → 提交
                            └────── 下一个增量 ──────┘
```

每步之后：`git commit`。出错时 `git revert` 一个 commit 即可，不用回退大段代码。

### 保持系统可工作

永远不要让系统处于损坏状态。每步之后：
```bash
pytest tests/ -q  # 全绿
```

### 每步一个关注点

一个 commit = 一个逻辑变更。不要在一个 commit 里同时改 Handler 和 time_out。

## 20k 项目的典型增量顺序

以新增支付渠道为例：

```
Commit 1: 数据库 — INSERT bank_type
Commit 2: Bank Handler 骨架 (pre_login_http)
Commit 3: Bank Handler OTP 流程 (send_otp + verify_otp)
Commit 4: Bank Handler UPI 流程 (get_upi_list + set_upi)
Commit 5: Controller dispatch 注册
Commit 6: Collector 实现
Commit 7: UPI Extractor (grab_upi.py)
Commit 8: time_out 映射表更新
Commit 9: 端到端验证脚本
```

## 反模式

- ❌ "先全部写完再测试" — 一测就 10 个错，不知道哪个改坏的
- ❌ "这个 commit 包含了 3 个不相关的改动" — 无法独立 revert
- ❌ "等我重构完再提交" — 重构到一半发现方向错了，没有回退点
