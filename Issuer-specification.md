# Credential Issuer Specification - AWMP

## Version: 1.1.0
## Date: 2026-01-23
## Protocol: Agent Write Mandate Protocol (AWMP)

---

## Table of Contents

1. [Overview](#overview)
2. [VC Structure](#vc-structure)
3. [Claims Database Specification](#claims-database-specification)
4. [Permissions Database Specification](#permissions-database-specification)
5. [Issuance Process](#issuance-process)
6. [Examples](#examples)
7. [Implementation Checklist](#implementation-checklist)
8. [Alternative: Decentralized Permissions Model](#alternative-decentralized-permissions-model)

---

## Overview

This specification defines the structure and issuance process for Verifiable Credentials (VCs) in the Agent Write Mandate Protocol (AWMP).

### Key Concepts

- **Verifiable Credential (VC)**: JWT-based credential granting agent permissions
- **Claims Database**: Defines available scopes and target resources
- **Permissions Database**: Defines agent authorizations and HITL requirements
- **Scope**: Permission identifier in format `resource:action`
- **DID (Decentralized Identifier)**: Cryptographic identifier for agents and issuers

### VC Purpose

VCs in AWMP serve to:
- Authenticate agents via DIDs
- Authorize operations via scopes
- Delegate permissions with constraints
- Enable auditing and accountability

---

## VC Structure

### Complete VC JWT Format

```json
{
  "header": {
    "alg": "EdDSA",
    "typ": "JWT"
  },
  "payload": {
    "sub": "did:key:z6MkpZqkFm9EHF8CjZVp6RNX8Fj3dCzMxQjLqGvZ4rB2W8yE",
    "iss": "did:key:z6MkrJVnaZkeFzdQyMZmb3dYZ9vpC5VdkA7VfK7CmJfRHaJR",
    "nbf": 1737561600,
    "exp": 1769097600,
    "vc": {
      "@context": [
        "https://www.w3.org/2018/credentials/v1",
        "https://awm-protocol.org/context/v1"
      ],
      "type": [
        "VerifiableCredential",
        "WriteIntentMandate"
      ],
      "credentialSubject": {
        "id": "did:key:z6MkpZqkFm9EHF8CjZVp6RNX8Fj3dCzMxQjLqGvZ4rB2W8yE",

        "agentName": "data-analytics-bot",
        "version": "2.1.0",
        "scopes": ["order:read", "customer:read"],

        "action": ["READ"],
        "target": "postgresql://db.example.com/production/orders",

        "constraints": {
          "maxQueriesPerHour": 100,
          "allowedFields": ["id", "status", "total", "created_at"]
        },

        "validFrom": "2026-01-22T10:00:00.000Z",
        "validUntil": "2026-02-22T10:00:00.000Z"
      }
    }
  },
  "signature": "..."
}
```

### Field Specifications

#### JWT Header

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `alg` | string | Yes | Algorithm (must be "EdDSA") |
| `typ` | string | Yes | Type (must be "JWT") |

#### JWT Payload (Top Level)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sub` | string | Yes | Subject DID (agent requesting the VC) |
| `iss` | string | Yes | Issuer DID (VC issuer's DID) |
| `nbf` | number | Yes | Not Before timestamp (Unix epoch) |
| `exp` | number | Yes | Expiration timestamp (Unix epoch) |

#### Credential Subject

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Subject DID (same as `sub`) |
| `agentName` | string | Yes | Identifier for the agent |
| `version` | string | No | Agent version |
| `scopes` | string[] | Yes | Array of permission scopes |
| `action` | string[] | No | AWMP actions (READ, UPDATE, DELETE, etc.) |
| `target` | string | No | Target resource URI |
| `constraints` | object | No | Operational constraints |
| `validFrom` | string | No | ISO 8601 timestamp for validity start |
| `validUntil` | string | No | ISO 8601 timestamp for validity end |

### Scope Format

Scopes follow the pattern: `{resource}:{action}`

**Examples:**
- `order:read` - Read order data
- `order:write` - Create or update orders
- `order:create` - Create new orders
- `order:update` - Update existing orders
- `order:delete` - Delete orders
- `customer:read` - Read customer data
- `analytics:read` - Query analytics

**Conventions:**
- Use lowercase
- Use singular resource names
- Common actions: `read`, `write`, `create`, `update`, `delete`

### Constraints

Constraints limit what agents can do:

```json
{
  "constraints": {
    "maxRowsPerDay": 50,
    "maxQueriesPerHour": 100,
    "allowedFields": ["id", "status", "total"],
    "valueLimit": { "total": 5000 },
    "timeWindow": "09:00-17:00",
    "allowedRegions": ["US", "EU"]
  }
}
```

Common constraint types:
- **Rate limits**: `maxRowsPerDay`, `maxQueriesPerHour`
- **Field restrictions**: `allowedFields`, `excludedFields`
- **Value limits**: `valueLimit`, `maxAmount`
- **Time restrictions**: `timeWindow`, `expiresAfter`
- **Geographic restrictions**: `allowedRegions`, `allowedCountries`

---

## Claims Database Specification

### Purpose

The Claims Database defines:
- Available permission scopes
- Target resources for each scope
- Scope type (read/write)

This database acts as the **source of truth** for what scopes exist and what they authorize.

### Schema

```typescript
interface ClaimDefinition {
  scope: string;        // Scope identifier (e.g., "order:delete")
  type: "read" | "write";  // Scope type
  target: string[];     // Valid target resources/MCPs
}
```

### Example

```json
[
  {
    "scope": "order:read",
    "type": "read",
    "target": ["mcp:orders-mcp:readorder"]
  },
  {
    "scope": "order:create",
    "type": "write",
    "target": ["mcp:orders-mcp:createorder"]
  },
  {
    "scope": "order:update",
    "type": "write",
    "target": ["mcp:orders-mcp:updateorder"]
  },
  {
    "scope": "order:delete",
    "type": "write",
    "target": ["mcp:orders-mcp:deleteorder"]
  },
  {
    "scope": "customer:read",
    "type": "read",
    "target": ["mcp:customers-mcp:readcustomer"]
  }
]
```

### Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `scope` | string | Yes | Unique scope identifier in format `resource:action` |
| `type` | enum | Yes | Either "read" or "write" |
| `target` | string[] | Yes | Array of valid target resources (MCP tools, APIs, etc.) |

### Usage Guidelines

1. **Scope Naming**
   - Format: `{resource}:{action}`
   - Examples: `order:read`, `customer:write`
   - Use lowercase, singular resource names

2. **Type Classification**
   - `read`: Query, list, search operations
   - `write`: Create, update, delete operations

3. **Target Resources**
   - List all valid MCP tools or API endpoints
   - Format: `mcp:{server}:{tool}` or full URIs
   - Used for validation and auditing

---

## Permissions Database Specification

### Purpose

The Permissions Database defines:
- Which agents are authorized for which scopes
- Agent DID to agent name mapping
- Human-in-the-Loop (HITL) requirements

### Schema

```typescript
interface PermissionDefinition {
  agent: string;    // Agent name
  did: string;      // Agent's DID
  scope: string;    // Authorized scope
  hitl: boolean;    // Whether HITL approval is required
}
```

### Example

```json
[
  {
    "agent": "claude-code-agent",
    "did": "did:key:z6MkpZqkFm9EHF8CjZVp6RNX8Fj3dCzMxQjLqGvZ4rB2W8yE",
    "scope": "order:read",
    "hitl": false
  },
  {
    "agent": "claude-code-agent",
    "did": "did:key:z6MkpZqkFm9EHF8CjZVp6RNX8Fj3dCzMxQjLqGvZ4rB2W8yE",
    "scope": "order:delete",
    "hitl": true
  },
  {
    "agent": "data-analytics-bot",
    "did": "did:key:z6MkfR8TqVvVHJxPQzN7RYx9vpC5VdkA7VfK7CmJfRHaXyZ",
    "scope": "order:read",
    "hitl": false
  }
]
```

### Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `agent` | string | Yes | Agent name (must match VC claim) |
| `did` | string | Yes | Agent's DID (must match VC subject) |
| `scope` | string | Yes | Scope the agent is authorized for |
| `hitl` | boolean | Yes | Whether human approval is required |

### Authorization Rules

1. Agent DID is used to identify the agent for authorization
2. Create separate entry for each scope an agent needs
3. One agent (DID) can have multiple scopes
4. Different agents can have the same scope with different HITL settings

### HITL (Human-in-the-Loop) Guidelines

- Set `true` for high-risk operations (delete, bulk updates, financial transactions)
- Set `false` for routine operations (standard reads, low-risk writes)
- HITL adds an approval step before VC issuance

---

## Issuance Process

### Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│                 AWMP VC Issuance Process                │
└─────────────────────────────────────────────────────────┘

1. Agent Requests VC
        │
        ▼
2. Validate Claims Against Claims Database
        │
        ├─► Invalid scope ─► HTTP 400 "Invalid scope"
        ├─► Missing target ─► HTTP 428 "Target required"
        │
        ▼ Valid
        │
3. Check Agent Permissions
        │
        ├─► Agent not in permissions DB ─► HTTP 403 "Unauthorized"
        ├─► DID mismatch ─► HTTP 403 "DID mismatch"
        ├─► Scope not authorized ─► HTTP 403 "Scope not authorized"
        │
        ▼ Authorized
        │
4. Check HITL Requirement
        │
        ├─► HITL: true ─► Wait for Human Approval
        │                      │
        │                      ├─► Denied ─► HTTP 403 "Approval denied"
        │                      └─► Approved ─► Continue
        │
        ▼ HITL: false or approved
        │
5. Generate VC
        │
        ├─► Create credential subject with scopes
        ├─► Set expiration (nbf, exp)
        ├─► Sign with issuer's DID key (EdDSA)
        │
        ▼
6. Return VC JWT
```

## Examples

### Example 1: Read-Only Access

#### Request

```http
POST /issue HTTP/1.1
Content-Type: application/json

{
  "subjectDid": "did:key:z6MkfR8TqVvVHJxPQzN7RYx9vpC5VdkA7VfK7CmJfRHaXyZ",
  "claims": {
    "agentName": "data-analytics-bot",
    "version": "2.1.0",
    "scopes": ["order:read", "customer:read"]
  }
}
```

#### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "vcJwt": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9...",
  "issuerDid": "did:key:z6MkrJVnaZkeFzdQyMZmb3dYZ9vpC5VdkA7VfK7CmJfRHaJR"
}
```

#### Decoded VC

```json
{
  "sub": "did:key:z6MkfR8TqVvVHJxPQzN7RYx9vpC5VdkA7VfK7CmJfRHaXyZ",
  "iss": "did:key:z6MkrJVnaZkeFzdQyMZmb3dYZ9vpC5VdkA7VfK7CmJfRHaJR",
  "nbf": 1737561600,
  "exp": 1769097600,
  "vc": {
    "@context": ["https://www.w3.org/2018/credentials/v1", "https://awm-protocol.org/context/v1"],
    "type": ["VerifiableCredential", "WriteIntentMandate"],
    "credentialSubject": {
      "id": "did:key:z6MkfR8TqVvVHJxPQzN7RYx9vpC5VdkA7VfK7CmJfRHaXyZ",
      "agentName": "data-analytics-bot",
      "version": "2.1.0",
      "scopes": ["order:read", "customer:read"]
    }
  }
}
```

### Example 2: Write Access with Constraints

#### Request

```http
POST /issue HTTP/1.1
Content-Type: application/json

{
  "subjectDid": "did:key:z6MkpZqkFm9EHF8CjZVp6RNX8Fj3dCzMxQjLqGvZ4rB2W8yE",
  "claims": {
    "agentName": "order-management-bot",
    "version": "1.0.0",
    "scopes": ["order:update"],
    "action": ["UPDATE"],
    "target": "postgresql://db.example.com/production/orders",
    "constraints": {
      "maxRowsPerDay": 50,
      "allowedFields": ["status", "shipping_address", "tracking_number"]
    }
  }
}
```

#### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "vcJwt": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9...",
  "issuerDid": "did:key:z6MkrJVnaZkeFzdQyMZmb3dYZ9vpC5VdkA7VfK7CmJfRHaJR"
}
```

### Example 3: Invalid Scope

#### Request

```http
POST /issue HTTP/1.1
Content-Type: application/json

{
  "subjectDid": "did:key:z6MkpZqkFm9EHF8CjZVp6RNX8Fj3dCzMxQjLqGvZ4rB2W8yE",
  "claims": {
    "agentName": "test-agent",
    "scopes": ["nonexistent:scope"]
  }
}
```

#### Response

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "Invalid scopes",
  "message": "The following scopes are not defined in claims-db: nonexistent:scope",
  "invalidScopes": ["nonexistent:scope"],
  "hint": "Please check the claims-db.json for valid scopes"
}
```

### Example 4: Unauthorized Agent

#### Request

```http
POST /issue HTTP/1.1
Content-Type: application/json

{
  "subjectDid": "did:key:z6MkUNKNOWN...",
  "claims": {
    "agentName": "unauthorized-agent",
    "scopes": ["order:delete"]
  }
}
```

#### Response

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": "Unauthorized scopes",
  "message": "Agent 'unauthorized-agent' with DID did:key:z6MkUNKNOWN... is not authorized for the requested scopes",
  "unauthorizedScopes": ["order:delete"],
  "agentName": "unauthorized-agent",
  "agentDid": "did:key:z6MkUNKNOWN...",
  "hint": "Ensure that BOTH the agent name AND DID match an entry in permissions-db.json"
}
```

---

## Implementation Checklist

### For VC Issuers

- [ ] Implement claims-db.json with scope definitions
  - [ ] Define all available scopes
  - [ ] Set type (read/write) for each scope
  - [ ] Specify target resources

- [ ] Implement permissions-db.json with agent authorizations
  - [ ] Add entries for each agent
  - [ ] Map agent names to DIDs
  - [ ] Set HITL requirements per scope

- [ ] Implement VC issuance endpoint
  - [ ] Validate incoming claims against claims-db
  - [ ] Check agent authorization against permissions-db
  - [ ] Verify DID matches agent name
  - [ ] Handle HITL workflow if required

- [ ] Implement VC signing
  - [ ] Use Ed25519 keys for signing
  - [ ] Generate proper JWT structure
  - [ ] Set appropriate expiration times

- [ ] Security measures
  - [ ] Protect issuer private keys
  - [ ] Validate all inputs
  - [ ] Log all VC issuance for audit trail
  - [ ] Rate limit VC requests

### For Agents

- [ ] Generate and manage DID keys
  - [ ] Use Ed25519 key pair
  - [ ] Derive did:key identifier
  - [ ] Store keys securely

- [ ] Request VCs from issuer
  - [ ] Include subjectDid and claims
  - [ ] Handle HTTP response codes (200, 400, 403, 428)
  - [ ] Store received VCs securely

- [ ] Use VCs to access protected resources
  - [ ] Include VC in Authorization header
  - [ ] Handle VC expiration
  - [ ] Request new VCs when needed

### For Resource Servers

- [ ] Verify incoming VCs
  - [ ] Verify JWT signature cryptographically
  - [ ] Check issuer DID is trusted
  - [ ] Validate expiration (exp, nbf)
  - [ ] Extract and validate scopes

- [ ] Authorize requests
  - [ ] Match endpoint to required scopes
  - [ ] Check VC scopes include required scope
  - [ ] Enforce constraints if present

- [ ] Audit and logging
  - [ ] Log all VC usage
  - [ ] Track agent actions
  - [ ] Monitor for abuse

---

## Security Considerations

### Cryptographic Requirements

1. **Signature Algorithm**: Must use EdDSA (Ed25519)
2. **Key Management**: Protect private keys, rotate periodically
3. **DID Verification**: Verify DID resolution and key ownership

### Validation Requirements

1. **Signature Verification**: Cryptographically verify all VCs
2. **Expiration Check**: Reject expired VCs (check `exp` claim)
3. **Issuer Trust**: Only trust VCs from known, trusted issuers
4. **Scope Validation**: Ensure scopes match endpoint requirements

### Best Practices

1. Use short expiration times (mins to hours, not days)
2. Implement VC revocation mechanism
3. Log all VC issuance and usage
4. Monitor for suspicious patterns
5. Use HITL for high-risk operations
6. Implement rate limiting
7. Regular security audits

---

## Compliance and Auditing

### Audit Trail Requirements

Every VC issuance should be logged with:
- Timestamp
- Agent DID and name
- Requested scopes
- Approval status (granted/denied)
- HITL approval if required
- Issuer DID

### Compliance Considerations

- **Data Privacy**: VCs may contain sensitive information
- **Retention**: Follow data retention policies
- **Access Control**: Limit who can issue VCs
- **Monitoring**: Track unusual patterns

---

## Alternative: Decentralized Permissions Model

**Note:** This section describes an alternative architecture for managing permissions, scopes, and HITL configuration in a decentralized manner.

### Overview

Instead of maintaining centralized databases, permissions and scope definitions can be **distributed across MCP servers** using manifest files. Each MCP server owns and exposes its own security policies.

### Architecture Comparison

#### Centralized (Standard)

```
┌─────────────────────────────────┐
│       VC Issuer                 │
│  ┌──────────────────────────┐   │
│  │  claims-db.json          │   │
│  │  permissions-db.json     │   │
│  │  (all MCPs combined)     │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

**Characteristics:**
- Single source of truth
- Simple to implement
- Manual updates when adding MCPs
- Tight coupling between issuer and MCPs

#### Decentralized (Alternative)

```
┌─────────────────────────────────┐
│       VC Issuer                 │
│  ┌──────────────────────────┐   │
│  │  Discovery & Aggregation │   │
│  │  (reads from MCPs)       │   │
│  └──────────────────────────┘   │
└───────────┬─────────────────────┘
            │
            │ Discovers manifests
            ▼
┌─────────────────────────────────┐
│      MCP Servers                │
│  ┌────────────────────────┐     │
│  │ orders-mcp             │     │
│  │  └─ .awmp/manifest.json│     │
│  └────────────────────────┘     │
│  ┌────────────────────────┐     │
│  │ customers-mcp          │     │
│  │  └─ .awmp/manifest.json│     │
│  └────────────────────────┘     │
└─────────────────────────────────┘
```

**Characteristics:**
- Distributed ownership
- MCP servers define their own policies
- Auto-discovery possible
- Loose coupling

### MCP Manifest Structure

Each MCP server exposes an AWMP manifest file containing scopes, permissions, and HITL configuration.

**File location:** `.awmp/manifest.json` (convention)

**Schema:**
```typescript
interface AWMPManifest {
  mcpServer: string;           // Unique identifier
  version: string;             // Manifest version
  description?: string;        // Human-readable description

  scopes: ScopeDefinition[];   // Replaces claims-db entries
  permissions: PermissionRule[]; // Replaces permissions-db entries
  hitlConfig?: HITLConfig;     // Optional HITL settings

  updatedAt?: string;          // ISO 8601 timestamp
}
```

### Example: orders-mcp Manifest

```json
{
  "mcpServer": "orders-mcp",
  "version": "1.0.0",
  "description": "Order management and fulfillment",

  "scopes": [
    {
      "scope": "order:read",
      "type": "read",
      "target": ["mcp:orders-mcp:read_order"],
      "description": "Read order information",
      "intentVerificationRequired": false
    },
    {
      "scope": "order:delete",
      "type": "write",
      "target": ["mcp:orders-mcp:delete_order"],
      "description": "Delete orders - high risk",
      "intentVerificationRequired": true
    }
  ],

  "permissions": [
    {
      "agent": "claude-code-agent",
      "did": "did:key:z6MkehBKDXpcwGpX8353kWwLQiiAw8ckXGAjMx51hBjcP4h6",
      "scope": "order:read",
      "hitl": false
    },
    {
      "agent": "claude-code-agent",
      "did": "did:key:z6MkehBKDXpcwGpX8353kWwLQiiAw8ckXGAjMx51hBjcP4h6",
      "scope": "order:delete",
      "hitl": true
    }
  ],

  "hitlConfig": {
    "enabled": true,
    "channels": {
      "slack": {
        "enabled": true,
        "channel": "#orders-approvals",
        "mentionUsers": ["@orders-team"]
      }
    }
  },

  "updatedAt": "2026-01-23T10:00:00.000Z"
}
```

### MCP Registration and Discovery

**Critical Requirement:** The issuer must know which MCP servers exist and how to reach them. This requires a registration mechanism.

#### Registration Flow

```
┌─────────────────────────────────────────────────┐
│  Step 1: MCP Server Registers with Issuer      │
└─────────────────────────────────────────────────┘

MCP Server (orders-mcp) starts
    │
    ▼
POST /api/mcp/register
{
  "mcpServer": "orders-mcp",
  "manifestType": "file|http|mcp-resource",
  "manifestLocation": "/path/to/.awmp/manifest.json",
  "version": "1.0.0"
}
    │
    ▼
VC Issuer stores in registry
    │
    ▼
Returns registration confirmation

┌─────────────────────────────────────────────────┐
│  Step 2: Agent Requests VC with MCP Attribution│
└─────────────────────────────────────────────────┘

Agent requests VC
{
  "subjectDid": "did:key:...",
  "claims": {
    "agentName": "claude-code-agent",
    "mcpServer": "orders-mcp",      ← NEW: Which MCP
    "scopes": ["order:read"]
  }
}
    │
    ▼
Issuer looks up "orders-mcp" in registry
    │
    ▼
Fetches manifest from registered location
    │
    ▼
Validates permissions and issues VC
```

#### Enhanced VC Request Format

**With MCP Attribution (Single MCP):**
```json
{
  "subjectDid": "did:key:z6MkehBKDXpcwGpX8353kWwLQiiAw8ckXGAjMx51hBjcP4h6",
  "claims": {
    "agentName": "claude-code-agent",
    "mcpServer": "orders-mcp",
    "scopes": ["order:read", "order:create"]
  }
}
```

**With Multiple MCPs (Cross-MCP VC):**
```json
{
  "subjectDid": "did:key:z6MkehBKDXpcwGpX8353kWwLQiiAw8ckXGAjMx51hBjcP4h6",
  "claims": {
    "agentName": "claude-code-agent",
    "scopeRequests": [
      {
        "mcpServer": "orders-mcp",
        "scopes": ["order:read", "order:create"]
      },
      {
        "mcpServer": "customers-mcp",
        "scopes": ["customer:read"]
      }
    ]
  }
}
```

#### MCP Registry

The issuer maintains a registry of registered MCP servers:

**Registry Structure (`mcp-registry.json`):**
```json
{
  "mcpServers": [
    {
      "mcpServer": "orders-mcp",
      "manifestType": "file",
      "manifestLocation": "/path/to/orders-mcp/.awmp/manifest.json",
      "version": "1.0.0",
      "registeredAt": "2026-01-23T10:00:00.000Z",
      "lastSeen": "2026-01-23T10:00:00.000Z",
      "status": "active"
    },
    {
      "mcpServer": "customers-mcp",
      "manifestType": "http",
      "manifestLocation": "http://localhost:3100/.well-known/awmp-manifest",
      "version": "1.0.0",
      "registeredAt": "2026-01-23T10:05:00.000Z",
      "lastSeen": "2026-01-23T10:05:00.000Z",
      "status": "active"
    }
  ]
}
```

#### Registration API

**Register MCP Server:**
```http
POST /api/mcp/register
Content-Type: application/json

{
  "mcpServer": "orders-mcp",
  "manifestType": "file",
  "manifestLocation": "/path/to/orders-mcp/.awmp/manifest.json",
  "version": "1.0.0",
  "description": "Order management MCP server"
}
```

**Response:**
```http
HTTP/1.1 200 OK

{
  "success": true,
  "mcpServer": "orders-mcp",
  "registeredAt": "2026-01-23T10:00:00.000Z",
  "message": "MCP server registered successfully"
}
```

**List Registered MCPs:**
```http
GET /api/mcp/registry

Response:
{
  "mcpServers": [
    {
      "mcpServer": "orders-mcp",
      "status": "active",
      "version": "1.0.0"
    }
  ]
}
```

**Refresh MCP Manifest:**
```http
POST /api/mcp/refresh/orders-mcp

Response:
{
  "success": true,
  "mcpServer": "orders-mcp",
  "scopesCount": 4,
  "permissionsCount": 6,
  "refreshedAt": "2026-01-23T11:00:00.000Z"
}
```

#### Discovery Methods

Three options for discovering MCP manifests:

#### Option 1: File-Based Discovery

**Configuration:**
```json
{
  "mcpServers": [
    {
      "name": "orders-mcp",
      "type": "file",
      "manifestPath": "/path/to/orders-mcp/.awmp/manifest.json"
    }
  ]
}
```

**Pros:** Simple, works locally
**Cons:** Requires file system access

#### Option 2: HTTP Endpoints

**Convention:** `/.well-known/awmp-manifest`

```bash
curl http://localhost:3100/.well-known/awmp-manifest
```

**Pros:** Standard web pattern, easy to cache
**Cons:** Requires HTTP server in MCP

### Issuer Implementation

**Registry Management:**
```javascript
// issuer.js

import fs from 'fs';
import path from 'path';

// Load MCP registry
let mcpRegistry = loadMcpRegistry();

function loadMcpRegistry() {
  try {
    const registryPath = path.join(process.cwd(), 'mcp-registry.json');
    const data = fs.readFileSync(registryPath, 'utf8');
    return JSON.parse(data);
  } catch (error) {
    return { mcpServers: [] };
  }
}

function saveMcpRegistry(registry) {
  const registryPath = path.join(process.cwd(), 'mcp-registry.json');
  fs.writeFileSync(registryPath, JSON.stringify(registry, null, 2));
}

// MCP Registration Endpoint
app.post('/api/mcp/register', (req, res) => {
  const { mcpServer, manifestType, manifestLocation, version, description } = req.body;

  if (!mcpServer || !manifestType || !manifestLocation) {
    return res.status(400).json({
      error: 'Missing required fields: mcpServer, manifestType, manifestLocation'
    });
  }

  // Check if already registered
  const existing = mcpRegistry.mcpServers.find(m => m.mcpServer === mcpServer);

  if (existing) {
    // Update existing registration
    existing.manifestType = manifestType;
    existing.manifestLocation = manifestLocation;
    existing.version = version;
    existing.lastSeen = new Date().toISOString();
    existing.status = 'active';
  } else {
    // New registration
    mcpRegistry.mcpServers.push({
      mcpServer,
      manifestType,
      manifestLocation,
      version: version || '1.0.0',
      description: description || '',
      registeredAt: new Date().toISOString(),
      lastSeen: new Date().toISOString(),
      status: 'active'
    });
  }

  saveMcpRegistry(mcpRegistry);

  console.log(`✅ MCP server registered: ${mcpServer}`);

  res.json({
    success: true,
    mcpServer,
    registeredAt: existing ? existing.registeredAt : new Date().toISOString(),
    message: `MCP server ${existing ? 'updated' : 'registered'} successfully`
  });
});

// Unregister MCP
app.delete('/api/mcp/register/:mcpServer', (req, res) => {
  const { mcpServer } = req.params;

  const index = mcpRegistry.mcpServers.findIndex(m => m.mcpServer === mcpServer);

  if (index === -1) {
    return res.status(404).json({ error: 'MCP server not found' });
  }

  mcpRegistry.mcpServers.splice(index, 1);
  saveMcpRegistry(mcpRegistry);

  res.json({ success: true, message: `MCP server ${mcpServer} unregistered` });
});

// List registered MCPs
app.get('/api/mcp/registry', (req, res) => {
  res.json({
    mcpServers: mcpRegistry.mcpServers.map(m => ({
      mcpServer: m.mcpServer,
      version: m.version,
      status: m.status,
      registeredAt: m.registeredAt,
      lastSeen: m.lastSeen
    }))
  });
});
```

**Load Manifest for Specific MCP:**
```javascript
async function loadMcpManifest(mcpServer) {
  const registration = mcpRegistry.mcpServers.find(m => m.mcpServer === mcpServer);

  if (!registration) {
    throw new Error(`MCP server not registered: ${mcpServer}`);
  }

  switch (registration.manifestType) {
    case 'file':
      const filePath = path.resolve(registration.manifestLocation);
      const data = fs.readFileSync(filePath, 'utf8');
      return JSON.parse(data);

    case 'http':
      const response = await fetch(registration.manifestLocation);
      if (!response.ok) {
        throw new Error(`Failed to fetch manifest: ${response.status}`);
      }
      return await response.json();

    case 'mcp-resource':
      // Future: Use MCP client
      throw new Error('MCP resource discovery not yet implemented');

    default:
      throw new Error(`Unknown manifest type: ${registration.manifestType}`);
  }
}
```

**Enhanced VC Issuance with MCP Attribution:**
```javascript
app.post('/issue', async (req, res) => {
  const { subjectDid, claims } = req.body;

  // Determine which MCP(s) are involved
  let mcpServers = [];

  if (claims.mcpServer) {
    // Single MCP
    mcpServers = [claims.mcpServer];
  } else if (claims.scopeRequests) {
    // Multiple MCPs
    mcpServers = [...new Set(claims.scopeRequests.map(sr => sr.mcpServer))];
  } else {
    return res.status(400).json({
      error: 'Missing MCP attribution',
      message: 'Please specify mcpServer or use scopeRequests format'
    });
  }

  // Load manifests for involved MCPs
  const manifests = {};
  for (const mcpServer of mcpServers) {
    try {
      manifests[mcpServer] = await loadMcpManifest(mcpServer);
    } catch (error) {
      return res.status(404).json({
        error: `MCP server not found: ${mcpServer}`,
        message: error.message
      });
    }
  }

  // Validate scopes against manifests
  const requestedScopes = claims.scopes ||
    claims.scopeRequests?.flatMap(sr => sr.scopes) || [];

  for (const scope of requestedScopes) {
    let found = false;

    for (const manifest of Object.values(manifests)) {
      if (manifest.scopes.find(s => s.scope === scope)) {
        found = true;
        break;
      }
    }

    if (!found) {
      return res.status(400).json({
        error: `Invalid scope: ${scope}`,
        message: `Scope not found in any registered MCP manifest`
      });
    }
  }

  // Check permissions from manifests
  for (const [mcpServer, manifest] of Object.entries(manifests)) {
    const relevantScopes = claims.mcpServer ? claims.scopes :
      claims.scopeRequests?.find(sr => sr.mcpServer === mcpServer)?.scopes || [];

    for (const scope of relevantScopes) {
      const permission = manifest.permissions.find(p =>
        p.scope === scope &&
        p.did === subjectDid &&
        p.agent === claims.agentName
      );

      if (!permission) {
        return res.status(403).json({
          error: 'Unauthorized',
          message: `Agent not authorized for scope ${scope} on ${mcpServer}`
        });
      }

      // Check HITL requirement
      if (permission.hitl) {
        const approval = await requestHumanApproval(subjectDid, claims,
          `HITL required for ${scope} on ${mcpServer}`);

        if (!approval.approved) {
          return res.status(403).json({ error: approval.error });
        }
      }
    }
  }

  // Issue VC (same as before)
  const vcJwt = await createVerifiableCredentialJwt(vcPayload, issuer);
  res.json({ vcJwt, issuerDid });
});
```

**Periodic Refresh:**
```javascript
// Optional: Refresh manifests periodically
setInterval(async () => {
  for (const registration of mcpRegistry.mcpServers) {
    try {
      const manifest = await loadMcpManifest(registration.mcpServer);
      console.log(`✅ Refreshed manifest for ${registration.mcpServer}`);
      registration.lastSeen = new Date().toISOString();
    } catch (error) {
      console.error(`❌ Failed to refresh ${registration.mcpServer}:`, error.message);
      registration.status = 'error';
    }
  }
  saveMcpRegistry(mcpRegistry);
}, 5 * 60 * 1000); // Every 5 minutes
```

### MCP Server Implementation

**Registration on Startup:**
```javascript
// orders-mcp/server.js

import fetch from 'node-fetch';
import path from 'path';

const MCP_SERVER_NAME = 'orders-mcp';
const ISSUER_URL = process.env.ISSUER_URL || 'http://localhost:3001';
const MANIFEST_PATH = path.join(process.cwd(), '.awmp/manifest.json');

async function registerWithIssuer() {
  try {
    const response = await fetch(`${ISSUER_URL}/api/mcp/register`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        mcpServer: MCP_SERVER_NAME,
        manifestType: 'file',
        manifestLocation: MANIFEST_PATH,
        version: '1.0.0',
        description: 'Order management and fulfillment'
      })
    });

    if (!response.ok) {
      throw new Error(`Registration failed: ${response.status}`);
    }

    const result = await response.json();
    console.log(`✅ Registered with VC issuer:`, result);

  } catch (error) {
    console.error(`❌ Failed to register with issuer:`, error.message);
    // Non-fatal - MCP can still operate without AWMP
  }
}

// Register on startup
registerWithIssuer();

// Re-register periodically (heartbeat)
setInterval(registerWithIssuer, 5 * 60 * 1000); // Every 5 minutes
```

**HTTP Endpoint Method:**
```javascript
// Alternative: Expose manifest via HTTP endpoint

import express from 'express';
import fs from 'fs';

const app = express();

app.get('/.well-known/awmp-manifest', (req, res) => {
  try {
    const manifest = JSON.parse(
      fs.readFileSync('.awmp/manifest.json', 'utf8')
    );
    res.json(manifest);
  } catch (error) {
    res.status(500).json({ error: 'Failed to load manifest' });
  }
});

app.listen(3100);
```

**MCP Resource Method:**
```javascript
// Alternative: Expose manifest as MCP resource

server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [
      {
        uri: "awmp://manifest",
        name: "AWMP Manifest",
        description: "Permissions and HITL configuration",
        mimeType: "application/json"
      }
    ]
  };
});

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  if (request.params.uri === "awmp://manifest") {
    const manifest = JSON.parse(
      fs.readFileSync('.awmp/manifest.json', 'utf8')
    );

    return {
      contents: [{
        uri: request.params.uri,
        mimeType: "application/json",
        text: JSON.stringify(manifest)
      }]
    };
  }
});
```

### Agent Implementation

**Enhanced VC Request with MCP Attribution:**
```javascript
// agent.js

async function requestVC(scopes, mcpServer) {
  const response = await fetch(`${ISSUER_URL}/issue`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      subjectDid: AGENT_DID,
      claims: {
        agentName: 'claude-code-agent',
        mcpServer: mcpServer,  // NEW: Specify which MCP
        scopes: scopes
      }
    })
  });

  if (!response.ok) {
    throw new Error(`VC request failed: ${response.status}`);
  }

  const { vcJwt } = await response.json();
  return vcJwt;
}

// Usage
const vc = await requestVC(['order:read', 'order:create'], 'orders-mcp');
```

**Cross-MCP VC Request:**
```javascript
async function requestCrossMcpVC() {
  const response = await fetch(`${ISSUER_URL}/issue`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      subjectDid: AGENT_DID,
      claims: {
        agentName: 'claude-code-agent',
        scopeRequests: [
          {
            mcpServer: 'orders-mcp',
            scopes: ['order:read', 'order:create']
          },
          {
            mcpServer: 'customers-mcp',
            scopes: ['customer:read']
          }
        ]
      }
    })
  });

  const { vcJwt } = await response.json();
  return vcJwt;
}
```

### Benefits

1. **Distributed Ownership**
   - Each MCP team owns their security policies
   - No central bottleneck for updates

2. **Scalability**
   - Add new MCPs without updating issuer
   - MCPs can be added/removed dynamically

3. **Modularity**
   - MCPs are self-contained
   - Easier to deploy and maintain

4. **Per-MCP Configuration**
   - Different HITL channels per MCP
   - Tailored security policies

5. **Dynamic Discovery**
   - MCPs register themselves on startup
   - No manual configuration needed
   - Automatic heartbeat and health checking

### Considerations

**Registry Management:**
- Maintain MCP registry persistence
- Handle MCP registration failures gracefully
- Implement heartbeat/health checking
- Prune stale/inactive MCPs
- Version control for registry schema

**Caching:**
- Cache manifests to avoid repeated reads
- Implement TTL and refresh strategy
- Consider manifest versioning
- Invalidate cache on MCP re-registration

**Validation:**
- Validate manifest schema on registration
- Check for scope conflicts across MCPs
- Verify DID formats
- Validate manifest location accessibility

**Security:**
- Authenticate MCP registration requests
- Validate manifest sources
- Prevent malicious manifests
- Implement signature verification (future)
- Rate limit registration attempts

**MCP Attribution:**
- Require mcpServer in all VC requests
- Reject requests without MCP attribution
- Validate MCP exists in registry
- Handle cross-MCP VC requests properly

**Error Handling:**
- Handle MCP unreachable scenarios
- Fallback when manifest fetch fails
- Clear error messages for missing MCPs
- Graceful degradation when MCP offline

**Backward Compatibility:**
- Support both centralized and decentralized modes
- Provide migration tooling
- Document upgrade path
- Allow gradual MCP migration

### When to Use Decentralized Model

**Use decentralized when:**
- ✅ Multiple MCP teams with independent ownership
- ✅ Frequently adding/removing MCPs
- ✅ MCPs deployed independently
- ✅ Need per-MCP HITL configuration
- ✅ Microservices architecture

**Use centralized when:**
- ⚪ Single team managing all MCPs
- ⚪ Stable set of MCPs
- ⚪ Simple deployment
- ⚪ Centralized governance required

---

## References

- [W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/)
- [DID Core Specification](https://www.w3.org/TR/did-core/)
- [did:key Method Specification](https://w3c-ccg.github.io/did-method-key/)
- [did-jwt Documentation](https://github.com/decentralized-identity/did-jwt)
- [did-jwt-vc Documentation](https://github.com/decentralized-identity/did-jwt-vc)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.1.0 | 2026-01-23 | Added alternative decentralized permissions model |
| 1.0.0 | 2026-01-22 | Initial AWMP VC specification |

---

**End of Specification**
