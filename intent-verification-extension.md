# Intent Verification Extension Specification

## Version: 1.0.0
## Date: 2026-01-22
## Extends: AWMP VC Specification v1.0.0

---

## Table of Contents

1. [Overview](#overview)
2. [Extension Architecture](#extension-architecture)
3. [Complete Request Workflow](#complete-request-workflow)
4. [Enhanced VC Structure](#enhanced-vc-structure)
5. [Intent Reasoning Specification](#intent-reasoning-specification)
6. [Verification Process](#verification-process)
7. [Claims Database Extension](#claims-database-extension)
8. [Examples](#examples)
9. [Implementation Guide](#implementation-guide)

---

## Overview

The Intent Verification Extension is an **security enhancement** for AWMP VC issuers that validates agent understanding before granting access to sensitive operations.

### Purpose

This extension allows VC issuers to:
- Require agents to demonstrate understanding of their actions
- Validate reasoning quality through automated checks
- Create detailed audit trails of agent decision-making
- Prevent mistakes and unintended operations

### Relationship to AWMP

```
┌──────────────────────────────────────────────┐
│         AWMP VC Specification                │
│         (Required - Base Protocol)           │
│  ─────────────────────────────────────       │
│  • DID-based authentication                  │
│  • Scope-based authorization                 │
│  • VC signing and verification               │
│  • Claims and permissions databases          │
└──────────────┬───────────────────────────────┘
               │
               │ Extends with optional fields
               ▼
┌──────────────────────────────────────────────┐
│    Intent Verification Extension             │
│    (Optional - Enhanced Security)            │
│  ─────────────────────────────────────       │
│  • Intent reasoning validation               │
│  • Multi-criteria analysis                   │
│  • Confidence scoring                        │
│  • Verification result embedding             │
└──────────────────────────────────────────────┘
```

### When to Use This Extension

**Recommended for:**
- ✅ High-risk operations (delete, bulk updates, financial transactions)
- ✅ Compliance-sensitive environments
- ✅ AI agents with varying levels of sophistication
- ✅ Operations requiring audit trails

**Not necessary for:**
- ⚪ Simple read operations
- ⚪ Low-risk workflows
- ⚪ Tightly controlled, pre-validated agents
- ⚪ Performance-critical systems where overhead is unacceptable

---

## Extension Architecture

### Modular Design

This extension is designed as a **pluggable module** that:
- Can be added to existing AWMP implementations
- Can be removed without breaking core functionality
- Can be enabled/disabled per scope
- Has no dependencies on core AWMP beyond standard VC fields

### Integration Points

```javascript
// issuer.js with extension

import { issueAWMPCredential } from './awmp-core.js';
import { verifyIntent } from './intent-verifier.js';  // Extension module

app.post('/issue', async (req, res) => {
  const { subjectDid, claims } = req.body;

  // 1. Core AWMP validation (always required)
  const validation = validateClaims(claims);
  const authorized = checkAgentPermission(subjectDid, claims);

  // 2. Optional: Intent Verification Extension
  if (INTENT_VERIFICATION_ENABLED) {
    const needsIntent = isIntentVerificationRequired(claims);

    if (needsIntent) {
      if (!claims.intentReasoning) {
        return res.status(428).json({ error: 'Intent required' });
      }

      const result = verifyIntent(claims.intentReasoning, claims, subjectDid);

      if (!result.verified) {
        return res.status(403).json({ error: 'Verification failed', ...result });
      }

      claims.intentVerificationResult = result;
    }
  }

  // 3. Issue VC (with or without intent fields)
  const vcJwt = await issueAWMPCredential(subjectDid, claims);
  res.json({ vcJwt });
});
```

---

## Complete Request Workflow

This section describes the end-to-end flow of a VC request with intent verification enabled.

### Workflow Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                  COMPLETE VC REQUEST WORKFLOW                   │
│              (With Intent Verification Extension)               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: Agent Preparation                                      │
└─────────────────────────────────────────────────────────────────┘

Agent receives user request
    │
    ▼
Analyzes task and context
    │
    ▼
Determines required scopes
    │
    ▼
Checks if scopes require intent verification
    │
    ├─► No intent required → Prepare standard VC request
    │
    └─► Intent required → Generate intent reasoning
             │
             ├─ taskContext: What user requested and why
             ├─ observedState: Current system state
             ├─ reasoningSteps: Step-by-step logic
             └─ confidence: Agent's confidence (0.0-1.0)

┌─────────────────────────────────────────────────────────────────┐
│  STEP 2: VC Request to Issuer                                   │
└─────────────────────────────────────────────────────────────────┘

POST /issue
{
  "subjectDid": "did:key:z6Mk...",
  "claims": {
    "agentName": "claude-code-agent",
    "mcpServer": "orders-mcp",              // Decentralized: specify MCP
    "scopes": ["order:delete"],
    "intentReasoning": {                    // Extension: intent data
      "taskContext": "User requested deletion of order ORD-123...",
      "observedState": "Order ORD-123 status: processing...",
      "reasoningSteps": [
        "Step 1: Confirmed ORD-123 is duplicate",
        "Step 2: Verified status allows deletion",
        "Step 3: Checked no dependencies",
        "Step 4: Determined deletion is safe"
      ],
      "confidence": 0.95
    }
  }
}

┌─────────────────────────────────────────────────────────────────┐
│  STEP 3: Issuer Validation (Core AWMP)                          │
└─────────────────────────────────────────────────────────────────┘

Issuer receives request
    │
    ▼
Validate request format
    │ Invalid → HTTP 400 "Invalid request"
    ▼
Extract claims and subjectDid
    │
    ▼
┌─────────────────────────────────────────────────┐
│ 3.1: Validate Scopes  from claims DB.           │
└─────────────────────────────────────────────────┘
    │
    ├─► Scope not found → HTTP 400 "Invalid scope"
    │
    ├─► Missing target → HTTP 428 "Target required"
    │
    ▼ Valid scopes
    │
┌─────────────────────────────────────────────────┐
│ 3.2: Check Agent Authorization                  │
│     (permission DB or MCP manifest)             │
└─────────────────────────────────────────────────┘
    │
    ├─► Agent not authorized → HTTP 403 "Unauthorized"
    │
    ├─► DID mismatch → HTTP 403 "DID mismatch"
    │
    ├─► Scope not permitted → HTTP 403 "Scope not authorized"
    │
    ▼ Authorized
    │
┌─────────────────────────────────────────────────┐
│ 3.3: Determine Requirements                     │
└─────────────────────────────────────────────────┘
    │
    ├─► Check if intent verification required
    │   (from claims-db.json: intentVerificationRequired)
    │
    └─► Check if HITL required
        (from permissions-db.json: hitl)

┌─────────────────────────────────────────────────────────────────┐
│  STEP 4: Intent Verification (Extension)                        │
└─────────────────────────────────────────────────────────────────┘

Is intent verification required for ANY scope?
    │
    ├─► NO → Skip to Step 5
    │
    └─► YES
         │
         ▼
    Is intentReasoning provided?
         │
         ├─► NO → HTTP 428 "Intent verification required"
         │         {
         │           "error": "Intent verification required",
         │           "requiresIntentVerification": true,
         │           "requiredFields": { ... },
         │           "hint": "Please retry with intentReasoning"
         │         }
         │
         └─► YES
              │
              ▼
    ┌─────────────────────────────────────────────┐
    │ 4.1: Validate Intent Structure              │
    └─────────────────────────────────────────────┘
              │
              ├─► Missing taskContext → HTTP 400
              ├─► Missing reasoningSteps → HTTP 400
              ├─► Invalid confidence → HTTP 400
              │
              ▼ Valid structure
              │
    ┌─────────────────────────────────────────────┐
    │ 4.2: Perform Intent Verification            │
    │     (verifyIntent function)                 │
    └─────────────────────────────────────────────┘
              │
              ▼
    Run 6 verification checks:
              │
              ├─ 1. Intent Present (20%)
              │     ✓ intentReasoning exists
              │
              ├─ 2. Task Context Valid (20%)
              │     ✓ taskContext >20 chars (recommended)
              │     ⚠ Warning if <20 chars
              │
              ├─ 3. Reasoning Steps Valid (20%)
              │     ✓ Array has 2+ steps
              │     ✓ Each step >10 chars
              │     ⚠ Warning if only 1 step
              │
              ├─ 4. Confidence Acceptable (15%)
              │     ✓ confidence is 0.0-1.0
              │     ⚠ Warning if <0.5
              │
              ├─ 5. Scopes Aligned (15%)
              │     ✓ Intent mentions resources/actions
              │     ⚠ Warning if no clear alignment
              │
              └─ 6. Constraints Respected (10%)
                    ✓ Constraints acknowledged
                    ⚠ Warning if not mentioned
              │
              ▼
    Calculate confidence score
    (weighted sum, threshold: 0.7)
              │
              ▼
    Verification result:
    {
      verified: true/false,
      confidenceScore: 0.0-1.0,
      issues: [],
      warnings: [],
      analysis: { ... }
    }
              │
              ▼
    Is verification successful?
    (verified = true AND score >= 0.7)
              │
              ├─► NO → HTTP 403 "Intent verification failed"
              │         {
              │           "error": "Intent verification failed",
              │           "issues": ["..."],
              │           "warnings": ["..."],
              │           "confidenceScore": 0.4,
              │           "analysis": { ... },
              │           "hint": "Provide more detailed reasoning"
              │         }
              │
              └─► YES
                   │
                   ▼
              Store verification result in claims:
              claims.intentVerificationResult = {
                verified: true,
                confidenceScore: 1.0,
                issues: [],
                warnings: [],
                analysis: { ... },
                verifiedAt: "2026-01-23T10:00:00.000Z"
              }

┌─────────────────────────────────────────────────────────────────┐
│  STEP 5: HITL Approval (If Required)                            │
└─────────────────────────────────────────────────────────────────┘

Is HITL required for ANY scope?
    │
    ├─► NO → Skip to Step 6
    │
    └─► YES
         │
         ▼
    Create approval request
         │
         ▼
    ┌─────────────────────────────────────────────┐
    │ 5.1: Send HITL Notifications                │
    │      (HITL Extension - Optional)            │
    └─────────────────────────────────────────────┘
         │
         ├─► Webhook → External system
         ├─► Slack → Approval channel
         └─► Email → Approvers
         │
         ▼
    Display in web UI
         │
         ▼
    Wait for human decision
         │
         ├─► APPROVED → Continue to Step 6
         │
         ├─► DECLINED → HTTP 403 "Access denied by administrator"
         │
         └─► TIMEOUT → HTTP 403 "Approval request timed out"

┌─────────────────────────────────────────────────────────────────┐
│  STEP 6: VC Issuance                                            │
└─────────────────────────────────────────────────────────────────┘

All validations passed
    │
    ▼
Create VC payload
    │
    ├─ Standard AWMP fields
    ├─ Intent verification result (if extension used)
    └─ Constraints
    │
    ▼
Sign VC with issuer's DID key (EdDSA)
    │
    ▼
Generate JWT
    │
    ▼
HTTP 200 OK
{
  "vcJwt": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9...",
  "issuerDid": "did:key:z6MkrJVnaZkeFzdQyMZmb3dYZ9vpC5VdkA7VfK7CmJfRHaJR"
}

┌─────────────────────────────────────────────────────────────────┐
│  STEP 7: Agent Uses VC                                          │
└─────────────────────────────────────────────────────────────────┘

Agent receives VC
    │
    ▼
Stores VC securely
    │
    ▼
Uses VC to access protected resources
    │
    ├─ Include in Authorization header
    ├─ Resource server verifies VC
    └─ Checks scopes and constraints
    │
    ▼
Operation completed
    │
    ▼
Audit trail created
    ├─ VC issuance logged
    ├─ Intent reasoning recorded
    ├─ Verification result stored
    └─ Operation outcome tracked
```

### Decision Points Summary

| Decision Point | Condition | Outcome if True | Outcome if False |
|----------------|-----------|-----------------|------------------|
| **Valid request format?** | Request has required fields | Continue | HTTP 400 "Invalid request" |
| **Scopes exist?** | All scopes in claims-db | Continue | HTTP 400 "Invalid scope" |
| **Agent authorized?** | Agent in permissions-db | Continue | HTTP 403 "Unauthorized" |
| **Intent required?** | `intentVerificationRequired: true` | Check intent | Skip intent check |
| **Intent provided?** | `claims.intentReasoning` exists | Verify intent | HTTP 428 "Intent required" |
| **Intent valid?** | Verification score ≥ 0.7 | Continue | HTTP 403 "Verification failed" |
| **HITL required?** | `hitl: true` | Wait for approval | Continue to VC issuance |
| **HITL approved?** | Human approves | Continue | HTTP 403 "Access denied" |

### HTTP Response Codes

| Code | Meaning | When Used |
|------|---------|-----------|
| **200** | Success | VC issued successfully |
| **400** | Bad Request | Invalid request format, invalid scope, malformed intent |
| **403** | Forbidden | Unauthorized agent, verification failed, HITL declined |
| **428** | Precondition Required | Intent required but not provided, target required |
| **500** | Internal Server Error | VC signing failed, database error |

### Example Flows

#### Flow 1: Read Operation (No Intent)

```
Agent → POST /issue (order:read)
     → Issuer validates scopes ✓
     → Issuer checks authorization ✓
     → Intent not required (type: read)
     → HITL not required
     → HTTP 200 + VC
```

#### Flow 2: Write Operation (Intent Required, Success)

```
Agent → POST /issue (order:delete + intentReasoning)
     → Issuer validates scopes ✓
     → Issuer checks authorization ✓
     → Intent required ✓
     → Intent provided ✓
     → Verification runs ✓
     → Score: 1.0 (>= 0.7) ✓
     → HITL required
     → Human approves ✓
     → HTTP 200 + VC (with intentVerificationResult)
```

#### Flow 3: Intent Missing

```
Agent → POST /issue (order:delete, no intent)
     → Issuer validates scopes ✓
     → Issuer checks authorization ✓
     → Intent required ✓
     → Intent provided ✗
     → HTTP 428 "Intent verification required"
```

#### Flow 4: Intent Verification Fails

```
Agent → POST /issue (order:delete + weak intent)
     → Issuer validates scopes ✓
     → Issuer checks authorization ✓
     → Intent required ✓
     → Intent provided ✓
     → Verification runs
     → Score: 0.4 (< 0.7) ✗
     → HTTP 403 "Intent verification failed"
         {
           "issues": ["Task context too brief"],
           "warnings": ["Low confidence"],
           "confidenceScore": 0.4
         }
```

#### Flow 5: HITL Declined

```
Agent → POST /issue (order:delete + intentReasoning)
     → Issuer validates scopes ✓
     → Issuer checks authorization ✓
     → Intent verification ✓
     → HITL required
     → Human declines ✗
     → HTTP 403 "Access denied by administrator"
```

### Retry Strategy for Agents

**HTTP 428 (Intent Required):**
```javascript
// Agent should generate intent and retry
try {
  const vc = await requestVC(scopes);
} catch (error) {
  if (error.status === 428) {
    const intent = generateIntentReasoning(userRequest);
    const vc = await requestVC(scopes, intent);
  }
}
```

**HTTP 403 (Verification Failed):**
```javascript
// Agent should improve reasoning and retry (once)
try {
  const vc = await requestVC(scopes, intent);
} catch (error) {
  if (error.status === 403 && error.confidenceScore < 0.7) {
    const improvedIntent = enhanceReasoning(intent, error.issues);
    const vc = await requestVC(scopes, improvedIntent);
  }
}
```

---

## Enhanced VC Structure

### Additional Fields

When this extension is enabled and intent is required, the VC includes:

```json
{
  "vc": {
    "credentialSubject": {
      // ... standard AWMP fields ...

      "intentReasoning": {
        "taskContext": "string",
        "observedState": "string (optional)",
        "reasoningSteps": ["string", "..."],
        "confidence": 0.95
      },

      "intentVerificationResult": {
        "verified": true,
        "confidenceScore": 1.0,
        "issues": [],
        "warnings": [],
        "analysis": {
          "intentPresent": true,
          "taskContextValid": true,
          "reasoningStepsValid": true,
          "confidenceAcceptable": true,
          "scopesAligned": true,
          "constraintsRespected": true
        },
        "verifiedAt": "2026-01-22T10:00:00.000Z"
      }
    }
  }
}
```

### Field Specifications

#### intentReasoning (Provided by Agent)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `taskContext` | string | Yes | What the user requested and why (>20 chars recommended) |
| `observedState` | string | No | Current system state observed by agent |
| `reasoningSteps` | string[] | Yes | Step-by-step reasoning (2+ steps recommended) |
| `confidence` | number | Yes | Agent's confidence in the operation (0.0-1.0) |

#### intentVerificationResult (Added by Issuer)

| Field | Type | Description |
|-------|------|-------------|
| `verified` | boolean | Whether verification passed |
| `confidenceScore` | number | Verification confidence (0.0-1.0) |
| `issues` | string[] | Critical issues found (empty if passed) |
| `warnings` | string[] | Non-critical warnings |
| `analysis` | object | Detailed breakdown of checks |
| `verifiedAt` | string | ISO 8601 timestamp of verification |

---

## Intent Reasoning Specification

### Structure

```typescript
interface IntentReasoning {
  taskContext: string;       // What and why
  observedState?: string;    // Current state (optional)
  reasoningSteps: string[];  // Step-by-step logic
  confidence: number;        // 0.0 - 1.0
}
```

### Requirements

#### taskContext (Required)

**Purpose:** Explain what the user requested and why

**Guidelines:**
- Minimum 20 characters recommended
- Clear, specific description
- Include both **what** and **why**

**Examples:**

✅ **Good:**
```
"User requested deletion of order ORD-123 because it was created
by mistake and is a duplicate of ORD-124"
```

❌ **Bad:**
```
"Delete order"  // Too vague, no context
```

#### observedState (Optional but Recommended)

**Purpose:** Describe current system state

**Guidelines:**
- Include relevant data, statuses, conditions
- Helps demonstrate agent's awareness

**Examples:**

✅ **Good:**
```
"Order ORD-123 has status 'processing'. Customer confirmed it's
a duplicate of ORD-124 which has status 'shipped'."
```

#### reasoningSteps (Required)

**Purpose:** Step-by-step breakdown of agent's logic

**Guidelines:**
- Minimum 2 steps (4-5 recommended)
- Each step should be a complete sentence
- Show logical progression

**Examples:**

✅ **Good:**
```json
[
  "Step 1: Confirmed order ORD-123 is a duplicate of ORD-124",
  "Step 2: Verified ORD-123 is in 'processing' status (safe to delete)",
  "Step 3: Checked no downstream dependencies on this order",
  "Step 4: Determined deletion is safe and aligns with user intent"
]
```

❌ **Bad:**
```json
["Delete it"]  // Only 1 step, too vague
```

#### confidence (Required)

**Purpose:** Agent's confidence in the operation

**Range:** 0.0 - 1.0 (0% - 100%)

**Guidelines:**
| Range | Meaning | When to Use |
|-------|---------|-------------|
| 0.9 - 1.0 | Very confident | Clear user intent, all info available |
| 0.7 - 0.9 | Confident | Standard operation, normal case |
| 0.5 - 0.7 | Moderate | Some uncertainty exists |
| < 0.5 | Low ⚠️ | Uncertain (triggers warnings) |

**Verification threshold:** ≥ 0.7 recommended

---

## Verification Process

### Verification Algorithm

The extension performs these checks on intent reasoning:

```
┌─────────────────────────────────────────────────────┐
│         Intent Verification Checks                  │
└─────────────────────────────────────────────────────┘

1. Intent Present
   ├─ Check: intentReasoning object exists
   └─ Fail if: Missing

2. Task Context Valid
   ├─ Check: taskContext is meaningful (>20 chars)
   └─ Warn if: Too brief (<20 chars)

3. Reasoning Steps Valid
   ├─ Check: Array has 2+ steps, each >10 chars
   └─ Warn if: Only 1 step or steps are too brief

4. Confidence Acceptable
   ├─ Check: confidence is 0.0-1.0
   └─ Warn if: <0.5 (low confidence)

5. Scopes Aligned
   ├─ Check: Intent mentions resources/actions in scopes
   └─ Warn if: No clear alignment

6. Constraints Respected
   ├─ Check: Constraints acknowledged if present
   └─ Warn if: Constraints not mentioned

Overall Result
├─ Pass if: No critical issues + score ≥ 0.7
└─ Fail if: Issues found or score < 0.7
```

### Confidence Scoring

```javascript
function calculateConfidenceScore(checks) {
  const weights = {
    intentPresent: 0.20,
    taskContextValid: 0.20,
    reasoningStepsValid: 0.20,
    confidenceAcceptable: 0.15,
    scopesAligned: 0.15,
    constraintsRespected: 0.10
  };

  let score = 0;
  for (const [check, passed] of Object.entries(checks)) {
    if (passed) score += weights[check];
  }

  return score;
}
```

### Pass/Fail Criteria

**Verification passes if ALL of:**
- ✅ No critical issues
- ✅ Confidence score ≥ 0.7 (70%)
- ✅ Intent reasoning is present
- ✅ Task context is valid

**Verification fails if ANY of:**
- ❌ Intent reasoning missing (when required)
- ❌ Task context empty or invalid
- ❌ Confidence score < 0.7
- ❌ Critical validation errors

---

## Claims Database Extension

### Additional Field

When implementing this extension, add to `claims-db.json`:

```typescript
interface ClaimDefinition {
  // ... standard AWMP fields ...
  scope: string;
  type: "read" | "write";
  target: string[];

  // Extension field (optional)
  intentVerificationRequired?: boolean;  // Default: false
}
```

### Example

```json
[
  {
    "scope": "order:read",
    "type": "read",
    "target": ["mcp:orders-mcp:readorder"],
    "intentVerificationRequired": false
  },
  {
    "scope": "order:delete",
    "type": "write",
    "target": ["mcp:orders-mcp:deleteorder"],
    "intentVerificationRequired": true
  }
]
```

### Configuration Guidelines

**Set `intentVerificationRequired: true` for:**
- Write operations (create, update, delete)
- High-risk operations
- Operations with irreversible consequences
- Compliance-critical operations

**Set `intentVerificationRequired: false` or omit for:**
- Read operations
- Low-risk operations
- Performance-critical operations

---

## Examples

### Example 1: With Intent Verification (Successful)

#### VC Request

```http
POST /issue HTTP/1.1
Content-Type: application/json

{
  "subjectDid": "did:key:z6MkpZqkFm9EHF8CjZVp6RNX8Fj3dCzMxQjLqGvZ4rB2W8yE",
  "claims": {
    "agentName": "claude-code-agent",
    "scopes": ["order:delete"],
    "intentReasoning": {
      "taskContext": "User requested deletion of order ORD-123 because it was created by mistake and is a duplicate of ORD-124",
      "observedState": "Order ORD-123 status: 'processing', ORD-124 exists with same items",
      "reasoningSteps": [
        "Step 1: Confirmed ORD-123 is duplicate of ORD-124",
        "Step 2: Verified ORD-123 is in 'processing' status",
        "Step 3: Checked no downstream dependencies",
        "Step 4: Determined deletion is safe and appropriate"
      ],
      "confidence": 0.95
    }
  }
}
```

#### Response

```http
HTTP/1.1 200 OK

{
  "vcJwt": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9...",
  "issuerDid": "did:key:z6MkrJVnaZkeFzdQyMZmb3dYZ9vpC5VdkA7VfK7CmJfRHaJR"
}
```

#### Decoded VC (includes verification result)

```json
{
  "credentialSubject": {
    "agentName": "claude-code-agent",
    "scopes": ["order:delete"],
    "intentReasoning": { /* ... */ },
    "intentVerificationResult": {
      "verified": true,
      "confidenceScore": 1.0,
      "issues": [],
      "warnings": [],
      "analysis": {
        "intentPresent": true,
        "taskContextValid": true,
        "reasoningStepsValid": true,
        "confidenceAcceptable": true,
        "scopesAligned": true,
        "constraintsRespected": true
      },
      "verifiedAt": "2026-01-22T10:00:00.000Z"
    }
  }
}
```

### Example 2: Intent Required But Not Provided

#### Request

```http
POST /issue HTTP/1.1
Content-Type: application/json

{
  "subjectDid": "did:key:z6MkpZqkFm9EHF8CjZVp6RNX8Fj3dCzMxQjLqGvZ4rB2W8yE",
  "claims": {
    "agentName": "claude-code-agent",
    "scopes": ["order:delete"]
    // No intentReasoning provided!
  }
}
```

#### Response (HTTP 428 Precondition Required)

```http
HTTP/1.1 428 Precondition Required
Content-Type: application/json

{
  "error": "Intent verification required",
  "message": "This request requires intentReasoning based on scope definitions",
  "requiresIntentVerification": true,
  "requiredFields": {
    "intentReasoning": {
      "taskContext": "string - description of the user's task",
      "observedState": "string - current state (optional)",
      "reasoningSteps": ["array of reasoning steps"],
      "confidence": "number (0-1) - agent's confidence"
    }
  },
  "hint": "Please retry your request with the intentReasoning field populated"
}
```

### Example 3: Intent Verification Failed

#### Request

```http
POST /issue HTTP/1.1
Content-Type: application/json

{
  "subjectDid": "did:key:z6MkpZqkFm9EHF8CjZVp6RNX8Fj3dCzMxQjLqGvZ4rB2W8yE",
  "claims": {
    "agentName": "claude-code-agent",
    "scopes": ["order:delete"],
    "intentReasoning": {
      "taskContext": "Delete order",  // Too vague
      "reasoningSteps": ["Delete it"],  // Only 1 step, vague
      "confidence": 0.4  // Low confidence
    }
  }
}
```

#### Response (HTTP 403 Forbidden)

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": "Intent verification failed",
  "message": "The provided intent reasoning does not meet verification requirements",
  "issues": [
    "Task context is too brief (must be >20 characters)"
  ],
  "warnings": [
    "Only one reasoning step provided - more detailed reasoning recommended",
    "Agent confidence is below 50% - risky operation"
  ],
  "confidenceScore": 0.4,
  "analysis": {
    "intentPresent": true,
    "taskContextValid": false,
    "reasoningStepsValid": true,
    "confidenceAcceptable": false,
    "scopesAligned": true,
    "constraintsRespected": false
  },
  "hint": "Review the issues and warnings, then provide more detailed intent reasoning"
}
```

---

## Implementation Guide

### Step 1: Install Extension Module

```bash
# Add intent-verifier module to your project
cp intent-verifier.js ./lib/
```

### Step 2: Enable Extension

```javascript
// config.js
export const INTENT_VERIFICATION_ENABLED =
  process.env.ENABLE_INTENT_VERIFICATION === 'true';
```

### Step 3: Update Claims Database

```json
// db/claims-db.json
[
  {
    "scope": "order:delete",
    "type": "write",
    "target": ["mcp:orders-mcp:deleteorder"],
    "intentVerificationRequired": true  // ← Add this
  }
]
```

### Step 4: Integrate into Issuer

```javascript
// issuer.js
import { INTENT_VERIFICATION_ENABLED } from './config.js';
import { verifyIntent, anyRequireIntentVerification } from './lib/intent-verifier.js';

app.post('/issue', async (req, res) => {
  const { subjectDid, claims } = req.body;

  // ... existing AWMP validation ...

  // Add intent verification
  if (INTENT_VERIFICATION_ENABLED) {
    const needsIntent = anyRequireIntentVerification(claims.scopes);

    if (needsIntent && !claims.intentReasoning) {
      return res.status(428).json({
        error: 'Intent verification required',
        hint: 'Include intentReasoning in claims'
      });
    }

    if (needsIntent && claims.intentReasoning) {
      const result = verifyIntent(
        claims.intentReasoning,
        claims,
        subjectDid
      );

      if (!result.verified) {
        return res.status(403).json({
          error: 'Intent verification failed',
          ...result
        });
      }

      claims.intentVerificationResult = result;
    }
  }

  // ... proceed with VC issuance ...
});
```

### Step 5: Test

```bash
# Enable extension
export ENABLE_INTENT_VERIFICATION=true

# Start issuer
node issuer.js

# Run tests
node test-intent-verifier.js
```

---

## Configuration Options

### Feature Flags

```javascript
// Enable/disable extension globally
ENABLE_INTENT_VERIFICATION=true|false

// Minimum confidence threshold (default: 0.7)
INTENT_VERIFICATION_MIN_CONFIDENCE=0.7

// Strict mode (fail on warnings)
INTENT_VERIFICATION_STRICT_MODE=false
```

### Per-Scope Configuration

```json
{
  "scope": "order:delete",
  "intentVerificationRequired": true,
  "intentVerificationMinConfidence": 0.8  // Optional override
}
```

---

## Migration Guide

### Adding Extension to Existing AWMP Issuer

**Phase 1: Deploy with extension disabled**
```bash
ENABLE_INTENT_VERIFICATION=false node issuer.js
```

**Phase 2: Add configuration**
- Update claims-db.json with `intentVerificationRequired` fields
- Start with `false` for all scopes

**Phase 3: Enable for high-risk scopes**
- Set `intentVerificationRequired: true` for critical operations
- Test with sample agents

**Phase 4: Go live**
```bash
ENABLE_INTENT_VERIFICATION=true node issuer.js
```

### Removing Extension

**Step 1: Disable**
```bash
ENABLE_INTENT_VERIFICATION=false
```

**Step 2: Remove configuration**
- Remove `intentVerificationRequired` from claims-db.json
- Or set all to `false`

**Step 3: Optional cleanup**
- Remove intent-verifier module if not needed
- Remove extension-related code

---

## References

- [AWMP VC Specification](./VC-SPECIFICATION.md) - Base protocol
- [Architecture Guide](./ARCHITECTURE-INTENT-AS-EXTENSION.md) - Design details
- [Implementation Example](./src/intent-verifier.js) - Reference code

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-01-22 | Initial intent verification extension spec |

---

**End of Specification**
