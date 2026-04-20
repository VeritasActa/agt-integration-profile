# Signet -> draft-farley-acta-signed-receipts-02 field mapping

**Status:** Maintainer-confirmed. All markers resolved by @willamhou in [VeritasActa/agt-integration-profile#1](https://github.com/VeritasActa/agt-integration-profile/pull/1). Source references below point to [Prismer-AI/signet/crates/signet-core/src/](https://github.com/Prismer-AI/signet/tree/main/crates/signet-core/src).

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
| `action.params_hash` | `payload.action_ref` | OPTIONAL | **Direct semantic match (confirmed).** Both are SHA-256 of canonicalized params. Signet: `params_hash = "sha256:" + hex(SHA-256(JCS(params)))`, canonicalization via `json_canon` (RFC 8785). Source: `sign.rs:29-40`. |
| `action.target` | (extension field) | - | Free-form URI, typically `mcp://server-name` or empty. Identifies the MCP server/endpoint. Not part of the tool name. Maps to an application-scoped extension field in the draft-02 payload; no canonical draft-02 equivalent. |
| `action.transport` | `payload.transport_hint` | OPTIONAL (draft-03) | Default value is `"stdio"`. Other values: `"http"`, `"sse"`. Free-form string, no closed enum at the Signet side. Maps to `transport_hint` with `custom` for non-standard values (draft-02 accepts `direct`, `ohttp`, `tor`, `custom`). |
| `action.session` | (application-scoped extension) | - | Optional MCP session ID. Not the agent session identifier; does not map to `agent_id`. Carried as an application-scoped extension field. |
| `action.call_id` | (application-scoped extension) | - | Optional JSON-RPC request ID. Used in bilateral co-signing to bind an agent-side receipt to the corresponding server-side response. Carried as an application-scoped extension field. |
| `action.response_hash` | (extension field, informational) | - | Signet emits **one** receipt per call. `response_hash` is optional; when present, it is `SHA-256` of the tool response and appears in the same receipt (not a separate post-execution receipt). This is a superset of draft-02's pre-execution-only decision-receipt model. Exception: Signet's LangChain callback handler emits two receipts (`tool_start` + `tool_end`), but that pattern is application-level, not part of the core Signet format. |
| `action.trace_id` | `payload.iteration_id` | OPTIONAL | **Direct semantic match.** Workflow-level correlation string. Signet carries `trace_id` inside the signature scope, so tampering is detectable. |
| `action.parent_receipt_id` | `payload.previousReceiptHash` | OPTIONAL from receipt 2+ | Chain linkage delta. Signet links by receipt ID; draft-02 links by SHA-256 of the prior canonical envelope. See [`DEVIATIONS.md`](./DEVIATIONS.md) Deviation 3. Signet is adding an optional `parent_hash` alongside `parent_receipt_id` in an upcoming point release so both schemes coexist. |
| `signer.pubkey` (`ed25519:<base64>`) | `signature.kid` (JWK thumbprint per RFC 7638) | REQUIRED | **Confirmed.** Adapter computes RFC 7638 JWK thumbprint from the raw Ed25519 pubkey and emits it as `signature.kid`. The raw pubkey is NOT carried in the draft-02 envelope. |
| `signer.name` | `payload.issuer_id` | OPTIONAL | Acceptable with a caveat: `signer.name` is application-assigned (e.g. `"deploy-bot"`), not a registered issuer identifier. Consumers should treat this field as informational. |
| `signer.owner` | (informational metadata) | - | Optional organizational label (e.g. `"acme-corp"`). Metadata only; **not a cryptographic identity**. Does not map to `holder_binding`. Multi-level authority in Signet lives in `authorization` (delegation chain), not in `signer.owner`. Carried as informational metadata. |
| `authorization` | `payload.holder_binding` (AIP-0003) | OPTIONAL | Delegation structure. Full definition per Signet maintainer: `struct Authorization { chain: Vec<DelegationToken>, chain_hash: String ("sha256:<hex>" of JCS(chain)), root_pubkey: String }`. **Important:** only `chain_hash` and `root_pubkey` enter the v4 signature scope (`build_v4_receipt_signable`). The full `chain` array is carried for storage but is NOT signed. Maps to `holder_binding` with mode `jwk_thumbprint` derived from `root_pubkey`. The full chain is Signet-specific, carried as extension or dropped at adapt time. Each `DelegationToken` additionally has an unsigned `correlation_id: Option<String>` annotation that does not appear in signed payloads. |
| `policy` (PolicyAttestation) | `payload.policy_digest`, `payload.policy_id`, `payload.decision` | OPTIONAL | Full struct per Signet maintainer: `struct PolicyAttestation { policy_hash: String ("sha256:<hex>" of JCS(policy)), policy_name: String, matched_rules: Vec<String>, decision: RuleAction, reason: String }`. Mapping: `policy_hash` → `policy_digest`, `policy_name` → `policy_id`, `decision` → `payload.decision`. `matched_rules` + `reason` carry richer evaluation evidence; suggest an `attestation_evidence` extension field at the draft level for lossless mapping. |
| `ts` (RFC 3339) | `payload.issued_at` (ISO 8601) | REQUIRED | RFC 3339 is a subset of ISO 8601, formats are compatible at the wire level. |
| `exp` (Option<RFC 3339>) | (informational extension or dropped) | - | Hard verification gate in Signet: `verify()` rejects expired receipts with `InvalidReceipt("receipt expired at {exp}")`. Separate `verify_allow_expired()` for forensic contexts. This is a verifier-behaviour delta vs draft-02, which places no expiration on decision receipts. The adapter SHOULD drop `exp` when producing a draft-02 envelope, or carry it as informational extension metadata; downstream verifiers using only `@veritasacta/verify` will not enforce expiration. |
| `nonce` | (Signet-specific freshness token; no draft-02 equivalent) | - | 128-bit cryptographically random freshness token (`OsRng`, 16 bytes, hex-encoded, prefixed `rnd_`). Inside signature scope in Signet. NOT a counter and NOT a VOPRF nullifier. draft-02's `nullifier` is semantically different (VOPRF issuer-blind metering artifact). Adapter should carry `nonce` as a Signet-specific extension or omit it. |
| `id` | (not emitted in draft-02) | - | Signet: `id = "rec_" + hex(SHA-256(sig_bytes))[..16]`. Derived from the signature, not from the canonical envelope hash. The adapter MUST compute `SHA-256` of the canonical draft-02 envelope at adapt time when the draft-02 receipt hash is needed (e.g. for `previousReceiptHash` on a subsequent receipt). |

## Decision enum

| Signet decision | draft-02 decision | Notes |
|---|---|---|
| `allow` | `allow` | Direct. |
| `deny` | `deny` | Direct. |
| `require_approval` | `require_approval` | Direct. |

Signet's `RuleAction` enum is `{allow, deny, require_approval}`. All three direct-match draft-02's decision enum; Signet does not currently use the extended values (`challenge`, `payment_required`, `escalate`, `override`, `compensated`).

## Mandatory / optional coverage summary

A Signet receipt maps cleanly onto draft-02 at the envelope level once the adapter is applied:

- **Directly mappable** (semantic equivalents): `action.tool`, `action.params_hash`, `action.trace_id`, `action.parent_receipt_id` (once Signet ships `parent_hash`), `signer.pubkey`, `signer.name`, `ts`, `sig`, and all three values of the `RuleAction` decision enum.
- **Dropped at adapt time** (superset, redacted, or Signet-specific verifier-behaviour): `action.params`, `action.response_hash` (if co-carried in the same receipt), `exp` (hard gate in Signet, not in draft-02), `nonce` (Signet-specific freshness token), `id` (regenerated from canonical envelope hash in draft-02).
- **Mapped to extensions**: `action.target`, `action.session`, `action.call_id`, `matched_rules` and `reason` from `PolicyAttestation`, the unsigned portion of `Authorization.chain`, `signer.owner`. All carried in application-scoped extension fields.

All markers originally tagged `[CONFIRM]` / `[FILL]` are resolved in this commit (mappings confirmed by the Signet maintainer). Residual wire-format deltas are documented in [`DEVIATIONS.md`](./DEVIATIONS.md); the adapter sketch in [`ADAPTER.md`](./ADAPTER.md) reflects the confirmed mapping.
