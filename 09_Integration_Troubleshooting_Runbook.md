# Integration Troubleshooting Runbook

> **Project**: Enterprise SaaS Implementation & Data Migration Governance
> **System**: NexaCloud Cloud CRM — API Integration Layer
> **Audience**: Implementation Consultants, Integration Engineers, Ops Team
> **Author**: Liyue Liu (@liyueliu43-arch)
> **Version**: 1.0 | Last Updated: 2026-11-25
> **On-Call Escalation**: ops-lead@nexacloud.io | Slack: `#data-ops`

---

## How to Use This Runbook

This runbook covers the **4 most common integration failure scenarios** encountered during
SaaS CRM implementation and post-go-live operations. For each error type:

1. **Identify** the HTTP status code and error message
2. **Follow** the step-by-step diagnosis procedure
3. **Apply** the documented fix
4. **Verify** resolution using the provided commands
5. **Log** the incident in the ops incident tracker

> ⚠️ **NEVER** manually insert or edit records in the production CRM without first resolving
> the root cause in the integration layer. Manual patches create audit gaps.

---

## Error Type 1: HTTP 401 Unauthorized — Token Expired or Invalid

### Symptom

```
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "status": "error",
  "http_code": 401,
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "The Bearer token has expired or is invalid.",
    "detail": "Token expiry: 2026-11-25T05:00:00Z. Current time: 2026-11-25T06:16:00Z."
  }
}
```

**Observed Behaviour**: All API calls return 401. n8n workflow executions show error on the
`🚀 Push: POST to Cloud CRM API` node. Slack/email alerts may also fail if credentials are shared.

### Root Cause Analysis

| Root Cause | Likelihood | Trigger |
|-----------|-----------|---------|
| Bearer token TTL (3600s / 1 hour) elapsed between n8n execution scheduling and actual execution | **High** | Long queues or delayed executions |
| API credentials revoked by admin (security rotation policy) | **Medium** | Periodic security review |
| Wrong `CLIENT_ID` or `CLIENT_SECRET` in environment variable | **Low** | Post-deployment misconfiguration |
| Clock skew between n8n server and auth server | **Low** | Server time drift >5 minutes |

### Step-by-Step Resolution

**Step 1**: Confirm the error is 401 (not 403):
```bash
# Check n8n execution log for the exact HTTP response code
# In n8n UI: Executions → Failed Execution → HTTP Request node → Response tab
# Look for: "statusCode": 401
```

**Step 2**: Test token validity manually:
```bash
curl -X GET "https://api.crm.nexacloud.io/v2/health" \
  -H "Authorization: Bearer <your_current_token>" \
  -v 2>&1 | grep "HTTP/"
# Expected if valid: HTTP/1.1 200 OK
# If 401: token is expired or invalid → proceed to Step 3
```

**Step 3**: Re-generate a new token:
```bash
curl -X POST "https://auth.nexacloud.io/oauth/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=${NEXACLOUD_CLIENT_ID}" \
  -d "client_secret=${NEXACLOUD_CLIENT_SECRET}" \
  -d "scope=crm:write crm:accounts crm:opportunities"
# Response: { "access_token": "eyJ...", "expires_in": 3600 }
```

**Step 4**: Update the token in n8n environment variable:
```
n8n UI → Settings → Environment Variables → NEXACLOUD_CRM_API_TOKEN
→ Paste new token value → Save → Restart affected workflow
```

**Step 5**: Re-run the failed execution:
```
n8n UI → Executions → Failed Execution → Click "Retry" button
```

**Step 6**: Implement long-term fix — add token auto-refresh:
```javascript
// Add a "Pre-flight" Code node BEFORE the HTTP Request node in n8n:
// This checks token age and refreshes if within 5 minutes of expiry
const tokenIssuedAt = $env.TOKEN_ISSUED_AT || 0;
const currentTime = Date.now() / 1000;
const tokenAge = currentTime - tokenIssuedAt;

if (tokenAge > 3300) { // 55-minute threshold (5 min buffer before 3600s expiry)
  // Trigger token refresh sub-workflow
  return [{ json: { needs_token_refresh: true } }];
}
return [{ json: { needs_token_refresh: false } }];
```

