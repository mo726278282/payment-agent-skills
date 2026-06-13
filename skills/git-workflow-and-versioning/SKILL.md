---
name: git-workflow-and-versioning
description: Git 工作流。做任何代码变更时使用——提交、分支、解决冲突、组织多路并行开发。
---

# Git 工作流

## 概述

Git 是安全网。commit 是存档点，branch 是沙盒，history 是文档。

## 核心原则

### Trunk-Based（推荐）

```
main 始终保持可部署
  ├── fix/payment-timeout    (1-2天)
  ├── feat/paytm-ios         (1-3天)
  └── feat/new-channel-navi  (1天)
```

- `main` 始终可部署
- 分支存活不超过 3 天
- 小步快合

### Commit 规范

一个 commit = 一个逻辑变更：
```
✅ "Add Paytm iOS device profile generation"
✅ "Fix UPI extraction for Navi bank"
❌ "Fix Navi UPI + update time_out + refactor BaseHandler"
```

Commit message 格式：
```
<type>: <简短描述>

<详细说明（可选）>

Ref: KPAY-123
```

Type: `feat` / `fix` / `refactor` / `test` / `docs` / `chore`

### 分支命名

```
fix/<描述>      — bug 修复
feat/<描述>     — 新功能
refactor/<描述> — 重构
```

20k 项目的目标分支：`final_dev`（或 `dev_1.0.2`，与 Jira skill 保持一致）。

### 代码审查后合并

```bash
# 从 final_dev 创建分支
git checkout final_dev
git pull origin final_dev
git checkout -b feat/new-bank

# 开发、提交
git add -A
git commit -m "feat: add new bank handler"

# 推送、创建 MR
git push origin feat/new-bank
# 在 GitLab UI 创建 MR，目标 final_dev
```

## GitLab MR 操作

```bash
# 通过 API 创建 MR
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  --data "source_branch=feat/new-bank&target_branch=final_dev&title=feat: add new bank" \
  "http://35.241.87.36/api/v4/projects/20kpay%2Fapi/merge_requests"
```

## 20k 项目注意

- GitLab 在 `http://35.241.87.36`（内网），不是 github.com
- 仓库路径：`20kpay/api`、`20kpay/skills` 等
- Token 在项目 `prod.md` 或技能配置中
