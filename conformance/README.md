# Conformance

An AGT implementation claims conformance to this profile by:

1. Emitting receipts that match the field mappings in
   [../profile.md](../profile.md).
2. Passing all positive test vectors in the shared testvectors repo.
3. Rejecting all negative test vectors in the shared testvectors repo.
4. Round-tripping canonicalization bit-for-bit.

## Test vectors

Conformance vectors live in
[ScopeBlind/agent-governance-testvectors](https://github.com/ScopeBlind/agent-governance-testvectors).

Relevant subdirectories for AGT implementations:

- `vectors/positive/` - receipts that MUST verify cleanly.
- `vectors/negative/` - receipts that MUST be rejected (embedded-key,
  malformed canonical form, etc.).
- `vectors/interop/` - cross-implementation fixtures (APS
  DecisionLineageReceipt, protect-mcp, sb-runtime, etc.).

## Self-certification

An implementation declares conformance by opening a PR against
[../IMPLEMENTATIONS.md](../IMPLEMENTATIONS.md) with:

- Implementation name, repo, and version claiming conformance.
- Evidence of passing conformance vectors (CI run or attached logs).
- Maintainer contact for conformance questions.

This profile is not gated. Any implementation may self-certify subject
to community review.
