# Conformant implementations

| Implementation | Repo | Status | Maintainer | Notes |
|---|---|---|---|---|
| sb-runtime | [ScopeBlind/sb-runtime](https://github.com/ScopeBlind/sb-runtime) | Claimed | @tomjwxf | Ring 2 + Ring 3 (Landlock + seccomp). Single Rust binary. |
| protect-mcp | [scopeblind/scopeblind-gateway](https://github.com/scopeblind/scopeblind-gateway) | Claimed | @tomjwxf | Claude Code hooks + MCP gateway. Operator-signed mode. |
| protect-mcp-adk | [scopeblind/protect-mcp-adk](https://github.com/scopeblind/protect-mcp-adk) | Claimed | @tomjwxf | Google ADK BasePlugin. Python. |
| aps-governance-hook | [aeoess/hermes-aps-delegation](https://github.com/aeoess/hermes-aps-delegation) (pending 0.1.0) | Claimed | @aeoess | Authority-chain-referenced mode with delegation_chain_root. |
| Signet | TBD | Under review | @willamhou | Added to AGT Tutorial 33 via [microsoft/agent-governance-toolkit#1201](https://github.com/microsoft/agent-governance-toolkit/pull/1201); conformance under confirmation. |

## How to add an implementation

Open a PR against this file with:

- Implementation name, repo URL, maintainer GitHub handle.
- Conformance evidence (link to CI run, attached logs, or conformance
  suite output).
- Notes on any optional fields implemented (e.g. `delegation_chain_root`
  for authority-chain mode).
