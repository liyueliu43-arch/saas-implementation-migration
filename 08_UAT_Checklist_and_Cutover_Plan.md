# User Acceptance Testing (UAT) Checklist & Cutover Plan

> **Project**: Enterprise SaaS Implementation & Data Migration Governance
> **Client**: NexaCloud Global
> **Environment**: Production (after Staging sign-off)
> **UAT Period**: 2026-11-25 to 2026-11-27
> **Go-Live (Cutover) Date**: 2026-11-28 02:00 UTC
> **Document Owner**: Liyue Liu, Implementation Consultant (@liyueliu43-arch)
> **Status**: 🔄 IN PROGRESS

---

## Part 1: UAT Test Cases

### 1.1 Functional Testing

| Test ID | Test Category | Test Case Description | Expected Result | Actual Result | Status | Tester | Date |
|---------|--------------|----------------------|-----------------|---------------|--------|--------|------|
| **FT-001** | Data Migration | Verify 21 clean records appear in CRM post-import | Exactly 21 accounts created in CRM with correct fields | 21 accounts found, all fields verified | ✅ PASS | L. Liu | 2026-11-25 |
| **FT-002** | Data Migration | Verify total imported ARR = $803,000.00 | Sum of `opportunity.amount` in CRM = $803,000.00 | $803,000.00 confirmed via SQL query | ✅ PASS | L. Liu | 2026-11-25 |
| **FT-003** | Data Migration | Verify no duplicate accounts exist post-import | Zero accounts with identical email + company name | Deduplication confirmed; 0 duplicates | ✅ PASS | L. Liu | 2026-11-25 |
| **FT-004** | Data Migration | Verify 9 quarantined records are NOT in CRM | Quarantined record emails (e.g. `invalid-email@`) absent from CRM | 0 quarantined records found in CRM | ✅ PASS | L. Liu | 2026-11-25 |
| **FT-005** | Field Accuracy | Verify all country codes are valid ISO 3166-1 Alpha-2 | All `billing_country` values are 2-letter codes | All 21 records: valid ISO codes | ✅ PASS | Client QA | 2026-11-25 |
| **FT-006** | Field Accuracy | Verify all phone numbers in E.164 format | All `phone` values match regex `^\+[1-9]\d{7,14}$` | 20/21 valid; L019 (SunTech Arabia) quarantined | ✅ PASS | Client QA | 2026-11-25 |
| **FT-007** | Field Accuracy | Verify all `close_date` values are in YYYY-MM-DD format | No mixed format dates in CRM | All dates ISO 8601 compliant | ✅ PASS | Client QA | 2026-11-26 |
| **FT-008** | API Integration | POST valid order payload → account created in CRM | HTTP 201 Created; CRM record visible within 5s | ✅ PASS — record ID `acc_7f3a92b1c4d5e6f7` created | ✅ PASS | L. Liu | 2026-11-26 |
| **FT-009** | API Integration | POST payload with missing `contact_email` → error intercepted | HTTP 422 from n8n; alert email sent to ops lead | 422 returned; ops alert email received in 12s | ✅ PASS | L. Liu | 2026-11-26 |
| **FT-010** | API Integration | POST payload with `annual_contract_usd = 0` → error intercepted | n8n IF node rejects; circuit breaker triggers | Circuit breaker activated; Slack alert sent | ✅ PASS | L. Liu | 2026-11-26 |

### 1.2 Data Accuracy & Financial Integrity Testing

| Test ID | Test Category | Test Case Description | Expected Result | Actual Result | Status | Tester | Date |
|---------|--------------|----------------------|-----------------|---------------|--------|--------|------|
| **DQ-001** | Financial Integrity | Pre/post migration contract value reconciliation | Pre: $903,500 (all unique) → Post: $803,000 (clean) | Variance $100,500 fully traced to 9 quarantined records | ✅ PASS | L. Liu | 2026-11-25 |
| **DQ-002** | Financial Integrity | Spot-check: EuroFinance Partners `$120,000` in CRM | `opportunity.amount = 120000.00` | Confirmed: $120,000.00 | ✅ PASS | Client Finance | 2026-11-26 |
| **DQ-003** | Financial Integrity | Spot-check: MedCore GmbH `$98,000` in CRM | `opportunity.amount = 98000.00` | Confirmed: $98,000.00 | ✅ PASS | Client Finance | 2026-11-26 |
| **DQ-004** | Data Accuracy | Verify `GlobalTrade AG` (L005) is in quarantine, not CRM | Record absent from CRM; present in `03_quarantined_error_records.csv` | Confirmed absent from CRM | ✅ PASS | L. Liu | 2026-11-26 |
| **DQ-005** | Data Accuracy | Verify `DataStream Inc` duplicates resolved correctly | Only 1 instance of DataStream Inc; both duplicates quarantined | 1 record in CRM (L012 quarantined for missing email; L030 quarantined as duplicate) | ✅ PASS | Client QA | 2026-11-26 |

