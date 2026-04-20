# Signet wire-format deviations from draft-farley-acta-signed-receipts-02

Four deltas that require adapter handling but do not affect semantic conformance. Each is adaptable with under ~20 lines of code per direction.

## 1. Envelope shape: flat struct vs `{payload, signature}` split

**Signet.** Single flat struct with `sig` as one field among the others:

```json
{
  "v": 1,
  "id": "...",
  "action": {"tool": "...", ...},
  "signer": {"pubkey": "ed25519:..."},
  "ts": "...",
  "sig": "ed25519:..."
}
```

**draft-02.** Envelope splits the signed material from the signature metadata:

```json
{
  "payload": {
    "action": "...",
    "issuer_id": "...",
    "issued_at": "..."
  },
  "signature": {
    "alg": "EdDSA",
    "kid": "...",
    "sig": "..."
  }
}
```

**Adapter direction (Signet -> draft-02):** Move Signet's `sig` out into `signature.sig` (stripping the `ed25519:` prefix); derive `signature.kid` from `signer.pubkey` (see deviation 4); lift all other Signet top-level and nested fields under `payload`; set `signature.alg = "EdDSA"`.

**Impact on conformance:** None. JCS canonicalization reproduces the same byte string for identical field sets regardless of nesting, as long as the adapter includes all originally-signed fields under `payload`.

## 2. Signature encoding: `ed25519:<base64>` vs `<base64url>`

**Signet.** Signature is a string of the form `ed25519:<standard-base64>` (canonical padded base64).

**draft-02.** Signature is base64url (URL-safe alphabet, no padding), no algorithm prefix. Algorithm is carried in `signature.alg`.

**Rationale for Signet's form.** Prefix-based encoding is self-identifying and forward-compatible with algorithm rotation (a post-quantum variant would be `ml-dsa-65:<base64>`). This is a reasonable design choice; draft-02 achieves the same via the separate `alg` field.

**Adapter direction (Signet -> draft-02):** Strip the `ed25519:` prefix, decode the base64 body, re-encode as base64url without padding, place in `signature.sig`. Set `signature.alg = "EdDSA"`.

**Possible path 3 (native alignment):** draft-02 could optionally accept a prefixed signature form in a future revision, matching Signet's convention. Discussion welcome.

## 3. Chain linkage: parent ID vs parent hash

**Signet.** `action.parent_receipt_id` is the **ID** of the prior receipt.

**draft-02.** `payload.previousReceiptHash` is the **SHA-256 hash of the canonicalized prior envelope**.

**Tradeoff.** Signet's form is shorter and human-linkable; draft-02's form is tamper-evident without needing access to the prior receipt. A verifier given only a chain tip can detect tampering in any intermediate receipt by recomputing hashes; with Signet's IDs, an attacker can substitute a tampered receipt with the same ID and the chain still "links".

**Adapter direction (Signet -> draft-02):** Compute the SHA-256 of the canonicalized Signet prior receipt at adapter time and emit as `previousReceiptHash`. The adapter MUST have access to the prior receipt (or its hash) to emit this field.

**Maintainer preference (selected).** The Signet maintainer has chosen **Option 2**: add a `parent_hash` field alongside `parent_receipt_id`. Dual-mode, additive, low-effort. Both fields coexist on the wire. Signet-native verifiers continue to use `parent_receipt_id`; draft-02 verifiers use `parent_hash`.

**Signet-side implementation note.** This requires the signer to have access to the prior receipt's canonical bytes at sign time; the signer API needs a minor extension to accept (or compute) the parent hash. Scheduled for an upcoming Signet point release. Once shipped, this deviation collapses to "emit `parent_hash` on the wire, read it on the draft-02 side", and the adapter's chain-linkage step becomes zero-cost.

## 4. Key identifier: pubkey string vs JWK thumbprint

**Signet.** `signer.pubkey = "ed25519:<base64-encoded-raw-pubkey>"`. The pubkey is carried directly; verification uses it by decoding.

**draft-02.** `signature.kid` is the RFC 7638 JWK thumbprint of the operator's public key. The pubkey itself is NOT in the receipt. Verifiers resolve the pubkey from an external source (`--jwks`, `--key`, or a pinned trust anchor).

**Rationale for draft-02's form.** Embedding the verification key in the receipt body does not protect against tampering. The key distribution concern is documented in Section 9 of draft-02 ("Key Distribution and Trust Anchors") and was the driver of the 0.4.0 embedded-key rejection across the protect-mcp and @veritasacta/verify stack.

**Adapter direction (Signet -> draft-02):**

- The adapter computes a JWK thumbprint over the Signet pubkey at adapt time and emits it as `signature.kid`.
- The adapter does **not** emit the raw pubkey in the draft-02 envelope.
- The adapter SHOULD publish the pubkey at an out-of-band discovery endpoint (e.g. operator-signed JWKS) for the verifier to resolve via `--jwks`.

**For Signet self-certification:** this is where the conformance bar has teeth. A Signet receipt with the raw pubkey in the payload would fail draft-02 verification on key-distribution grounds. The adapter-produced envelope MUST omit the raw pubkey and MUST carry only the thumbprint.

Not a blocker for conformance; just a real constraint on what the adapter emits.

## Summary

| Delta | Severity | Impact on adapter |
|---|---|---|
| Envelope shape | Structural | ~10 lines, boilerplate. |
| Signature encoding | Cosmetic | ~5 lines, strip prefix + re-encode. |
| Chain linkage (ID vs hash) | Semantic | Resolved. Signet is adding `parent_hash` alongside `parent_receipt_id` in an upcoming point release; dual-mode, zero-cost after the point release ships. |
| Key identifier (pubkey vs thumbprint) | Policy | Adapter MUST NOT emit raw pubkey. External key discovery is the verification path. |

None of these deltas are structural blockers to conformance. All are adapter-addressable; two could be additionally softened by thin Signet-side extensions.
