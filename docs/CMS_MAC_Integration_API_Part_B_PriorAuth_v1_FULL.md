# CMS → MAC Integration API  
## PART B — Prior Authorization Interaction API (v1)

---

## 1. Purpose

This document specifies the **MAC ↔ CMS** interaction surface for **Prior Authorization (PA)** requests that originate from providers and are received by CMS through existing HL7 Da Vinci implementation guides (provider-side interactions are out of scope here).

This API enables MACs to:
1. Retrieve a PA request submitted by a provider (intake).
2. Acknowledge receipt into MAC intake (reconciliation).
3. Request additional documentation (trigger CMS-to-provider documentation request workflows).
4. Send high-level status updates (external-facing visibility only).
5. Submit final decisions (line-level adjudication).

> **Note:** This specification intentionally excludes MAC internal workflow/case management design.

---

## 2. Out of Scope

The following are explicitly out of scope:
- MAC internal workflow/case routing and adjudication systems
- MAC portals and internal UI experiences
- CMS consent/legal disclosure governance (handled off-band)
- CMS platform observability, onboarding/certification, and audit/provenance (assumed to exist at platform level)
- Provider-facing interaction details (already implemented using Da Vinci IGs)

---

## 3. Authentication (Assumed)

All endpoints are assumed to be protected by CMS-approved system-to-system security controls 

Authentication and authorization details are outside this specification.

---

## 4. Content Negotiation via `_format`

Every endpoint in PART B supports a `_format` selector that controls the **representation** of the payload while keeping the **API contract and behavior consistent**.

### Supported Formats (v1)

| `_format` | Meaning | Intended Usage |
|---|---|---|
| `FHIR` | Full **Da Vinci PAS** representation | MACs able to consume/produce FHIR bundles/resources |
| `SIMPLE` | Simplified structured JSON/PDF representation | Low-friction adoption; similar to current portal-based thinking |
| `CUSTOM` | MAC-specific schema representation | Preserve existing MAC-specific intake/response mappings |

### Rules
- If `_format` is omitted, default is **FHIR**.
- `_format` applies to both request and response bodies (where applicable).
- Regardless of format, **attachments are retrieved through download links** (never embedded inline as large binaries).

---

## 5. Identifiers and MAC Identity

- `priorAuthId`: CMS identifier for the PA request (primary handle for MAC interactions).
- `trackingId`: CMS tracking identifier correlating to Event Communication (PART A).
- `macOid`: The MAC organization identifier. **Only one MAC identifier is used (OID).**

---

# Endpoint 1 — Retrieve Prior Authorization Request

## 1.1 Purpose

Retrieve a PA request by `priorAuthId` in a representation selected by `_format`.

This endpoint is the MAC equivalent of “download the PA request from a portal,” with optional standards-native (FHIR) delivery.

## 1.2 Endpoint

**GET** `/mac/v1/prior-auth/{priorAuthId}?_format=FHIR|SIMPLE|CUSTOM`

### Path Parameters

| Name | Type | Description |
|---|---|---|
| `priorAuthId` | string | CMS PA identifier |

### Query Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `_format` | string | No | `FHIR` (default), `SIMPLE`, `CUSTOM` |

## 1.3 Common Response Envelope (All Formats)

The response always contains a **common envelope** with routing/tracking metadata only.  
**Patient/beneficiary and code/service details appear only inside the format-specific `payload`.**

```json
{
  "priorAuthId": "PA-998877",
  "trackingId": "CMS-COMM-556677",
  "macOid": "2.16.840.1.113883.3.249.7.3.1",
  "jurisdiction": "J5",
  "program": "PART_B",
  "submittedAt": "2026-03-01T17:00:00Z",
  "format": "FHIR",
  "payload": {}
}
```

## 1.4 Payload — FHIR

FHIR representation is delivered as a **PAS submission bundle** (Da Vinci style).

```json
{
  "bundleDownloadUrl": "/mac/v1/bundles/PA-998877",
  "bundleType": "PAS_SUBMISSION_BUNDLE",
  "fhirVersion": "R4",
  "attachments": [
      {
        "fileName": "progress_notes.pdf",
        "downloadUrl": "/mac/v1/files/223344"
      },
      {
        "fileName": "encounter_note.pdf",
        "downloadUrl": "/mac/v1/files/223355"
      }
    ]
  }
}
```

