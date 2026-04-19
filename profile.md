# AGT Integration Profile v0.1

Draft mapping between Microsoft Agent Governance Toolkit primitives and
Veritas Acta decision receipt fields.

**Status:** Draft. Field mappings are subject to revision until v1.0.

## Overview

An AGT deployment emits a Veritas Acta decision receipt each time a
governed agent action is evaluated. The receipt is signed with the
operator's key, is verifiable offline using `@veritasacta/verify`, and
composes into a chain via `previousReceiptHash`.

## Receipt fields mapped from AGT primitives

| Veritas Acta field | AGT source | Notes |
|---|---|---|
| `kid` | Operator signing key identifier | RFC 7638 JWK thumbprint recommended. |
| `issuer` | AGT operator identity (e.g. `agt:operator:<id>`) | Opaque to verifier; MUST be stable for a given operator. |
| `issued_at` | Wall-clock time of policy evaluation | ISO 8601, UTC. |
| `algorithm` | `ed25519` (default), `es256` (legacy) | Ed25519 is recommended. |
| `payload.type` | `agt:ring_2_decision` or `agt:ring_3_decision` | Differentiates userspace-only vs sandbox-backed. |
| `payload.policy_id` | AGT policy pack identifier | SHOULD resolve to a specific policy version. |
| `payload.policy_hash` | SHA-256 of policy content evaluated | Binds decision to exact ruleset. |
| `payload.decision` | `allow` \| `deny` \| `require_approval` \| `compensated` | Matches AGT's standard decision enum. |
| `payload.ring` | `2` or `3` | AGT execution ring. |
| `payload.reason` | AGT reason code or human-readable rationale | OPTIONAL; MUST be short when present (<= 256 chars). |
| `payload.action` | The action evaluated (tool name, resource, parameters) | Serialization SHOULD be canonical. |
| `payload.agent_id` | Identity of the agent whose action was evaluated | REQUIRED. |
| `payload.previousReceiptHash` | Hash of prior receipt in this agent's chain | Present from second receipt onward. |
| `payload.delegation_chain_root` | Hash of delegation chain root | OPTIONAL; present in authority-chain-referenced mode. See draft-farley-acta-signed-receipts §6. |
| `signature` | Ed25519 signature over JCS-canonicalized envelope (excluding signature) | Hex, 128 chars. |

## Identity-binding modes

AGT implementations MAY implement either or both:

- **Operator-signed mode** - the only signer is the AGT operator's key.
  Sufficient when the operator is the authority being audited.
- **Authority-chain-referenced mode** - receipt additionally references
  `delegation_chain_root` proving agent authority from a delegation
  root. Required for cross-organization agent commerce and multi-tenant
  regulated deployments.

See Tutorial 33 §6 in microsoft/agent-governance-toolkit for the
expanded discussion.

## Canonicalization

All receipts MUST be JCS-canonicalized (RFC 8785) with AIP-0001 §JCS
adaptations:

- ASCII-only keys at ingest.
- Whole-number floats collapse to integers.

## Verification

All conformant receipts MUST verify against `@veritasacta/verify` with
an externally-sourced key (via `--key`, `--jwks`, or `--trust-anchor`).
Implementations MUST NOT rely on keys embedded in the receipt body for
verification.

## Version history

- **v0.1** (draft) - initial mapping, subject to revision.
