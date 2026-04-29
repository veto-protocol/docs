# Architecture

## The 5-layer agent-commerce stack

```
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 5 — Commerce flow (catalog / cart / checkout)                 │
│   ACP (OpenAI + Stripe), Shopify, native APIs                       │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 4 — User authorization (USER → AGENT consent)                 │
│   Verifiable Intent (Mastercard + Google) — SD-JWT, layered         │
│   AP2 (Google + payment partners) — Intent / Cart Mandates          │
│   ↑ user signs to authorize agent's right to spend                  │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 3 — Operational policy (OPERATOR → AGENT governance)          │
│   ★ VETO ★ — YAML policy + Ed25519 receipts + reason codes          │
│   ↑ operator's policy + signed evidence of every decision           │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 2 — Payment rails (settlement)                                │
│   MPP (Stripe SPT, HTTP 402)                                        │
│   x402 (USDC, HTTP 402)                                             │
│   Stripe Issuing (cards) | ACH | onchain (direct)                   │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 1 — Cryptography (signatures, key distribution)               │
│   SD-JWT, JWS, JWK, JWKS, EdDSA, ECDSA — what everyone shares       │
└─────────────────────────────────────────────────────────────────────┘
```

## Why Veto sits at Layer 3 specifically

The big payment networks (Visa, Mastercard, Stripe, Google, OpenAI, PayPal) are all racing to ship payment rails (Layer 2) and user-authorization specs (Layer 4) for AI agents.

**Layer 3 — operational policy + signed evidence — is empty.** Why?

Because their customer is the rail/network, not the company deploying the agent. Operator-side governance is a pain felt by *companies* deploying agents (e.g., a marketing agency running ad-buying bots, a SaaS company shipping customer-support agents, a Web3 startup running x402 micropayment workers), not by the payment networks themselves.

That's the Veto wedge. We're the layer where:

- **Companies declare what their agents are allowed to spend on** — caps, allowlists, escalation thresholds, multi-rail
- **Every decision produces an Ed25519-signed receipt** — verifiable offline by anyone who has the public key (from `/.well-known/jwks.json`)
- **The receipt cites the exact policy version and content hash** — so an auditor 12 months later can prove which policy governed any past transaction
- **The format composes with Layers 4 and 2** — receipts can cite upstream user-authorization mandates (AP2 / Verifiable Intent), and the policy specifies which payment rails are permitted

## How a transaction flows

```
┌─────────┐  1. action     ┌─────────┐  2. authorize  ┌──────────────┐
│  AGENT  │ ──────────────▶│  YOUR   │ ──────────────▶│     VETO     │
│ runtime │                │  CODE   │                │   ENGINE     │
└─────────┘                │ (CLI/   │                │ (8-step pipe)│
                           │  SDK)   │                └──────┬───────┘
                                                             │ 3. signed receipt
                                                             ▼
                                                     ┌──────────────┐
                                                     │  Ed25519 JWS │
                                                     │   compact    │
                                                     └──────┬───────┘
                                                             │
                          4. if approved → settle           │ 5. saved + verifiable
                                                             ▼
┌─────────────────────────────────────────────┐    ┌────────────────────┐
│ LAYER 2 — settlement                        │    │  Receipt is a      │
│ Stripe MPP / x402 / Stripe Issuing / onchain│    │  permanent audit   │
└─────────────────────────────────────────────┘    │  artifact. Verify  │
                                                    │  offline against   │
                                                    │  veto-ai.com/      │
                                                    │  .well-known/      │
                                                    │  jwks.json.        │
                                                    └────────────────────┘
```

## The 8-step engine pipeline

Every `authorize` call passes through:

1. **Pre-checks** — kill switch active? agent operational?
2. **Policy enforcement** — per-tx / daily / monthly caps, merchant + address + chain + token allowlists/blocklists. Allowlist violations are hard deny at any amount.
3. **Prompt injection detection** — 40+ regex patterns over the action's `description` and `context` fields.
4. **Merchant fraud screening** — known-fraud DB, typosquat detection (SequenceMatcher against well-known brands), suspicious TLDs, hyphen-heavy domains.
5. **Intent verification** — Claude Sonnet (or keyword fallback) compares the action to the agent's declared mission.
6. **Anomaly detection** — amount spikes (>3× 30-day avg), velocity bursts, merchant-diversity anomalies.
7. **LLM final verdict** — Claude reviews the full signal collection.
8. **Reputation weighting** — agent's reputation tier modulates the final risk score.

Output: `approve` | `deny` | `escalate` plus a `risk_score` (0–1), structured `reason_codes` (`AMOUNT_CAP_EXCEEDED`, `KNOWN_FRAUD_MERCHANT`, etc.), and a signed receipt.

## What's signed

The receipt is a JWS-compact (RFC 7515) string with `alg: EdDSA, kid: veto-receipts-v1`. The payload includes:

- The decision (`approve` / `deny` / `escalate`)
- The risk score
- Reason codes (canonical, mapped from internal signal names)
- The policy ID, version_number, and **content hash** (SHA-256 over canonical policy contents)
- The input fingerprint (SHA-256 over canonical decision inputs)
- The engine version
- The agent_id and client_id
- A reserved `mandate_ref` field for future Safe-guard-module integration

For full receipt format spec, see [receipts.md](./receipts.md).

## Why this matters strategically

**Operator-governance is a moat unique to Layer 3.** Payment networks won't build it (no customer for it from their perspective). User-authorization specs (AP2, VI) won't build it (different problem). Wallet vendors won't build it (different relationship to customer).

The companies deploying agents NEED:
- A way to author rules in code (YAML, in our case)
- Cryptographic evidence the rules were followed
- Audit-ready records that hold up against insurers, regulators, customer disputes
- Composability with whatever payment rail / mandate spec wins

Veto is positioned to be the durable "Stripe Radar for agent operators." Same role, different category, no incumbent.
