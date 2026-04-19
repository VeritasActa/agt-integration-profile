# Signet -> draft-farley-acta-signed-receipts-02 field mapping

**Status:** Template. Signet maintainer sign-off required on each `[CONFIRM]` marker.

All field mappings below are inferred from the public Signet source at [Prismer-AI/signet/crates/signet-core/src/](https://github.com/Prismer-AI/signet/tree/main/crates/signet-core/src).

## Top-level structure

| Signet receipt | draft-02 envelope | Notes |
|---|---|---|
| Flat struct with `sig` field | `{payload: {...}, signature: {alg, kid, sig}}` | Structural delta. Requires adapter. See [`DEVIATIONS.md`](./DEVIATIONS.md). |
| `v` (u8) | (no equivalent) | Signet receipt version. Not mapped; the draft relies on `algorithm` and the IETF draft version for versioning. |
| `sig` (String, `ed25519:<base64>`) | `signature.sig` (base64url) + `signature.alg = "EdDSA"` | Encoding delta. See [`DEVIATIONS.md`](./DEVIATIONS.md). |

## Payload field mapping

| Signet field | draft-02 field | Required in draft-02? | Notes |
|---|---|---|---|
| `action.tool` | `payload.action` or `payload.tool_name` | REQUIRED | Tool identifier. draft-02 accepts a string; Signet stores the same. |
| `action.params` | (no equivalent) | - | draft-02 redacts parameter values by default (privacy). See §3 of the draft. Signet carrying the raw `params` is a superset: the adapter drops this field before signing the draft-02 payload. |
| `action.params_hash` | `payload.action_ref` | OPTIONAL | Both are SHA-256 of canonicalized params. `action_ref` is the cross-engine correlation anchor added in draft-02. **Direct semantic match.** `[CONFIRM: is Signet's params_hash a SHA-256 of JCS-canonicalized params? If canonicalization differs, this is a delta.]` |
| `action.target` | (no direct equivalent) | - | Signet-specific resource identifier. `[CONFIRM: can this be expressed as part of the action string, or does it need a new payload extension?]` |
| `action.transport` | `payload.transport_hint` | OPTIONAL (draft-03) | transport_hint accepts `direct`, `ohttp`, `tor`, `custom`. `[CONFIRM: what values does Signet's transport field take, and do any require a new enum variant?]` |
| `action.session` | (no equivalent) | - | `[CONFIRM: is this an agent session ID? If yes, could map to `agent_id` scoped to a session suffix.]` |
| `action.call_id` | (no direct equivalent) | - | `[CONFIRM: is this an external correlation ID? Could map to an application-scoped extension field.]` |
| `action.response_hash` | (no equivalent in a decision receipt) | - | draft-02 decision receipts are signed BEFORE the action executes. Signet pattern of carrying `response_hash` in the same receipt suggests a post-execution receipt variant; `[CONFIRM: does Signet emit one receipt or two (pre + post)? If two, only the pre-receipt is comparable to draft-02 decision receipts.]` |
| `action.trace_id` | `payload.iteration_id` | OPTIONAL | `iteration_id` is the draft-02 field for logical grouping of multi-step workflows. **Direct semantic match.** |
| `action.parent_receipt_id` | `payload.previousReceiptHash` | OPTIONAL from receipt 2+ | Chain linkage delta. Signet links by receipt ID; draft-02 links by SHA-256 of the prior canonical envelope. See [`DEVIATIONS.md`](./DEVIATIONS.md). |
| `signer.pubkey` (`ed25519:<base64>`) | `signature.kid` (JWK thumbprint per RFC 7638) | REQUIRED | Key identifier format delta. See [`DEVIATIONS.md`](./DEVIATIONS.md). `[CONFIRM: is the adapter expected to compute a JWK thumbprint from the Signet pubkey at conversion time?]` |
| `signer.name` | `payload.issuer_id` | OPTIONAL | Operator identity. |
| `signer.owner` | (no direct equivalent) | - | `[CONFIRM: is this a multi-level identity (owner -> named signer)? Could map to the `holder_binding` extension with mode=`jwk_thumbprint`.]` |
| `authorization` | `payload.holder_binding` (AIP-0003) | OPTIONAL | Both represent delegated authority. Signet's `Authorization` struct contents need to match one of the `holder_binding` modes: `jwk_thumbprint`, `dpop`, or `attested_credential`. `[FILL: what fields does Signet's Authorization carry? A direct dump of the Rust struct would resolve this.]` |
| `policy` (PolicyAttestation) | `payload.policy_digest` + `payload.policy_id` | OPTIONAL | `[FILL: what fields does Signet's PolicyAttestation carry? If it is a policy-pack ID + digest, the mapping is trivial. If it carries evaluation evidence, map to `attestation_mode`.]` |
| `ts` (RFC 3339) | `payload.issued_at` (ISO 8601) | REQUIRED | RFC 3339 is a subset of ISO 8601, formats are compatible at the wire level. |
| `exp` (Option<RFC 3339>) | (no equivalent in a decision receipt) | - | Signet receipts have an optional expiration; draft-02 decision receipts do not. `[CONFIRM: is `exp` a soft hint (informational) or a hard verification gate? If the latter, this is a verifier-behavior delta too.]` |
| `nonce` | `payload.nullifier` (VOPRF mode only) | OPTIONAL | Different semantics. `nullifier` in draft-02 is a VOPRF mode artifact (issuer-blind metering); Signet's `nonce` appears to be replay protection. `[CONFIRM: is Signet's nonce a freshness token, a replay counter, or a privacy token? Different answers map to different draft-02 fields or to none.]` |
| `id` | (no direct equivalent; receipt is identified by its hash) | - | Signet assigns each receipt a string ID. draft-02 uses the SHA-256 of the canonical envelope as the identifier. |

## Decision enum

| Signet decision | draft-02 decision | Notes |
|---|---|---|
| `allow` | `allow` | Direct. |
| `deny` | `deny` | Direct. |
| `[FILL]` (any additional) | `require_approval`, `challenge`, `payment_required`, `escalate`, `override`, `compensated` | draft-02 extended enum. `[CONFIRM: does Signet only have allow / deny, or are there additional decision values?]` |

## Mandatory / optional coverage summary

A Signet receipt maps cleanly onto draft-02 **at the envelope level** once the adapter is applied:

- **Directly mappable:** `action.tool`, `action.params_hash`, `action.trace_id`, `action.parent_receipt_id`, `signer.pubkey`, `signer.name`, `ts`, `sig`.
- **Dropped at adapt time (superset):** `action.params` (redacted per draft-02 privacy model), `action.response_hash` (if present and post-execution).
- **Needs maintainer input:** `action.target`, `action.session`, `action.call_id`, `action.response_hash`, `authorization`, `policy`, `exp`, `nonce`, `signer.owner`, decision enum.

Every `[CONFIRM]` / `[FILL]` marker above either resolves to a direct mapping, a deviation (documented in [`DEVIATIONS.md`](./DEVIATIONS.md)), or a proposal for a new OPTIONAL draft-02 field (tracked for draft-03 consideration).
