---
name: distributed-log-tracing
description: 分布式日志链路追踪 — 跨多个索引/主题按业务 ID 串联请求全链路。适用于微服务架构下排查支付、订单、回调等问题。
tags: [logging, tracing, distributed, debugging, elk, elasticsearch]
---

# Distributed Log Tracing（分布式日志追踪）

微服务架构下，一次支付请求可能经过 3-4 个服务，日志分散在不同索引/主题中。核心技能：**按业务 ID 跨索引串联全链路**。

## 日志基础设施

典型支付系统的日志索引/主题划分：

| 索引/主题 | 内容 | 用途 |
|----------|------|------|
| `order-verification-*` | 订单超时验证流程 | 解冻/取消决策追踪 |
| `api-gateway-*` | HTTP API 请求日志 | 回调、接口调用 |
| `channel-a-*` | 渠道 A 采集端 | 渠道 A 操作日志 |
| `channel-b-*` | 渠道 B 采集端 | 渠道 B 操作日志 |
| ... | ... | ... |

索引通常按天分片：`{prefix}-{YYYY.MM.dd}`

## 关键前提：知道日志在哪个字段

不同的日志收集方案（Filebeat → Logstash → ES / Fluentd / Loki），原始日志内容可能在不同字段：

- `event.original` — Filebeat 原始日志
- `message` — 解析后的消息（可能是子集）
- `log` — 某些 agent 的字段名

**先确认你的系统用哪个字段**。不确定时用 `_search` 不加过滤先看一条。

## 查询模板

```bash
# 通用模板
curl -s -u 'user:password' \
  'https://es-host:9200/INDEX-*/_search' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "bool": {
        "must": [
          {"match_phrase": {"LOG_FIELD": "KEYWORD"}},
          {"match_phrase": {"LOG_FIELD": "BUSINESS_ID"}}
        ]
      }
    },
    "_source": ["@timestamp", "LOG_FIELD", "level"],
    "size": 20,
    "sort": [{"@timestamp": "desc"}]
  }'
```

关键参数：
- `INDEX-*` — 通配所有日期分片
- `match_phrase` — 精确短语匹配（注意日志中的换行符可能阻断匹配）
- `_source` — 只返回需要的字段，减少传输
- `sort` — 按时间排序还原事件顺序

## 常见追踪场景

### 查询某个业务 ID 的完整生命周期

```bash
for idx in order-verification api-gateway channel-a channel-b; do
  echo "=== $idx-* ==="
  curl -s -u 'user:password' \
    "https://es-host:9200/${idx}-*/_search" \
    -H 'Content-Type: application/json' \
    -d "{\"query\":{\"match_phrase\":{\"LOG_FIELD\":\"BUSINESS_ID\"}},\"_source\":[\"@timestamp\",\"LOG_FIELD\"],\"size\":5,\"sort\":[{\"@timestamp\":\"asc\"}]}"
done
```

### 查询验证流水线失败

```bash
curl -s -u 'user:password' \
  'https://es-host:9200/order-verification-*/_search' \
  -d '{"query":{"bool":{"must":[{"match_phrase":{"LOG_FIELD":"VERIFY_TAG"}},{"match_phrase":{"LOG_FIELD":"FAILED"}}]}},"size":20,"sort":[{"@timestamp":"desc"}]}'
```

### 查询回调是否触发

在 API 网关索引中搜索回调路径 + 业务 ID。

### 时间范围过滤

```json
{
  "query": {
    "bool": {
      "must": [...],
      "filter": [
        {"range": {"@timestamp": {"gte": "2026-06-10T00:00:00", "lte": "2026-06-13T23:59:59"}}}
      ]
    }
  }
}
```

## 日志标记规范（推荐）

为了让跨索引搜索更高效，日志应包含结构化的 TAG：

```
[ORDER_TIMEOUT][UPI_CHECK] 描述: field=value, field2=value2
[ORDER_TIMEOUT][BILL_PROBE] 描述: channel=A, result=ok
[COLLECTOR_HEALTH] channel=A, online=true
```

TAG 固定前缀，便于 `match_phrase` 精确搜索。`key=value` 格式便于后续 grep/awk 处理。

## 常见坑

1. **日志字段名不统一** — 不同服务可能用不同字段（`event.original` vs `message` vs `log`）。先确认。
2. **索引名格式** — 确认是 `prefix-YYYY.MM.dd` 还是 `prefix-YYYY-MM-dd`。
3. **match_phrase 与换行符** — 如果日志中有 `\n`，用不含换行的子串搜索，或用 `match` 代替 `match_phrase`。
4. **size 限制** — ES 默认返回 10 条，需要更多时设大 `size`。
5. **时区** — `@timestamp` 通常是 UTC，搜索时注意时区转换。
