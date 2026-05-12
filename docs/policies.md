# Policies

A Veto policy describes everything an operator wants to constrain about an agent's spending — caps, time windows, allowlists, rate limits, categories, intent keywords. The engine reads the active policy on every authorize call and refuses spends that violate it.

Policies are immutable and versioned: every change creates a new row with an incremented `version_number` and a content hash. Every receipt cites the exact `policy_version` and `policy_hash` that produced its verdict, so any past decision can be re-derived deterministically.

This page documents the YAML format, the eight policy knobs, and the two paths to edit a policy: from the shell (`veto policy push`) and from inside an agent (MCP tool calls).

---

## Minimal example

```yaml
schema_version: 1
name: "AI Inference"
scope: agent

# Spending caps (USD, decimal)
max_per_transaction: 5.00
daily_limit: 50.00
monthly_limit: 500.00

# Approval thresholds
auto_approve_below: 1.00
require_human_approval_above: 5.00

# Merchant controls — only these merchants are approved
merchant_allowlist:
  - api.anthropic.com
  - api.openai.com
merchant_blocklist: []
```

That's a complete policy. Every additional knob below is optional.

---

## The eight policy knobs (v1)

In addition to the basic caps and merchant lists, v1 adds eight new fields. Each one fires its own hard-deny (or escalate) signal and lands its own reason code on the receipt.

### Time windows

The agent is only allowed to spend during the listed windows. Empty list = 24/7.

```yaml
time_windows:
  - start: "09:00"    # HH:MM, 24-hour
    end:   "17:00"
    tz:    "America/Los_Angeles"   # IANA timezone, default UTC
    days:  [0, 1, 2, 3, 4]          # 0=Mon..6=Sun, omit for every day
  - start: "22:00"    # overnight window OK — end < start wraps midnight
    end:   "06:00"
    tz:    "UTC"
```

Outside any window → hard deny with reason `TIME_WINDOW_VIOLATION`.

### Rate limits (count, not dollars)

Independent of monetary caps. Useful for capping API call volume even when each call is cheap.

```yaml
rate_limit_per_hour: 60      # max 60 transactions per rolling 60 min
rate_limit_per_day: 500      # max 500 transactions per rolling 24 h
```

Exceeded → hard deny with `RATE_LIMITED_HOURLY` or `RATE_LIMITED_DAILY`.

### Categories

Semantic buckets resolved automatically from the merchant string. The catalog covers `weather`, `search`, `inference`, `finance`, `scraping`, `image`, `crypto_rpc`, `news`, `qa`, `code`, `gambling`, `adult`. Anything not in the curated catalog gets the `unknown` category.

```yaml
categories_allowlist:
  - weather
  - search
  - inference
categories_blocklist:
  - gambling
  - adult
```

Block hits → hard deny with `CATEGORY_BLOCKED`. Allowlist miss → hard deny with `CATEGORY_NOT_ALLOWED`.

### Per-merchant caps

Fine-grained dollar limits for individual merchants. Matched exactly OR as a domain prefix — a cap on `proxy.apihub.io` covers `proxy.apihub.io/weather/current`.

```yaml
per_merchant_caps:
  proxy.apihub.io:      "0.50"
  api.exa.x402.io:      "0.10"
  api.firecrawl.x402.io: "0.10"
```

Over the cap → hard deny with `OVER_PER_MERCHANT_CAP`.

### Intent keywords

Content-based gating on the description and conversation context the agent submits with each authorize call.

```yaml
intent_keywords_required:
  - weather             # if non-empty, intent text must contain at least one
intent_keywords_forbidden:
  - gamble              # if intent text contains any of these, hard deny
  - casino
```

Forbidden hit → hard deny with `INTENT_KEYWORD_FORBIDDEN`. Required absent → escalate with `INTENT_KEYWORD_MISSING`.

---

## All fields (complete reference)

| Field | Type | Required | Meaning |
|---|---|---|---|
| `schema_version` | int | Yes (or `version` for legacy) | Format version. Currently `1`. |
| `name` | string | Yes | Human-readable policy name. |
| `scope` | string | No (default `"agent"`) | `"agent"` or `"global"`. |
| `agent_id` | string (UUID) | If scope=agent and client has multiple agents | Which agent this policy applies to. |
| `max_per_transaction` | decimal | Yes | USD per-tx cap. Hard deny over this. |
| `daily_limit` | decimal | Yes | USD daily spend cap. Hard deny when exceeded. |
| `monthly_limit` | decimal | Yes | USD monthly spend cap. Hard deny when exceeded. |
| `auto_approve_below` | decimal \| null | No | Skip intent verification below this amount. |
| `require_human_approval_above` | decimal \| null | No | Force escalation above this amount. |
| `merchant_allowlist` | string[] | No | If non-empty, ONLY these merchants allowed. Hard deny on miss. |
| `merchant_blocklist` | string[] | No | These merchants always blocked. |
| `chain_allowlist` | string[] | No | If non-empty, only these blockchains. Lowercase names: `["base", "polygon"]`. |
| `token_allowlist` | string[] | No | If non-empty, only these token contract addresses. Empty = native asset. |
| `address_allowlist` | string[] | No | If non-empty, only these on-chain destinations. Hard deny on miss. |
| `address_blocklist` | string[] | No | These on-chain destinations always blocked. |
| `crypto_daily_limit_usd` | decimal \| null | No | Separate daily USD cap for crypto transfers. Falls back to `daily_limit` if null. |
| `time_windows` | object[] | No | List of `{start, end, tz, days}` windows. Empty = 24/7. |
| `rate_limit_per_hour` | int \| null | No | Max transactions per rolling 60 min. |
| `rate_limit_per_day` | int \| null | No | Max transactions per rolling 24 h. |
| `categories_allowlist` | string[] | No | Allowed semantic categories. |
| `categories_blocklist` | string[] | No | Always-blocked semantic categories. |
| `per_merchant_caps` | object | No | `{merchant: cap-string}` dollar caps per merchant. |
| `intent_keywords_required` | string[] | No | Intent text must contain at least one. |
| `intent_keywords_forbidden` | string[] | No | Intent text containing any of these is denied. |

