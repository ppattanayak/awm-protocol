# Agent Write Mandate Protocol (AWMP)

[![Spec Version](https://img.shields.io/badge/spec-v1.0-blue)](https://github.com/ppattanayak/awmp-protocol)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-orange)](CONTRIBUTING.md)

**AWMP** is an open pattern for secure, auditable delegation of high-risk "write" operations to AI agents. It generalizes the mandate model from the [Agent Payments Protocol (AP2)](https://ap2-protocol.org) beyond payments to any irreversible state changes.

Use cases include:
- Database inserts, updates, or deletes
- API calls that mutate resources
- File system or configuration changes
- Smart contract executions or on-chain transactions

The protocol requires all write actions to be backed by cryptographically verifiable proof of user intent, supporting both **Human-in-the-Loop (HITL)** and **autonomous** execution.

Built on standard **W3C Verifiable Credentials (VCs)** for interoperability, with seamless integration to **Agent2Agent (A2A)** and **Model Context Protocol (MCP)**.

## Goals
- Prevent erroneous, unauthorized, or hallucinated writes
- Provide full audit trails for compliance and debugging
- Enable safe scaling of agentic systems in production

**Scope**: High-risk mutations only (reads/queries remain unrestricted).

## Core Concepts

- **Mandate**: A W3C Verifiable Credential representing delegated authority.
  - Issued/signed by the user (or wallet/controller).
  - References Decentralized Identifiers (DIDs).
  - Types:
    - **Intent Mandate**: Pre-issued for autonomous use (broad scopes).
    - **Proposal Mandate**: Agent-generated summary for HITL review.
    - **Approval Mandate**: User-signed grant for a specific action.
    - **Execution Mandate**: Final credential attached to the write request.

- **Scope Constraints** (in VC claims):
  - Action type (`CREATE`, `UPDATE`, `DELETE`)
  - Target resource (DB table, API endpoint)
  - Parameter restrictions (fields, values, row limits)
  - Temporal validity
  - Usage quotas
  - Delegation rules
  - Revocation pointer

- **Verifier**: Gateway/service that validates the mandate chain before allowing the write.

## Protocol Flows

### Autonomous (No Human in the Loop)
1. User issues Intent Mandate to agent.
2. Agent checks proposed write against constraints.
3. If compliant → Attach/derive Execution Mandate → Execute.
4. If non-compliant → Escalate to HITL.
5. Verifier validates chain, scopes, and revocation.

### Human-in-the-Loop
1. Agent pauses on high-risk/out-of-scope write.
2. Agent generates Proposal Mandate (human + machine readable).
3. User reviews and signs Approval Mandate.
4. Agent creates Execution Mandate → Execute.
5. Verifier validates.

### Revocation & Fallback
- Mandates reference revocation lists (VC status or on-chain).
- Check revocation before execution.
- On issue: Abort or fall back to HITL.

## Integration
- **Communication**: A2A for mandate exchange
- **Identity**: DIDs for users/agents
- **Tool Access**: Pair with MCP
- **Compatibility**: Reuse AP2 schemas for payment writes

## Security Considerations
- Non-repudiation via cryptographic chain
- Least-privilege scopes
- Revocation support
- Short-lived mandates recommended

## Example: Intent Mandate (JSON-LD)
```json
{
  "@context": ["https://www.w3.org/ns/credentials/v2", "https://awmp.org/context/v1"],
  "type": ["VerifiableCredential", "WriteIntentMandate"],
  "credentialSubject": {
    "id": "did:example:agent-abc123",
    "action": ["UPDATE", "INSERT"],
    "target": "postgresql://db.example.com/production/orders",
    "constraints": {
      "maxRowsPerDay": 50,
      "allowedFields": ["status", "shipping_address"],
      "valueLimit": {"total": 5000}
    },
    "validFrom": "2026-01-16T00:00:00Z",
    "validUntil": "2026-12-31T23:59:59Z"
  },
  "issuer": "did:example:user-xyz456",
  "proof": { /* JWT or Data Integrity proof */ }
}
