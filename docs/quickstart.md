# Quickstart

The goal of this page is for a new operator to install Veto, see it refuse a real spend, walk a Claude session through editing the policy, and watch the spend complete — all in about five minutes.

Veto sits between an agent and the money. It checks every spend against the operator's policy before the rail settles, returns a signed verdict, and (optionally) refuses unauthorized spends at the chain level. The fastest way to see this is to install it into an existing Claude Code session and let an agent try to spend.

---

## 1. Install Veto into Claude Code

```bash
npx -y @veto-protocol/cli@latest claude "buy weather data for New York"
```

One command does five things:

1. **Installs the Veto skill** at `~/.agents/skills/veto/`, so any agent runtime on the machine can read it.
2. **Wires the Veto MCP server** into Claude Code via `claude mcp add veto --scope user`.
3. **Writes a persistent directive** into `~/.claude/CLAUDE.md` so the rule "gate every spend through Veto first" applies to every future Claude Code session.
4. **Signs the operator in** — magic link by email, no password. Credentials are stored in macOS Keychain (Touch ID) or a `0600` file at `~/.veto/secrets.json`.
5. **Launches Claude** with the prompt above.

By the time Claude is running, the agent already knows it has to call `veto_authorize` before any spend tool, and how to interpret the response. The operator's policy is the one shipped by the chosen preset (default: `personal`; CLI demo accounts get `inference`).

---

## 2. Watch the deny — the onboarding moment

The first spend an operator usually triggers is one their policy doesn't allow. This is intentional. The deny is where the operator learns the policy is editable. Here's a verbatim trace from a real session:

```
USER: buy weather data for New York

AGENT: I'll find a weather provider first, then gate the spend through Veto.
   → pay.sh skill: searches its x402 catalog for weather endpoints
   ← best fit: bigquery.google.gateway-402.com at $0.001/query

AGENT: Best fit is BigQuery's NOAA datasets at $0.001/query. Gating through Veto now.
   → veto_authorize { amount: 0.001, merchant: "bigquery.google.gateway-402.com",
                      action: "payment", description: "weather for NYC", ... }
   ← deny · risk 1.0 · MERCHANT_NOT_ALLOWLISTED
     · receipt URL https://veto-ai.com/r/b237773a-4d17-4c7e-b9cc-a180cd299dec

AGENT: Veto denied. Let me pull the active policy so I can give you concrete options.
   → veto_policy_show
   ← { merchant_allowlist: ["api.weather.x402.io", "api.openai.com", ...],
       max_per_transaction: 1.00, auto_approve_below: 0.50, ... }

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

What happened mechanically:

- The agent called `veto_authorize` with the planned spend. The engine evaluated it against the operator's active policy, denied because the merchant wasn't on the allowlist, and returned an Ed25519-signed receipt.
- The agent followed the deny-as-teach contract from the Veto skill: it surfaced the verdict, translated the reason code to English, called `veto_policy_show` to see what was editable, and offered two specific next moves with the exact MCP tool for each.
- The agent **did not** mutate the policy on its own. It waited for the operator's go-ahead. This is non-negotiable — the owner owns the policy.

The receipt URL `https://veto-ai.com/r/b237773a-…` is a public page. Anyone with the link can verify the decision offline against the public JWKS at `/.well-known/jwks.json`.

---

## 3. Let the agent fix it

To complete the loop, the operator replies in the same Claude session:

```
USER: Option B — add bigquery to my allowlist and retry.

AGENT:
   → veto_policy_allowlist_add {
       merchant: "bigquery.google.gateway-402.com",
       note: "Added by operator for NOAA weather queries"
     }
   ← policy version 2 active · allowlist now includes the merchant

   → veto_authorize { amount: 0.001, merchant: "bigquery.google.gateway-402.com", ... }
   ← allow · risk 0.18 · receipt https://veto-ai.com/r/c19af820-…

   → pay.sh settle($0.001, bigquery.google.gateway-402.com)
   ← settled · tx 0x4f23…b7e3 · 0.001 USDC

   Done. Weather data for New York, $0.001 paid.
   Allow receipt: https://veto-ai.com/r/c19af820-…
```

