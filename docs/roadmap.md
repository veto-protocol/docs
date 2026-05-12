# Roadmap

## Today — v1 (npm-first onboarding, multi-rail enforcement, all eight policy knobs)

What's shipped:

- ✅ **`npx @veto-protocol/cli` one-line install** — no pip, no manual MCP config. One command installs the Veto skill, wires the MCP server into Claude Code / Codex / Cursor, signs the operator in (magic-link email), and launches the agent.
- ✅ **The deny-as-teach contract** — when Veto refuses a spend, the agent shows the receipt URL, translates the reason code to English, calls `veto_policy_show`, offers specific edits with the exact MCP tool that performs each, and waits for the operator's go-ahead. Owner stays in control of the policy at every step.
- ✅ **The eight policy knobs** — caps (per-tx, daily, monthly, per-merchant), time windows, count-based rate limits, merchant + chain + token + address allowlists/blocklists, semantic categories (weather / search / inference / finance / etc.), required and forbidden intent keywords. All editable from inside Claude via MCP tools or from the shell via `veto policy push`.
- ✅ **Ed25519-signed receipts** — JWS-compact, public URL at `/r/<uuid>`, offline-verifiable against `/.well-known/jwks.json`. Every receipt cites the exact policy version and content hash that produced it.
- ✅ **Public transparency feed** — `veto-ai.com/transparency` shows every recent decision across the network with click-through to its receipt. Trust by being inspectable.
- ✅ **Policy versioning + atomic rollback** — every change creates an immutable new version; `veto policy activate <prior-id>` is an instant state flip.
- ✅ **Multi-rail composition** — Veto auto-composes with pay.sh (zero code change). Wraps x402 via `@veto-protocol/pay`. Wraps Stripe via `vetoAuthorize()` pre-check. Wraps any wallet via the same pattern.
- ✅ **`VetoGuardedAccount` smart wallet on Base Sepolia** — [`0xCBbb…92c5`](https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5). Real executions, real replay-revert (`MandateAlreadySpent()`). Source at [`veto-protocol/contracts`](https://github.com/veto-protocol/contracts).
- ✅ **EVM multi-chain CREATE2 deploys** — same contract address on Base, Optimism, Arbitrum, Ethereum, Polygon. Domain separator binds `chainId` + `verifyingContract` → no cross-chain replay.
- ✅ **Solana variant on Devnet** — `veto_guarded_program` (Anchor, Ed25519 sysvar). Same JWT mandate format, native verification.
- ✅ **Backend dual-signs mandates** — Ed25519 JWT for off-chain consumers + secp256k1 EIP-712 for on-chain `ecrecover`.
- ✅ **8-stage engine** — kill switch, policy, prompt-injection, canonical-merchant typosquat (catches `api-anthropc.com` for every operator), OFAC live feed, address poisoning, intent verification, per-agent behavioral baselines, reputation weighting, LLM final review.
- ✅ **`@veto/mandate-verifier`** — TS package for offline mandate verification, zero runtime deps.

**Today's enforcement model is cooperative by default, hard-stop by opt-in.** Most agents run cooperatively (the agent asks Veto, Veto answers, the agent obeys — same model as Stripe Radar). For adversarial threat models, set `WALLET_CONTRACT` in the agent's `.env` and the smart wallet refuses settlement without a fresh, in-scope, Veto-signed mandate. The contract is unaudited and testnet-only today; production audited contracts on mainnet ship next.

---

## Next — audit + mainnet hard-stop

The Sepolia stub proves the architecture. Next:

- **External smart-contract audit** of `VetoGuardedAccount`. Engagement scoping, then 4–6 weeks audit + fix cycle.
- **Mainnet deploys** — Base, Ethereum, Optimism, Arbitrum, Polygon, all five at once via CREATE2 (same address everywhere).
- **`veto wallet upgrade` CLI** — interactive wizard to deploy + fund + wire `WALLET_CONTRACT` into an existing agent.
- **Solana Mainnet-Beta** — same program, gated behind a typed-phrase acknowledgement once audited.

---

## ERC-4337 module variant

For operators already on ERC-4337 smart accounts (CDP, Privy, Alchemy, ZeroDev, Pimlico, Dynamic), Veto ships an entryPoint validator that requires the same Veto-signed mandate. Same scope, same `jti`, same JWKS. Operators who want `VetoGuardedAccount` semantics on a 4337 account get them without changing wallet vendor.

## Safe Guard module

For Safes (treasury / multisig customers): a Guard contract that requires a Veto-signed mandate before any tx is allowed. Same primitive, different host.

---

## Stripe Issuing webhook (fiat hard-stop, no MSB)

For fiat: integrate with the operator's existing Stripe Issuing account. Stripe pings Veto's authorize endpoint on every card auth (~2 second window); the policy engine returns approve/deny; Stripe enforces card-side. **Operator keeps custody of card + funding; Veto never holds money.** Real fiat hard-stop without Veto becoming a money transmitter.

---

## Custodial signing — *not* on the roadmap (intentionally)

The "easiest" enforcement path would be for Veto to hold the operator's signing key directly. We deliberately stay non-custodial. The moment Veto holds operator keys:

- Money transmitter / MSB licensure across US states, MiCA in EU, FCA in UK, Israel's regime
- Single-point-of-failure key compromise = operator funds gone
- 24/7 keystore HSM operational burden
- Brand inversion (Veto becomes "the wallet vendor" instead of "the policy layer")

The on-chain enforcement paths above give us real chain-level enforcement without custody. That's the architectural prize.

---

## Smaller items (soon)

- **Dynamic.xyz / Privy embedded wallet templates** — `--wallet-source dynamic` option in `veto agent init`
- **`veto-sdk` Python package** — thin re-export wrapper for SDK-first Python usage
- **Hosted MCP endpoint** — for operators who can't run a local subprocess
- **Telegram approval bot** — escalation handler for `decision == "escalate"`
- **Operator dashboard refresh** — light-mode, surfaced v1 policy knobs, per-merchant breakdown, first-time-signup funnel

Order of execution depends on customer pull. We don't pre-build features nobody asked for.

---

## What this means for a technical evaluator

A sharp question we expect: *"Can Veto actually stop a malicious agent today?"*

Honest answer: *"On Base Sepolia and Solana Devnet, yes — the deployed `VetoGuardedAccount` and `veto_guarded_program` refuse settlement without a fresh, in-scope, Veto-signed mandate. Replay attempts revert on chain (`MandateAlreadySpent()`). Production-audited contracts on mainnet ship next. For agents that don't opt into the smart wallet, Veto still provides cryptographic decision evidence on every authorize call — the cooperative model that handles the bug / hallucination / runaway threat models that account for most real failures. Veto deliberately stays non-custodial to avoid the regulatory cliff that custody implies."*

That holds up under any depth of due diligence.