### 1.3 Security & Access Control Testing

| Test ID | Test Category | Test Case Description | Expected Result | Actual Result | Status | Tester | Date |
|---------|--------------|----------------------|-----------------|---------------|--------|--------|------|
| **SEC-001** | Authentication | Expired Bearer token → API call rejected | HTTP 401 Unauthorized returned | 401 received; n8n auto-refreshes token | ✅ PASS | L. Liu | 2026-11-26 |
| **SEC-002** | Access Control | Standard user cannot access Finance-tier CRM records | 403 Forbidden returned for unpermissioned user | 403 confirmed | ✅ PASS | Client IT | 2026-11-27 |
| **SEC-003** | Secrets Management | API tokens stored only in environment variables | No hardcoded credentials in `etl_migration_validator.py` or n8n workflow | Code review confirmed: 0 hardcoded secrets | ✅ PASS | L. Liu | 2026-11-27 |
| **SEC-004** | Audit Log | All API calls logged with `X-Request-ID` and timestamp | n8n execution logs show full call history | Logs confirmed in n8n execution history | ✅ PASS | Client IT | 2026-11-27 |

---

## Part 2: UAT Sign-Off Summary

| Test Suite | Total Cases | Passed | Failed | Blocked | Pass Rate |
|-----------|------------|--------|--------|---------|----------|
| Functional Testing | 10 | 10 | 0 | 0 | 100% |
| Data Accuracy & Financial | 5 | 5 | 0 | 0 | 100% |
| Security & Access Control | 4 | 4 | 0 | 0 | 100% |
| **TOTAL** | **19** | **19** | **0** | **0** | **100%** |

> ✅ **UAT SIGN-OFF GRANTED** — All 19 test cases passed. System is approved for production cutover.

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Implementation Consultant | Liyue Liu | *[signed]* | 2026-11-27 |
| Client Project Manager | Sarah Mitchell | *[signed]* | 2026-11-27 |
| Client IT Lead | James Okafor | *[signed]* | 2026-11-27 |
| Client Finance Rep | Anna Bergström | *[signed]* | 2026-11-27 |

---

## Part 3: Cutover Runbook (Go-Live Plan)

### Pre-Cutover Checklist (T-48 hours: 2026-11-26 02:00 UTC)

- [ ] All 19 UAT test cases marked PASS
- [ ] Client stakeholders have signed UAT sign-off sheet
- [ ] `02_clean_target_crm_import.csv` locked and checksummed (`SHA-256: computed at runtime`)
- [ ] n8n workflow set to ACTIVE (not paused)
- [ ] CRM API credentials rotated and updated in n8n environment variables
- [ ] Ops lead and client PM added to `#data-ops` Slack channel
- [ ] Rollback plan reviewed and confirmed by all parties
- [ ] Maintenance window communicated to end users (email sent 2026-11-26)
- [ ] Legacy CRM set to READ-ONLY mode to prevent new writes

### Cutover Execution (T=0: 2026-11-28 02:00 UTC — Low-Traffic Window)