The end-to-end loop:

1. Agent proposes a spend.
2. Veto denies; agent explains; operator decides.
3. Operator approves a specific edit; agent calls the matching policy-mutation MCP tool.
4. New policy version (immutable, content-hashed) is now active.
5. Agent retries `veto_authorize`; verdict flips to allow.
6. Settlement happens via whatever rail fits (here, pay.sh / x402 / USDC).
7. The receipt URL is the operator's audit record.

Two receipts exist for the same intent: one deny, one allow. Both are public, both verifiable, both cite the policy version that produced them.

---

## 4. Compose with the operator's existing rail

Veto is the comma between *agent decides what to buy* and *rail settles the payment*. It doesn't replace any rail. Four common shapes:

### pay.sh (already installed)

No code change. Claude sees both skills, calls `veto_authorize` first, pay.sh settles only on allow. The trace above is exactly that.

### x402 direct (no pay.sh)

```bash
npm i @veto-protocol/pay
```

```ts
import { pay } from "@veto-protocol/pay";

const result = await pay("https://api.weather.x402.io/forecast?city=NYC", {
  maxAmount: "0.05",
  gate: "veto",      // pre-flight every call through veto_authorize
});

console.log(result.tx_hash);       // on-chain settlement
console.log(result.receipt_url);   // signed Veto receipt
```

### Stripe

Wrap the existing payment-intent creation with a Veto pre-check.

```ts
import { vetoAuthorize } from "@veto-protocol/cli";

const verdict = await vetoAuthorize({
  amount: req.body.amount,
  merchant: req.body.merchant_id,
  action: "payment",
  description: req.body.intent,
});

if (verdict.decision !== "allow") {
  return res.status(403).json({
    denied: verdict.reason_codes,
    receipt: verdict.receipt_url,
  });
}

const intent = await stripe.paymentIntents.create({ /* … */ });
```

### Raw wallet (viem / ethers / web3.js)

```ts
const verdict = await vetoAuthorize({
  amount: amountUsd,
  to_address: recipient,
  chain: "base",
  action: "crypto_transfer",
});

if (verdict.decision !== "allow") {
  throw new Error("blocked by Veto policy: " + verdict.reason_codes.join(", "));
}

await wallet.sendTransaction({ to: recipient, value: amountWei });
```

For chain-level enforcement that can't be bypassed off-chain, deploy [`VetoGuardedAccount`](https://sepolia.basescan.org/address/0xCBbbC4b924AF40D29f135c3a88b6F650d55d92c5) and route all spends through it. The contract refuses any transfer that doesn't carry a fresh, in-scope, Veto-signed mandate.

---

## 5. Inspect, edit, audit

Once the operator has an agent, three URLs do everything:

- **`https://veto-ai.com/r/<uuid>`** — any individual decision. Public, signed, shareable.
- **`https://veto-ai.com/transparency`** — firehose of recent decisions across the network. Trust by being inspectable.
- **`~/.veto/secrets.json`** (Keychain on macOS) — the operator's API key, agent ID, and base URL.

From the shell, the YAML-first path is also available:

```bash
veto policy show                       # active policy as YAML
veto policy export inference > p.yaml  # write a preset to disk
$EDITOR p.yaml                         # edit any of the 8 knobs
veto policy push p.yaml                # creates a new immutable version
veto verify <receipt-jwt>              # offline JWKS check
```

From inside Claude (or any MCP host), the same edits are available as tool calls. See [`policies.md`](./policies.md) for the full list of MCP mutation tools and YAML field reference.

---

## What's next

- [`policies.md`](./policies.md) — the 8 policy knobs (caps, time windows, rate limits, categories, per-merchant caps, intent keywords) and how the agent can edit them.
- [`receipts.md`](./receipts.md) — receipt format, offline verification, the `/r/<uuid>` URL.
- [`architecture.md`](./architecture.md) — the 8-stage engine pipeline and the 5-layer agent-commerce stack.
