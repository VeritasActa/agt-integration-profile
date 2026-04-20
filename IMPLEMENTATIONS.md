# Conformant implementations

| Implementation | Repo | Status | Maintainer | Notes |
|---|---|---|---|---|
| sb-runtime | [ScopeBlind/sb-runtime](https://github.com/ScopeBlind/sb-runtime) | Claimed | @tomjwxf | Ring 2 + Ring 3 (Landlock + seccomp). Single Rust binary. |
| protect-mcp | [scopeblind/scopeblind-gateway](https://github.com/scopeblind/scopeblind-gateway) | Claimed | @tomjwxf | Claude Code hooks + MCP gateway. Operator-signed mode. |
| protect-mcp-adk | [scopeblind/protect-mcp-adk](https://github.com/scopeblind/protect-mcp-adk) | Claimed | @tomjwxf | Google ADK BasePlugin. Python. |
| aps-governance-hook | [aeoess/hermes-aps-delegation](https://github.com/aeoess/hermes-aps-delegation) (pending 0.1.0) | Claimed | @aeoess | Authority-chain-referenced mode with delegation_chain_root. |
| Signet | [Prismer-AI/signet](https://github.com/Prismer-AI/signet) | Self-certified (adapter crate pending) | @willamhou | Mapping confirmed by @willamhou in [agt-integration-profile#1](https://github.com/VeritasActa/agt-integration-profile/pull/1); conformance/signet/ captures the agreed field mapping and four documented wire-format deltas. Signet is adding `parent_hash` alongside `parent_receipt_id` and shipping a `--emit-draft02` flag plus an in-workspace `signet-draft02-adapter` crate in an upcoming point release. Cross-documentation at [aeoess/agent-governance-vocabulary#37](https://github.com/aeoess/agent-governance-vocabulary/pull/37). |

## How to add an implementation

Open a PR against this file with:

- Implementation name, repo URL, maintainer GitHub handle.
- Conformance evidence (link to CI run, attached logs, or conformance
  suite output).
- Notes on any optional fields implemented (e.g. `delegation_chain_root`
  for authority-chain mode).
