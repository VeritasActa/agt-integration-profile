# Signet conformance mapping

**Status:** Maintainer-confirmed. All [CONFIRM] / [FILL] markers in the original template were resolved by @willamhou in [VeritasActa/agt-integration-profile#1](https://github.com/VeritasActa/agt-integration-profile/pull/1). The field mapping and deviations are the agreed reference for the Signet ([Prismer-AI/signet](https://github.com/Prismer-AI/signet)) <-> draft-farley-acta-signed-receipts-02 correspondence.

**Signet-side follow-ups (in flight, not blocking this artifact):**

- **`parent_hash` field** added alongside `parent_receipt_id` to close Deviation 3 (chain linkage by ID vs hash). Scheduled for an upcoming Signet point release; additive, non-breaking.
- **`--emit-draft02` flag** on `signet sign`, emitting a draft-02 envelope in parallel with the Signet-native envelope per operation.
- **`signet-draft02-adapter` crate** landing in the Signet Cargo workspace, implementing the co-signer pattern sketched in [`ADAPTER.md`](./ADAPTER.md).

Signet lands as a self-certified implementation in [`../../IMPLEMENTATIONS.md`](../../IMPLEMENTATIONS.md) and as an entry in the draft-farley-acta-signed-receipts-02 Implementation Status appendix when -02 submits to IETF datatracker.

## Files

- [`FIELD-MAPPING.md`](./FIELD-MAPPING.md) - Signet receipt fields mapped onto draft-farley-acta-signed-receipts-02 fields, with source-file references and maintainer-confirmed struct definitions.
- [`DEVIATIONS.md`](./DEVIATIONS.md) - Documented wire-format differences that DO NOT affect semantic conformance but require adapter handling (signature prefix, envelope shape, chain linkage style, key identifier format). Deviation 3 (chain linkage) is being closed by an upcoming Signet point release.
- [`ADAPTER.md`](./ADAPTER.md) - Co-signer adapter reference. Being implemented in the Signet Cargo workspace as `signet-draft02-adapter` and wired behind `signet sign --emit-draft02`.
- `fixtures/` - Empty at the time of this merge. Signet fixtures land when the point release ships.

## Context

Triggered by conversation on [microsoft/agent-governance-toolkit#1201](https://github.com/microsoft/agent-governance-toolkit/pull/1201) between @willamhou (Signet maintainer) and @tomjwxf. Signet shares the same crypto primitives as draft-02 (JCS RFC 8785 canonicalization, Ed25519 signatures, SHA-256 hashing). The wire-format gap is isolated to field naming and encoding conventions; with the maintainer's review, semantic conformance is confirmed.

Signet's own cross-documentation of this mapping is at [aeoess/agent-governance-vocabulary#37](https://github.com/aeoess/agent-governance-vocabulary/pull/37).