**Expected contents (illustrative):**
- PAS-related FHIR resources (e.g., Claim / Coverage / supporting clinical context)
- DocumentReference resources referencing Binary attachments
- QuestionnaireResponse resources if applicable
- Any PAS-required correlation references

## 1.5 Payload — SIMPLE

SIMPLE representation provides a structured JSON summary that is easier to ingest for non-FHIR MAC systems with a PDF request like a faxed coversheet which can be used with OCR engines.

```json
{
  "package": {
    "manifestUrl": "/mac/v1/communications/CMS-COMM-556677",
    "pdfRequestUrl": "/mac/v1/files/223367/request.pdf",
    "attachments": [
      {
        "fileName": "progress_notes.pdf",
        "downloadUrl": "/mac/v1/files/223344"
      },
      {
        "fileName": "encounter_note.pdf",
        "downloadUrl": "/mac/v1/files/223355"
      }
    ]
  }
}
```



## 1.6 Payload — CUSTOM

CUSTOM representation points to a MAC-specific payload that CMS produces based on MAC onboarding mapping.

```json
{
  "customPayloadUrl": "/mac/v1/files/custom/2.16.840.1.113883.3.249.7.3.1/PA-998877.json",
  "schemaVersion": "1.0",
  "attachments": [
      {
        "fileName": "progress_notes.pdf",
        "downloadUrl": "/mac/v1/files/223344"
      },
      {
        "fileName": "encounter_note.pdf",
        "downloadUrl": "/mac/v1/files/223355"
      }
    ]
  }
}
```

## 1.7 Behavioral Rules

- Retrieval is **idempotent** and can be repeated.
- Retrieval does **not** require acknowledgement first.
- If CMS re-sends events (PART A) due to retry/reconciliation, retrieval remains stable.

---

# Endpoint 2 — Acknowledge Receipt (Intake Confirmation)

## 2.1 Purpose

Confirm that the MAC has successfully retrieved the PA request and accepted it into MAC intake.  
This is not a workflow state; it is an intake receipt confirmation for reconciliation.

## 2.2 Endpoint

**POST** `/mac/v1/prior-auth/{priorAuthId}/acknowledgement?_format=FHIR|SIMPLE|CUSTOM`

### Path Parameters

| Name | Type | Description |
|---|---|---|
| `priorAuthId` | string | CMS PA identifier |


## 2.3 Request Body (Format-neutral)

```json
{
  "macOid": "2.16.840.1.113883.3.249.7.3.1",
  "acknowledgedAt": "2026-03-01T18:00:00Z",
  "intakeReferenceId": "WPS-INTAKE-445566"
}
```

### Fields

| Field | Required | Description |
|---|---:|---|
| `macOid` | Yes | MAC organization OID |
| `acknowledgedAt` | Yes | Timestamp PA entered MAC intake |
| `intakeReferenceId` | No | MAC internal intake reference (useful for support/tracing) |

## 2.4 Response Example

```json
{
  "status": "ACKNOWLEDGEMENT_RECORDED",
  "priorAuthId": "PA-998877",
  "recordedAt": "2026-03-01T18:00:02Z"
}
```

## 2.5 Behavioral Rules

- Acknowledgement is **idempotent** (retries return success).
- **Status updates and decisions SHOULD follow acknowledgement**.
- CMS may use acknowledgement to stop retrying event delivery (PART A) and for reconciliation.

---

# Endpoint 3 — Request Additional Documentation (ADR-like request)

## 3.1 Purpose

Allow the MAC to request additional documentation required to complete PA review.

CMS will translate this request into provider-facing actions using Da Vinci-aligned workflows (provider side out of scope here). Provider responses return back to MAC via **PART A events** and retrieval.

## 3.2 Endpoint

**POST** `/mac/v1/prior-auth/{priorAuthId}/request-documentation?_format=FHIR|SIMPLE|CUSTOM`

### Path Parameters

| Name | Type | Description |
|---|---|---|
| `priorAuthId` | string | CMS PA identifier |

### Query Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `_format` | string | No | Default `FHIR` |

## 3.3 Common Request Envelope

