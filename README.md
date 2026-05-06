<div align="center">

<br>

<img src="https://veto-ai.com/veto-logo.png" alt="Veto" width="220">

<br><br>

<h3>Documentation for Veto.</h3>

<p>
  Architecture, integration guides, receipt format, on-chain hard-stop.<br>
  Any agent. Any payment rail. Safe transactions.
</p>

<p>
  <a href="https://pypi.org/project/veto-cli/"><img src="https://img.shields.io/pypi/v/veto-cli.svg?style=flat-square&color=06B6D4&label=pypi" alt="PyPI"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-ELv2-22d3ee?style=flat-square" alt="license"></a>
  <a href="https://x402.org"><img src="https://img.shields.io/badge/x402-native-10B981?style=flat-square" alt="x402"></a>
  <a href="https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5"><img src="https://img.shields.io/badge/EVM-Base%20%C2%B7%20Optimism%20%C2%B7%20Arbitrum%20%C2%B7%20Ethereum%20%C2%B7%20Polygon-0052FF?style=flat-square" alt="EVM rails live"></a>
  <a href="https://github.com/veto-protocol/solana-program"><img src="https://img.shields.io/badge/Solana-Devnet-9945FF?style=flat-square" alt="Solana Devnet live"></a>
  <a href="https://veto-ai.com/.well-known/jwks.json"><img src="https://img.shields.io/badge/Ed25519-signed_receipts-22d3ee?style=flat-square" alt="signed receipts"></a>
</p>

<p>
  <a href="#60-second-quickstart">Quickstart</a> ·
  <a href="#what-the-engine-catches">Engine</a> ·
  <a href="#on-chain-hard-stop-live-on-base-sepolia">On-chain</a> ·
  <a href="#documentation">Docs</a> ·
  <a href="#composability">Composes</a> ·
  <a href="#public-artifacts">Artifacts</a>
</p>

</div>

---

## Veto in one paragraph

Veto sits between AI agents and the money. Every spend gets checked against an operator-defined policy in real time, every decision ships with a cryptographically-signed receipt anyone can verify offline, and an optional smart wallet contract refuses unauthorized spends at the chain level. Composes with x402, AP2, Stripe MPP, Verifiable Intent — Veto is one layer above the rail, not the rail itself.

> **Layer 3 — Operational Policy**
>
> The agent-commerce stack has rails (x402, MPP, AP2) and consent (Verifiable Intent, AP2 mandates).
> The layer between agent and rail — *operator-side governance, signed evidence, refusal at the moment of decision* — has been empty.
> Veto is that layer.

This repo holds the public docs. For implementations:

