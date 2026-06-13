---
name: spec-driven-development
description: 先写 spec 再写代码。新项目/新功能/需求不明确时使用。
---

# 规格驱动开发

## 概述

写代码之前先写结构化规格。Spec 是你和工程师之间的共用真相源——定义要做什么、为什么、以及怎么算做完。

无 spec 写代码 = 猜测。

## 何时使用

- 启动新项目或功能
- 需求模糊或残缺
- 改动涉及多文件多模块
- 要做架构决策
- 预估实现超过 30 分钟

**不用**: 单行修复、配置变更、纯重构。

## Spec 结构

```markdown
# Spec: [功能名]

## 目标
一句话描述这个功能做什么。

## 背景
为什么需要？解决什么问题？

## 范围
### 包含
- 具体交付物

### 不包含
- 明确排除的内容

## 设计
### 数据流
[架构图或数据流描述]

### API
[接口定义]

### 数据库变更
[表/字段变更]

## 任务拆解
1. [任务列表]
...

## 验收标准
- [ ] 功能正常工作的可验证标志
- [ ] 边界情况覆盖
- [ ] 测试通过

## 风险
- 潜在问题和缓解方案
```

## 20k 项目示例

以新增支付渠道为例：

```markdown
# Spec: 新增 XYZ Bank 支付渠道

## 目标
支持 XYZ Bank 作为新的 UPI 支付渠道。

## 背景
XYZ Bank 是印度新兴数字银行，支持 UPI VPA 查询 API。
认证方式: OTP + OAuth2 token。

## 范围
### 包含
- Bank Handler (pre_login / send_otp / verify_otp / get_upi_list)
- Collector 采集端
- UPI Extractor (grab_upi.py)
- time_out 映射更新

### 不包含
- SMS 到账通知（XYZ 无 SMS 通知）
- WebView 登录（仅 API）

## API
- VPA 查询: POST https://api.xyzbank.com/v1/upi/vpa
- OTP 发送: POST https://api.xyzbank.com/v1/auth/otp/send
- OAuth2: POST https://api.xyzbank.com/v1/auth/token

## 数据库
- INSERT INTO bank_type (name, status) VALUES ('XYZBANK', 1)

## 任务
1. DB: bank_type 表
2. Bank Handler: XYZBank 类
3. Controller dispatch
4. Collector: jobs/xyzbank/xyzbank.py
5. UPI Extractor
6. time_out 映射

## 验收
- [ ] POST /api/v1/login/pre_login bankname=xyzbank 返回 session
- [ ] Collector 能登录并获取 UPI
- [ ] /order/Success 回调正常更新 payment.upi
- [ ] ES 日志完整追踪链路
```

## 结合 planning-and-task-breakdown

写完 spec 后，用 `planning-and-task-breakdown` 技能拆成具体实现任务。
