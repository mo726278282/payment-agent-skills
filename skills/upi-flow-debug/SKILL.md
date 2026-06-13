---
name: payment-data-flow-debugging
description: 支付数据流调试方法论 — 追踪数据从采集端到回调到持久化存储的完整链路。适用于排查"数据不一致"类问题。
tags: [payment, data-flow, debugging, consistency, reconciliation]
---

# Payment Data Flow Debugging（支付数据流调试）

当系统记录的关键字段与实际数据源不一致时，按以下链路逐段排查。

## 数据流全貌

```
采集端拉取 → 回调通知 → 后端接收 → 持久化更新
fetch_data()   callback    handler     UPDATE db
     │             │           │            │
     数据源 API     POST /callback  解析+校验    写入存储
```

**核心原则**: 不一致一定发生在链路上的某一个环节。不要猜测，逐段验证。

## 诊断步骤

### ① 确认当前存储的值

```sql
SELECT key_field, updated_at FROM data_table WHERE id = <business_id>;
```

### ② 确认数据源实时值

通过数据提取器直接从数据源 API 查询当前真实值：

```
live_data = fetch_from_source(business_id, channel_type)
# → { "success": true, "value": "real_value", "error": null }
```

### ③ 比对差异

| 存储值 | 实时值 | 说明 |
|--------|--------|------|
| `abc@bank` | `abc@bank` | ✅ 一致，没问题 |
| `abc@bank` | `xyz@bank` | ❌ 不一致 → 继续排查 |
| `abc@bank` | `null` | ⚠️ 数据源查询失败 → 检查数据源可用性 |
| `null` | `abc@bank` | ❌ 回调未触发 → 排查回调链路 |

### ④ 排查回调链路

在 API 日志中搜索回调记录：

```
# 搜索回调接口 + 业务 ID
curl -s 'https://es-host:9200/api-gateway-*/_search' \
  -d '{"query":{"match_phrase":{"LOG_FIELD":"/callback/endpoint"}},
       "_source":["@timestamp","LOG_FIELD"],"size":10,"sort":[{"@timestamp":"desc"}]}'
```

如果找不到回调记录 → 采集端未发送或发送失败。

### ⑤ 排查采集端

在采集端日志中搜索上报记录，确认：
- 是否成功拉取到数据
- 是否成功发送回调
- 回调参数是否正确

## 数据提取器架构

将各渠道的数据提取统一到一个模块：

```
fetch_value_by_id(id, channel)
  → call_channel_api(id, channel)     # 调渠道 API
  → extractors[channel](response)     # 按渠道解析
  → normalized_value                  # 标准格式返回
```

每个渠道的提取器是一个纯函数，输入 API 响应 JSON，输出标准化字段值。

**新增渠道时**：优先参考 API Handler 的调用方式（纯 HTTP），而不是参考 Collector（可能混杂 UI 自动化逻辑）。

## 渠道变更检测差异

部分渠道在检测到数据变更时的行为不同。记录这些差异很重要：

| 渠道 | 检测变更 | 记录历史 | 维护列表 |
|------|:---:|:---:|:---:|
| A | ✅ 比对后更新 | ✅ old→new | ✅ 追加 |
| B | ❌ 直接覆盖 | ❌ | ❌ 只存当前值 |

变更检测逻辑影响数据一致性——渠道 B 可能覆盖掉正确值。

## 快捷诊断脚本

```bash
#!/bin/bash
# 支付数据流一键诊断
ID=$1
echo "=== 存储 ==="
mysql -e "SELECT key_field, updated_at FROM data_table WHERE id=$ID"
echo "=== 实时 ==="
python -c "from extractors import fetch; print(fetch('$ID', 'channel_name'))"
echo "=== 回调日志 ==="
curl -s -u 'user:pass' "https://es-host:9200/api-gateway-*/_search" \
  -d "{\"query\":{\"match_phrase\":{\"LOG_FIELD\":\"$ID\"}},\"size\":5}"
echo "=== 采集端日志 ==="
curl -s -u 'user:pass' "https://es-host:9200/channel-a-*/_search" \
  -d "{\"query\":{\"match_phrase\":{\"LOG_FIELD\":\"$ID\"}},\"size\":5}"
```

## 常见坑

1. **参考错代码** — 新渠道的数据提取调用方式，参考 API Handler（纯 HTTP），不是 Collector（交互式 UI 登录）。
2. **部分渠道不检测变更** — 直接覆盖旧值，不做 `old != new` 比对。
3. **返回值格式不统一** — 不同渠道可能返回不同格式（VPA、QR ID、账号），提取时需要 normalize。
4. **缓存污染** — 数据提取可能有缓存层，排查问题时确认拿到的是实时值。