```json
{
  "macOid": "2.16.840.1.113883.3.249.7.3.1",
  "requestId": "ADR-1122",
  "requestedAt": "2026-03-01T19:10:00Z",
  "dueDate": "2026-03-10",
  "payload": {}
}
```

### Fields

| Field | Required | Description |
|---|---:|---|
| `macOid` | Yes | MAC organization OID |
| `requestId` | Yes | MAC identifier for this documentation request |
| `requestedAt` | Yes | Timestamp request created |
| `dueDate` | No | Requested due date |
| `payload` | Yes | Format-specific request content |

## 3.4 Payload — FHIR

FHIR payload may reference PAS/CDex-aligned request intent. CMS will generate appropriate FHIR Task/Communication (internally).

```json
{
  "reasonCode": "MISSING_FACE_TO_FACE",
  "reasonDisplay": "Face-to-face encounter documentation required"
}
```

## 3.5 Payload — SIMPLE

Simple human-readable request details.

```json
{
  "requestTitle": "Additional Documentation Required",
  "requestDetails": "Please submit face-to-face encounter documentation supporting medical necessity.",
  "documentTypesRequested": [
    "Face-to-face encounter note",
    "Recent progress notes"
  ]
}
```

## 3.6 Payload — CUSTOM

MAC provides or references its own request schema as configured during onboarding.

```json
{
  "customPayloadUrl": "/mac/upload/2.16.840.1.113883.3.249.7.3.1/ADR-1122.json",
  "schemaVersion": "1.0"
}
```

## 3.7 Response Example

```json
{
  "status": "DOCUMENTATION_REQUEST_ACCEPTED",
  "priorAuthId": "PA-998877",
  "requestId": "ADR-1122",
  "recordedAt": "2026-03-01T19:10:02Z"
}
```

## 3.8 Behavioral Rules

- Multiple documentation requests are allowed.
- Documentation requests are **not allowed after final decision**.
- Provider responses will generate **new PART A events**, which the MAC retrieves via communications endpoints.

---

# Endpoint 4 — Status Updates (High-level only)

## 4.1 Purpose

Allow the MAC to communicate high-level status of PA processing to CMS for external visibility and tracking.

This is not MAC workflow. It is a simple externally relevant status indicator.

## 4.2 Endpoint

**POST** `/mac/v1/prior-auth/{priorAuthId}/status?_format=FHIR|SIMPLE|CUSTOM`

### Status Codes (v1)

| Status | Meaning |
|---|---|
| `RECEIVED` | Request acknowledged by MAC |
| `IN_REVIEW` | Under active review |
| `PENDING_DOCUMENTATION` | Awaiting additional documentation |
| `ON_HOLD` | Temporarily suspended |
| `CANCELLED` | Cancelled before decision |

## 4.3 Request Body

```json
{
  "macOid": "2.16.840.1.113883.3.249.7.3.1",
  "status": "IN_REVIEW",
  "updatedAt": "2026-03-01T20:15:00Z",
  "statusReason": "Clinical documentation under evaluation"
}
```

### Fields

| Field | Required | Description |
|---|---:|---|
| `macOid` | Yes | MAC organization OID |
| `status` | Yes | One of the defined status codes |
| `updatedAt` | Yes | Timestamp of the status update |
| `statusReason` | No | Human-readable reason/context |

## 4.4 Response Example

```json
{
  "status": "STATUS_RECORDED",
  "priorAuthId": "PA-998877",
  "recordedAt": "2026-03-01T20:15:02Z"
}
```

## 4.5 Behavioral Rules

- Status updates SHOULD follow acknowledgement, but CMS should accept late updates for resiliency.
- CMS should apply minimal validation (e.g., no status updates after final decision).
- Status transitions are not strictly enforced to avoid constraining MAC internal systems.

---

# Endpoint 5 — Final Decision (Line-level Adjudication)

## 5.1 Purpose

Submit the final decision for a PA request. Decisions are expressed at the **requested code/service** level and can include MAC-defined adjudication reason codes.

## 5.2 Endpoint

**POST** `/mac/v1/prior-auth/{priorAuthId}/decision?_format=FHIR|SIMPLE|CUSTOM`

## 5.3 Decision Codes (v1)

