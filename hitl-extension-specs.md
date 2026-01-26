# HITL (Human-in-the-Loop) Extension Specification

## Version: 1.0.0
## Date: 2026-01-23
## Extends: AWMP VC Specification v1.0.0

---

## Table of Contents

1. [Overview](#overview)
2. [Extension Architecture](#extension-architecture)
3. [HITL Request Structure](#hitl-request-structure)
4. [Notification Channels](#notification-channels)
5. [Configuration](#configuration)
6. [Integration Guide](#integration-guide)
7. [Examples](#examples)
8. [Security Considerations](#security-considerations)

---

## Overview

The HITL (Human-in-the-Loop) Extension is an **optional enhancement** for AWMP VC issuers that extends the basic web UI approval workflow with external notification capabilities.

### Purpose

This extension allows VC issuers to:
- Send approval requests to external systems via webhooks
- Notify approvers through Slack messages
- Send email notifications for approval requests
- Integrate HITL approvals with existing workflows
- Maintain audit trails across multiple notification channels

### Relationship to AWMP

```
┌──────────────────────────────────────────────┐
│         AWMP VC Specification                │
│         (Required - Base Protocol)           │
│  ─────────────────────────────────────       │
│  • DID-based authentication                  │
│  • Scope-based authorization                 │
│  • VC signing and verification               │
│  • Permissions database (hitl field)         │
└──────────────┬───────────────────────────────┘
               │
               │ Extends with approval workflows
               ▼
┌──────────────────────────────────────────────┐
│    Basic HITL (Issuer Web UI)                │
│    (Built-in - Simple approval)              │
│  ─────────────────────────────────────       │
│  • Web-based approval interface              │
│  • Manual approve/decline                    │
│  • Password protection                       │
└──────────────┬───────────────────────────────┘
               │
               │ Extends with notification channels
               ▼
┌──────────────────────────────────────────────┐
│    HITL Extension                            │
│    (Optional - External Notifications)       │
│  ─────────────────────────────────────       │
│  • Webhook notifications                     │
│  • Slack integration                         │
│  • Email notifications                       │
│  • Custom approval workflows                 │
└──────────────────────────────────────────────┘
```

### When to Use This Extension

**Recommended for:**
- ✅ Remote teams needing notifications in Slack/Email
- ✅ Integration with external approval systems
- ✅ Automated approval workflows
- ✅ Multi-channel notification requirements
- ✅ Audit trails in external systems

**Not necessary for:**
- ⚪ Single-user deployments
- ⚪ Always-available web UI monitors
- ⚪ Fully automated systems (no human approval)
- ⚪ Simple development environments

---

## Extension Architecture

### Modular Design

This extension is designed as a **pluggable notification layer** that:
- Can be added to existing AWMP implementations with basic HITL
- Can be removed without breaking core functionality
- Can be enabled/disabled per notification channel
- Has no dependencies on core AWMP beyond HITL approval hooks

### Integration Points

```javascript
// issuer.js with HITL extension

import { requestHumanApproval } from './hitl-core.js';  // Basic HITL
import { notifyHITLRequest } from './hitl-notifier.js';  // Extension module

app.post('/issue', async (req, res) => {
  const { subjectDid, claims } = req.body;

  // 1. Core AWMP validation (always required)
  const validation = validateClaims(claims);
  const authorized = checkAgentPermission(subjectDid, claims);

  // 2. Check if HITL is required (from permissions DB)
  if (hitlRequired) {
    // 3. Basic HITL: Web UI approval (always available)
    const approvalPromise = requestHumanApproval(subjectDid, claims, reason);

    // 4. Optional: HITL Extension - Send notifications
    if (HITL_NOTIFICATIONS_ENABLED) {
      await notifyHITLRequest({
        requestId: approvalRequest.requestId,
        subjectDid,
        agentName: claims.agentName,
        scopes: claims.scopes,
        reason,
        approvalUrl: `${BASE_URL}/issuer.html#approval-${requestId}`
      });
    }

    // Wait for approval (from web UI or external system)
    const approval = await approvalPromise;

    if (!approval.approved) {
      return res.status(403).json({ error: approval.error });
    }
  }

  // 5. Issue VC
  const vcJwt = await issueAWMPCredential(subjectDid, claims);
  res.json({ vcJwt });
});
```

---

## HITL Request Structure

### Approval Request Object

```typescript
interface HITLApprovalRequest {
  requestId: string;           // Unique identifier
  timestamp: string;           // ISO 8601 timestamp
  subjectDid: string;          // Agent DID requesting access
  agentName: string;           // Agent name
  scopes: string[];            // Requested scopes
  target: string;              // Target resource
  reason: string;              // Why HITL is required
  intentReasoning?: {          // Optional intent verification data
    taskContext: string;
    reasoningSteps: string[];
    confidence: number;
  };
  approvalUrl: string;         // URL to approve in web UI
  status: 'pending' | 'approved' | 'declined';
}
```

### Notification Payload

```typescript
interface HITLNotificationPayload {
  requestId: string;
  subjectDid: string;
  agentName: string;
  scopes: string[];
  reason: string;
  approvalUrl: string;
  timestamp: string;
  expiresAt?: string;          // Optional expiration
  metadata?: {                 // Optional metadata
    intentReasoning?: object;
    userContext?: string;
  };
}
```

---

## Notification Channels

### 1. Webhook Notifications

Send HITL approval requests to external HTTP endpoints.

**Configuration:**
```json
{
  "webhook": {
    "enabled": true,
    "url": "https://api.example.com/approvals",
    "method": "POST",
    "headers": {
      "Authorization": "Bearer YOUR_API_KEY",
      "Content-Type": "application/json"
    },
    "timeout": 5000,
    "retries": 3
  }
}
```

**Request Format:**
```http
POST https://api.example.com/approvals
Content-Type: application/json
Authorization: Bearer YOUR_API_KEY

{
  "event": "hitl_approval_requested",
  "requestId": "req_123",
  "timestamp": "2026-01-23T10:00:00.000Z",
  "data": {
    "agentDid": "did:key:z6Mk...",
    "agentName": "claude-code-agent",
    "scopes": ["order:delete"],
    "reason": "HITL required for scope: order:delete",
    "approvalUrl": "http://localhost:3001/issuer.html#approval-req_123"
  }
}
```

**Response Handling:**
- **200-299**: Notification successful
- **400-499**: Configuration error (logged, notification marked failed)
- **500-599**: Temporary error (retried based on `retries` config)
- **Timeout**: Logged, marked as failed

---

### 2. Slack Notifications

Send approval requests as Slack messages with action buttons.

**Configuration:**
```json
{
  "slack": {
    "enabled": true,
    "webhookUrl": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
    "channel": "#approvals",
    "username": "AWMP Approval Bot",
    "iconEmoji": ":lock:",
    "mentionUsers": ["@admin", "@security-team"]
  }
}
```

**Message Format:**
```json
{
  "channel": "#approvals",
  "username": "AWMP Approval Bot",
  "icon_emoji": ":lock:",
  "text": "🔐 *HITL Approval Required*",
  "attachments": [
    {
      "color": "warning",
      "fields": [
        {
          "title": "Agent",
          "value": "claude-code-agent",
          "short": true
        },
        {
          "title": "Requested Scopes",
          "value": "order:delete",
          "short": true
        },
        {
          "title": "Reason",
          "value": "HITL required for scope: order:delete",
          "short": false
        },
        {
          "title": "Request ID",
          "value": "req_123",
          "short": true
        }
      ],
      "actions": [
        {
          "type": "button",
          "text": "Approve in Web UI",
          "url": "http://localhost:3001/issuer.html#approval-req_123",
          "style": "primary"
        }
      ],
      "footer": "AWMP HITL Extension",
      "ts": 1737633600
    }
  ]
}
```

**Features:**
- Rich formatting with color-coded attachments
- Action button linking to approval UI
- Optional user mentions for urgent requests
- Custom bot name and icon

---

### 3. Email Notifications

Send email notifications to designated approvers.

**Configuration:**
```json
{
  "email": {
    "enabled": true,
    "provider": "smtp",
    "smtp": {
      "host": "smtp.gmail.com",
      "port": 587,
      "secure": false,
      "auth": {
        "user": "approvals@example.com",
        "pass": "your-app-password"
      }
    },
    "from": "AWMP Approvals <approvals@example.com>",
    "to": ["admin@example.com", "security@example.com"],
    "subject": "HITL Approval Required - {{agentName}}",
    "template": "html"
  }
}
```

**Email Template (HTML):**
```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Arial, sans-serif; line-height: 1.6; }
    .header { background: #f4f4f4; padding: 20px; }
    .content { padding: 20px; }
    .button {
      background: #007bff;
      color: white;
      padding: 10px 20px;
      text-decoration: none;
      border-radius: 5px;
      display: inline-block;
    }
    .info { background: #fff3cd; padding: 15px; border-radius: 5px; }
  </style>
</head>
<body>
  <div class="header">
    <h2>🔐 HITL Approval Required</h2>
  </div>
  <div class="content">
    <p>A new approval request requires your attention.</p>

    <div class="info">
      <p><strong>Agent Name:</strong> {{agentName}}</p>
      <p><strong>Agent DID:</strong> {{subjectDid}}</p>
      <p><strong>Requested Scopes:</strong> {{scopes}}</p>
      <p><strong>Reason:</strong> {{reason}}</p>
      <p><strong>Request ID:</strong> {{requestId}}</p>
      <p><strong>Timestamp:</strong> {{timestamp}}</p>
    </div>

    <p>
      <a href="{{approvalUrl}}" class="button">Review and Approve</a>
    </p>

    <p style="color: #666; font-size: 12px;">
      This is an automated notification from AWMP HITL Extension.
    </p>
  </div>
</body>
</html>
```

**Email Template (Plain Text):**
```
🔐 HITL APPROVAL REQUIRED

A new approval request requires your attention.

Agent Name: {{agentName}}
Agent DID: {{subjectDid}}
Requested Scopes: {{scopes}}
Reason: {{reason}}
Request ID: {{requestId}}
Timestamp: {{timestamp}}

Review and Approve: {{approvalUrl}}

---
This is an automated notification from AWMP HITL Extension.
```

---

## Configuration

### Configuration File Structure

**`hitl-config.json`:**
```json
{
  "enabled": true,
  "channels": {
    "webhook": {
      "enabled": false,
      "url": "",
      "method": "POST",
      "headers": {},
      "timeout": 5000,
      "retries": 3
    },
    "slack": {
      "enabled": false,
      "webhookUrl": "",
      "channel": "#approvals",
      "username": "AWMP Approval Bot",
      "iconEmoji": ":lock:",
      "mentionUsers": []
    },
    "email": {
      "enabled": false,
      "provider": "smtp",
      "smtp": {
        "host": "",
        "port": 587,
        "secure": false,
        "auth": {
          "user": "",
          "pass": ""
        }
      },
      "from": "",
      "to": [],
      "subject": "HITL Approval Required - {{agentName}}",
      "template": "html"
    }
  },
  "defaults": {
    "timeout": 300000,
    "notifyOnApproval": true,
    "notifyOnDecline": true,
    "includeIntentReasoning": true
  }
}
```

### Environment Variables

```bash
# Enable HITL Extension
HITL_NOTIFICATIONS_ENABLED=true

# Webhook
HITL_WEBHOOK_URL=https://api.example.com/approvals
HITL_WEBHOOK_TOKEN=your-api-token

# Slack
HITL_SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
HITL_SLACK_CHANNEL=#approvals

# Email (SMTP)
HITL_EMAIL_HOST=smtp.gmail.com
HITL_EMAIL_PORT=587
HITL_EMAIL_USER=approvals@example.com
HITL_EMAIL_PASS=your-app-password
HITL_EMAIL_FROM=approvals@example.com
HITL_EMAIL_TO=admin@example.com,security@example.com
```

### Feature Flags

```javascript
// config.js
export const HITL_NOTIFICATIONS_ENABLED =
  process.env.HITL_NOTIFICATIONS_ENABLED === 'true';

export const HITL_WEBHOOK_ENABLED =
  process.env.HITL_WEBHOOK_URL ? true : false;

export const HITL_SLACK_ENABLED =
  process.env.HITL_SLACK_WEBHOOK_URL ? true : false;

export const HITL_EMAIL_ENABLED =
  process.env.HITL_EMAIL_HOST ? true : false;
```

---

## Integration Guide

### Step 1: Install Dependencies

```bash
npm install node-fetch nodemailer
```

### Step 2: Create HITL Notifier Module

```bash
cp hitl-notifier.js ./lib/
```

### Step 3: Configure Notification Channels

Create `hitl-config.json` or set environment variables:

```bash
export HITL_NOTIFICATIONS_ENABLED=true
export HITL_SLACK_WEBHOOK_URL=https://hooks.slack.com/...
export HITL_SLACK_CHANNEL=#approvals
```

### Step 4: Integrate into Issuer

```javascript
// issuer.js
import { notifyHITLRequest, notifyHITLResponse } from './lib/hitl-notifier.js';
import { HITL_NOTIFICATIONS_ENABLED } from './config.js';

function requestHumanApproval(subjectDid, claims, reason) {
  return new Promise((resolve, reject) => {
    const requestId = String(nextRequestId++);

    const approvalRequest = {
      requestId,
      timestamp: new Date().toISOString(),
      subjectDid,
      agentName: claims.agentName,
      scopes: claims.scopes,
      reason,
      approvalUrl: `${BASE_URL}/issuer.html#approval-${requestId}`,
      status: 'pending'
    };

    pendingApprovals.set(requestId, { request: approvalRequest, resolve, reject });

    // Send notifications via extension
    if (HITL_NOTIFICATIONS_ENABLED) {
      notifyHITLRequest(approvalRequest).catch(err => {
        console.error('Failed to send HITL notifications:', err);
      });
    }

    // Set timeout
    setTimeout(() => {
      if (pendingApprovals.has(requestId)) {
        pendingApprovals.delete(requestId);
        resolve({ approved: false, error: "Approval timed out" });
      }
    }, 5 * 60 * 1000);
  });
}

// When approval is processed
app.post("/api/approvals/:requestId/approve", async (req, res) => {
  // ... existing approval logic ...

  // Send approval notification
  if (HITL_NOTIFICATIONS_ENABLED) {
    await notifyHITLResponse(requestId, 'approved');
  }

  res.json({ success: true });
});
```

### Step 5: Test

```bash
# Enable extension
export HITL_NOTIFICATIONS_ENABLED=true
export HITL_SLACK_WEBHOOK_URL=your-webhook-url

# Start issuer
node issuer.js

# Trigger HITL request (should send Slack notification)
node test-hitl-extension.js
```

---

## Examples

### Example 1: Webhook Only

**Configuration:**
```bash
export HITL_NOTIFICATIONS_ENABLED=true
export HITL_WEBHOOK_URL=https://api.example.com/approvals
export HITL_WEBHOOK_TOKEN=secret-token
```

**Request Flow:**
1. Agent requests VC with `order:delete` scope
2. Permissions DB marks scope as `hitl: true`
3. Issuer creates approval request
4. **HITL Extension sends webhook:**
   ```http
   POST https://api.example.com/approvals
   Authorization: Bearer secret-token

   {
     "event": "hitl_approval_requested",
     "requestId": "req_123",
     "data": { ... }
   }
   ```
5. Approver visits web UI via `approvalUrl`
6. Approves request
7. VC is issued

---

### Example 2: Slack + Email

**Configuration:**
```bash
export HITL_NOTIFICATIONS_ENABLED=true
export HITL_SLACK_WEBHOOK_URL=https://hooks.slack.com/...
export HITL_SLACK_CHANNEL=#security-approvals
export HITL_EMAIL_HOST=smtp.gmail.com
export HITL_EMAIL_USER=approvals@company.com
export HITL_EMAIL_PASS=app-password
export HITL_EMAIL_TO=security@company.com,admin@company.com
```

**Result:**
- Slack message posted to #security-approvals channel
- Email sent to security@company.com and admin@company.com
- Both contain approval link to web UI

---

### Example 3: All Channels

**Configuration via `hitl-config.json`:**
```json
{
  "enabled": true,
  "channels": {
    "webhook": {
      "enabled": true,
      "url": "https://workflow.company.com/approvals"
    },
    "slack": {
      "enabled": true,
      "webhookUrl": "https://hooks.slack.com/...",
      "channel": "#approvals"
    },
    "email": {
      "enabled": true,
      "to": ["admin@company.com"]
    }
  }
}
```

**Result:**
- Webhook sent to workflow system
- Slack message posted
- Email sent to admin
- All notifications contain same `requestId` for correlation

---

## Security Considerations

### Authentication

1. **Webhook Authentication:**
   - Use Bearer tokens or API keys
   - Rotate credentials regularly
   - Use HTTPS only

2. **Email Security:**
   - Use app-specific passwords
   - Enable 2FA on email accounts
   - Use TLS/STARTTLS

3. **Slack Security:**
   - Protect webhook URLs (treat as secrets)
   - Use Slack app tokens instead of webhooks for production
   - Validate callback requests

### Data Privacy

1. **Sensitive Information:**
   - Don't include DID private keys in notifications
   - Consider redacting sensitive claim data
   - Use secure channels only

2. **Logging:**
   - Log notification attempts
   - Don't log credentials
   - Sanitize error messages

3. **Approval URLs:**
   - Use short-lived tokens in URLs
   - Implement CSRF protection
   - Require authentication

### Rate Limiting

```javascript
// Prevent notification spam
const notificationRateLimiter = {
  maxPerMinute: 10,
  maxPerHour: 100
};
```

### Retry Logic

```javascript
// Exponential backoff for failed notifications
const retryConfig = {
  attempts: 3,
  backoff: 'exponential',
  initialDelay: 1000,
  maxDelay: 10000
};
```

---

## Monitoring and Observability

### Metrics to Track

```javascript
{
  "hitl_notifications_sent": {
    "channel": "slack|email|webhook",
    "status": "success|failure",
    "count": 42
  },
  "hitl_notification_latency_ms": {
    "channel": "slack",
    "p50": 150,
    "p95": 300,
    "p99": 500
  },
  "hitl_approval_time_seconds": {
    "p50": 60,
    "p95": 300,
    "p99": 600
  }
}
```

### Logging

```javascript
// Structured logs for HITL notifications
{
  "timestamp": "2026-01-23T10:00:00.000Z",
  "event": "hitl_notification_sent",
  "requestId": "req_123",
  "channel": "slack",
  "status": "success",
  "latencyMs": 245,
  "metadata": {
    "agentName": "claude-code-agent",
    "scopes": ["order:delete"]
  }
}
```

---

## Migration Guide

### Adding Extension to Existing AWMP Issuer

**Phase 1: Deploy with extension disabled**
```bash
HITL_NOTIFICATIONS_ENABLED=false node issuer.js
```

**Phase 2: Configure one channel**
- Start with email or Slack
- Test with non-production approvals

**Phase 3: Enable for production**
```bash
HITL_NOTIFICATIONS_ENABLED=true node issuer.js
```

**Phase 4: Add additional channels**
- Enable webhook integration
- Configure all desired channels

### Removing Extension

**Step 1: Disable**
```bash
HITL_NOTIFICATIONS_ENABLED=false
```

**Step 2: Remove configuration**
- Delete `hitl-config.json`
- Clear environment variables

**Step 3: Optional cleanup**
- Remove hitl-notifier module
- Remove extension-related code

**Core HITL (web UI) continues working!**

---

## References

- [AWMP VC Specification](./VC-SPECIFICATION.md) - Base protocol
- [Slack Incoming Webhooks](https://api.slack.com/messaging/webhooks)
- [Nodemailer Documentation](https://nodemailer.com/)
- [Webhook Best Practices](https://webhooks.fyi/)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-01-23 | Initial HITL extension specification |

---

**End of Specification**