- 🪙 [`veto-pay`](https://github.com/veto-protocol/veto-pay) — One-line agent payments, any rail. `pip install veto-pay` / `npm i @veto/pay`
- 🐍 [`veto-cli`](https://github.com/veto-protocol/veto-cli) — Headline command surface (register, authorize, agent init/fund/deploy/status, verify)
- 📐 [`x402-policy-schema`](https://github.com/veto-protocol/x402-policy-schema) — APPS, the open policy spec (MIT)
- 🔓 [`veto-policies`](https://github.com/veto-protocol/veto-policies) — Veto's own operator policies, in the open
- 🟦 [`mandate-verifier`](https://github.com/veto-protocol/mandate-verifier) — Verify a Veto mandate offline (TS)
- ⛓️ [`contracts`](https://github.com/veto-protocol/contracts) — `VetoGuardedAccount` (EVM) — Base, Optimism, Arbitrum, Ethereum, Polygon
- 🌀 [`solana-program`](https://github.com/veto-protocol/solana-program) — `veto_guarded_program` (Anchor, Ed25519) — Devnet today, Mainnet-Beta gated

---

## 60-second quickstart

```bash
# 1. Install — agent-friendly (Claude Code skills, MCP plugins, scripts)
curl -fsSL https://veto-ai.com/install.sh | bash

# Or human-friendly
pip install veto-cli

# 2. Register an account from the terminal
veto register --email me@example.com --preset inference

# 3. Ask Veto whether an action is allowed
veto authorize --amount 0.05 --merchant api.openai.com --action payment
# → APPROVED · risk 0.18 · signed receipt issued

# 4. Verify the receipt offline (no contact with Veto's runtime)
veto authorize ... --json | jq -r .receipt | veto verify -
# → ✓ VERIFIED — Ed25519 / engine 0.1.1 + decoded payload
```

That's the cooperative path. To set up a runnable agent with hard-stop:

```bash
veto agent init --name my-agent --dir ./my-agent
veto agent fund      # auto-open faucet, poll for funds
veto agent deploy    # deploy the smart wallet on Base Sepolia (default)
veto agent status    # see agent + wallet + contract + policy state

# Switch network or rail without leaving the CLI:
veto agent init --name my-agent --network base-mainnet     # mainnet (typed-phrase ack)
veto agent init --name my-agent --network optimism-sepolia # any EVM
veto agent init --name my-agent --rail solana              # Solana Devnet
```

The `pay()` one-liner — drop into any agent runtime:

```python
from veto_pay import pay

result = await pay(
    url="https://api.weather.x402.io/forecast",
    max_amount="$0.05",
    rail="auto",          # auto | evm | solana
)
print(result.tx_hash)     # on-chain settlement
print(result.receipt)     # signed Veto receipt
```

---

## Rail-agnostic enforcement

One JWT mandate format. Per-rail native verifier. Same policy, same receipt.

| Rail | Verifier | Status |
|------|----------|--------|
| **EVM** — Base, Optimism, Arbitrum, Ethereum, Polygon | `VetoGuardedAccount` (Solidity, secp256k1 + EIP-712 + `ecrecover`) | Testnet live · Mainnet via typed-phrase ack |
| **Solana** | `veto_guarded_program` (Anchor, Ed25519 sysvar) | Devnet live · Mainnet-Beta via typed-phrase ack |
| **Card** (Stripe Issuing) | Webhook auth-time verify | Coming next |
| **Sui / Aptos** | Move module (Ed25519 native) | On the roadmap |

### EVM hard-stop — live and proven

A minimal smart wallet (`VetoGuardedAccount`) holds the agent's funds and only releases them on a fresh, in-scope, Veto-signed mandate. Single-use, time-bound, scope-locked. Same address across every EVM chain via CREATE2.

| | |
|---|---|
| **Contract (Base Sepolia)** | [`0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5`](https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5) |
| **First execution** | [`0x2f9ec…d2af`](https://sepolia.basescan.org/tx/0x2f9ec691a6f5958bea296c5f630b26d1be1d93667dc3c974671cce0773cad2af) — 0.000001 ETH transferred |
| **Second execution** | [`0xe4112b…5217`](https://sepolia.basescan.org/tx/0xe4112b29c4a80ad337ca45a1599d669fd9853a3a7a8977d52251ce4e8c0e5217) — fresh mandate |
| **Replay rejected** | `MandateAlreadySpent()` selector `0xffa64355` — chain refused a duplicate |
| **Off-chain verifier** | [`@veto/mandate-verifier`](https://github.com/veto-protocol/mandate-verifier) (TS, Node 18+) |
| **Source** | [`veto-protocol/contracts`](https://github.com/veto-protocol/contracts) |

Verification: secp256k1 EIP-712 + `ecrecover` (~3k gas). Domain separator binds `chainId` + `verifyingContract` → no cross-chain or cross-contract replay either. Audited contracts ship in v2.

### Solana hard-stop — Devnet today

`veto_guarded_program` mirrors the EVM contract on Solana with native Ed25519 verification via the instructions sysvar. Same JWT mandate, no paired signature needed.

- **Source**: [`veto-protocol/solana-program`](https://github.com/veto-protocol/solana-program)
- **Pattern**: 2-instruction transaction (Ed25519 verify + `execute_with_mandate`)
- **Status**: v0.6 stub — Devnet live, Mainnet-Beta gated by typed-phrase ack, audit pending

---

## Where Veto sits

```
LAYER 5 — Commerce flow (catalog/cart/checkout)
   ACP (OpenAI + Stripe), Shopify, native APIs

LAYER 4 — User authorization (USER → AGENT consent)
   Verifiable Intent (Mastercard + Google), AP2 (Google)
   ↑ user signs a mandate authorizing their agent to spend

LAYER 3 — Operational policy (OPERATOR → AGENT governance)
   ★ VETO ★ — multi-dim policy + signed receipts + on-chain hard-stop
   ↑ operator's policy + cryptographic decision evidence

LAYER 2 — Payment rails (settlement)
   x402 (USDC, HTTP 402)
   MPP (Stripe SPT, HTTP 402)
   AP2, ACH, direct on-chain

LAYER 1 — Cryptography (signatures, key distribution)
   SD-JWT, JWS, JWK, JWKS, EdDSA, ECDSA
```

Visa, Mastercard, Stripe, Google, OpenAI, PayPal — all racing to ship Layers 2 and 4. **Layer 3 is empty.** Payment networks don't build it because their customer is the rail, not the company deploying the agent. Operator-side governance is a pain felt by *companies* deploying agents — that's the Veto wedge.

For a deeper walk-through, see [`docs/architecture.md`](./docs/architecture.md).

---

## What's in a receipt

Every authorize call produces a JWS-compact receipt (`<header>.<payload>.<signature>`) signed with Ed25519. Decoded:

```json
{
  "iss": "veto",
  "sub": "<transaction-uuid>",
  "iat": 1745783400,
  "decision": "deny",
  "decision_layer": "operator_policy",
  "risk_score": 1.0,
  "reason_codes": [
    "AMOUNT_CAP_EXCEEDED",
    "MERCHANT_NOT_ALLOWLISTED",
    "TYPOSQUAT_CANONICAL"
  ],
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

Plus the authorize response now carries `engine_trace` (per-stage breakdown) and `engine_aggregate` (final scoring math). For the full spec, see [`docs/receipts.md`](./docs/receipts.md).

---

## What the engine catches

Eight stages. Every signal that fires lands in the receipt's `engine_trace`.

| # | Stage | What it catches |
|---|---|---|
| 1 | Your rules | Caps, daily limits, allow- and blocklists for merchants, chains, tokens, addresses |
| 2 | Prompt-injection | "Ignore previous instructions" patterns and similar agent-context attacks |
| 3 | Misspelled merchants | `api-anthropc.com`, `аpple.com` (Cyrillic homoglyph) — for *every* user, allowlist or not |
| 4 | Crypto safety | OFAC sanctioned addresses (live feed), address-poisoning attacks, known-drainer contracts |
| 5 | Intent | Does the spend match the agent's mission and recent context? Crypto-aware. |
| 6 | Anomaly | Velocity bursts, merchant-diversity spikes, off-pattern amounts |
| 7 | Behavior baseline | Per-agent rolling stats (p99 amount, hour-of-day, recipient set) |
| 8 | Final decision | Weighted aggregation, fraud floor, human-required floor, signed receipt with full trace |

---

## Composability

Veto is the **operator-policy** layer. It composes with everything around it:

| Spec | One-line | How it composes with Veto |
|---|---|---|
| [x402](https://www.x402.org/) | Payment rail (HTTP 402 + USDC on Base) | Veto authorizes, x402 settles |
| [pay.sh](https://pay.sh) (Solana + Google Cloud) | Hosted x402 aggregator on Solana, 75+ APIs | Veto pre-flight checks every call; hard-stop via `veto_guarded_program` |
| [Stripe MPP](https://stripe.com/blog/machine-payments-protocol) | Payment rail (HTTP 402 + Stripe SPT) | Veto sits above; rail handles settlement |
| [AP2](https://ap2-protocol.org/) (Google) | User → Agent authorization (mandate JWTs) | Different layer (Layer 4); receipts can cite an upstream AP2 mandate via `mandate_ref` |
| [Verifiable Intent](https://verifiableintent.dev/) (Mastercard + Google) | Sibling of AP2; same primitive | Same composition pattern |
| [ACP](https://www.agenticcommerce.dev/) (OpenAI + Stripe) | Catalog/cart/checkout API | Layer 5; orthogonal |

When all four layers exist on a transaction, the agent carries: AP2/VI mandate (legal consent), Veto receipt (operational compliance), ACP merchant flow (commerce), MPP/x402 settlement (money). All four produce signed credentials. Together they're the audit trail.

---

## Documentation

- 📐 [`docs/architecture.md`](./docs/architecture.md) — the 5-layer stack, the engine pipeline, on-chain hard-stop
- 🚀 [`docs/quickstart.md`](./docs/quickstart.md) — first 60 seconds (install → authorize → verify → agent init)
- 📜 [`docs/receipts.md`](./docs/receipts.md) — receipt format spec, signing, verification, audit guarantees
- 🛡️ [`docs/policies.md`](./docs/policies.md) — YAML policy format, presets, customization, lifecycle
- 🗺️ [`docs/roadmap.md`](./docs/roadmap.md) — what's shipped (v0.6) → what's next (audit + mainnet hard-stop)

---

## Public artifacts

| Artifact | URL |
|---|---|
| `veto-pay` (PyPI) | https://pypi.org/project/veto-pay/ |
| `@veto/pay` (npm) | https://www.npmjs.com/package/@veto/pay |
| `veto-cli` (PyPI) | https://pypi.org/project/veto-cli/ |
| CLI source | https://github.com/veto-protocol/veto-cli |
| APPS open policy spec | https://github.com/veto-protocol/x402-policy-schema |
| Mandate verifier (TS) | https://github.com/veto-protocol/mandate-verifier |
| EVM contract source | https://github.com/veto-protocol/contracts |
| Solana program source | https://github.com/veto-protocol/solana-program |
| Veto's own published policies (transparency) | https://github.com/veto-protocol/veto-policies |
| JWKS endpoint (public key) | https://veto-ai.com/.well-known/jwks.json |
| Live contract on Base Sepolia | https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5 |

---

## Status

**v0.6** — public, dogfooded end-to-end, contract live on Base Sepolia. Active development.

## License

Elastic License v2 (ELv2). See [LICENSE](LICENSE) for the full text and copyright.

---

<div align="center">

**Documentation for Veto.**<br>
<sub>The safety layer for AI agents that spend money.</sub>

<br><br>

<sub>Built by <a href="https://veto-ai.com">Investech Global LLC</a> · part of the <a href="https://github.com/veto-protocol">veto-protocol</a> family.</sub>

</div>
