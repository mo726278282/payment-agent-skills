---
name: payment-channel-onboard
description: Multi-channel payment adapter pattern — a repeatable checklist for onboarding new banks, wallets, or third-party payment channels.
tags: [payment, channel, adapter, architecture, onboarding]
---

# Payment Channel Adapter Pattern

If you're building a system that talks to multiple banks or payment providers, use this pattern to keep each new integration consistent and contained.

## Architecture

```
                    ┌──────────────────┐
                    │   Dispatch Layer  │  ← routes by channel_name
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
        │       Base Channel Handler           │  ← shared infrastructure
        │  DB ops · Session mgmt · Retry/lock  │
        │  Credential crypto · Proxy mgmt      │
        │  Error logging · Health checks       │
        └──────────────────────────────────────┘
```

The dispatch layer doesn't need to know how each channel works. Every channel exposes the same interface. The base class owns the boring stuff — database connections, session storage, retry logic, proxy rotation, error logging. Subclasses only write the channel-specific API calls and response parsers.

## Onboarding Checklist

### 1. Research the channel API

Three questions you need answered before writing a line:

- **Auth**: OTP? OAuth2? API key? Mutual TLS?
- **Response shape**: What does the API return? How deep is the field you need?
- **Network**: Region-locked? Proxy required?

### 2. Implement the Channel Handler

A `<Channel>Handler` class with these methods:

```
pre_login()       — initialize session, verify user credentials
send_otp()        — deliver OTP (if applicable)
verify_otp()      — validate OTP
fetch_accounts()  — pull account/wallet list
set_active()      — select active account
```

If this is your second channel, **extract a BaseHandler first**. Don't let the second handler copy-paste 400 lines of infrastructure from the first. It only gets worse from there.

### 3. Register in the dispatch layer

```python
if channel_name == 'channel_a':
    handler = ChannelAHandler(request)
elif channel_name == 'channel_b':
    handler = ChannelBHandler(request)
else:
    raise UnsupportedChannelError(channel_name)
```

Do this in every endpoint (pre_login, send_otp, verify_otp, fetch_accounts, set_active). Missing one means a 404 with no clear error.

### 4. Build the Collector (background worker)

If the channel needs a persistent process for polling or long-lived connections:

- Standalone process or worker
- Login → maintain session → poll on interval → POST results to backend
- Handle token expiry, proxy failures, rate limits

The collector-to-backend contract:

```json
{
  "channel_id": "xxx",
  "type": "ACCOUNT_UPDATE",
  "status": 1,
  "accounts": [{"id": "abc@bank", "name": "Bank Name"}]
}
```

### 5. Build the data extractor

Every channel returns a different JSON shape. Normalize it:

```
fetch_value_by_id(id, channel)
  → call_channel_api(id, channel)
  → extractors[channel](response_json)
  → normalized_value
```

Each extractor is a pure function: raw API response in, normalized field out.

### 6. Update the verification pipeline

If you have a timeout/hold verification pipeline, new channels need:

- `channel_type_id → channel_name` mapping
- `channel_name → data_source` mapping (different channels may use different scraping tools)
- Any channel-specific verification steps

### 7. Database

```sql
INSERT INTO channel_type (name, status) VALUES ('CHANNEL_NAME', 1);
```

## Verification

- [ ] Handler passes HTTP tests for every endpoint
- [ ] Collector can login and retrieve data
- [ ] Callback correctly updates backend state
- [ ] Extractor returns live data for the channel
- [ ] Log system shows the full login → fetch → callback trace
- [ ] Verification pipeline (if present) handles this channel correctly

## Pitfalls

1. **Phone/account format** varies per channel (with/without country code). Normalize before comparing.
2. **Proxy requirements** are channel-specific. Store proxy configs in a config store, not hardcoded.
3. **HTTP client** — some channels detect and block vanilla `requests`. You may need TLS fingerprint emulation (`curl_cffi`).
4. **Handler duplication** — the first channel can be standalone. The second channel demands a BaseHandler. Don't wait until channel five.
5. **Dispatch gaps** — every HTTP method needs the channel branch. Easy to miss one.
6. **Naming drift** — the same channel might be spelled `paytm_business` in one place and `paytmbusiness` in another. Pick one and enforce it everywhere.
