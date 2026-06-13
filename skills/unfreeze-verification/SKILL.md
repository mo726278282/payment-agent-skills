---
name: unfreeze-verification
description: Multi-step verification pipeline for held orders — checks system health before releasing funds. For payment, lending, and withdrawal systems that need risk-control verification.
tags: [payment, verification, pipeline, timeout, hold, risk-control]
---

# Order Verification Pipeline

When an order times out and enters HOLD, it should pass a series of checks before automatic release or cancellation. Don't release funds on a hunch.

## Dual-Path Pattern

```
Order timeout
  ├── TIMEOUT path (first timeout)
  │     └── Query: pending orders created N~M minutes ago
  └── HOLD_RETRY path (retry)
        └── Query: already-held orders, retry verification on interval
```

**The rule that prevents most bugs**: both paths run the same verification logic. Change one, change the other. Every time.

## The Pipeline

```
① Collector health        — is the data collector alive
② Source probe            — can we actively reach the data source
③ Data consistency        — does stored data match live source
④ Temporal ordering       — is source data newer than the order
         │
         ▼
   can_finalize = ① ∧ ② ∧ ③ ∧ ④
         │
    ┌────┴────┐
  true       false
   │           │
 release      keep HOLD
```

### ① Collector health

Check if the background collector process is alive via shared state:

```
is_online = state_store.is_member('collector_online_set', channel_id)
```

When the state store is unreachable, **reject**. Don't release funds because you can't check.

### ② Source probe (active verification)

Before unfreezing, actively call the data source to confirm the collector can actually retrieve data:

```
result = probe_data_source(channel_id, channel_type)
if not result.success:
    return BLOCK  # dead token, network failure, anything
```

This step needs two lookup tables:
- `channel_type_id → channel_name` (numeric ID to string name)
- `channel_name → data_source` (name to internal tool identifier)

**Both tables must be updated when adding a channel.** Missing one gives you "unrecognized channel, can't verify."

### ③ Data consistency

Compare the stored key field against the live source:

```
local = db.query("SELECT key_field FROM orders WHERE id = ?", order_id)
live  = fetch_from_source(channel_id, channel_type)
if local != live:
    return BLOCK  # possible tampering or collector malfunction
```

Gate this per-channel — some data sources don't support real-time queries.

### ④ Temporal ordering

Make sure the data source was last crawled *after* the order was created:

```
last_crawl = state_store.get(f"last_crawl:{channel_id}")
if last_crawl < order_create_time:
    return BLOCK  # source hasn't seen post-order transactions yet
```

## Adding a new check

1. Add fields to the SQL query if the check needs new data
2. Add fields to the HOLD persistence so retries have the data
3. Insert the check in **both** TIMEOUT and HOLD_RETRY paths
4. Add the condition to `can_finalize`
5. Record the result in `hold_reason` for traceability
6. Log every step with `[TAG][STEP]` format
7. If the check is channel-specific, gate it with `if channel in ('A', 'B'):`

## Hold reason structure

```json
{
  "collector_online": true,
  "reason_collector": "online_set",
  "source_probe": true,
  "reason_probe": "probe_ok",
  "data_consistency": true,
  "reason_consistency": "field_match",
  "time_order": true,
  "reason_time": "crawl_after_order"
}
```

Every check result and its reason is recorded. When something goes wrong, you can trace exactly which step failed.

## Pitfalls

1. **Dual-path drift** — the most common bug. Fix one path, forget the other.
2. **Channel ID type mismatch** — sometimes `int`, sometimes `string`. Normalize before comparing.
3. **Mapping table gaps** — two tables, both need the new channel. One missing = unverifiable.
4. **State store failure policy** — decide once: reject or pass through? Financial systems should reject.
5. **Scattered finalize logic** — all conditions feed into a single `can_finalize`. Don't decide in multiple places.
