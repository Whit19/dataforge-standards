# AFAS Project — Data Issues Tracker
**Active issues only. Resolved items move to Decision Log with date closed.**
Last updated: 2026-07-01

---

## How to Use
- **Open:** Issue identified, not yet resolved
- **In Progress:** Actively being investigated
- **Resolved:** Add resolution date and move ruling to Decision Log, then remove from this file

---

## Active Issues

### ISSUE-008 — Baird Plaid Connection Failure
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-06-01 |
| Priority | Medium |
| Description | Plaid connection to Robert Baird Online (ins_117067) fails. Originally INTERNAL_SERVER_ERROR / API_ERROR. Baird support advised a platform migration disrupted Plaid. Retried 2026-06-15 — still fails with generic "Couldn't connect to your institution" error. Second email sent. CSV fallback pipeline (import_baird_holdings.py) now operational as permanent workaround — all 11 Baird accounts covered via monthly manual CSV export. |
| Last Action | Second email sent to Baird Online Support. CSV fallback pipeline built and tested 2026-06-17. |
| Next Step | Await Baird response. If Plaid resolves, holdings will auto-populate via Plaid Investments endpoint. CSV pipeline remains as monthly fallback regardless. |

---

### ISSUE-009 — Principal Financial 401k Token Pending
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-06-02 |
| Priority | Low |
| Description | Principal Financial 401k not yet connected via Plaid. get_plaid_tokens.py is ready. NWM diagnostic confirmed Plaid Investments returns no holdings for life insurance accounts — Principal is the primary target for Plaid Investments data. |
| Next Step | Run get_plaid_tokens.py, click Connect Principal Financial — 401k, complete login, store token as PLAID_ACCESS_TOKEN_PRINCIPAL in local.settings.json and Azure App Settings. Then run investments_diagnostic.py to confirm holdings data returned. |

---

### NOTE — 8 Apple Uncategorized rows intentionally parked
Pending manual review. All have `category_source = manual`, `in_budget = 1`. Not a data quality issue — held for owner classification decision.

---

### ISSUE-012 — Category taxonomy drift
category_map contains subcategories not documented in Category_Taxonomy.md (Dividends, Mobile Deposit, Pension, Tax Refund under Other Income — discovered 2026-07-01 during Interest reclassification). Needs a full taxonomy audit next session: every category/subcategory/in_budget flag reviewed against actual category_map and merchant_patterns contents, not just spot-checked reactively.

---

### ISSUE-013 — Chase/Amex/Associated transaction verification needed
timer_sync (renamed cadence: now monthly, was daily) confirmed running successfully in Azure Monitor every day this past month with zero errors, and plaid_sync.py's _TOKEN_KEYS correctly lists all three institutions — but actual presence of Chase/Amex/Associated transactions in dbo.transactions was never directly confirmed. Verify with:
```sql
SELECT source, COUNT(*) AS cnt, MAX([date]) AS most_recent
FROM dbo.transactions
GROUP BY source
ORDER BY cnt DESC;
```
before assuming ingestion is working just because the timer fires successfully.

---

### ISSUE-014 (recurrence note on prior fix) — subcategory-mirror violations recurred
2026-06-01 taxonomy fix corrected subcategory=category mirror violations (Clothing, Dining Out, Gifts/Charity, Groceries, Payment). By 2026-07-01, 294 merchant_patterns rows and 11 category_map rows had the same violation again (Clothing, Groceries, Car, Property Tax, Payment, Dining Out, Interest). Root cause of the recurrence not yet investigated — worth checking whether a specific import/enrichment script is reintroducing these values, rather than treating each occurrence as an isolated one-off fix.

---

## Resolved Issues (archive — recent)

### RESOLVED — ISSUE-010 — Plaid Balance Product Not Authorized
- Resolved: 2026-06-17
- Balance product approved by Plaid. plaid_client.py fixed (request object passed directly, not .to_dict()). [cursor] reserved keyword fix in balance_sync.py run_log INSERT. Associated balance pull live: 7 accounts, all verified.

### RESOLVED — ISSUE-011 — balance_sync.py error_log NULL error_id
- Resolved: 2026-06-15
- Fixed via str(uuid.uuid4()) inline

### RESOLVED — ISSUE-005 — enrichment.py Bonus Rule Amount Threshold
- Resolved: 2026-06-03
- Already implemented; retry path bug also fixed

### RESOLVED — ISSUE-004 — Uncategorized/VENMO-Review (46 rows)
- Resolved: 2026-06-02
- All rows classified

### RESOLVED — ISSUE-006 — May 2026 Apple Card CSV
- Resolved: 2026-06-01

### RESOLVED — ISSUE-007 — Historical Amount Sign Audit
- Resolved: 2026-06-01
