# Signet -> draft-02 adapter sketch

Pseudocode for a thin shim that translates a Signet receipt (as emitted by `Prismer-AI/signet@main` at the date of this PR) into a draft-farley-acta-signed-receipts-02 envelope verifiable against `@veritasacta/verify`.

**Status:** Sketch. The Signet maintainer has confirmed the co-signer shape below and committed to shipping this as a `--emit-draft02` flag on `signet sign` (main repo), plus an in-workspace `signet-draft02-adapter` crate. Production implementation is scheduled on the Signet side; this file remains as the architectural reference. Open questions at the bottom are resolved.

## Input

A signed Signet receipt, as emitted by `signet_core::sign::sign()`.

## Output

Either:

- **A draft-02 envelope that verifies against `@veritasacta/verify --key operator.pem`**, IF the adapter has access to the Signet operator's pubkey and can publish a JWKS at the correct discovery endpoint; OR
- **A draft-02 envelope that fails verification with a deterministic `invalid_signature` error** — because the signature was generated over Signet's canonical form, not draft-02's. To produce a verifying draft-02 envelope, the adapter must re-sign.

The latter is usually what is wanted: the adapter acts as a **co-signer** at the governance gateway, producing a draft-02 envelope in parallel with the Signet receipt. Both receipts are emitted; downstream consumers choose which to verify.

## Pseudocode (co-signer mode)

```python
def signet_to_draft02(signet_receipt: dict, operator_signer: Ed25519Signer) -> dict:
    """Co-sign a Signet receipt as a draft-02 envelope.

    signet_receipt: the decoded JSON of a Signet receipt.
    operator_signer: the same Ed25519 key that signed the Signet receipt,
                     wrapped for draft-02-style JCS canonicalization.

    Returns: a draft-02 envelope.
    """
    action = signet_receipt["action"]

    payload = {
        "type": "signet:decision",
        "action": action["tool"],
        "agent_id": signet_receipt["signer"].get("name", "unknown"),
        "issuer_id": signet_receipt["signer"].get("owner", "unknown"),
        "issued_at": signet_receipt["ts"],
        "decision": "allow",  # Or pull from signet's decision field if present
    }

    # Optional mappings (only if present in Signet)
    if "params_hash" in action:
        payload["action_ref"] = action["params_hash"]
    if "trace_id" in action:
        payload["iteration_id"] = action["trace_id"]
    if "parent_receipt_id" in action:
        # Requires the prior receipt (or its hash) to be resolvable.
        # See DEVIATIONS.md section 3.
        payload["previousReceiptHash"] = resolve_prior_hash(action["parent_receipt_id"])

    # Map PolicyAttestation -> policy_digest / policy_id / decision.
    # Signet's PolicyAttestation struct uses policy_name and policy_hash.
    if signet_receipt.get("policy"):
        policy = signet_receipt["policy"]
        payload["policy_id"] = policy.get("policy_name")
        payload["policy_digest"] = policy.get("policy_hash")
        if policy.get("decision"):
            payload["decision"] = policy["decision"]
        # Suggested: attestation_evidence extension for richer fields
        if policy.get("matched_rules") or policy.get("reason"):
            payload["attestation_evidence"] = {
                "matched_rules": policy.get("matched_rules", []),
                "reason": policy.get("reason"),
            }

    # Map Authorization -> holder_binding (AIP-0003).
    # Signet's Authorization struct: { chain, chain_hash, root_pubkey }.
    # Only chain_hash and root_pubkey enter the signed scope; full chain
    # is carried for storage and dropped at adapt time.
    if signet_receipt.get("authorization"):
        auth = signet_receipt["authorization"]
        payload["holder_binding"] = {
            "mode": "jwk_thumbprint",
            "thumbprint": jwk_thumbprint_of_ed25519_pubkey(auth["root_pubkey"]),
            "delegation_chain_hash": auth["chain_hash"],
        }

    # Re-sign under draft-02 canonicalization (JCS RFC 8785, AIP-0001 ASCII-keys)
    canonical = jcs_canonicalize(payload)
    signature_bytes = operator_signer.sign(canonical)

    return {
        "payload": payload,
        "signature": {
            "alg": "EdDSA",
            "kid": jwk_thumbprint(operator_signer.public_key()),
            "sig": base64url_encode(signature_bytes).rstrip("="),
        },
    }
```

## Verification after adapt

```bash
# Write adapted envelope
echo "$DRAFT02_ENVELOPE" > /tmp/signet-as-draft02.json

# Publish operator pubkey as JWKS at well-known URL, then:
npx @veritasacta/verify /tmp/signet-as-draft02.json --jwks https://operator.example/jwks

# Or pin the key locally:
npx @veritasacta/verify /tmp/signet-as-draft02.json --key ./operator-public.pem
```

Expected output on a successful adapt + verify:

```
✓ Signature valid
✓ Issuer: signet:operator:...
✓ Decision: allow
✓ Issued: 2026-04-19T...
✓ Tier: T1 (basic)
```

## Alternative: transcode mode (no re-sign)

If the operator specifically wants the SAME Ed25519 signature to verify under BOTH Signet and draft-02 rules, the transcoding is stricter: the canonicalization MUST produce the same byte string under both systems, which in practice requires the Signet canonical form and the draft-02 payload canonical form to be byte-identical.

That is achievable but constrains Signet's field set (cannot include fields that draft-02 does not canonicalize, and vice versa). Given the known deltas on `params` (redacted in draft-02), this mode is not straightforward.

**Co-sign mode is the recommended default.** Transcode mode is a design conversation for Path 3 (native alignment), not for an adapter.

## Signet maintainer answers

Resolved in the review of [VeritasActa/agt-integration-profile#1](https://github.com/VeritasActa/agt-integration-profile/pull/1):

1. **Feature flag.** The co-signer emits as a `--emit-draft02` flag on `signet sign`. When set, both the Signet-native envelope and the draft-02 envelope are produced per operation. Lives in the main Signet repo (discoverable, tested in the main CI).

2. **In-workspace adapter crate.** A `signet-draft02-adapter` crate is welcome inside the Signet Cargo workspace. Timing depends on the conformance bar; maintainer will prioritise after field mapping is confirmed (confirmed by this PR).

3. **Co-signer mode is the right default.** The operator private key is accessible at the adapter site, so the adapter can produce a draft-02 envelope that verifies under the operator's public key. No cross-custody complications at the adapter site.

The adapter therefore lives in the Signet main repo (not this repo, not third-party). This repo retains the architectural reference (this file) and the normative field mapping ([`FIELD-MAPPING.md`](./FIELD-MAPPING.md)).
