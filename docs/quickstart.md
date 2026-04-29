# Quickstart

Goal: in 60 seconds, you have a working Veto account, a signed authorize decision, and an offline-verified receipt.

## Prerequisites

- Python 3.9+
- Internet access (to reach `https://veto-ai.com`)

## 1. Install

```bash
pip install veto-cli
```

This pulls in `PyYAML` (policy authoring) and `cryptography` (offline receipt verification).

## 2. Register

```bash
veto register --email me@example.com --preset inference
```

You'll see something like:

```
✓ Welcome to Veto.
  Email:      me@example.com
  Org:        me's Veto
  Agent:      default-agent  (237d9ea3-…)
  Mission:    Make paid API calls to AI inference providers...
  Policy:     AI Inference preset
              max $5.00/tx · $50.00/day · $500.00/mo
              auto-approve below $1.00
              escalate above $5.00
              allows: api.anthropic.com, api.openai.com, ...
  API key:    saved to /Users/you/.veto/config.json
```

The API key is saved locally with mode `0600` — only your user can read it.

**Pick a preset that fits your agent.** Available presets:

| Preset | For |
|---|---|
| `personal` *(default)* | General-purpose agent |
| `inference` | AI API calls (allowlists Anthropic, OpenAI, Replicate, etc.) |
| `x402-micropay` | x402 testing — Base chain only, $1/tx cap |
| `ad-spend` | Meta/Google ads — $1k/tx cap |
| `dev` | Dogfooding/testing — loose limits |

## 3. Authorize an action

```bash
veto authorize --amount 0.05 --merchant api.openai.com --action payment
```

Output:

```
✓ APPROVED — risk 0.18
  reason_codes: REPUTATION_NEW
  transaction_id: …
```

Exit code: `0` for approved, `1` for denied, `2` for escalated, `3` for error.

Try a deliberately denied action:

```bash
veto authorize --amount 50000 --merchant amazon.com --action payment
```

Output:

```
✗ DENIED — risk 1.00
  reason: Amount $50000 exceeds per-tx limit $5.00.
  reason_codes: AMOUNT_CAP_EXCEEDED, DAILY_CAP_EXCEEDED, ...
```

Both decisions ship with a signed receipt — denials are evidence too.

## 4. Verify the receipt offline

```bash
veto authorize --amount 0.05 --merchant api.openai.com --action payment --json | jq -r .receipt | veto verify -
```

Output:

```
✓ VERIFIED — Ed25519 / 0.1.1

  decision:        APPROVE
  decision_layer:  operator_policy
  risk_score:      0.18
  reason_codes:    REPUTATION_NEW
  policy:          AI Inference v1
  policy_id:       …
  policy_hash:     53aa6184…
  transaction_id:  …
  agent_id:        …
  fingerprint:     8674c29af8d3…
  signed_at:       2026-04-29T10:46:34+00:00
```

The verifier:
1. Parses the JWS-compact receipt (`<header>.<payload>.<signature>`)
2. Reads the `kid` from the header (`veto-receipts-v1`)
3. Fetches `https://veto-ai.com/.well-known/jwks.json` (cached locally for 1 hour)
4. Validates the Ed25519 signature
5. Decodes and prints the payload

**No network connection required after the JWKS is cached.** You can verify any Veto receipt offline forever as long as you have the public key.

## 5. Customize your policy (optional)

When the preset isn't enough:

```bash
# Export the current preset as YAML
veto policy export inference > my-policy.yaml

# Edit the YAML in any text editor
$EDITOR my-policy.yaml

# Push your edited version (auto-versioned, auto-active)
veto policy push my-policy.yaml
# → ✓ Policy v2 pushed — now active

# See your active policy
veto policy show

# Dry-run an action without recording a transaction
veto policy check '{"action":"payment","amount":50,"merchant":"amazon.com"}'

# List your policy versions, newest first
veto policy list

# Roll back to a prior version (instant)
veto policy activate <prior-policy-id>
```

For YAML policy format, see [policies.md](./policies.md).

## 6. (Optional) Wire Veto into Claude Desktop / Cursor

If you have a local AI client that speaks MCP:

```bash
veto init
```

This auto-detects Claude Desktop, Cursor, Zed, Continue, and configures each one to spawn Veto's MCP server as a child process. Your AI client can then call `veto_authorize`, `veto_policy_show`, `veto_policy_check`, etc. as tools.

After running it, restart the AI client.

## You're done

You have:
- A registered Veto account (data: `~/.veto/config.json`)
- A working authorize loop with signed receipts
- Verified one of those receipts offline against the public JWKS
- (Optionally) a custom policy and/or MCP integration

Next: integrate `veto authorize` into your agent's pre-payment hook. Your agent calls Veto before each costly action; if Veto approves, the agent proceeds; if Veto denies, the agent refuses. The signed receipt is your audit trail.

For deeper docs:
- [Architecture](./architecture.md) — where Veto fits in the agent-commerce stack
- [Receipts](./receipts.md) — receipt format, signing algorithm, verification
- [Policies](./policies.md) — YAML format, presets, lifecycle
- [Roadmap](./roadmap.md) — cooperative today, on-chain enforcement Q3/Q4
