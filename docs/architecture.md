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
│   ★ VETO ★ — YAML policy + Ed25519 receipts + on-chain hard-stop    │
│   ↑ operator's policy + signed evidence + optional chain enforcement│
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 2 — Payment rails (settlement)                                │
│   x402 (USDC, HTTP 402)                                             │
│   MPP (Stripe SPT, HTTP 402)                                        │
│   AP2 settlement | ACH | direct EVM/Solana                          │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 1 — Cryptography (signatures, key distribution)               │
│   SD-JWT, JWS, JWK, JWKS, EdDSA, ECDSA — what everyone shares       │
└─────────────────────────────────────────────────────────────────────┘
```

## Why Veto sits at Layer 3 specifically

The big payment networks (Visa, Mastercard, Stripe, Google, OpenAI, PayPal) are all racing to ship payment rails (Layer 2) and user-authorization specs (Layer 4) for AI agents.

**Layer 3 — operational policy + signed evidence — is empty.** Why?

Because their customer is the rail/network, not the company deploying the agent. Operator-side governance is a pain felt by *companies* deploying agents (a marketing team running ad-buying bots, a SaaS company shipping customer-support agents, a Web3 startup running x402 micropayment workers), not by the payment networks themselves.

That's the Veto wedge. We're the layer where:

- **Companies declare what their agents are allowed to spend on** — caps, allowlists, escalation thresholds, multi-rail
- **Every decision produces an Ed25519-signed receipt** — verifiable offline by anyone who has the public key (from `/.well-known/jwks.json`)
- **The receipt cites the exact policy version and content hash** — so an auditor 12 months later can prove which policy governed any past transaction
- **Optional on-chain hard-stop** — funds in a smart wallet that refuses to settle without a fresh, in-scope, Veto-signed mandate; a compromised agent can't move money even if its code is bypassed
- **The format composes with Layers 4 and 2** — receipts can cite upstream user-authorization mandates (AP2 / Verifiable Intent), and the policy specifies which payment rails are permitted

## How a transaction flows

### Cooperative path (default)

```
┌─────────┐  1. action     ┌─────────┐  2. authorize  ┌──────────────┐
│  AGENT  │ ──────────────▶│  YOUR   │ ──────────────▶│     VETO     │
│ runtime │                │  CODE   │                │   ENGINE     │
└─────────┘                │ (CLI/   │                │ (8-stage)    │
                           │  SDK)   │                └──────┬───────┘
                                                             │ 3. signed receipt
                                                             ▼
                                                     ┌──────────────┐
                                                     │  Ed25519 JWS │
                                                     └──────┬───────┘
                                                             │
                          4. if approved → settle           │ 5. saved + verifiable
                                                             ▼
┌─────────────────────────────────────────────┐    ┌────────────────────┐
│ LAYER 2 — settlement                        │    │  Receipt is a      │
│ x402 / Stripe MPP / direct chain            │    │  permanent audit   │
└─────────────────────────────────────────────┘    │  artifact. Verify  │
                                                    │  offline against   │
                                                    │  /.well-known/     │
                                                    │  jwks.json.        │
                                                    └────────────────────┘
```

### Hard-stop path (opt-in via smart wallet)

```
┌─────────┐  1. action     ┌─────────┐  2. authorize    ┌──────────────┐
│  AGENT  │ ──────────────▶│  YOUR   │ ───────────────▶│     VETO     │
└─────────┘                │  CODE   │ + wallet_contract│   ENGINE     │
                           └────┬────┘ + chain_id       └──────┬───────┘
                                │                              │
                                │      3. signed receipt + signed mandate (EIP-712 / secp256k1)
                                │              ◀───────────────┘
                                │
                                ▼ 4. executeWithMandate(mandate, sig, amount)
                       ┌─────────────────────────────┐
                       │  VetoGuardedAccount.sol     │
                       │  on Base Sepolia            │
                       │                             │
                       │  - ecrecover signer         │
                       │  - check jti not spent      │
                       │  - check exp > now          │
                       │  - check scope matches      │
                       │  - settle or revert         │
                       └─────────────────────────────┘