| Decision | Meaning |
|---|---|
| `AFFIRMED` | All requested services affirmed |
| `PARTIAL_AFFIRMATION` | Some affirmed, some non-affirmed |
| `NON_AFFIRMED` | No services affirmed |
| `REJECTED` | Administrative/technical rejection (no adjudication) |
| `CANCELLED` | Cancelled without adjudication |

> **Administrative failures** resulting in inability to adjudicate should use `REJECTED`.

## 5.4 Common Request Envelope

```json
{
  "macOid": "2.16.840.1.113883.3.249.7.3.1",
  "decision": "PARTIAL_AFFIRMATION",
  "decisionDate": "2026-03-02",
  "payload": {}
}
```

## 5.5 Payload — SIMPLE (Line-level)

SIMPLE payload expresses decisions per requested code/service.

```json
{
  "serviceDecisions": [
    {
      "codeSystem": "HCPCS",
      "code": "E1390",
      "decision": "AFFIRMED",
      "validityPeriodDays": 90
    },
    {
      "codeSystem": "HCPCS",
      "code": "A9999",
      "decision": "NON_AFFIRMED",
      "reasonCode": "MAC_REASON_CODE",
      "reasonText": "See MAC adjudication reason code set"
    }
  ],
  "overallNotes": "Primary device affirmed. Accessories non-affirmed."
}
```

### Reason Codes
- `reasonCode` values are drawn from **existing MAC adjudication reason code sets**.
- CMS will reference and validate against the MAC’s configured reason code set where applicable.

## 5.6 Payload — FHIR (Full Da Vinci PAS ClaimResponse)

FHIR payload contains the **entire Da Vinci PAS-style ClaimResponse** (as a resource or bundle).

Two supported patterns (v1):

### Pattern A — Inline FHIR Bundle (preferred when payload sizes allow)

```json
{
  "claimResponseBundle": {
    "resourceType": "Bundle",
    "type": "collection",
    "entry": [
      {
        "resource": {
          "resourceType": "ClaimResponse"
          /* Full Da Vinci PAS ClaimResponse content here */
        }
      }
    ]
  }
}
```

### Pattern B — Bundle by Reference (preferred for large payloads)

```json
{
  "claimResponseBundleUrl": "/mac/v1/bundles/PA-998877-claimresponse",
  "fhirVersion": "R4",
  "profile": "davinci-pas-claimresponse"
}
```

> CMS will translate the submitted ClaimResponse into provider-facing PAS response interactions.

## 5.7 Payload — CUSTOM

MAC provides or references its own decision schema as configured during onboarding.

```json
{
  "customPayloadUrl": "/mac/upload/2.16.840.1.113883.3.249.7.3.1/decisions/PA-998877.json",
  "schemaVersion": "1.0"
}
```

## 5.8 Response Example

```json
{
  "status": "DECISION_RECORDED",
  "priorAuthId": "PA-998877",
  "recordedAt": "2026-03-02T10:15:02Z"
}
```

## 5.9 Behavioral Rules

- **Decision closes lifecycle**: after decision, no further status updates or documentation requests are allowed.
- Decision submission is **idempotent** (retries succeed).
- `REJECTED` indicates administrative/technical rejection and should include `overallNotes` or appropriate details in the payload.
- Line-level decisions should align with the originally requested services/codes.

---

## 6. Attachment Handling (All Formats)

- Attachments are never embedded as large binary content in these payloads.
- Attachments are provided as **download URLs** with appropriate authorization controls.
- The communications/package manifest (referenced from Endpoint 1 SIMPLE payload, and PART A communications retrieval) lists attachments consistently.

---

## 7. End-to-End Loop Summary (How it fits together)

1. Provider submits PA → CMS receives (provider side via Da Vinci IGs).  
2. CMS notifies MAC (PART A).  
3. MAC retrieves PA via **GET /prior-auth/{priorAuthId}**.  
4. MAC acknowledges receipt via **/acknowledgement**.  
5. If needed, MAC requests documentation via **/request-documentation**.  
6. Provider responds → CMS emits a new PART A event → MAC retrieves new documentation.  
7. MAC sends status updates via **/status** (optional).  
8. MAC submits final decision via **/decision** → CMS responds to provider via PAS.

---

**End of PART B — Prior Authorization Interaction API (v1)**
