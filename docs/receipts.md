# Receipts

## Format

Every Veto authorize call produces a **JWS-compact (RFC 7515)** receipt:

```
<base64url(header)>.<base64url(payload)>.<base64url(signature)>
```

### Header

```json
{
  "alg": "EdDSA",
  "typ": "JWT",
  "kid": "veto-receipts-v1"
}
```

- `alg: EdDSA` — Ed25519 signature algorithm (RFC 8037)
- `kid: veto-receipts-v1` — key identifier; verifier looks this up in JWKS

### Payload

```json
{
  "iss": "veto",
  "sub": "<transaction-uuid>",
  "iat": 1745783400,
  "decision": "approve" | "deny" | "escalate",
  "decision_layer": "operator_policy",
  "risk_score": 0.0,
  "reason_codes": ["AMOUNT_CAP_EXCEEDED", "..."],
  "engine_version": "0.1.1",
  "input_fingerprint": "<sha256-hex>",
  "agent_id": "<uuid>",
  "client_id": "<uuid>",
  "policy": {
    "id": "<uuid>",
    "version_number": 3,
    "name": "Custom",
    "hash": "<sha256-hex>"
  },
  "mandate_ref": null
}
```

#### Field reference

| Field | Type | Meaning |
|---|---|---|
| `iss` | string | Issuer. Always `"veto"`. |
| `sub` | string | Subject. The transaction UUID. |
| `iat` | int | Issued-at, unix seconds. |
| `decision` | string | `"approve"` / `"deny"` / `"escalate"` |
| `decision_layer` | string | Always `"operator_policy"`. Distinguishes Veto receipts from user-authorization mandates (AP2, Verifiable Intent). |
| `risk_score` | float | 0.0–1.0. Aggregate risk from the engine pipeline. |
| `reason_codes` | string[] | Canonical reason codes (`AMOUNT_CAP_EXCEEDED`, `KNOWN_FRAUD_MERCHANT`, etc.). See [reason codes](#reason-codes). |
| `engine_version` | string | Semver. Bumped when engine semantics change meaningfully (e.g., 0.1.0 → 0.1.1 added hard-deny on allowlist violations). |
| `input_fingerprint` | string | SHA-256 hex over canonical decision inputs. Same input → same fingerprint. Foundation for deterministic replay. |
| `agent_id` | string | UUID of the agent that submitted the action. |
| `client_id` | string | UUID of the customer (client) account. |
| `policy.id` | string | UUID of the policy lineage. |
| `policy.version_number` | int | Revision count for that lineage. |
| `policy.name` | string | Human-readable policy name. |
| `policy.hash` | string | SHA-256 hex over canonical policy contents at sign-time. Closes the in-place tampering gap. |
| `mandate_ref` | string \| null | Reserved for future Safe-guard-module integration where a Veto-signed mandate JWT is required by an on-chain guard. |

## Signing

Veto signs receipts with an **Ed25519** private key (RFC 8032). Production keys are loaded from `VETO_RECEIPT_PRIVATE_KEY_B64` env var (32-byte seed, base64-encoded). Dev environments auto-generate a key and cache it locally (gitignored).

The signing input is the byte sequence:

```
<base64url(header)>.<base64url(payload)>
```

The signature is the Ed25519 signature over those bytes, base64url-encoded. The full receipt is the three components joined by dots.

## Public key distribution (JWKS)

Veto publishes the public key at:

```
https://veto-ai.com/.well-known/jwks.json
```

Format follows RFC 7517:

```json
{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "veto-receipts-v1",
    "alg": "EdDSA",
    "use": "sig",
    "x": "<base64url-public-key>"
  }]
}
```

JWKS is a public document. Cacheable. Anyone with internet access can fetch it.

## Verification

Five steps, all client-side:

1. **Parse** the receipt: split on `.` → `[header_b64, payload_b64, signature_b64]`. Each is base64url-decoded.
2. **Read the `kid`** from the header.
3. **Fetch the JWKS** from `<base_url>/.well-known/jwks.json` (cached locally; 1-hour TTL).
4. **Find the key** with the matching `kid` and `kty: OKP`, `crv: Ed25519`.
5. **Verify the Ed25519 signature** over the bytes `<header_b64>.<payload_b64>` using the public key.

If the signature verifies, the payload is trustworthy. If not, the receipt was tampered with.

**Reference implementation:** [`veto verify` CLI command](https://github.com/veto-protocol/veto-cli/blob/main/veto_cli/receipts.py). Uses `cryptography.hazmat.primitives.asymmetric.ed25519`. ~50 lines of code, no external services beyond the JWKS fetch.

## What the policy_hash provides

The policy_hash field is a SHA-256 over the canonical policy contents at the moment the receipt was signed. Specifically:

1. Build a dict from the policy via `policies.serializers.policy_to_dict`. Fields:
   `schema_version, name, scope, agent_id, max_per_transaction, daily_limit, monthly_limit, auto_approve_below, require_human_approval_above, crypto_daily_limit_usd, merchant_allowlist, merchant_blocklist, category_allowlist, chain_allowlist, token_allowlist, address_allowlist, address_blocklist`.
2. Drop the `_meta` block (it changes on every save — timestamps, is_active).
3. JSON-encode with `sort_keys=True, separators=(",", ":")`.
4. SHA-256 the UTF-8 bytes. Hex-encoded.

**Threat it closes:** in-place admin tampering. Receipt cites `(policy_id, version_number, hash)`. If a malicious admin later edits the SecurityPolicy row contents without bumping `version_number`, the pointer (`policy_id, version_number`) still resolves but to *different bytes*. Future receipts will sign a *different* `policy.hash`. Old receipts (from before tampering) carry the OLD hash. Verifier compares: mismatch detected.

The hash represents pure policy semantics. `version_number` deliberately lives outside the canonical hash; the receipt's `policy.version_number` sibling field still pins which revision was in effect.

## Reason codes

Canonical reason codes that appear in `receipt.reason_codes`. Internal engine signals are mapped to these codes; unknown signals fall back to `SIGNAL_<NAME_UPPER>`.

### Pre-checks

- `KILL_SWITCH_ACTIVE` — client kill switch on
- `AGENT_NOT_OPERATIONAL` — agent suspended

### Spending caps

- `AMOUNT_CAP_EXCEEDED` — over per-tx limit
- `DAILY_CAP_EXCEEDED` — over daily spend
- `MONTHLY_CAP_EXCEEDED` — over monthly spend

### Merchant + address gates

- `MERCHANT_BLOCKLISTED` — merchant on blocklist
- `MERCHANT_NOT_ALLOWLISTED` — non-empty allowlist + merchant not in it
- `KNOWN_FRAUD_MERCHANT` — merchant on Veto's fraud DB
- `TYPOSQUAT_SUSPECTED` — name similar to a well-known brand keyword (legacy)
- `TYPOSQUAT_CANONICAL` — name similar to an entry in the canonical merchant registry (e.g., `api-anthropc.com` resembles `api.anthropic.com`). Fires for *every* user without an allowlist. Homoglyph-aware.
- `HIGH_RISK_CATEGORY` — keyword match (gambling, mixer, adult, etc.)
- `SUSPICIOUS_TLD` — `.xyz`, `.top`, `.click`, etc.
- `LONG_DOMAIN` / `HYPHEN_HEAVY_DOMAIN` — phishing patterns
- `ADDRESS_BLOCKLISTED` — destination on operator's blocklist
- `ADDRESS_NOT_ALLOWLISTED` — non-empty address allowlist + dest not in it
- `ADDRESS_POISONING_SUSPECTED` — destination shares first/last hex chars with a previously-paid recipient — classic address-poisoning attack pattern
- `KNOWN_DRAINER_ADDRESS` — destination on Veto's known-drainer list (hard deny)
- `CHAIN_NOT_ALLOWLISTED` — chain not in operator's chain_allowlist
- `TOKEN_NOT_ALLOWLISTED` — token contract not in operator's token_allowlist
- `OFAC_SANCTIONED_ADDRESS` — destination on the OFAC SDN crypto list (live feed)

### Approval

- `HUMAN_APPROVAL_REQUIRED` — over operator's `require_human_approval_above`

### Behavioral / context

- `PROMPT_INJECTION_DETECTED` — injection patterns in agent context
- `INTENT_MATCH` / `INTENT_PARTIAL` / `INTENT_MISMATCH` — alignment with agent mission
- `INTENT_NO_CONTEXT` — crypto_transfer arrived with no description/context to evaluate intent against (soft signal, neutral)
- `AMOUNT_ANOMALY` — amount > 3× 30-day rolling avg (legacy fixed-threshold)
- `VELOCITY_ANOMALY_EXTREME` / `_HIGH` / `_MODERATE` — burst rates
- `MERCHANT_DIVERSITY_ANOMALY` — many new merchants in 24h
- `NEW_MERCHANT` — first-time merchant for this agent
- `NEW_DESTINATION` — first-time crypto recipient for this agent
- `REPUTATION_NEW` — agent has limited tx history
- `LLM_RISK_FINAL` — Claude reviewed the case and contributed to the score

### Per-agent baseline signals (v0.6+)

Distinguish "trading bot at 20 tx/min (normal)" from "inference agent suddenly at 20 tx/min (suspicious)". Cold-start at 10 settled transactions.

- `BASELINE_WARMING` — under 10 settled txs, baseline isn't ready yet (neutral)
- `AMOUNT_ABOVE_BASELINE_P99` — current amount > 2× this agent's p99 baseline
- `AMOUNT_IN_BASELINE` — current amount within p99 (informational)
- `VELOCITY_ABOVE_BASELINE` — current 1h velocity > 2× lifetime max for this agent
- `UNUSUAL_HOUR_FOR_AGENT` — agent has never settled at this UTC hour before
- `NEW_RECIPIENT_FOR_AGENT` — recipient not in this agent's recent rolling window

## engine_trace and engine_aggregate

Starting in v0.6, the authorize response carries two additional fields alongside the receipt:

```json
{
  "transaction_id": "...",
  "status": "approved",
  "receipt": "<JWS>",
  "mandate": "<JWT>",
  "engine_trace": [
    { "stage": "policy",            "signals": [...], "decision_impact": "soft_signal" },
    { "stage": "merchant_fraud",    "signals": [...], "decision_impact": "soft_signal" },
    { "stage": "crypto",            "signals": [...], "decision_impact": "no_signal" },
    ...
  ],
  "engine_aggregate": {
    "raw_score": 0.18,
    "rep_multiplier": 1.0,
    "rep_adjusted_score": 0.18,
    "fraud_floor_applied": false,
    "human_required_floor_applied": false,
    "llm_final_score": null,
    "decision_score": 0.18,
    "auto_approve_threshold": 0.3,
    "escalate_threshold": 0.7,
    "decision": "approve"
  }
}
```

`engine_trace` answers "which stage caught what" and `engine_aggregate` answers "what was the math". Together they make "why was this denied?" structurally answerable for any decision.

## Example: end-to-end verification in pure Python

```python
import base64, json, urllib.request
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey

def b64url_decode(s):
    return base64.urlsafe_b64decode(s + "=" * (-len(s) % 4))

def verify_veto_receipt(receipt: str, base_url: str = "https://veto-ai.com") -> dict:
    header_b64, payload_b64, sig_b64 = receipt.split(".")
    header = json.loads(b64url_decode(header_b64))
    
    # Fetch JWKS
    jwks = json.loads(urllib.request.urlopen(f"{base_url}/.well-known/jwks.json").read())
    key = next(k for k in jwks["keys"] if k["kid"] == header["kid"])
    pub = Ed25519PublicKey.from_public_bytes(b64url_decode(key["x"]))
    
    # Verify signature
    signing_input = f"{header_b64}.{payload_b64}".encode("ascii")
    pub.verify(b64url_decode(sig_b64), signing_input)  # raises if invalid
    
    return json.loads(b64url_decode(payload_b64))
```

That's the entire verification logic. ~15 lines, stdlib + `cryptography`, no Veto runtime dependency.
