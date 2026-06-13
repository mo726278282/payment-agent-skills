---
name: es-log-tracing
description: Cross-index distributed log tracing — stitch request lifecycles across multiple indices by business ID. For debugging payment flows, callbacks, and order processing in microservice architectures.
tags: [logging, tracing, distributed, debugging, elk, elasticsearch]
---

# Distributed Log Tracing

In a payment system, one request might touch 3-4 services. Logs end up scattered across different indices. The skill: trace the full lifecycle by business ID.

## Index layout

A typical payment system splits logs across indices by function:

| Index/topic | What's in it | Used for |
|-------------|-------------|----------|
| `order-verification-*` | Timeout verification flow | Release/cancel decisions |
| `api-gateway-*` | HTTP request logs | Callbacks, API calls |
| `channel-a-*` | Channel A collector | Channel A operations |
| `channel-b-*` | Channel B collector | Channel B operations |

Indices are usually time-sharded: `{prefix}-{YYYY.MM.dd}`

## First: find the right field

Different logging pipelines (Filebeat → Logstash → ES, Fluentd, Loki) put the raw log in different fields:

- `event.original` — Filebeat raw log
- `message` — parsed subset (may not have everything)
- `log` — some agents use this

Don't guess. Run a query without filters on one index and look at a result to see where the text lives.

## Query template

```bash
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

- `INDEX-*` hits all date shards
- `match_phrase` for exact phrase matching (watch for newlines breaking the match)
- `_source` limits what comes back — cut transfer size
- `sort` by time to reconstruct event order

## Common traces

### Full lifecycle for one business ID

```bash
for idx in order-verification api-gateway channel-a channel-b; do
  echo "=== $idx-* ==="
  curl -s -u 'user:password' \
    "https://es-host:9200/${idx}-*/_search" \
    -H 'Content-Type: application/json' \
    -d '{"query":{"match_phrase":{"LOG_FIELD":"BUSINESS_ID"}},"_source":["@timestamp","LOG_FIELD"],"size":5,"sort":[{"@timestamp":"asc"}]}'
done
```

### Failed verifications

```bash
curl -s -u 'user:password' \
  'https://es-host:9200/order-verification-*/_search' \
  -d '{"query":{"bool":{"must":[{"match_phrase":{"LOG_FIELD":"VERIFY_TAG"}},{"match_phrase":{"LOG_FIELD":"FAILED"}}]}},"size":20,"sort":[{"@timestamp":"desc"}]}'
```

### Time range filter

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

## Log tagging convention

Structured tags make cross-index search efficient:

```
[ORDER_TIMEOUT][UPI_CHECK] field=value, field2=value2
[ORDER_TIMEOUT][BILL_PROBE] channel=A, result=ok
[COLLECTOR_HEALTH] channel=A, online=true
```

Fixed-prefix tags for `match_phrase`. `key=value` pairs for downstream grep/awk.

## Pitfalls

1. **Field name fragmentation** — different services use different field names. Check before searching.
2. **Index date format** — confirm whether it's `prefix-YYYY.MM.dd` or `prefix-YYYY-MM-dd`.
3. **match_phrase vs newlines** — if logs contain `\n`, search with a substring that doesn't cross the line break, or use `match` instead.
4. **size cap** — ES defaults to 10 results. Bump it when you need more.
5. **Timezone** — `@timestamp` is usually UTC. Adjust your mental clock.
