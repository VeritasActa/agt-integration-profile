# Signet conformance mapping (template)

This directory is a **template** for the Signet ([Prismer-AI/signet](https://github.com/Prismer-AI/signet)) maintainer team to fill in, not a finished conformance claim. It is populated with everything inferrable from the public Signet source (`crates/signet-core/src/`) as of the date of this PR. Entries marked `[CONFIRM]` are best-guess inferences that need maintainer sign-off. Entries marked `[FILL]` require information that is not in the public repo.

The intent is to reduce Signet-side activation energy from "design and draft a conformance mapping" to "confirm / correct / complete a pre-filled mapping". Once that is done, Signet lands as a self-certified implementation in [`../../IMPLEMENTATIONS.md`](../../IMPLEMENTATIONS.md) and as an entry in the [draft-farley-acta-signed-receipts](https://datatracker.ietf.org/doc/draft-farley-acta-signed-receipts/) Implementation Status appendix.

## Files

- [`FIELD-MAPPING.md`](./FIELD-MAPPING.md) - Signet receipt fields mapped onto draft-farley-acta-signed-receipts-02 fields, with `[CONFIRM]` / `[FILL]` markers on anything uncertain.
- [`DEVIATIONS.md`](./DEVIATIONS.md) - Documented wire-format differences that DO NOT affect semantic conformance but require adapter handling (signature prefix, envelope shape, chain linkage style, key identifier format).
- [`ADAPTER.md`](./ADAPTER.md) - Sketch of the thin shim that converts a Signet receipt into a draft-02 envelope for verification with `@veritasacta/verify`. Pseudocode only; a production adapter is out of scope for this template.
- `fixtures/` - Empty. Populated by the Signet maintainer with signed example receipts.

## What the Signet maintainer needs to do

1. Review [`FIELD-MAPPING.md`](./FIELD-MAPPING.md) and resolve each `[CONFIRM]` / `[FILL]` marker.
2. Review [`DEVIATIONS.md`](./DEVIATIONS.md). Confirm the four documented deltas (envelope shape, signature prefix, chain linkage by ID vs. hash, key identifier format). Flag any additional ones.
3. Drop at least one signed Signet receipt into `fixtures/` so the conformance claim is testable against a real artifact.
4. Optionally: build the adapter sketched in [`ADAPTER.md`](./ADAPTER.md) and point this directory at it.
5. Open a PR (or push a commit to this branch) with the above, and update the [`IMPLEMENTATIONS.md`](../../IMPLEMENTATIONS.md) Signet row from "Self-certification template staged" to "Self-certified" with a repo link to the Signet commit/tag that matches the fixtures.

## What this template is NOT

- Not a binding claim of conformance. Signet is not conformant until the maintainer confirms the mapping and the `[CONFIRM]` markers are resolved.
- Not an adapter implementation. `ADAPTER.md` is a sketch, not code.
- Not a fork-and-rewrite proposal. Signet's existing wire format is its own design; this directory documents how a Signet receipt translates into the draft-02 envelope, nothing more.

## Context

Triggered by conversation on [microsoft/agent-governance-toolkit#1201](https://github.com/microsoft/agent-governance-toolkit/pull/1201) between @willamhou (Signet maintainer) and @tomjwxf.

Signet shares the same crypto primitives as draft-02 (JCS RFC 8785 canonicalization, Ed25519 signatures, SHA-256 hashing). The wire-format gap is isolated to field naming and encoding conventions; semantic conformance is plausible and this template is the fastest path to confirming it.