**Verification**:
```bash
# After fix, send a test payload:
curl -X POST "https://api.crm.nexacloud.io/v2/accounts/create-with-opportunity" \
  -H "Authorization: Bearer <new_token>" \
  -H "Content-Type: application/json" \
  -d '{"account":{"name":"Test Co","billing_country":"US","type":"Customer","status":"Trial"},...}'
# Expected: HTTP 201 Created
```

---

## Error Type 2: HTTP 422 Unprocessable Entity — Field Validation Failure

### Symptom

```
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

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
      }
    ]
  }
}
```

**Observed Behaviour**: The n8n circuit breaker is triggered. Alert email sent to ops lead.
Record is NOT created in CRM. The problematic record lands in the quarantine log.

### Root Cause Analysis

| Root Cause | Likelihood |
|-----------|-----------|
| Source system submitted record with invalid/missing email | **High** |
| Amount field sent as string with currency symbol (e.g. `"$12,000"`) instead of float | **High** |
| Country value not normalised to ISO code before submission | **Medium** |
| Webhook payload structure doesn't match API schema (field name mismatch) | **Medium** |
| Required field omitted from payload (e.g. `company_name` absent) | **Low** |

### Step-by-Step Resolution

**Step 1**: Read the full `validation_errors` array from the 422 response:
```bash
# The error response will list EVERY failing field.
# Example multi-field failure:
# validation_errors: [
#   { "field": "contact.email", "value": "invalid@", "rule": "RFC_5322_FORMAT" },
#   { "field": "opportunity.amount", "value": "$12,000", "rule": "NUMERIC_FLOAT" }
# ]
```

**Step 2**: Locate the source record in the quarantine file:
```bash
# Open 03_quarantined_error_records.csv
# Search for the company name or Legacy_ID mentioned in the alert email
# Check the _quarantine_reason column for the exact issue
```

**Step 3**: Depending on error field, apply the correct fix:

**If `contact.email` invalid:**
```python
import re
email = "invalid-email@"
pattern = r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$'
is_valid = bool(re.match(pattern, email))
# False → flag for client to provide correct email, do NOT guess
```

**If `opportunity.amount` not numeric:**
```python
import re
raw = "$12,000"
clean = float(re.sub(r'[^\d.]', '', raw))  # → 12000.0
# Always run amount normalisation BEFORE submitting to API
```

**If `account.billing_country` not ISO:**
```python
COUNTRY_MAP = {"USA": "US", "United States": "US", "Germany": "DE", ...}
iso = COUNTRY_MAP.get(raw_country.strip(), raw_country)
```

**Step 4**: Correct the source record and re-submit through the ETL pipeline:
```bash
# Update the record in the source CSV
# Re-run etl_migration_validator.py
# Verify the corrected record appears in 02_clean_target_crm_import.csv
# Re-trigger the n8n webhook with the corrected payload
```

**Step 5**: Confirm resolution:
```bash
# After re-submission, check n8n execution log for HTTP 201
# Verify the account appears in CRM search
# Update the quarantine CSV to mark the record as "RESOLVED"
```

**Prevention Measures**:
- Enforce input validation at the source sales form (regex on email field)
- Run ETL pipeline on ALL records before ANY webhook submission
- Never bypass the n8n IF validation node in production

---

## Error Type 3: HTTP 429 Too Many Requests — API Rate Limit Exceeded

### Symptom

```
HTTP/1.1 429 Too Many Requests
Retry-After: 47
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1732513847
Content-Type: application/json

{
  "status": "error",
  "http_code": 429,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "API rate limit exceeded. Maximum 100 requests per minute per token.",
    "retry_after_seconds": 47
  }
}
```

**Observed Behaviour**: Bulk migration batch partially completes; subsequent requests fail with 429.
n8n workflow shows multiple consecutive errors on HTTP Request node. Some records may not be imported.

### Root Cause Analysis

| Root Cause | Likelihood |
|-----------|-----------|
| Bulk migration submitted all records simultaneously without rate limiting | **Very High** |
| Multiple n8n workflow instances running in parallel (triggered by retry) | **High** |
| API tier is Professional (100 req/min) but migration volume requires Enterprise tier | **Medium** |
| Another system (e.g. reporting tool) consuming API quota simultaneously | **Low** |

### Step-by-Step Resolution

