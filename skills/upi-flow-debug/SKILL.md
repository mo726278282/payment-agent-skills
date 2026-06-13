---
name: upi-flow-debug
description: Payment data flow debugging — trace data from collector to callback to persistence. For investigating "stored value doesn't match source" issues.
tags: [payment, data-flow, debugging, consistency, reconciliation]
---

# Payment Data Flow Debugging

When what's in your database doesn't match what the data source says, the discrepancy is on one link in the chain. Find it.

## The chain

```
Collector pull → Callback → Backend handler → Persist
fetch_data()     callback    handler           UPDATE db
     │               │           │                │
   source API    POST /callback  parse+validate   write to store
```

Don't guess which link. Check each one in order.

## Diagnosis steps

### ① Check stored value

```sql
SELECT key_field, updated_at FROM data_table WHERE id = <business_id>;
```

### ② Check live source

```python
live_data = fetch_from_source(business_id, channel_type)
# → { "success": true, "value": "real_value", "error": null }
```

### ③ Compare

| Stored | Live | Meaning |
|--------|------|---------|
| `abc@bank` | `abc@bank` | ✅ All good |
| `abc@bank` | `xyz@bank` | ❌ Mismatch — keep digging |
| `abc@bank` | `null` | ⚠️ Source query failed — check source availability |
| `null` | `abc@bank` | ❌ Callback never fired — trace the callback path |

### ④ Trace the callback

Search API logs for the callback:

```bash
curl -s 'https://es-host:9200/api-gateway-*/_search' \
  -d '{"query":{"match_phrase":{"LOG_FIELD":"/callback/endpoint"}},
       "_source":["@timestamp","LOG_FIELD"],"size":10,"sort":[{"@timestamp":"desc"}]}'
```

No callback found? The collector didn't send it or the send failed.

### ⑤ Trace the collector

Search collector logs for that business ID. Check:
- Did it successfully pull data?
- Did it attempt to send the callback?
- Were the callback parameters correct?

## The extractor pattern

All channel data extraction lives in one place:

```
fetch_value_by_id(id, channel)
  → call_channel_api(id, channel)
  → extractors[channel](response_json)
  → normalized_value
```

Each extractor is a pure function: raw JSON in, normalized field out. No side effects.

When writing an extractor for a new channel, **reference the API handler, not the collector**. The API handler does pure HTTP calls — the collector may mix in UI automation or WebView logic that you don't want in a headless extraction.

## Channel change detection differences

Not all channels detect value changes the same way. This matters for consistency:

| Channel | Detects change | Records history | Maintains list |
|---------|:---:|:---:|:---:|
| A | ✅ compares then updates | ✅ old→new tracked | ✅ appends |
| B | ❌ overwrites blindly | ❌ | ❌ stores current only |

Channel B can silently overwrite a correct value with a stale one. Know which type you're dealing with.

## Quick diagnosis script

```bash
#!/bin/bash
# One-shot data flow diagnosis
ID=$1
echo "=== Stored ==="
mysql -e "SELECT key_field, updated_at FROM data_table WHERE id=$ID"
echo "=== Live ==="
python -c "from extractors import fetch; print(fetch('$ID', 'channel_name'))"
echo "=== Callback logs ==="
curl -s -u 'user:pass' "https://es-host:9200/api-gateway-*/_search" \
  -d "{\"query\":{\"match_phrase\":{\"LOG_FIELD\":\"$ID\"}},\"size\":5}"
echo "=== Collector logs ==="
curl -s -u 'user:pass' "https://es-host:9200/channel-a-*/_search" \
  -d "{\"query\":{\"match_phrase\":{\"LOG_FIELD\":\"$ID\"}},\"size\":5}"
```

## Pitfalls

1. **Wrong reference code** — for extractors, look at the API handler (pure HTTP), not the collector (interactive login).
2. **Silent overwrites** — some channels don't compare before updating. They'll overwrite a correct value without warning.
3. **Format variance** — different channels return different formats (VPA, QR ID, account number). Normalize in the extractor.
4. **Cache pollution** — extractors may cache results. When debugging, make sure you're looking at live data.
