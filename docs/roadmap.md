# Roadmap

## Today (v0.6 — cooperative + chain-level hard-stop on testnet)

What's shipped:

- ✅ Multi-dimensional YAML policy authoring (caps + allowlists + thresholds + multi-rail)
- ✅ Ed25519-signed decision receipts (JWS-compact + JWKS)
- ✅ Policy versioning + atomic rollback
- ✅ Offline receipt verification (`veto verify` + Python lib)
- ✅ MCP server with 6 tools — works in Claude Desktop, Cursor, Zed, Continue
- ✅ 5 policy presets out of the box
- ✅ Composability with x402, MPP, AP2 / Verifiable Intent (`mandate_ref` reserved)
- ✅ **Engine improvements** — canonical-merchant typosquat (catches `api-anthropc.com` for every user, no allowlist required), address-poisoning detection, OFAC live feed, per-agent behavioral baselines, structured `engine_trace` per stage
- ✅ **Mandate JWT format + issuance** — Ed25519 JWT alongside every approve receipt (`typ=mandate+jwt`)
- ✅ **`@veto/mandate-verifier`** — TS package for offline mandate verification
- ✅ **`veto agent init/fund/deploy/status`** — scaffold a runnable governed agent in 60 seconds, including an opt-in smart wallet on Base Sepolia
- ✅ **`VetoGuardedAccount` smart wallet stub on Base Sepolia** — [`0xCBbb…92c5`](https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5). Real executions, real replay-revert. Source at [`veto-protocol/contracts`](https://github.com/veto-protocol/contracts).
- ✅ Backend dual-signs mandates (Ed25519 JWT for off-chain consumers + secp256k1 EIP-712 for on-chain `ecrecover`)
- ✅ Dogfooded end-to-end: agent → veto.authorize → backend signs → contract verifies → settlement; replay attempt rejected on chain (`MandateAlreadySpent()`)

**Today's enforcement model is cooperative by default, hard-stop by opt-in.** Most agents run in cooperative mode (your code asks Veto, Veto answers, your code obeys — same model as Stripe Radar). For adversarial threat models, set `WALLET_CONTRACT` in the agent's `.env` and the smart wallet refuses settlement without a fresh, in-scope, Veto-signed mandate. The contract is unaudited and testnet-only — production audited contracts ship next.

## Next — audit + mainnet hard-stop

The Sepolia stub proves the architecture. Next:

- **External smart-contract audit** of `VetoGuardedAccount`. Engagement scoping, then 4–6 weeks audit + fix cycle.
- **Mainnet deploys** — Base, Ethereum, Optimism, Arbitrum. Same contract, audited.
- **`veto wallet upgrade` CLI** — interactive wizard to deploy + fund + wire `WALLET_CONTRACT` into a customer's existing agent.

## ERC-4337 module variant

For customers already on ERC-4337 smart accounts (CDP, Privy, Alchemy, ZeroDev, Pimlico, Dynamic), Veto ships an entryPoint validator that requires the same Veto-signed mandate. Same scope, same `jti`, same JWKS. Customers who want `VetoGuardedAccount` semantics on a 4337 account get them without changing wallet vendor.

## Safe Guard module

For Safes (treasury / multisig customers): a Guard contract that requires a Veto-signed mandate before any tx is allowed. Same primitive.

## Solana variant

`VetoGuardedAccount` is EVM-specific (`ecrecover`). The Solana variant is a program that verifies Ed25519 signatures on-chain (Solana has the precompile). Same mandate format, different verification surface.

## Stripe Issuing webhook (fiat hard-stop, no MSB)

For fiat: integrate with the customer's existing Stripe Issuing account. Stripe pings Veto's authorize endpoint on every card auth (~2 second window); Veto's policy engine returns approve/deny; Stripe enforces card-side. **Customer keeps custody of card + funding; Veto never holds money.** Real fiat hard-stop without becoming a money transmitter.

## Custodial signing — *not* on the roadmap (intentionally)

The "easiest" enforcement path would be for Veto to hold the customer's signing key directly. We deliberately stay non-custodial. The moment Veto holds customer keys:

- Money transmitter / MSB licensure across US states, MiCA in EU, FCA in UK, Israel's regime
- Single-point-of-failure key compromise = customer funds gone
- 24/7 keystore HSM operational burden
- Brand inversion (Veto becomes "the wallet vendor" instead of "the policy layer")

The on-chain enforcement paths above give us **real chain-level enforcement without custody**. That's the architectural prize.

## Other items (smaller, soon)

- **Dynamic.xyz / Privy embedded wallet templates** — `--wallet-source dynamic` option in `veto agent init`
- **`veto-sdk` Python package** — thin re-export wrapper over `veto-cli` for SDK-first Python usage
- **Hosted MCP endpoint** — for users who can't run a local subprocess
- **Telegram approval bot** — escalation handler for `decision == "escalate"`
- **Stripe Checkout for Pro/Scale tiers** — paid tier UX
- **Custom dashboard pages** — replace `/admin/` with a proper metrics + receipts UI

Order of execution depends on customer pull. We don't pre-build features that nobody asked for.

## What this means for an investor or technical evaluator

A sharp question we expect: *"Can Veto actually stop a malicious agent today?"*

Honest answer: *"On Base Sepolia, yes — the deployed `VetoGuardedAccount` contract refuses settlement without a fresh, in-scope, Veto-signed mandate. Replay attempts revert with `MandateAlreadySpent()` on chain. Production-audited contracts on mainnet ship next. For agents that don't opt into the smart wallet, we still provide cryptographic decision evidence on every authorize call — the cooperative model that handles the bug/hallucination/runaway threat models that account for most real failures. We deliberately stay non-custodial to avoid the regulatory cliff that custody implies."*

That holds up under any depth of due diligence.
