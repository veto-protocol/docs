# Policies

## YAML format

Policies are authored in YAML and pushed to Veto via `veto policy push`. The format mirrors the `SecurityPolicy` model 1:1.

### Minimal example

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

# Crypto controls
chain_allowlist: []
token_allowlist: []
address_allowlist: []
address_blocklist: []
```

### All fields

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
| `category_allowlist` | string[] | No | Allowed merchant category codes (MCCs). |
| `chain_allowlist` | string[] | No | If non-empty, only these blockchains. Lowercase names: `["base", "polygon"]`. |
| `token_allowlist` | string[] | No | If non-empty, only these token contract addresses. Empty string = native asset. |
| `address_allowlist` | string[] | No | If non-empty, only these on-chain destinations. Hard deny on miss. |
| `address_blocklist` | string[] | No | These on-chain destinations always blocked. |
| `crypto_daily_limit_usd` | decimal \| null | No | Separate daily USD cap for crypto transfers. Falls back to `daily_limit` if null. |

### Allowlist semantics

When an `_allowlist` field is **non-empty**, it acts as a **hard gate**: only entries in the list are permitted; anything else is hard-denied at any amount. This matches the security pattern *"if you set an allowlist, only those entries are allowed."*

When an `_allowlist` field is **empty** (`[]`), it means *"all allowed unless denied"*: the field is opt-in. Engine treats empty as no constraint.

> **Engine version 0.1.1 (April 2026) closed a fail-open gap** where allowlist violations fired as soft signals (0.6/0.7) and could be diluted to "approve" at low amounts. They're now hard deny (1.0) at any amount.

## Lifecycle

### 1. Pick a preset to start from

`veto register` applies one of five presets out of the box:

| Preset | For |
|---|---|
| `personal` *(default)* | General-purpose agent — $500/tx, blocks gambling/mixers/adult |
| `inference` | AI API calls — $5/tx, allowlists Anthropic/OpenAI/Replicate/Together/Cohere/Perplexity/Groq/DeepSeek |
| `x402-micropay` | x402 testing — $1/tx, Base chain only, auto-approve <$0.10 |
| `ad-spend` | Meta/Google ads — $1k/tx, escalate >$1k |
| `dev` | Dogfooding/testing — loose limits |

### 2. Export and customize

```bash
veto policy export inference > my-policy.yaml
$EDITOR my-policy.yaml
veto policy push my-policy.yaml
# → ✓ Policy v2 pushed — now active
```

Every push:
- Creates a new SecurityPolicy row with `version_number = max + 1`
- Auto-activates (deactivates the prior active policy)
- Receipt for any subsequent authorize cites the new `(policy_id, version_number, hash)`

### 3. Inspect

```bash
# Show the active policy as YAML
veto policy show

# List versions, newest first, with relative timestamps
veto policy list

# Dry-run an action — no transaction recorded, just policy evaluation
veto policy check '{"action":"payment","amount":50,"merchant":"amazon.com"}'
```

### 4. Roll back

```bash
veto policy activate <prior-policy-id>
# → ✓ Activated v1 — AI Inference
```

True state flip on existing rows — no data drift, the same DB record gets re-activated. Receipts always cite the policy that was active *at decision time*, not the latest-ever-pushed version.

## Versioning model (v1)

For simplicity in v1: **one active policy per client at a time.** Push deactivates ALL prior actives in an atomic transaction. Multi-policy parallel rules (different agents under one client with different policies, parallel scopes, etc.) come later.

The `_meta` block in `policy show` output shows `policy_id`, `version_number`, `is_active`, and `created_at` for the active row. These fields are read-only — they're stripped by the CLI before pushing back.

## Composability

A Veto policy is **APPS-conformant** in spirit but stricter and more opinionated than the abstract APPS spec. The flat field structure makes it easy to author by hand; the strict typing makes it fast to validate. See [`x402-policy-schema`](https://github.com/veto-protocol/x402-policy-schema) for the abstract APPS spec that Veto and other engines implement.

## Veto's own published policies

Veto publishes the policies governing its own internal agents at [`veto-protocol/veto-policies`](https://github.com/veto-protocol/veto-policies). It's a transparency artifact — anyone can fetch the YAML and verify Veto receipts cite a matching policy_hash.