---

## Allowlist semantics

When an `_allowlist` field is **non-empty**, it's a **hard gate**: only entries in the list are permitted; anything else is hard-denied at any amount. This matches the security pattern *"if you set an allowlist, only those entries are allowed."*

When an `_allowlist` field is **empty** (`[]`), it's *"all allowed unless denied"*: opt-in. The engine treats empty as no constraint.

> **Engine 0.1.1 (April 2026) closed a fail-open gap** where allowlist violations fired as soft signals (0.6/0.7) and could be diluted to "approve" at low amounts. They're now hard deny (1.0) at any amount.

---

## Editing policy from inside Claude (MCP tools)

A Veto-installed Claude Code session exposes seven MCP tools that mutate the active policy. Each call fetches the active policy, merges the patch, and pushes a new immutable version. The skill instructs the agent to always confirm with the operator before calling any of them.

| Tool | What it changes |
|---|---|
| `veto_policy_allowlist_add` | Append a merchant to `merchant_allowlist`. |
| `veto_policy_set_caps` | Update any of `max_per_transaction`, `daily_limit`, `monthly_limit`, `require_human_approval_above`, `auto_approve_below`. |
| `veto_policy_set_time_windows` | Replace `time_windows` (empty array = remove all restrictions). |
| `veto_policy_set_rate_limits` | Set or remove `rate_limit_per_hour` / `rate_limit_per_day`. |
| `veto_policy_categories_set` | Replace `categories_allowlist` and/or `categories_blocklist`. |
| `veto_policy_per_merchant_cap_set` | Add or remove a single per-merchant cap. |
| `veto_policy_intent_keywords_set` | Replace `intent_keywords_required` and/or `intent_keywords_forbidden`. |

Each mutation creates a new policy version. The old version stays in the audit history; rollback is `veto policy activate <prior-policy-id>`.

---

## Editing policy from the shell (YAML)

```bash
# Export the current preset as YAML
veto policy export inference > my-policy.yaml

# Edit any field
$EDITOR my-policy.yaml

# Push — creates a new immutable version, auto-activates
veto policy push my-policy.yaml
# → ✓ Policy v2 pushed — now active

# Show the active policy
veto policy show

# Dry-run any action without recording a transaction
veto policy check '{"action":"payment","amount":50,"merchant":"amazon.com"}'

# List all versions, newest first
veto policy list

# Roll back to a prior version (instant, no data drift)
veto policy activate <prior-policy-id>
```

Every push:

- Creates a new `SecurityPolicy` row with `version_number = max + 1`
- Auto-activates (deactivates the prior active policy in a single atomic transaction)
- Receipts for any subsequent authorize cite the new `(policy_id, version_number, hash)`

---

## Presets

`veto register` and the cli-demo signup flow apply one of five presets out of the box:

| Preset | For |
|---|---|
| `personal` *(default)* | General-purpose agent — $500/tx, blocks gambling/mixers/adult |
| `inference` | AI API calls — $5/tx, allowlists Anthropic/OpenAI/Replicate/Together/Cohere/Perplexity/Groq/DeepSeek |
| `x402-micropay` | x402 testing — $1/tx, Base chain only, allowlists 10+ x402 categories with per-merchant caps |
| `ad-spend` | Meta/Google ads — $1k/tx, escalate >$1k |
| `dev` | Dogfooding/testing — loose limits |

The `x402-micropay` preset is the one designed to exercise the v1 policy knobs: it ships with `categories_allowlist`, `categories_blocklist`, `per_merchant_caps`, and `rate_limit_per_hour/day` populated. A good starting point for any agent that calls multiple x402 endpoints.

---

## Versioning model (v1)

For simplicity in v1: **one active policy per client at a time.** Push deactivates ALL prior actives in an atomic transaction. Multi-policy parallel rules (different agents under one client with different policies, parallel scopes) come later.

The `_meta` block in `policy show` output shows `policy_id`, `version_number`, `is_active`, and `created_at` for the active row. These fields are read-only — they're stripped by the CLI before pushing back.

---

## Composability and the APPS spec

Veto policies are **APPS-conformant** in spirit but stricter and more opinionated than the abstract APPS spec. The flat field structure makes them easy to author by hand; the strict typing makes them fast to validate. See [`x402-policy-schema`](https://github.com/veto-protocol/x402-policy-schema) for the abstract APPS spec that Veto and other engines implement.

---

## Veto's own published policies

Veto publishes the policies governing its own internal agents at [`veto-protocol/veto-policies`](https://github.com/veto-protocol/veto-policies). Anyone can fetch the YAML and verify that Veto's own receipts cite a matching `policy_hash` — transparency from the operator side, not just the receipt side.
