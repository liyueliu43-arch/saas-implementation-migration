# API Field Mapping & JSON Payload Specification

> **Project**: Enterprise SaaS Implementation & Data Migration Governance
> **Integration**: Legacy CRM → NexaCloud Cloud CRM (REST API v2)
> **Endpoint Base URL**: `https://api.crm.nexacloud.io/v2`
> **Author**: Liyue Liu (@liyueliu43-arch)
> **Version**: 1.0.0 | Last Updated: 2026-11-25
> **Specification Status**: ✅ Approved for Implementation

---

## 1. Overview

This document defines the authoritative field mapping between the **source legacy CRM system**
and the **target NexaCloud Cloud CRM API**, including data types, transformation rules,
validation constraints, and example payloads. It is the single source of truth for all
integration development, QA testing, and cutover activities.

### Integration Pattern
```
Source: Legacy On-Premise CRM (CSV export / webhook event)
  ↓  [ETL / n8n formatter node]
Target: NexaCloud Cloud CRM REST API v2 (JSON POST)
```

---

## 2. Authentication

| Parameter | Value |
|-----------|-------|
| **Method** | Bearer Token (OAuth 2.0 Client Credentials) |
| **Header** | `Authorization: Bearer <token>` |
| **Token Endpoint** | `POST https://auth.nexacloud.io/oauth/token` |
| **Token Lifetime** | 3600 seconds (1 hour) |
| **Scopes Required** | `crm:write crm:accounts crm:opportunities` |
| **Rotation Policy** | Tokens must be refreshed before expiry; never hardcode in code |

**Token Request Example:**
```http
POST https://auth.nexacloud.io/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={{CLIENT_ID}}
&client_secret={{CLIENT_SECRET}}
&scope=crm:write crm:accounts crm:opportunities
```

