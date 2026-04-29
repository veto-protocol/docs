# Veto Docs

> The operator-governance layer for AI agent payments. Multi-dimensional policy + Ed25519-signed decision receipts + offline verification. Composes with Stripe MPP, x402, AP2, and Verifiable Intent.

[![PyPI](https://img.shields.io/pypi/v/veto-cli)](https://pypi.org/project/veto-cli/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

This repo holds the public docs, architecture explainers, and integration guides for [Veto](https://veto-ai.com). For the implementation, see:

- 🐍 [`veto-cli`](https://github.com/veto-protocol/veto-cli) — `pip install veto-cli`
- 📐 [`x402-policy-schema`](https://github.com/veto-protocol/x402-policy-schema) — APPS, the open policy spec
- 🔓 [`veto-policies`](https://github.com/veto-protocol/veto-policies) — Veto's own operator policies, published

---

## What Veto is, in one paragraph

Stripe Radar gives merchants a policy + signed-decision layer for fraud detection. Veto gives **agent operators** the same primitive for AI agent payments. The company deploying an agent declares a YAML policy (caps, allowlists, escalation thresholds, multi-rail). Every authorize request goes through Veto's engine and produces a signed Ed25519 receipt anyone can verify offline against [our JWKS endpoint](https://veto-ai.com/.well-known/jwks.json). Composes with Stripe MPP / x402 / AP2 / Verifiable Intent — different layer of the agent-commerce stack, complementary to all of them.

## On-chain proof point

April 29, 2026 — first end-to-end live transaction:

> **Tx:** [`0xb13f9ddbd9ff1edba341e96aaeef0cee47510c275656c4cafd1b78375a639b4b`](https://basescan.org/tx/0xb13f9ddbd9ff1edba341e96aaeef0cee47510c275656c4cafd1b78375a639b4b)

A real agent paid Exa.ai $0.007 USDC on Base mainnet for a search query. Veto's engine evaluated the request against the agent's policy, signed the decision, and the agent honored the approve. Three test scenarios (legit / denied-by-allowlist / denied-by-cap) all behaved correctly. Receipt is verifiable forever — the signed JWT is reproducible from policy + input + key, and the on-chain settlement is immutable.

---

## 60-second quickstart

```bash
# 1. Install
pip install veto-cli

# 2. Register an account from the terminal
veto register --email me@example.com --preset x402-micropay
# → API key saved locally; default agent created

# 3. Ask Veto before doing anything costly
veto authorize --amount 0.05 --merchant api.openai.com --action payment
# → APPROVED / DENIED / ESCALATED. Exit code 0 / 1 / 2 / 3.

# 4. Verify any signed receipt offline
veto authorize ... --json | jq -r .receipt | veto verify -
# → ✓ VERIFIED — Ed25519 / 0.1.1 + decoded payload
```

That's it. No website, no form, no MCP setup required. Just terminal-native auth → terminal-native authorization → cryptographic evidence.

---

## The 5-layer agent-commerce stack

```
LAYER 5 — Commerce flow (catalog/cart/checkout)
   ACP (OpenAI + Stripe), Shopify, native APIs

LAYER 4 — User authorization (USER → AGENT consent)
   Verifiable Intent (Mastercard + Google), AP2 (Google)
   ↑ user signs a mandate authorizing their agent to spend

LAYER 3 — Operational policy (OPERATOR → AGENT governance)
   ★ VETO ★ — multi-dim policy + signed receipts + reason codes
   ↑ operator's policy + cryptographic decision evidence

LAYER 2 — Payment rails (settlement)
   MPP (Stripe SPT, HTTP 402)
   x402 (USDC, HTTP 402)
   Stripe Issuing (cards) | ACH | onchain

LAYER 1 — Cryptography (signatures, key distribution)
   SD-JWT, JWS, JWK, JWKS, EdDSA, ECDSA
```

Visa, Mastercard, Stripe, Google, OpenAI, PayPal — all racing to ship Layers 2 and 4. **Layer 3 is empty.** Payment networks don't build it because their customer is the rail/network, not the company deploying the agent. Operator-side governance is a pain felt by *companies* deploying agents, and that's the Veto wedge.

For a deeper walk-through, see [`docs/architecture.md`](./docs/architecture.md).

---

## What's in the receipt

Every authorize call produces a JWS-compact receipt (`<header>.<payload>.<signature>`) signed with Ed25519. Decoded:

```json
{
  "iss": "veto",
  "sub": "<transaction-uuid>",
  "iat": 1745783400,
  "decision": "deny",
  "decision_layer": "operator_policy",
  "risk_score": 1.0,
  "reason_codes": ["AMOUNT_CAP_EXCEEDED", "MERCHANT_NOT_ALLOWLISTED"],
  "engine_version": "0.1.1",
  "input_fingerprint": "5f3c8a9b21e4...",
  "agent_id": "<uuid>",
  "client_id": "<uuid>",
  "policy": {
    "id": "<uuid>",
    "version_number": 3,
    "name": "Custom",
    "hash": "53aa6184..."
  },
  "mandate_ref": null
}
```

- **`input_fingerprint`** — SHA-256 over the canonical decision inputs. Same input → same fingerprint. Foundation for deterministic replay.
- **`policy.hash`** — SHA-256 over the canonical policy contents. Closes the in-place tampering gap: an admin can't edit a policy row without invalidating future receipts that cite the old hash.
- **`decision_layer: "operator_policy"`** — distinguishes Veto receipts from user-authorization mandates (AP2, Verifiable Intent).
- **`mandate_ref`** — reserved for future Safe-guard-module integration where a Veto-signed mandate JWT is required by an on-chain guard module.

For the full receipt format spec, see [`docs/receipts.md`](./docs/receipts.md).

---

## Today: cooperative enforcement. Roadmap: on-chain.

Today, Veto is cooperative. The agent calls `veto authorize`, gets a signed decision, and chooses to obey. If a malicious agent ignored a deny, Veto can't physically stop the on-chain transaction.

This is exactly how Stripe Radar works: your code asks Stripe, Stripe answers, your code obeys. Nobody calls Stripe Radar "fake fraud prevention" because the threat model that matters — bugs, hallucinations, runaway logic — is cooperative-friendly. Same is true for AI agents. Most operators deploy agents they own. The failure modes are LLM hallucinations and bugs, not adversarial intent.

For the adversarial case, three real-enforcement paths are on the roadmap. See [`docs/roadmap.md`](./docs/roadmap.md):

| Path | Status |
|---|---|
| ERC-4337 session keys with policy-derived caps | Planned Q3 |
| Safe guard modules requiring Veto-signed mandate JWTs | Planned Q4 |
| Custodial signing | **Intentionally NOT on the roadmap** — would make Veto a regulated money transmitter. Different category. |

---

## Composability with other agent-commerce specs

Veto is the **operator-governance** layer. It composes with everything around it:

| Spec | One-line | How it composes with Veto |
|---|---|---|
| [Stripe MPP](https://stripe.com/blog/machine-payments-protocol) | Payment rail (HTTP 402 + Stripe SPT) | Veto sits above; rail handles settlement |
| [x402](https://www.x402.org/) | Payment rail (HTTP 402 + USDC on Base) | Same — Veto authorizes, x402 settles |
| [AP2](https://ap2-protocol.org/) (Google) | User → Agent authorization (mandate JWTs) | Different layer (Layer 4); Veto receipts can cite an upstream AP2 mandate via `mandate_ref` |
| [Verifiable Intent](https://verifiableintent.dev/) (Mastercard + Google) | Sibling of AP2; same primitive | Same composition pattern as AP2 |
| [ACP](https://www.agenticcommerce.dev/) (OpenAI + Stripe) | Catalog/cart/checkout API | Different layer (Layer 5); orthogonal |

When all four layers exist on a transaction, the agent carries: AP2/VI mandate (legal consent), Veto receipt (operational compliance), ACP merchant flow (commerce), MPP/x402 settlement (money). All four produce signed credentials. Together they're the audit trail.

---

## Public artifacts

| Artifact | URL |
|---|---|
| **PyPI package** | https://pypi.org/project/veto-cli/ |
| **CLI source** | https://github.com/veto-protocol/veto-cli |
| **APPS open policy spec** | https://github.com/veto-protocol/x402-policy-schema |
| **Veto's own policies (transparency)** | https://github.com/veto-protocol/veto-policies |
| **JWKS endpoint** (Ed25519 public key for receipt verification) | https://veto-ai.com/.well-known/jwks.json |
| **Live JWKS proof** | `curl https://veto-ai.com/.well-known/jwks.json` |

---

## Documentation

- [`docs/architecture.md`](./docs/architecture.md) — the 5-layer stack, where Veto sits, how it composes
- [`docs/quickstart.md`](./docs/quickstart.md) — first 60 seconds (pip install → authorize → verify)
- [`docs/receipts.md`](./docs/receipts.md) — receipt format spec, signing, verification, audit guarantees
- [`docs/policies.md`](./docs/policies.md) — YAML policy format, presets, customization, lifecycle
- [`docs/roadmap.md`](./docs/roadmap.md) — what's shipped, what's next (cooperative → ERC-4337 → Safe guards)

---

## Status

v0.5.4 — public, dogfooded, on-chain proof captured. Active development.

## License

This docs repo: MIT. See [LICENSE](LICENSE). Copyright Investech Global LLC.