| Step | Time (UTC) | Action | Owner | Verification |
|------|-----------|--------|-------|-------------|
| 1 | 02:00 | Begin maintenance window; legacy CRM read-only enforced | IT Lead | CRM admin confirms read-only flag |
| 2 | 02:05 | Final backup of legacy CRM database | IT Lead | Backup checksum logged |
| 3 | 02:10 | Run ETL pipeline one final time against latest legacy export | Liyue Liu | Confirm output counts match T-48h run |
| 4 | 02:20 | Execute bulk import: `02_clean_target_crm_import.csv` → Cloud CRM | Liyue Liu | Monitor n8n execution logs; watch for errors |
| 5 | 02:35 | Run post-import SQL reconciliation query | Liyue Liu | Confirm: 21 records, $803,000.00 |
| 6 | 02:40 | Activate n8n webhook for live order sync | Liyue Liu | Send test webhook; verify CRM record created |
| 7 | 02:45 | Update DNS/portal config to point to new CRM | IT Lead | Verify new CRM URL accessible |
| 8 | 02:50 | Smoke test: Sales team creates 1 test account end-to-end | Sales Lead | Account appears in CRM + Slack notification |
| 9 | 03:00 | ✅ GO/NO-GO decision point | All stakeholders | All green → proceed; any red → rollback |
| 10 | 03:05 | End maintenance window; communicate go-live to all users | PM | Email sent to all users |
| 11 | 03:10 | Begin 48-hour hypercare monitoring period | Liyue Liu | Daily stand-up at 09:00 UTC |

### Go/No-Go Criteria

| Criterion | Required | Consequence if Failed |
|-----------|---------|----------------------|
| Import record count = 21 | ✅ Must be exact | STOP — investigate before continuing |
| Total ARR in CRM = $803,000.00 | ✅ Must be exact | STOP — do not go live until reconciled |
| n8n webhook test creates account < 5s | ✅ Required | STOP — check API connectivity |
| Zero error records in import log | ✅ Required | STOP — investigate errors |
| Slack notification received | ⬜ Preferred | Continue but escalate |

---

## Part 4: Rollback Plan (Emergency Contingency)

> **Rollback Decision Authority**: Client Project Manager (Sarah Mitchell) or IT Lead (James Okafor)
> **Rollback Trigger Threshold**: Any Go/No-Go criterion fails at Step 9, or critical defect discovered within 4 hours of go-live.
> **Maximum Rollback Window**: 4 hours post-cutover (before business day begins in US Eastern time)

### Rollback Procedure

| Step | Action | Time Required | Owner |
|------|--------|--------------|-------|
| RB-1 | Declare rollback; notify all stakeholders via Slack + email | 5 min | PM |
| RB-2 | Pause n8n webhook (set workflow to INACTIVE) | 2 min | Liyue Liu |
| RB-3 | Restore legacy CRM to READ-WRITE mode | 5 min | IT Lead |
| RB-4 | Delete all records created during failed cutover from Cloud CRM (use `migration_batch = 'NEXACLOUD_2026_11'` as filter) | 15 min | Liyue Liu |
| RB-5 | Verify legacy CRM is fully operational; run smoke test | 10 min | Sales Lead |
| RB-6 | Communicate rollback completion to all users | 5 min | PM |
| RB-7 | Conduct root cause analysis (RCA); schedule re-cutover | 24 hrs | Liyue Liu + IT Lead |

**Rollback SQL (Cloud CRM cleanup):**
```sql
-- Identify rollback candidates
SELECT id, name, created_at
FROM accounts
WHERE migration_batch = 'NEXACLOUD_2026_11';

-- Delete opportunities first (FK constraint)
DELETE FROM opportunities
WHERE migration_batch = 'NEXACLOUD_2026_11';

-- Then delete contacts
DELETE FROM contacts
WHERE account_id IN (
    SELECT id FROM accounts WHERE migration_batch = 'NEXACLOUD_2026_11'
);

-- Finally delete accounts
DELETE FROM accounts
WHERE migration_batch = 'NEXACLOUD_2026_11';
```

> ⚠️ **CAUTION**: Rollback SQL must be executed by the IT Lead only, with a secondary approver present.
> Always run a `SELECT` first to verify record count before `DELETE`.

---

## Part 5: Post Go-Live Hypercare Plan (48 hours)

| Hour | Activity | Owner |
|------|----------|-------|
| 0–4 | Real-time monitoring: n8n execution logs, CRM API error rate | Liyue Liu |
| 4–24 | Daily SQL reconciliation: verify record counts + ARR totals | Liyue Liu |
| 24 | First hypercare check-in call with client (30 min) | Liyue Liu + PM |
| 24–48 | Monitor Slack #data-ops for any error alerts | Client IT |
| 48 | Hypercare end; handover to client operations team | Liyue Liu |
| 72 | 3-day post go-live review; sign-off on project closure | All stakeholders |

---

*Document prepared by: Liyue Liu | github.com/liyueliu43-arch | Implementation Consultant*
*For support during cutover: reach Liyue Liu on Slack @liyueliu or +1-xxx-xxx-xxxx*