**Token Response:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "crm:write crm:accounts crm:opportunities"
}
```

---

## 3. Primary Endpoint: Create Account + Opportunity

```
POST /v2/accounts/create-with-opportunity
Content-Type: application/json
Authorization: Bearer <token>
X-Request-ID: <uuid>           (idempotency key — use n8n execution ID)
X-Migration-Batch: NEXACLOUD_2026_11
```

---

## 4. Field Mapping Table

### 4.1 Account Object

| Source Field (Legacy CSV) | Target Field (CRM API) | Data Type | Transformation Rule | Required | Validation Constraint |
|---------------------------|------------------------|-----------|--------------------|---------|-----------------------|
| `Company_Name` | `account.name` | `string` | Trim whitespace; title-case if all-caps | ✅ Yes | Min 2, Max 255 chars |
| `Country` | `account.billing_country` | `string` | Normalise to ISO 3166-1 alpha-2 (e.g. `USA` → `US`) | ✅ Yes | Must be valid ISO code |
| `Phone_Raw` | `account.phone` | `string` | Normalise to E.164 format (e.g. `+14158882301`) | ⬜ No | Regex: `^\+[1-9]\d{7,14}$` |
| *(derived)* | `account.type` | `string` | Hardcode: `"Customer"` | ✅ Yes | Enum: `Customer`, `Prospect`, `Partner` |
| *(derived)* | `account.industry` | `string` | Hardcode: `"Technology"` unless known | ⬜ No | CRM picklist value |
| `Status` | `account.status` | `string` | Map: `Active`→`Active`, `Churned`→`Inactive`, `Trial`→`Trial` | ✅ Yes | Enum: `Active`, `Inactive`, `Trial` |

### 4.2 Contact Object

| Source Field (Legacy CSV) | Target Field (CRM API) | Data Type | Transformation Rule | Required | Validation Constraint |
|---------------------------|------------------------|-----------|--------------------|---------|-----------------------|
| `Contact_Email` | `contact.email` | `string` | Lowercase, trim; validate RFC-5322 | ✅ Yes | Regex RFC-5322; unique in CRM |
| `Phone_Raw` | `contact.phone` | `string` | Same E.164 normalisation as account | ⬜ No | Regex: `^\+[1-9]\d{7,14}$` |
| `Company_Name` | `contact.account_name` | `string` | Must match `account.name` exactly | ✅ Yes | Foreign key linkage |
| *(derived)* | `contact.role` | `string` | Default: `"Primary Contact"` | ⬜ No | CRM picklist value |

### 4.3 Opportunity Object

| Source Field (Legacy CSV) | Target Field (CRM API) | Data Type | Transformation Rule | Required | Validation Constraint |
|---------------------------|------------------------|-----------|--------------------|---------|-----------------------|
| `Company_Name` + `Plan_Tier` | `opportunity.name` | `string` | Template: `"{Company} — {Plan} Plan"` | ✅ Yes | Max 255 chars |
| `Annual_Contract_USD` | `opportunity.amount` | `float` | Strip `$`, `€`, `,`, text; parse float; must be ≥ 0 | ✅ Yes | `amount > 0` for non-Trial |
| `Plan_Tier` | `opportunity.plan_tier` | `string` | Pass-through; validate against picklist | ✅ Yes | Enum: `Starter`, `Professional`, `Enterprise` |
| `Signup_Date` | `opportunity.close_date` | `string` | Normalise to ISO 8601: `YYYY-MM-DD` | ✅ Yes | Valid date; not in future (warn if >today) |
| *(derived)* | `opportunity.stage` | `string` | Hardcode: `"Closed Won"` for migration | ✅ Yes | Enum per CRM stage definition |
| *(derived)* | `opportunity.type` | `string` | Hardcode: `"New Business"` | ⬜ No | Enum: `New Business`, `Expansion`, `Renewal` |
| *(derived)* | `opportunity.lead_source` | `string` | Hardcode: `"API Integration"` | ⬜ No | CRM picklist value |
| *(derived)* | `opportunity.migration_batch` | `string` | Hardcode: `"NEXACLOUD_2026_11"` | ✅ Yes | Used for post-migration SQL reconciliation |

---

## 5. Status Field Mapping (Enum Translation)

| Legacy `Status` Value | Target CRM `account.status` | Notes |
|-----------------------|-----------------------------|-------|
| `Active` | `Active` | Direct mapping |
| `Churned` | `Inactive` | Customer has cancelled |
| `Trial` | `Trial` | Still in evaluation period |
| `Prospect` | `Prospect` | Pre-sale; not yet customer |
| *(blank / unknown)* | `Trial` | Default to Trial if unset |

---

## 6. Plan Tier Mapping

| Legacy `Plan_Tier` Value | Target CRM `plan_tier` | ARR Range (Guidance) |
|--------------------------|------------------------|----------------------|
| `Starter` | `Starter` | $0 – $10,000/yr |
| `Professional` | `Professional` | $10,001 – $50,000/yr |
| `Enterprise` | `Enterprise` | $50,001+/yr |

> ⚠️ **Business Rule**: If `plan_tier = Starter` and `annual_contract_usd > 10000`, flag for
> manual review. This indicates a potential data entry error in the legacy system.

---

## 7. Standard Request Payload (JSON)

### 7.1 Valid Request Example (Happy Path)

```http
POST /v2/accounts/create-with-opportunity
Host: api.crm.nexacloud.io
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
X-Request-ID: exec-n8n-20261125-001
X-Migration-Batch: NEXACLOUD_2026_11
```

```json
{
  "account": {
    "name": "Meridian Health Partners",
    "billing_country": "GB",
    "phone": "+442079460845",
    "type": "Customer",
    "industry": "Healthcare",
    "status": "Active"
  },
  "contact": {
    "email": "r.patel@meridianhealth.co.uk",
    "phone": "+442079460845",
    "account_name": "Meridian Health Partners",
    "role": "Primary Contact"
  },
  "opportunity": {
    "name": "Meridian Health Partners — Enterprise Plan",
    "stage": "Closed Won",
    "close_date": "2024-03-10",
    "amount": 95000.00,
    "plan_tier": "Enterprise",
    "type": "New Business",
    "lead_source": "API Integration",
    "migration_batch": "NEXACLOUD_2026_11"
  },
  "metadata": {
    "source_system": "Legacy CRM Migration",
    "submitted_at": "2026-11-25T06:16:00Z",
    "sync_version": "1.0.0"
  }
}
```

### 7.2 Success Response (HTTP 201 Created)

```json
{
  "status": "success",
  "http_code": 201,
  "data": {
    "id": "acc_7f3a92b1c4d5e6f7",
    "account": {
      "id": "acc_7f3a92b1c4d5e6f7",
      "name": "Meridian Health Partners",
      "billing_country": "GB",
      "created_at": "2026-11-25T06:16:01Z"
    },
    "contact": {
      "id": "con_1a2b3c4d5e6f7890",
      "email": "r.patel@meridianhealth.co.uk",
      "created_at": "2026-11-25T06:16:01Z"
    },
    "opportunity": {
      "id": "opp_a1b2c3d4e5f6a7b8",
      "name": "Meridian Health Partners — Enterprise Plan",
      "amount": 95000.00,
      "stage": "Closed Won",
      "created_at": "2026-11-25T06:16:01Z"
    }
  },
  "request_id": "exec-n8n-20261125-001",
  "migration_batch": "NEXACLOUD_2026_11"
}
```

---

## 8. Error Response Catalogue

### 8.1 HTTP 400 Bad Request — Malformed JSON

```json
{
  "status": "error",
  "http_code": 400,
  "error": {
    "code": "MALFORMED_REQUEST",
    "message": "Request body is not valid JSON.",
    "detail": "Unexpected token at line 7, column 3"
  },
  "request_id": "exec-n8n-20261125-002"
}
```

### 8.2 HTTP 401 Unauthorized — Invalid or Expired Token

```json
{
  "status": "error",
  "http_code": 401,
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "The Bearer token has expired or is invalid.",
    "detail": "Token expiry: 2026-11-25T05:00:00Z. Current time: 2026-11-25T06:16:00Z.",
    "resolution": "Re-authenticate via POST /oauth/token and retry with new token."
  },
  "request_id": "exec-n8n-20261125-003"
}
```

### 8.3 HTTP 422 Unprocessable Entity — Field Validation Failure

```json
{
  "status": "error",
  "http_code": 422,
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "One or more fields failed validation.",
    "validation_errors": [
      {
        "field": "contact.email",
        "value": "invalid-email@",
        "rule": "RFC_5322_FORMAT",
        "message": "Email address does not conform to RFC-5322 standard."
      },
      {
        "field": "opportunity.amount",
        "value": -500,
        "rule": "POSITIVE_NUMBER",
        "message": "Contract amount must be greater than 0."
      }
    ]
  },
  "request_id": "exec-n8n-20261125-004"
}
```

### 8.4 HTTP 429 Too Many Requests — Rate Limit Exceeded

```json
{
  "status": "error",
  "http_code": 429,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "API rate limit exceeded. Maximum 100 requests per minute per token.",
    "retry_after_seconds": 47,
    "current_usage": "100/100 requests in current window"
  },
  "request_id": "exec-n8n-20261125-005",
  "headers": {
    "X-RateLimit-Limit": "100",
    "X-RateLimit-Remaining": "0",
    "X-RateLimit-Reset": "1732513847",
    "Retry-After": "47"
  }
}
```

### 8.5 HTTP 409 Conflict — Duplicate Record

```json
{
  "status": "error",
  "http_code": 409,
  "error": {
    "code": "DUPLICATE_ACCOUNT",
    "message": "An account with this email already exists.",
    "existing_record": {
      "account_id": "acc_existing_12345",
      "created_at": "2024-01-15T10:00:00Z"
    },
    "resolution": "Use PATCH /v2/accounts/{id} to update existing record, or DELETE first if replacing."
  },
  "request_id": "exec-n8n-20261125-006"
}
```

---

## 9. API Rate Limits & Throttling Policy

| Tier | Requests/Min | Requests/Day | Burst Allowance |
|------|-------------|-------------|----------------|
| Starter | 20 | 1,000 | 30 req/min (10s window) |
| Professional | 60 | 10,000 | 80 req/min (10s window) |
| Enterprise | 300 | 100,000 | 400 req/min (10s window) |

**Recommended n8n Configuration for Bulk Migration:**
- Use `Split In Batches` node: batch size = 10 records
- Add `Wait` node: 2 seconds between batches
- Maximum safe throughput at Professional tier: ~300 records/minute

---

## 10. Idempotency & Retry Policy

| Scenario | Recommended Action |
|----------|-------------------|
| `2xx` success | No retry needed |
| `400` Bad Request | Fix payload, do NOT retry same payload |
| `401` Unauthorized | Refresh token, then retry once |
| `422` Validation Error | Log to quarantine file, escalate for manual review |
| `429` Rate Limited | Wait `Retry-After` seconds, then retry with exponential backoff |
| `500` Server Error | Retry after 30s delay, up to 3 attempts |
| `504` Gateway Timeout | Retry after 60s delay; check API status page |

> **Idempotency Key**: Always send `X-Request-ID` header with a unique value (e.g. n8n execution ID).
> The CRM API uses this to deduplicate retried requests within a 24-hour window.

---

*Specification maintained by: Liyue Liu | github.com/liyueliu43-arch*
*For integration questions, open a GitHub issue on the project repository.*
