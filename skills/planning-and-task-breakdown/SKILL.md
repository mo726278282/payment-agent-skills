---
name: planning-and-task-breakdown
description: 需求拆解成可执行任务。当你拿到 spec 或需求文档、任务太大无法下手、需要评估工作量、或需要并行分配时使用。
---

# 任务规划和拆解

## 概述

把工作拆成小粒度的、可验证的任务，每个任务附明确的验收标准。好的任务拆解是 agent 稳定交付 vs 产出混乱代码的关键区别。每个任务应该小到能在单次 session 中实现、测试、验证。

## 何时使用

- 有 spec，需要拆成可实现的单元
- 任务太模糊或太大，无从下手
- 需要并行化（多 agent 协作）
- 估算工作量

## 拆解原则

### 小任务

每个任务：
- 单文件或最多 2-3 个相关文件
- 一道明确的变更，不是"重构认证模块"
- 可独立测试，不依赖其他未完成的任务
- 有明确完成标准

### 有序依赖

标注任务间的依赖关系：
```
任务 1: 创建 bank_type 表记录          ← 无依赖
任务 2: 实现 Bank Handler               ← 依赖 1
任务 3: 注册 HTTP Controller dispatch   ← 依赖 2
任务 4: 实现 Collector                  ← 依赖 1
任务 5: 添加 UPI Extractor             ← 依赖 2
任务 6: 更新 time_out 映射表            ← 依赖 1
```

### 验收标准

每个任务写清楚"怎么算做完"：
```
任务 2: 实现 Bank Handler
  ✅ pre_login_http() 通过 HTTP 接口测试
  ✅ send_otp_http() 成功发送 OTP
  ✅ verify_otp_http() 成功验证
  ✅ get_upi_list_http() 返回正确 UPI 列表
```

### 并行标记

标注哪些任务可以并行：
```
任务 4 (Collector) ∥ 任务 2 (Handler)  ← 可并行
任务 5 (UPI Extractor) ∥ 任务 6 (time_out) ← 可并行
```

## 输出格式

```markdown
## 任务拆解: [功能名]

### 前置
- [ ] 确认银行 API 文档
- [ ] 获取测试账号

### 任务列表
- [ ] 1. 数据库准备 (bank_type 表)
- [ ] 2. Bank Handler 实现 ∥
- [ ] 3. Controller dispatch 注册
- [ ] 4. Collector 实现 ∥
- [ ] 5. UPI Extractor
- [ ] 6. time_out 映射表更新
- [ ] 7. 端到端验证

### 预估
- 总任务: 7 个
- 可并行: 2 组
- 预估时间: 4-6 小时
```

## 结合 20k 项目

对于 20k 支付系统，`payment-channel-onboard` 技能提供了标准的任务模板，直接用那个拆。