**Step 1**: Immediately stop all running n8n executions:
```
n8n UI → Executions → Active executions → Pause/Stop all
```

**Step 2**: Calculate how many records were successfully imported before throttling:
```sql
-- Run in CRM database console:
SELECT COUNT(*) as imported_count
FROM accounts
WHERE migration_batch = 'NEXACLOUD_2026_11'
  AND created_at > '2026-11-25T02:00:00Z';
-- Compare to expected 21; find the gap
```

**Step 3**: Wait for rate limit window to reset:
```bash
# Read the X-RateLimit-Reset header (Unix timestamp)
date -d @1732513847  # → converts to human-readable time
# Wait until that time + 30 seconds buffer before retrying
```

**Step 4**: Modify n8n workflow to include rate limiting:
```
1. In n8n, add a "Split In Batches" node after the formatter:
   - Batch size: 10 records

2. After the HTTP Request node, add a "Wait" node:
   - Wait: 2 seconds

3. This gives throughput of ~5 req/s = 300 req/min MAX
   → Safe for Enterprise tier (300/min limit)
   → Adjust batch size to 5 for Professional tier (60/min limit)
```

**Step 5**: Resume migration with only the unimported records:
```bash
# Create a "resume file" containing only records not yet in CRM:
# 1. Export currently imported account emails from CRM
# 2. Filter 02_clean_target_crm_import.csv to exclude already-imported rows
# 3. Re-run ETL with the filtered file only
```

**Step 6**: Monitor the retry:
```bash
# Watch n8n execution success rate in real-time
# In n8n UI: Executions → sort by Start Time → watch for 201 responses
# Target: 0 errors; 100% success rate
```

**Long-term Prevention**:
```
1. Add exponential backoff to n8n HTTP node: on 429 → wait (retry_after + 10s) → retry
2. Request API tier upgrade to Enterprise if volume > 10,000 records/migration
3. Schedule large migrations during off-peak hours (02:00-04:00 UTC)
4. Use separate API tokens for migration vs. real-time order sync
```

---

## Error Type 4: HTTP 504 Gateway Timeout — Network/Infrastructure Issue

### Symptom

```
HTTP/1.1 504 Gateway Timeout
Content-Type: application/json

{
  "status": "error",
  "http_code": 504,
  "error": {
    "code": "GATEWAY_TIMEOUT",
    "message": "The upstream server did not respond within the timeout period.",
    "detail": "Request timed out after 15000ms.",
    "upstream": "crm-api-service-pod-3"
  }
}
```

**Observed Behaviour**: n8n HTTP Request node times out after 15 seconds. No CRM record is created.
May occur for single requests (transient) or all requests (systemic infrastructure issue).

### Root Cause Analysis

| Root Cause | Likelihood | Evidence |
|-----------|-----------|---------|
| CRM API server overloaded (high traffic period) | **High** | Check CRM status page |
| Network connectivity issue between n8n server and CRM API | **High** | Ping test fails |
| Payload too large (e.g. binary attachments accidentally included) | **Medium** | Check payload size |
| CRM database performing heavy index rebuild or backup | **Medium** | Check CRM maintenance schedule |
| n8n outbound firewall rule blocking requests | **Low** | Check firewall logs |
| DNS resolution failure | **Low** | nslookup test |

### Step-by-Step Resolution

**Step 1**: Check CRM API status page immediately:
```
https://status.crm.nexacloud.io
→ Look for: "API Service — Degraded" or "Outage"
→ If outage reported: WAIT; do not retry repeatedly (worsens overload)
→ Subscribe to status page notifications
```

**Step 2**: Test basic connectivity from n8n server:
```bash
# Test DNS resolution:
nslookup api.crm.nexacloud.io
# Expected: returns IP address like 54.x.x.x

# Test TCP connectivity on HTTPS port:
Test-NetConnection -ComputerName api.crm.nexacloud.io -Port 443
# Expected: TcpTestSucceeded : True

# Test HTTP layer:
curl -I "https://api.crm.nexacloud.io/v2/health" --max-time 10
# Expected: HTTP/2 200
```

