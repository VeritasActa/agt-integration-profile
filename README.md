# AGT Integration Profile

This repository formalizes how implementations of the
[Microsoft Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit)
claim conformance to the
[Veritas Acta decision receipt format](https://datatracker.ietf.org/doc/draft-farley-acta-signed-receipts/).

## What's here

- **[profile.md](profile.md)** - the normative mapping between AGT
  primitives (execution rings, policy engine outputs, audit log entries)
  and Veritas Acta receipt fields.
- **[conformance/](conformance/)** - how an implementation demonstrates
  it meets the profile, with pointer to the shared testvectors repo.
- **[examples/](examples/)** - reference fixtures showing each execution
  ring's receipt output.
- **[IMPLEMENTATIONS.md](IMPLEMENTATIONS.md)** - the current list of
  conformant implementations and their status.

## Who this is for

- **AGT operators** deciding which Ring 2/3 backend to deploy.
- **Implementers** shipping a backend that plugs into AGT and produces
  receipts external auditors can verify.
- **Auditors and procurement** verifying that an AGT deployment produces
  receipts in a portable, standardized format.

## Scope

This profile covers the receipt emission side of AGT. Policy authoring,
discovery, identity, and scoring are handled by AGT itself; this repo
only specifies how the outcomes of those subsystems get serialized into
externally verifiable receipts.

## Relationship to other repos

- [ScopeBlind/agent-governance-testvectors](https://github.com/ScopeBlind/agent-governance-testvectors) - shared conformance vectors.
- [VeritasActa/verify](https://github.com/VeritasActa/verify) - reference verifier.
- [draft-farley-acta-signed-receipts](https://datatracker.ietf.org/doc/draft-farley-acta-signed-receipts/) - the IETF spec this profile implements against.

## Maintenance

Maintained under the VeritasActa GitHub organization. Contributions
welcome from AGT maintainers, backend implementers, and independent
reviewers.

## License

Apache-2.0
