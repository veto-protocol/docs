<div align="center">

<br>

<img src="https://veto-ai.com/veto-logo.png" alt="Veto" width="220">

<br><br>

<h3>The governance layer for AI agent payments.</h3>

<p>
  Veto stands between an agent and the money it's about to spend.<br>
  Any agent. Any payment rail. Operator-owned policy. Signed receipts. Optional on-chain hard-stop.
</p>

<p>
  <a href="https://www.npmjs.com/package/@veto-protocol/cli"><img src="https://img.shields.io/npm/v/@veto-protocol/cli.svg?style=flat-square&color=06B6D4&label=npm" alt="npm"></a>
  <a href="https://pypi.org/project/veto-cli/"><img src="https://img.shields.io/pypi/v/veto-cli.svg?style=flat-square&color=06B6D4&label=pypi" alt="PyPI"></a>
  <a href="https://veto-ai.com/transparency"><img src="https://img.shields.io/badge/live%20feed-public-22d3ee?style=flat-square" alt="transparency"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-ELv2-22d3ee?style=flat-square" alt="license"></a>
  <a href="https://x402.org"><img src="https://img.shields.io/badge/x402-composes-10B981?style=flat-square" alt="x402"></a>
  <a href="https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5"><img src="https://img.shields.io/badge/EVM-Base%20%C2%B7%20Optimism%20%C2%B7%20Arbitrum%20%C2%B7%20Ethereum%20%C2%B7%20Polygon-0052FF?style=flat-square" alt="EVM rails live"></a>
  <a href="https://github.com/veto-protocol/solana-program"><img src="https://img.shields.io/badge/Solana-Devnet-9945FF?style=flat-square" alt="Solana Devnet live"></a>
</p>

<p>
  <a href="#what-veto-does">What it does</a> ·
  <a href="#install-and-run-in-one-line">Install</a> ·
  <a href="#a-real-session">A real session</a> ·
  <a href="#compose-with-any-rail">Compose with any rail</a> ·
  <a href="#what-the-engine-catches">Engine</a> ·
  <a href="#documentation">Docs</a>
</p>

</div>

---

## What Veto does

An AI agent is about to spend its operator's money. Veto checks the spend against the operator's policy first &mdash; caps, time windows, allowlists, rate limits, categories, intent keywords &mdash; and returns a verdict before the rail settles. Every verdict is an Ed25519-signed receipt, viewable at a public URL, verifiable offline. An optional smart wallet refuses any spend the engine didn't bless, on-chain.

**Veto governs. The rail executes.** It composes with x402, pay.sh, Stripe, raw wallets, anything that moves money. It isn't a payment rail; it's the policy a rail has to clear first.

The owner stays in control. The agent stays useful. The receipt is the audit trail.

> **Layer 3 &mdash; Operational policy**
>
> The agent-commerce stack has rails (x402, MPP, AP2) and user consent (Verifiable Intent, AP2 mandates). The layer between agent and rail &mdash; operator-side governance, signed evidence, refusal at the moment of decision &mdash; has been empty. Veto is that layer.

---

## Install and run in one line

Built for Claude Code, Codex, Cursor &mdash; any agent runtime that speaks MCP.

```bash
npx -y @veto-protocol/cli@latest claude "buy weather data for New York"
```

That one command installs the Veto skill, wires the MCP server into Claude Code, signs the operator in (magic link by email, no password), and launches the agent. The skill instructs the agent to gate every spend through `veto_authorize` first. The verdict is what controls whether the rail ever sees the request.

For non-MCP runtimes, install the same surface as a library:

```bash
npm i @veto-protocol/pay     # one-line agent payments, any rail
pip install veto-cli         # YAML policy authoring + offline receipt verify
```

---

## A real session