**Step 3**: Check if issue is transient (single request) or systemic (all requests):
```bash
# Send 3 lightweight health check requests at 10-second intervals:
for i in 1 2 3; do
    curl -s -o /dev/null -w "Attempt $i: %{http_code} (%{time_total}s)\n" \
        -H "Authorization: Bearer <token>" \
        "https://api.crm.nexacloud.io/v2/health"
    sleep 10
done
# If all return 200: transient issue → retry original request
# If all return 504: systemic issue → escalate to CRM vendor
```

**Step 4**: If transient, configure n8n retry with exponential backoff:
```javascript
// In n8n HTTP Request node → Settings → On Error: Continue (error output)
// Add a "Wait" node after → 60 seconds
// Add another "HTTP Request" node (the retry)
// Limit to 3 retry attempts:
const retryCount = $node["Retry Counter"].json.count || 0;
if (retryCount >= 3) {
  // Send alert email: max retries exceeded
  return [{ json: { escalate: true, retryCount } }];
}
return [{ json: { escalate: false, retryCount: retryCount + 1 } }];
```

**Step 5**: If systemic (all requests failing > 10 minutes):
```
1. Declare incident — post to #data-ops Slack channel:
   "🚨 INCIDENT: CRM API returning 504. Migration paused. ETA unknown. 
    Monitoring status.crm.nexacloud.io"

2. Contact CRM vendor support with:
   - Error timestamp (UTC)
   - Request IDs from failed n8n executions
   - Volume of requests in the period
   - Status page incident ID (if any)
   Vendor SLA: Enterprise P1 = 1-hour response

3. Pause n8n migration workflow to prevent further failed executions
   (They will not succeed anyway and consume API quota)

4. Resume only after CRM status page shows "All Systems Operational"
```

**Step 6**: Post-resolution — verify records submitted during outage:
```sql
-- Check CRM for records created after outage end time
SELECT COUNT(*), MIN(created_at), MAX(created_at)
FROM accounts
WHERE migration_batch = 'NEXACLOUD_2026_11';
-- Compare to expected count (21)
-- If gap found: re-submit missing records only
```

**Long-term Prevention**:
```
1. Set n8n HTTP timeout to 30s (double the API's 15s server timeout) for better error capture
2. Implement idempotency: use X-Request-ID; API will deduplicate retried requests
3. Subscribe to CRM API status page via email/Slack webhook
4. Schedule large batch migrations between 00:00–04:00 UTC (lowest API load)
5. Maintain a "pending queue" file of records not yet confirmed as created
```

---

## Quick Reference: HTTP Status Code Decision Tree

```
API Call fails
     │
     ├── 401 Unauthorized → Refresh Bearer token → Retry
     │
     ├── 403 Forbidden → Check API scope / user permissions (not a token issue)
     │
     ├── 404 Not Found → Wrong endpoint URL; check API documentation
     │
     ├── 409 Conflict → Duplicate record; use PATCH instead of POST, or skip
     │
     ├── 422 Unprocessable → Fix data payload (email/amount/country format) → Re-run ETL → Retry
     │
     ├── 429 Too Many Requests → Wait Retry-After seconds → Implement batch + sleep → Retry
     │
     ├── 500 Internal Server Error → Likely transient; retry after 30s, up to 3 times
     │
     ├── 502 Bad Gateway → Load balancer issue; retry after 60s
     │
     └── 504 Gateway Timeout → Check status page; wait for resolution; retry with backoff
```

---

## Escalation Matrix

| Severity | Condition | First Responder | Escalation Path | SLA |
|----------|-----------|----------------|----------------|-----|
| **P1 Critical** | Full API outage; all requests failing; migration blocked | Ops Lead | Liyue Liu → CRM Vendor P1 Support | 1 hour |
| **P2 High** | Validation errors > 10% of batch; partial migration blocked | Liyue Liu | Client PM → Data Steward Team | 4 hours |
| **P3 Medium** | Rate limiting causing delays; migration > 2x estimated time | Liyue Liu | Adjust batch parameters | 24 hours |
| **P4 Low** | Single record failure; isolated 422 or 401 | Liyue Liu | Log in quarantine file; resolve in next batch | 72 hours |

---

*Runbook maintained by: Liyue Liu | github.com/liyueliu43-arch*
*For additions or corrections, submit a PR to the project repository.*
*Last tested: 2026-11-25 against CRM API v2.3.1*
