# Roadmap

## Today (v0.5.x — cooperative + verifiable)

What's shipped:

- ✅ Multi-dimensional YAML policy authoring (caps + allowlists + thresholds + multi-rail)
- ✅ Ed25519-signed decision receipts (JWS-compact + JWKS)
- ✅ Policy versioning + atomic rollback
- ✅ Offline receipt verification (`veto verify` + Python lib)
- ✅ MCP server with 6 tools — works in Claude Desktop, Cursor, Zed, Continue
- ✅ 5 policy presets out of the box
- ✅ x402, MPP, on-chain — same engine, same policy, multi-rail by design
- ✅ Composability with AP2 / Verifiable Intent (`mandate_ref` field reserved)
- ✅ Dogfooded end-to-end: real agent transaction settled via the engine

**Today's enforcement model is cooperative.** Same as Stripe Radar — your code asks Veto, Veto answers, your code obeys. For the threat model that matters most (LLM hallucinations, runaway logic, cost-runaway, bug-induced spend), that's exactly the right tool.

## Q3 — ERC-4337 session keys

For the adversarial case (a non-cooperative agent that ignores Veto's deny), Veto provisions and rotates **session keys** on the customer's ERC-4337 / smart account wallet. Session keys carry on-chain enforced caps and allowlists derived from the customer's APPS policy.

**Properties:**

- Veto never holds the customer's main key
- Session keys have on-chain expiry, max amount, allowlisted destinations
- The smart account contract refuses out-of-policy transactions at the chain level
- Bypass requires breaking the smart account itself
- Works wherever ERC-4337 is supported: Base, Arbitrum, Optimism, Polygon zkEVM, etc.

This corresponds to internal task #16 (ZeroDev AA co-signer integration). Activation gated on user-pulled need — when the first paying customer asks for hard enforcement on a non-cooperative agent, this ships.

## Q4 — Safe guard modules + mandate JWTs

For wallets that aren't ERC-4337 (e.g., Safe), the customer attaches a **guard module** that requires a Veto-signed **mandate JWT** for every transaction. The chain itself rejects any transaction that isn't accompanied by a valid Veto mandate.

**Mandate JWT contents:**

- Authorized recipient (chain + address)
- Maximum amount
- Nonce (single-use)
- Expiry (typically 5 minutes)
- Signature (Ed25519, same key as decision receipts)

**Forward compatibility:** the receipt format already reserves a `mandate_ref` field (currently null). When Q4 ships, every authorize call also issues a mandate JWT and `mandate_ref` cites it. **Same Ed25519 key, same JWKS endpoint, same audit trail** — just shifts the mandate consumer from "the agent's cooperative code" to "the smart account's guard module."

This corresponds to internal task #17 (Phase 3 crypto stack). Same activation gate as Q3 — user-pulled.

## Custodial signing — *not* on the roadmap (intentionally)

The "easiest" enforcement path is for Veto to hold the customer's signing key directly: agent says "pay X"; Veto checks policy; Veto signs (or refuses). One field flip on `crypto_signer.py:mode` from `advisory` to `enforce` and we're there.

**We deliberately stay non-custodial.**

The moment Veto holds customer keys:

- **Money transmitter / MSB licensure** — depending on jurisdiction. US: state-by-state. EU: MiCA. UK: FCA. Israel: separate regime. We aren't equipped for that, and shouldn't be.
- **Single point of failure** — one private key compromise = customer funds gone.
- **Operational scale** — 24/7 keystore HSM, key rotation, separation-of-concerns audits, formal compliance posture.
- **Brand inversion** — Veto stops being "the policy layer" and starts being "the wallet vendor." Different category, different competition (Coinbase Custody, Fireblocks).

The Q3 (ERC-4337 session keys) and Q4 (Safe guards) paths give us **real on-chain enforcement without custody**. That's the architectural prize.

## Why this roadmap order is the right one

1. **Cooperative + cryptographic evidence (today)** is enough for 80–90% of real threat models. Most operators deploy agents they own. The failure modes are bugs and hallucinations.
2. **ERC-4337 session keys (Q3)** is the right next move because ERC-4337 adoption is genuinely growing (post-Pectra) and the integration is well-understood (ZeroDev does it for many wallets). Customer pull → ship the integration.
3. **Safe guard modules (Q4)** is where adversarial-agent threat models converge with the AP2 / Verifiable Intent ecosystem. By Q4 the on-chain mandate-JWT pattern will be more widely adopted across the industry, and Veto receipts will compose with it natively.
4. **Custody is the wrong category** for Veto, period.

## What this means for an investor or technical evaluator

A sharp question we expect: *"Can Veto actually stop a malicious agent today?"*

Honest answer: *"No. Today we provide cryptographic decision evidence for cooperative agents. For the adversarial case, Q3 ERC-4337 session keys and Q4 Safe guard modules are the on-chain enforcement paths. Both leverage the existing Ed25519 receipt infrastructure — the receipt format already includes a `mandate_ref` field reserved for the Q4 guard-module integration. We deliberately stay non-custodial to avoid the regulatory cliff that custody implies."*

That holds up under any depth of due diligence.

## Other roadmap items (smaller, soon)

- **`veto-sdk` Python package** — thin re-export wrapper over `veto-cli` for SDK-first Python usage (`import veto_sdk`). Ships when SDK-import users complain about `from veto_cli import api` being awkward.
- **Hosted MCP endpoint** — for users who can't run a local MCP server. Currently local-only via `veto init`.
- **Telegram approval bot** — escalation handler for `decision == "escalate"`. Operator gets a Telegram message and approves/denies via inline buttons.
- **Stripe Checkout for Pro/Scale tiers** — paid tier UX. Blocked on Mercury LLC bank approval → Stripe LLC migration.
- **Curated allowlist in `x402-micropay` preset** — out-of-the-box gating to known-safe x402 merchants from the CDP Bazaar.
- **Dedicated `/whoami` endpoint** — replaces the current `verify_key` probe that uses `/authorize` with empty body.
- **Custom dashboard pages** — replace bare-bones `/admin/` with a proper metrics + receipts UI.

Order of execution depends on customer pull. v1 is about gathering signal, not pre-building features.