The trace below is verbatim from a Claude Code session with Veto and [pay.sh](https://pay.sh) both installed. The agent finds a merchant via pay.sh's discovery skill, gates the spend through Veto, gets denied because the merchant isn't on the operator's allowlist, then walks the operator through what's editable. Nothing is scripted; the agent narrates because the Veto skill instructs it to.

```
USER: buy weather data for New York

AGENT: I'll find a weather provider first, then gate the spend through Veto.
   → pay.sh skill: searches its x402 catalog
   ← best fit: bigquery.google.gateway-402.com at $0.001/query

AGENT: Best fit is BigQuery's NOAA datasets at $0.001/query. Gating through Veto now.
   → veto_authorize { amount: 0.001, merchant: "bigquery.google.gateway-402.com", ... }
   ← deny · risk 1.0 · MERCHANT_NOT_ALLOWLISTED · https://veto-ai.com/r/b237773a-...

AGENT: Veto denied. Let me pull the active policy so I can give you concrete options.
   → veto_policy_show
   ← { merchant_allowlist: ["api.weather.x402.io", "api.openai.com", ...], ... }

AGENT:
   Veto verdict: DENY
   - Receipt: https://veto-ai.com/r/b237773a-4d17-4c7e-b9cc-a180cd299dec
   - Rule that fired: MERCHANT_NOT_ALLOWLISTED — bigquery.google.gateway-402.com
     isn't in your allowlist. Caps weren't the issue ($0.001 is well under your
     $1.00 per-tx / $0.50 auto-approve thresholds).

   Good news — your policy already allows a weather merchant.

   Your allowlist contains api.weather.x402.io, which looks purpose-built for this.
   Two paths forward:

   Option A — pivot (no policy change): retry against api.weather.x402.io.
   Option B — extend allowlist: add bigquery.google.gateway-402.com via
     veto_policy_allowlist_add. Useful for SQL-style historical queries.

   Which do you want? I won't touch the policy or settle anything without
   your go-ahead.
```

Two things to notice. First &mdash; the agent showed the operator the receipt URL, not a 2KB JWT. Anyone with that link can verify the decision offline at `/.well-known/jwks.json`. Second &mdash; the agent waited for the operator's go-ahead before mutating the policy. The owner stays in control.

The deny is the onboarding moment. The operator learns the policy is editable by being denied, then offered specific edits with the exact MCP tool that performs them.

For the full walk-through, see [`docs/quickstart.md`](./docs/quickstart.md).

---

## Compose with any rail

Veto is the comma between *agent decides what to buy* and *rail settles the payment*. It doesn't replace the rail; it gates it. Four common shapes:

| Rail | Composition |
|------|-------------|
| **[pay.sh](https://pay.sh)** | Already installed? Veto auto-composes. Claude sees both skills, calls `veto_authorize` first, pay.sh settles only on allow. The trace above is exactly that. |
| **[x402](https://x402.org) direct** | `npm i @veto-protocol/pay` &rarr; `await pay(url, { gate: "veto" })` calls authorize then sends USDC. Same one-liner, governance built in. |
| **Stripe** | Wrap `stripe.PaymentIntent.create()` with a `vetoAuthorize()` check first. Deny &rarr; don't create the intent. Allow &rarr; settle normally. |
| **Raw wallet (viem / ethers / web3.js)** | Call `vetoAuthorize()` before `wallet.sendTransaction()`. For chain-level enforcement that can't be bypassed off-chain, deploy [`VetoGuardedAccount`](https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5) and route spends through it. |

The verdict shape is identical across rails. The receipt is identical. The mandate JWT a smart wallet checks on-chain is identical. One policy, one audit surface, every rail.

---

## Live transparency

Every Veto decision lives at a public URL:

- **`https://veto-ai.com/transparency`** &mdash; firehose of recent allow / deny / escalate decisions across the network, with reason codes and click-through receipts. Anyone can audit the engine. Trust comes from being inspectable, not from a logo.
- **`https://veto-ai.com/r/<uuid>`** &mdash; individual receipt page. Public, signed, shareable.
- **`https://veto-ai.com/.well-known/jwks.json`** &mdash; Veto's public Ed25519 keys. Offline receipt verification.

Privacy of individual agents comes from their UUIDs being unguessable; the decision itself is public by design, the way Etherscan or Certificate Transparency logs are.

---

## What the engine catches

Eight stages. Every signal that fires lands in the receipt's `engine_trace`. Every reason code lands in `reason_codes`.

| # | Stage | What it catches |
|---|---|---|
| 1 | Operator policy | Caps (per-tx, daily, monthly, per-merchant), time windows, rate limits (count), allowlists, blocklists, categories, required/forbidden intent keywords |
| 2 | Prompt injection | "Ignore previous instructions" and similar agent-context attacks |
| 3 | Misspelled merchants | `api-anthropc.com`, `аpple.com` (Cyrillic homoglyph) &mdash; for every operator, allowlist or not |
| 4 | Crypto safety | OFAC sanctioned addresses (live feed), address poisoning, known-drainer contracts |
| 5 | Intent | Does the spend match the agent's mission and recent context? Crypto-aware. |
| 6 | Anomaly | Velocity bursts, merchant-diversity spikes, off-pattern amounts |
| 7 | Behavior baseline | Per-agent rolling stats (p99 amount, hour-of-day, recipient set) |
| 8 | Final decision | Weighted aggregation, fraud floor, human-required floor, signed receipt with full trace |

Policy is YAML, versioned, content-hashed; every push creates a new immutable version, every receipt cites the exact `policy_version` and `policy_hash` that produced it.

---

## Where Veto sits

```
LAYER 5  Commerce flow (catalog/cart/checkout)
        ACP (OpenAI + Stripe), Shopify, native APIs

LAYER 4  User authorization (USER &rarr; AGENT consent)
        Verifiable Intent (Mastercard + Google), AP2 (Google)
        &uarr; user signs a mandate authorizing their agent to spend

LAYER 3  Operational policy (OPERATOR &rarr; AGENT governance)
        &#9733; VETO &#9733;  multi-dim policy + signed receipts + on-chain hard-stop
        &uarr; operator's policy + cryptographic decision evidence

LAYER 2  Payment rails (settlement)
        x402 (USDC, HTTP 402)
        MPP (Stripe SPT, HTTP 402)
        AP2, ACH, direct on-chain

LAYER 1  Cryptography (signatures, key distribution)
        SD-JWT, JWS, JWK, JWKS, EdDSA, ECDSA
```

Visa, Mastercard, Stripe, Google, OpenAI &mdash; all racing to ship Layers 2 and 4. **Layer 3 is empty.** Payment networks don't build it because their customer is the rail, not the company deploying the agent. Operator-side governance is a pain felt by the *companies* deploying agents. That's the Veto wedge.

For a deeper walk-through, see [`docs/architecture.md`](./docs/architecture.md).

---

## On-chain hard-stop (optional, live on Base Sepolia)

For operators who can't trust an off-chain check alone, Veto ships a smart wallet that holds the agent's funds and only releases them on a fresh, in-scope, Veto-signed mandate. Single-use, time-bound, scope-locked. Same address across every EVM chain via CREATE2.

| | |
|---|---|
| **Contract (Base Sepolia)** | [`0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5`](https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5) |
| **First execution** | [`0x2f9ec…d2af`](https://sepolia.basescan.org/tx/0x2f9ec691a6f5958bea296c5f630b26d1be1d93667dc3c974671cce0773cad2af) &mdash; 0.000001 ETH transferred |
| **Replay rejected on-chain** | `MandateAlreadySpent()` selector `0xffa64355` |
| **Off-chain verifier** | [`@veto/mandate-verifier`](https://github.com/veto-protocol/mandate-verifier) (TS, Node 18+) |

Verification: secp256k1 EIP-712 + `ecrecover` (~3k gas). Domain separator binds `chainId` + `verifyingContract` &mdash; no cross-chain or cross-contract replay. Solana mirrors this with `veto_guarded_program` (Anchor, Ed25519 sysvar) live on Devnet today.

Audited mainnet contracts ship in v2.

---

## Documentation

- 🚀 [`docs/quickstart.md`](./docs/quickstart.md) &mdash; install, dogfood the deny-then-fix loop, compose with your rail
- 🛡️ [`docs/policies.md`](./docs/policies.md) &mdash; YAML format, the 8 policy knobs, edit-from-Claude tools
- 📜 [`docs/receipts.md`](./docs/receipts.md) &mdash; receipt format, signing, offline verification, the `/r/<uuid>` URL
- 📐 [`docs/architecture.md`](./docs/architecture.md) &mdash; the 5-layer stack, the engine pipeline, on-chain hard-stop
- 🗺️ [`docs/roadmap.md`](./docs/roadmap.md) &mdash; what shipped, what's next, what's gated by audit

---

## Public artifacts

| Artifact | URL |
|---|---|
| `@veto-protocol/cli` (npm) | https://www.npmjs.com/package/@veto-protocol/cli |
| `@veto-protocol/pay` (npm) | https://www.npmjs.com/package/@veto-protocol/pay |
| `veto-pay` (PyPI) | https://pypi.org/project/veto-pay/ |
| `veto-cli` (PyPI) | https://pypi.org/project/veto-cli/ |
| Live transparency feed | https://veto-ai.com/transparency |
| Public receipt URL pattern | https://veto-ai.com/r/&lt;uuid&gt; |
| JWKS (public key) | https://veto-ai.com/.well-known/jwks.json |
| APPS open policy spec | https://github.com/veto-protocol/x402-policy-schema |
| Mandate verifier (TS) | https://github.com/veto-protocol/mandate-verifier |
| EVM contract source | https://github.com/veto-protocol/contracts |
| Solana program source | https://github.com/veto-protocol/solana-program |
| Veto's own operator policies | https://github.com/veto-protocol/veto-policies |
| Live contract on Base Sepolia | https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5 |

---

## Status

**v1.0** &mdash; public. CLI shipping on npm and PyPI. Engine deployed. Transparency feed live. Contract live on Base Sepolia and Solana Devnet. Audited mainnet contracts ship in v2.

## License

Elastic License v2 (ELv2). See [LICENSE](LICENSE) for the full text and copyright.

---

<div align="center">

**The governance layer for AI agent payments.**<br>
<sub>Owner-defined policy. Signed verdicts. Any rail.</sub>

<br><br>

<sub>Built by <a href="https://veto-ai.com">Investech Global LLC</a> · part of the <a href="https://github.com/veto-protocol">veto-protocol</a> family.</sub>

</div>
