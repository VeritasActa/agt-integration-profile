# Contributing to the AGT Integration Profile

This repository defines the normative mapping between Microsoft Agent Governance Toolkit (AGT) primitives and the [Veritas Acta decision receipt format](https://datatracker.ietf.org/doc/draft-farley-acta-signed-receipts/). It tracks which implementations claim conformance, what deviations they carry, and what adapters close those deviations.

We welcome two kinds of contributions:

- **New conformance claims.** Your project emits Ed25519-signed decision receipts and you want to list it as conformant.
- **Corrections and extensions.** The profile mapping, a deviation writeup, or an adapter sketch is wrong or incomplete.

If you're here as a prospective implementer, start with the [self-certification workflow](#self-certification-workflow) below. It's designed to make the conformance claim low-effort for you and high-signal for reviewers.

---

## Self-certification workflow

The pattern: **a template is pre-filled with everything inferrable from your public source, you confirm what we got right and correct what we got wrong, we merge.** The effort asymmetry is deliberate. We do the inference work; you do the sign-off work. Reference prior example: [Signet self-certification](https://github.com/VeritasActa/agt-integration-profile/pull/1) (merged).

### Step 1: Open a self-certification issue

Open an issue titled `Self-certification: <project-name>`. Include:

- Repo URL and the specific commit/tag you're claiming conformance against
- Short description of your signing path (language, wire format, which of the draft-02 receipt types you emit)
- Confirmation that your receipts use Ed25519 (and/or hybrid PQ variants) with JCS RFC 8785 canonicalization
- Maintainer GitHub handle(s) authorized to sign off on the mapping

If your project's wire format differs from draft-02 (field names, signature encoding, chain linkage model, key identifier style), that is fine — we handle deviations in the [`conformance/<project>/DEVIATIONS.md`](#the-conformance-directory-layout) file.

### Step 2: A profile maintainer pre-fills a template

Within a working week, a profile maintainer will open a PR that adds `conformance/<project>/` with four files pre-populated from your public source:

- `FIELD-MAPPING.md` — field-by-field mapping from your receipt format onto draft-02
- `DEVIATIONS.md` — documented wire-format deltas requiring adapter handling
- `ADAPTER.md` — pseudocode sketch of a co-signer or transcode adapter
- `README.md` — what's here and what the maintainer needs to do

Entries inferred from your public code are tagged `[CONFIRM]`. Entries that could not be inferred are tagged `[FILL]`. The PR Cc's the maintainer handle from the issue.

### Step 3: You confirm, correct, or extend

You review the PR and reply with:

- Maintainer-authoritative answers to each `[CONFIRM]` marker (confirmed, or corrected with the right information)
- Information for each `[FILL]` marker (struct definitions, field semantics, enum values — whatever resolves the inference gap)
- Your preference on any deviation that has multiple reasonable paths (e.g. chain linkage by ID vs hash vs additive)

Push commits directly to the PR branch if easier than leaving review comments. The PR is your canvas.

### Step 4: Merge + draft-02 appendix entry

When the maintainer confirms the mapping (all markers resolved, deviations chosen, any corrections applied), the profile maintainer:

- Pushes a follow-up commit resolving all markers per your answers (with `Co-Authored-By: <you>` attribution)
- Updates [`IMPLEMENTATIONS.md`](./IMPLEMENTATIONS.md) to list your project as self-certified with a link to your conformance directory
- Merges the PR
- Adds an entry for your project to the `draft-farley-acta-signed-receipts-02` Implementation Status appendix on the next draft revision

### What "self-certified" means and doesn't mean

- **It means:** the mapping of your receipt format to draft-02 is agreed between the profile maintainers and your project's maintainers, and a draft-02 verifier can produce a meaningful result for receipts in your format (possibly via an adapter, possibly natively).
- **It doesn't mean:** receipts from your project verify against `@veritasacta/verify` unchanged. Where deviations exist, the adapter or future wire-format evolution closes them.
- **It doesn't mean:** this repository or the Veritas Acta project audits your security or certifies your correctness. Conformance is semantic, not adversarial. Operators evaluating multiple implementations still need their own security review.

---

## Contributing without a self-certification claim

You don't need to be an implementer to contribute. Other welcome contributions:

- **Corrections** to existing mappings, deviations, or adapter sketches. Open a PR with the correction and a one-line explanation of the current error.
- **New test vectors.** Negative vectors are especially valuable; see [ScopeBlind/agent-governance-testvectors](https://github.com/ScopeBlind/agent-governance-testvectors).
- **Profile clarifications.** If the base [`profile.md`](./profile.md) is ambiguous for your implementation, open an issue. Ambiguity in the mapping is often a spec-side issue that needs fixing.
- **Implementation listings** for consumer-only integrations (receipt validators, audit tools, archival systems) that don't emit receipts themselves but process them. These are listed in `IMPLEMENTATIONS.md` with appropriate scope notes.

---

## The conformance directory layout

Every self-certified implementation lives under `conformance/<project-name>/`. The expected layout is:

```
conformance/<project>/
├── README.md              # status + Signet-side follow-ups (if any)
├── FIELD-MAPPING.md       # field-by-field mapping with source-file refs
├── DEVIATIONS.md          # wire-format deltas with adapter guidance
├── ADAPTER.md             # reference adapter or co-signer sketch
└── fixtures/              # (optional) signed example receipts
```

The [Signet conformance directory](./conformance/signet/) is the canonical example of what a completed self-certification looks like.

---

## Style and repository conventions

- Markdown files use ATX headings (`#`, `##`, etc.) and no trailing whitespace.
- Source file references use GitHub permalink format: `Prismer-AI/signet/blob/<sha>/path/to/file#L<line>` when pinning to a specific commit matters.
- Commit messages follow Conventional Commits (`feat:`, `fix:`, `docs:`, `conformance(project):` for mapping changes).
- `Co-Authored-By: <name> <email>` is used on any commit that incorporates a maintainer's review content. See Step 4 above.

---

## Normative sources

- **Draft spec:** [draft-farley-acta-signed-receipts-02](https://datatracker.ietf.org/doc/draft-farley-acta-signed-receipts/)
- **Base profile:** [`profile.md`](./profile.md)
- **Test vectors:** [ScopeBlind/agent-governance-testvectors](https://github.com/ScopeBlind/agent-governance-testvectors)
- **Reference verifier:** [`@veritasacta/verify`](https://github.com/ScopeBlind/verify) (Apache-2.0, offline)

---

## Reporting issues

Mapping errors, ambiguities, or adapter bugs: open an issue with the affected file path and the specific item in question.

Security issues (e.g. an adapter pattern that leaks key material): email `tommy@scopeblind.com` privately rather than filing a public issue.
