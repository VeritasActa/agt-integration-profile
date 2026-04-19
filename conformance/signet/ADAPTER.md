# Signet -> draft-02 adapter sketch

Pseudocode for a thin shim that translates a Signet receipt (as emitted by `Prismer-AI/signet@main` at the date of this PR) into a draft-farley-acta-signed-receipts-02 envelope verifiable against `@veritasacta/verify`.

**Status:** Sketch only. Production implementation is out of scope for this template. Provided so the Signet maintainer can evaluate whether the adapter path is thin enough to pursue, or whether native alignment (Path 3 on [microsoft/agent-governance-toolkit#1201](https://github.com/microsoft/agent-governance-toolkit/pull/1201)) is the better move.

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

    # Optional: map policy_attestation -> policy_digest/policy_id
    if signet_receipt.get("policy"):
        policy = signet_receipt["policy"]
        payload["policy_id"] = policy.get("id")
        payload["policy_digest"] = policy.get("digest")

    # Optional: map authorization -> holder_binding (AIP-0003)
    if signet_receipt.get("authorization"):
        payload["holder_binding"] = {
            "mode": "jwk_thumbprint",  # Adjust based on Signet's auth shape
            "thumbprint": jwk_thumbprint(signet_receipt["authorization"]),
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

## Open questions for the Signet maintainer

1. Does a co-sign adapter make sense as a Signet feature flag (emit both Signet-native and draft-02 envelopes per operation), or as an out-of-tree tool?
2. Would a Rust-native adapter (`signet-draft02-adapter` crate) be accepted upstream into the Signet workspace?
3. Is the operator's private key accessible at the adapter site, or is co-signing constrained by key-custody boundaries?

The answers shape whether the adapter lives in the Signet repo, this repo, or as a third-party crate.
