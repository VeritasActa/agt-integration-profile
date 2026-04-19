# Examples

Reference fixtures showing receipt output for each AGT execution ring.

## Planned examples

- `ring-2-userspace.json` - a Ring 2 policy-only decision (no sandbox).
- `ring-3-sandboxed.json` - a Ring 3 decision with Landlock+seccomp
  attestation.
- `ring-3-authority-chain.json` - a Ring 3 decision including a
  delegation chain root (cross-org mode).

These fixtures will be generated from the
[examples/sb-runtime-governed/](https://github.com/microsoft/agent-governance-toolkit/tree/main/examples/sb-runtime-governed)
and
[examples/protect-mcp-governed/](https://github.com/microsoft/agent-governance-toolkit/tree/main/examples/protect-mcp-governed)
reference examples in AGT.

Each fixture will include:

- The canonical form as-signed.
- The SHA-256 digest before signing.
- A reference to the operator's verification key (JWKS published
  separately, NOT embedded in the receipt).
- Expected `@veritasacta/verify --key <key>` exit status and output.