```

In hard-stop mode, an attacker who compromises the agent code can't drain funds: the smart wallet refuses to settle without a fresh, in-scope, Veto-signed mandate. The chain enforces. Same policy, same receipt, additional chain-level guarantee.

## The 8-stage engine pipeline

| # | Stage | What it catches | Source |
|---|---|---|---|
| 1 | **Pre-checks** | Kill switch active? Agent operational? | `safety/services/engine.py` |
| 2 | **Policy enforcement** | Per-tx / daily / monthly caps, merchant + address + chain + token allowlists/blocklists. Allowlist violations are hard deny at any amount. | `safety/services/engine.py` |
| 3 | **Prompt-injection detection** | Regex patterns over `conversation_context` — "ignore previous instructions" and friends. | `safety/services/engine.py` |
| 4 | **Merchant fraud screening** | Canonical-merchant typosquat (catches `api-anthropc.com` for *every* user without an allowlist), homoglyph-aware, suspicious TLDs, hyphen-heavy domains, brand-keyword similarity. | `safety/services/canonical_merchants.py` |
| 5 | **Crypto safety** *(crypto_transfer only)* | OFAC sanctioned addresses (live SDN feed), address-poisoning detection, known-drainer contracts, chain + token allowlist. | `onchain/services/address_safety.py` |
| 6 | **Intent verification** | Claude Sonnet checks the action against the agent's declared mission. Crypto-aware variant for `crypto_transfer`. Soft signal — informs aggregation. | `safety/services/engine.py` |
| 7 | **Anomaly detection** | Amount spikes, velocity bursts, merchant-diversity anomalies (rolling 30-day window). | `safety/services/engine.py` |
| 8 | **Behavioral baseline** | Per-agent rolling stats — amount p99, recipient set, hour-of-day, max txs/hour. Distinguishes "trading bot at 20 tx/min (normal)" from "inference agent suddenly at 20 tx/min (suspicious)". Cold-start at 10 settled txs. | `safety/services/baselines.py` |
| 9 | **Reputation weighting** | Agent's reputation tier modulates the final risk score. | `safety/services/engine.py` |
| 10 | **LLM final verdict + aggregator** | Claude reviews the full signal collection. Output: decision + risk score + structured `reason_codes` + `engine_trace` (per-stage breakdown) + `engine_aggregate` (final scoring math). | `safety/services/engine.py` |

Output: `approve` / `deny` / `escalate` plus a `risk_score` (0–1), `reason_codes` (`AMOUNT_CAP_EXCEEDED`, `TYPOSQUAT_CANONICAL`, `ADDRESS_POISONING_SUSPECTED`, `OFAC_SANCTIONED_ADDRESS`, etc.), and a signed receipt with the full trace.

## What's signed

The receipt is a JWS-compact (RFC 7515) string with `alg: EdDSA, kid: veto-receipts-v1`. The payload includes:

- The decision (`approve` / `deny` / `escalate`)
- The risk score
- Reason codes (canonical, mapped from internal signal names)
- The policy ID, version_number, and **content hash** (SHA-256 over canonical policy contents)
- The input fingerprint (SHA-256 over canonical decision inputs)
- The engine version
- The agent_id and client_id
- A reserved `mandate_ref` field for upstream user-authorization credentials (AP2, Verifiable Intent)

For full receipt format spec, see [receipts.md](./receipts.md).

## On-chain hard-stop — the contract

`VetoGuardedAccount.sol` is a minimal smart wallet that holds funds and only releases them via `executeWithMandate(mandate, signature, amount)`. The mandate is an EIP-712 typed-data struct signed by Veto's secp256k1 signer; the contract verifies via `ecrecover` (~3k gas).

| Property | Detail |
|---|---|
| **Live deploy** | [`0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5`](https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5) on Base Sepolia |
| **Source** | https://github.com/veto-protocol/contracts |
| **Mandate scope** | `{ jti, exp, recipient, maxAmount, token }` — single-use, time-bound, scope-locked |
| **Domain separator** | binds `chainId` + `verifyingContract` → no cross-chain or cross-contract replay |
| **Replay protection** | contract tracks `spent[jti]` on chain |
| **Verifier (off-chain)** | [`@veto/mandate-verifier`](https://github.com/veto-protocol/mandate-verifier) — TS, Node 18+ |

Why two signature schemes? EVM has `ecrecover` as a precompile (~3k gas). Verifying an Ed25519 JWT directly on chain would cost ~1M gas through a Solidity library. Veto issues the off-chain Mandate JWT (Ed25519, for off-chain consumers like agents and settlement services) AND a parallel secp256k1 EIP-712 signature (for on-chain consumers like `VetoGuardedAccount`). Same scope fields, paired by `jti`.

## Why this matters strategically

**Operator-governance is a moat unique to Layer 3.** Payment networks won't build it (no customer for it from their perspective). User-authorization specs (AP2, VI) won't build it (different problem). Wallet vendors won't build it (different relationship to customer).

The companies deploying agents NEED:

- A way to author rules in code (YAML, in our case)
- Cryptographic evidence the rules were followed
- Audit-ready records that hold up against insurers, regulators, customer disputes
- Optional chain-level enforcement for adversarial threat models
- Composability with whatever payment rail / mandate spec wins

Veto is positioned to be the durable "Stripe Radar for agent operators." Same role, different category, no incumbent.
