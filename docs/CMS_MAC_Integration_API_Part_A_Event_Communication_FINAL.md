# CMS → MAC Integration API
## PART A — Event Communication API (v1)

---

## 1. Purpose
The Event Communication API enables Medicare Administrative Contractors (MACs) to:
1. Be notified when new provider communication is available in CMS.
2. Identify the associated workflow resource.
3. Retrieve the resource using the appropriate workflow-specific APIs.
4. Acknowledge successful event processing.

This API replaces portal, fax, and batch-file intake with a standardized, secure, event-driven model.

---

## 2. Scope
This API covers provider → CMS → MAC communications including:
- Prior Authorization requests
- Additional documentation submissions
- ADR responses
- Appeals or reopening documentation
- Provider clarifications

This API does NOT define MAC internal workflow or review processes.

---

## 3. Authentication (Assumed)
All endpoints are protected using CMS-approved system-to-system authentication:
- OAuth2 Client Credentials or JWT
---

## 4. Identifier Model
Two identifiers are used:

| Identifier | Purpose |
|------------|---------|
| trackingId | Identifies the communication message |
| resourceIds | Identifies workflow resource tied to the communication |

Example:
{
  "resourceIds": {
    "priorAuthId": "PA-998877"
  }
}

---

## 5. CommunicationType Code Set (v1.0)

| Code | Description |
|------|-------------|
| PRIOR_AUTH_REQUEST | New prior authorization request |
| PRIOR_AUTH_ADDITIONAL_DOCUMENTS | Additional documentation for existing PA |
| ADR_RESPONSE | Response to ADR |
| APPEAL_DOCUMENTATION | Appeal documentation |
| GENERAL_DOCUMENTATION | Non‑PA documentation |
| PROVIDER_CLARIFICATION | Clarification or follow‑up |

---

## 6. Subscription API

### POST /mac/v1/subscriptions

Request:
{
  "macOid": "2.16.840.1.113883.3.249.7.3.1",
  "jurisdictions": ["J5","J8"],
  "programs": ["PART_A","PART_B","DME"],
  "notificationMode": "WEBHOOK",
  "webhookUrl": "https://mac.example.com/cms-events",
  "pollingAllowed": true
}

Response:
{
  "subscriptionId": "sub-12345",
  "status": "ACTIVE"
}

---

## 7. Webhook Event Delivery

Event Envelope:
{
  "eventId": "evt-PA-78910",
  "eventType": "NEW_COMMUNICATION_AVAILABLE",
  "timestamp": "2026-03-01T17:00:00Z",
  "macOid": "2.16.840.1.113883.3.249.7.3.1",
  "communicationType": "PRIOR_AUTH_REQUEST",
  "trackingId": "CMS-COMM-556677",
  "resourceIds": {
    "priorAuthId": "PA-998877"
  }
}

MAC returns HTTP 200 on success.

---

## 8. Polling Events

GET /mac/v1/events?since=timestamp

Response:
{
  "events": [
    {
      "eventId": "evt-PA-78910",
      "communicationType": "PRIOR_AUTH_REQUEST",
      "trackingId": "CMS-COMM-556677",
      "resourceIds": {
        "priorAuthId": "PA-998877"
      }
    }
  ]
}

---

## 9. Event Acknowledgement

POST /mac/v1/events/{eventId}/acknowledgement

Request:
{
  "macOid": "2.16.840.1.113883.3.249.7.3.1",
  "status": "RECEIVED"
}

Response:
{
  "eventId": "evt-PA-78910",
  "status": "ACKNOWLEDGED"
}

---

## 10. Reliability
- At-least-once delivery
- Idempotent event processing
- Polling replay supported

---
End of Part A
