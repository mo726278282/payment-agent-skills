---
name: payment-channel-adapter-pattern
description: 多渠道支付适配器架构 — 新增支付渠道的通用模式。适用于任何需要对接多个银行/钱包/第三方支付渠道的支付系统。
tags: [payment, channel, adapter, architecture, onboarding]
---

# Payment Channel Adapter Pattern（多渠道支付适配器）

任何对接多个银行/钱包/第三方支付渠道的支付系统，都可以用这套模式统一新渠道接入。

## 核心架构

```
                    ┌──────────────────┐
                    │   Dispatch Layer  │  ← 按 channel_name 路由
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │Channel A │  │Channel B │  │Channel C │  ← Channel Handler
        │ Handler  │  │ Handler  │  │ Handler  │
        └──────────┘  └──────────┘  └──────────┘
              │              │              │
              ▼              ▼              ▼
        ┌──────────────────────────────────────┐
        │       Base Channel Handler           │  ← 共享基础设施
        │  • 数据库操作  • Session 管理         │
        │  • 重试/锁    • 凭证加密              │
        │  • 代理管理   • 错误日志              │
        └──────────────────────────────────────┘
```

## 接入清单（7 步）

### 1. 研究渠道 API

回答三个问题：
- **认证方式**：OTP / OAuth2 / API Key / 证书？
- **数据获取**：API 返回什么？关键字段在哪层 JSON？
- **网络要求**：需要特定地区 IP？代理？

### 2. 实现 Channel Handler

创建 `<Channel>Handler` 类。关键方法：

```
pre_login()       — 初始化会话、验证用户凭证
send_otp()        — 发送验证码（如适用）
verify_otp()      — 验证验证码
fetch_accounts()  — 获取账户/钱包列表
set_active()      — 选择活跃账户
```

**务必提取 BaseHandler** — 数据库操作、Session 管理、重试逻辑、锁、错误日志等一律放基类。子类只写渠道特有的 API 调用和响应解析逻辑。避免每个渠道复制粘贴几百行基础设施代码。

### 3. 注册 Dispatch

在路由层添加渠道分发：

```
if channel_name == 'channel_a':
    handler = ChannelAHandler(request)
elif channel_name == 'channel_b':
    handler = ChannelBHandler(request)
else:
    raise UnsupportedChannelError(channel_name)
```

每个 HTTP 接口方法（pre_login / send_otp / verify_otp / fetch_accounts / set_active）都需要添加对应分支。

### 4. 实现 Collector（后台采集端）

如果渠道需要后台定时轮询/长连接采集数据：

- 独立进程或 worker
- 登录 → 维持会话 → 定时拉取数据 → 回调后端
- 处理 token 过期、代理失效等异常

Collector ↔ Backend 通信契约：

```json
{
  "channel_id": "xxx",
  "type": "ACCOUNT_UPDATE",
  "status": 1,
  "accounts": [{"id": "abc@bank", "name": "Bank Name"}]
}
```

### 5. 实现数据提取器

每个渠道的 API 响应结构不同，需要一个统一的提取层：

```
extract_value_by_id(id, channel_name)
  → call_channel_api(id)
  → extractor[channel](response_json)  ← 渠道特定解析
  → normalized_value
```

提取器从 API 响应 JSON 中取出关键字段（账户 ID、余额、状态等），返回标准化格式。

### 6. 更新验证流水线

如果系统有订单超时/冻结后的验证流水线，需要：
- 新增渠道的 **ID → 名称** 映射
- 新增渠道的 **名称 → 数据源名称** 映射（不同渠道可能用不同爬取工具）
- 如果新渠道需要特殊验证逻辑，加入对应的检查步骤

### 7. 数据库

渠道元数据表：
```sql
INSERT INTO channel_type (name, status) VALUES ('CHANNEL_NAME', 1);
```

## 验证清单

- [ ] Handler 通过 HTTP API 测试（各接口返回正确）
- [ ] Collector 能登录并获取数据
- [ ] 数据回调正常更新后端状态
- [ ] 数据提取器能查询到实时数据
- [ ] 日志系统能看到完整的登录→获取→回调链路
- [ ] 验证流水线（如有）对该渠道正确执行

## 常见坑

1. **手机号/账号格式** — 不同渠道格式不同（带/不带国家码），统一规范化后再比较。
2. **代理 IP** — 部分渠道需要特定地区 IP，用配置中心管理（如 Redis key `proxy_ip_{channel}`）。
3. **HTTP 客户端选择** — 部分渠道需要 TLS 指纹伪装（如 `curl_cffi`），普通 `requests` 可能被拦截。
4. **Handler 重复代码** — 第一个渠道可能是独立实现，第二个渠道加入时必须提取 BaseHandler。不要等到第 5 个渠道再重构。
5. **Dispatch 遗漏** — 每个 HTTP 接口方法都要加分支，遗漏任何一个都会导致 404/不支持错误。
6. **渠道名映射不一致** — Handler、Dispatch、Collector、验证流水线中渠道名拼写必须一致（注意大小写和下划线）。
