---
name: code-review-and-quality
description: 多维度代码审查。合并前使用——自查、审查 agent 产出、审查同事代码。
---

# Code Review（代码审查）

## 概述

5 轴代码审查，带质量门禁。每次变更合并前必须审查，无例外。

**审批标准**: 变更明确改善整体代码健康度即可通过，不必完美。不要因为"不是你的写法"就阻止。

## 5 轴审查

### 1. 正确性 — 它做对了吗？

- 逻辑正确？边界条件覆盖？
- 空值处理？异常路径？
- 并发安全？事务边界正确？

### 2. 可读性 — 半年后能看懂吗？

- 命名是否清晰表达意图？
- 复杂逻辑有注释？
- 函数长度合理（< 50 行）？

### 3. 架构 — 放对位置了吗？

- 是否遵循已有模式？
- 引入新依赖有充分理由吗？
- 是否在正确的层（handler / service / model）？

### 4. 安全性 — 有漏洞吗？

- SQL 注入（参数化查询）？
- 敏感信息泄露（日志中打印密码/token）？
- 输入验证充分？

### 5. 性能 — 会拖慢系统吗？

- N+1 查询？
- 不必要的大数据加载？
- 缺少索引？

## 20k 项目特有检查

### time_out.py 同步修改检查
如果改了 TIMEOUT 路径，HOLD_RETRY 路径也得同步改。这是最容易出错的地方。

### Bank Handler 重复代码
新 Handler 是否复用了 BaseBankHandler？还是又复制了 400 行？

### ES 日志
新增的关键操作是否有对应的 log（`[TAG][STEP]` 格式）？

### 银行映射表
新增银行时，`bank_id_to_name` 和 `bank_mapping` 是否都更新了？

## GitLab MR 审查流程

```bash
# 获取 MR 变更
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "http://35.241.87.36/api/v4/projects/20kpay%2Fapi/merge_requests/MR_ID/changes"

# 查看 diff
git diff final_dev...feature/branch-name
```

## 审查报告模板

```markdown
## MR Review: [!MR_ID] 标题

### 总结
[一句话概述]

### 发现
| # | 严重度 | 轴 | 问题 |
|---|--------|-----|------|
| 1 | 🔴 高 | 正确性 | ... |
| 2 | 🟡 中 | 架构 | ... |
| 3 | 🟢 低 | 可读性 | ... |

### 建议
- [具体改动建议]

### 审批
✅ 通过（修改建议后） / ❌ 需修改
```
