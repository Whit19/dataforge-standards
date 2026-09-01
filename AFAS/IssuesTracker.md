# AFAS Project — Data Issues Tracker
**Active issues only. Resolved items move to Decision Log with date closed.**
Last updated: 2026-09-01

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

### NOTE — 8 Apple Uncategorized rows intentionally parked
Pending manual review. All have `category_source = manual`, `in_budget = 1`. Not a data quality issue — held for owner classification decision.

---

### ISSUE-012 — Category taxonomy drift
category_map contains subcategories not documented in Category_Taxonomy.md (Dividends, Mobile Deposit, Pension, Tax Refund under Other Income — discovered 2026-07-01 during Interest reclassification). Needs a full taxonomy audit next session: every category/subcategory/in_budget flag reviewed against actual category_map and merchant_patterns contents, not just spot-checked reactively.

---

### ISSUE-014 (recurrence note on prior fix) — subcategory-mirror violations recurred
2026-06-01 taxonomy fix corrected subcategory=category mirror violations (Clothing, Dining Out, Gifts/Charity, Groceries, Payment). By 2026-07-01, 294 merchant_patterns rows and 11 category_map rows had the same violation again (Clothing, Groceries, Car, Property Tax, Payment, Dining Out, Interest). Root cause of the recurrence not yet investigated — worth checking whether a specific import/enrichment script is reintroducing these values, rather than treating each occurrence as an isolated one-off fix.

---

### ISSUE-016 — run_log missing entries for daily transaction syncs
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-01 |
| Priority | Medium |
| Description | plaid_sync.py's daily CHASE/AMEX/ASSOCIATED_PERSONAL transaction sync never writes to dbo.run_log, despite dbo.plaid_sync_state showing successful syncs. Only BAIRD_HOLDINGS, ASSOCIATED_PERSONAL_BALANCE, NWM_SYNC, and PRINCIPAL_401K currently log to run_log. This gap made the August 2026 outage harder to diagnose than it should have been. |
| Next Step | Add run_log INSERT calls to plaid_sync.py's _sync_institution(), matching the pattern already used in balance_sync.py/nwm_sync.py/principal_sync.py. |

---

### ISSUE-017 — HSA-Baird and other non-canonical Baird accounts never imported
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-01 |
| Priority | Medium |
| Description | Confirmed via SELECT DISTINCT account_name FROM dbo.baird_holdings that HSA-Baird and "401k Baird Profit Share" (as a Baird-CSV-tracked account, separate from the now-connected Principal/Plaid version) have never appeared in a single monthly import. Not a Plaid gap — these accounts simply aren't part of what gets exported/tagged in the monthly CSV procedure. (Distinct from the new HSA-BOFA-MANUAL account built in Session 13 — that's a separate Bank of America custodial HSA with its own manual CSV pipeline; this issue is specifically about the Baird-administered HSA-Baird account.) |
| Next Step | Decide whether HSA-Baird needs to be added to the monthly export procedure, or confirm it's intentionally tracked elsewhere / excluded. |

---

### ISSUE-020 — category_confidence not populated for Plaid-synced sources

| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-01 |
| Priority | Low |
| Description | Data Health page's vw_category_health shows real low_confidence_count for CHASE (318), ASSOCIATED_PERSONAL (193), AMEX (14), while APPLE and HSA (both CSV-imported sources) show 0. Hypothesis, not confirmed: enrich_transactions.py (used for Plaid-synced sources) may not be setting category_confidence='HIGH' on newer rows the way enrich_apple_csv.py/enrich_hsa_csv.py do. |
| Next Step | Review enrich_transactions.py directly to confirm whether/how it sets category_confidence, rather than continuing to infer from the symptom. |

---

### ISSUE-021 — import_baird_holdings.py has a broken local.settings.json path

| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-01 |
| Priority | Low |
| Description | Found while fixing get_plaid_tokens.py's hardcoded credentials: import_baird_holdings.py's local.settings.json path resolution points at Run_Monthly/local.settings.json, which doesn't exist. Currently harmless — that script never actually reads a PLAID_* value from local.settings.json — but would silently break if a future edit added one. |
| Next Step | Fix the path resolution to match the working pattern used elsewhere (same fix applied to get_plaid_tokens.py this session). Not urgent — dead code path today. |

---

### ISSUE-022 — Extent of pre-existing March 2026 HSA merchant_patterns batch unconfirmed

| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-01 |
| Priority | Low |
| Description | Two separate attempts to add new HSA merchant_patterns this session (Payroll Deduction, Employer Contribution) both turned out to be case-insensitive duplicate-key collisions with patterns already created 2026-03-05/07 — before this session's HSA reconnection work began. Suggests a fuller categorization pass was already done on this account at some point while it was still Plaid-connected. A diagnostic query (patterns created 2026-03-04 to 2026-03-08) was given to Tom to confirm the full extent — not yet run. |
| Next Step | Run the diagnostic query, review results, note in Category_Taxonomy.md or BestMethods whether this session's merchant_patterns additions filled real gaps or were mostly redundant. |

---

### ISSUE-024 — plaid_sync.py's description column is dead code; no fallback against Plaid merchant_name drift
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-03 |
| Priority | Medium |
| Description | tx.get("description", "") in plaid_sync.py always returns empty — Plaid's transaction object has no field called description, so this column has never been populated. Discovered while investigating why "Center" and "Product" displayed instead of "North Shore CTR LLC" and "TradingViewV*Product" — Plaid's merchant_name field is sometimes over-resolved/generic, and the code only falls back to the fuller name field when merchant_name is completely absent, not merely low-quality. Confirmed separately that Plaid's own merchant_name resolution for a given vendor can change year over year (TradingView: matched an existing pattern in 2025, resolved to generic "Product" in 2026), meaning a merchant_patterns rule that works today can silently stop matching later with no code change on our end. |
| Next Step | Repurpose the description column to store Plaid's raw name field at ingestion, giving a fallback text source when merchant_name is low-quality, and a way to detect/diagnose future merchant-name drift without waiting for a human to recognize a truncated name. |

---

### ISSUE-025 — enrich_apple_csv.py suspected of the same type/in_budget fallback bug as enrich_transactions.py
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-03 |
| Priority | Medium |
| Description | Apple-sourced transactions were still not appearing correctly under Expense on Power BI's Monthly Spend page after the CHASE/ASSOCIATED_PERSONAL/AMEX fix (ISSUE-019) was deployed and verified. Apple transactions are enriched by a separate script, enrich_apple_csv.py, which was not reviewed this session — the same apply_fallback()-style type='Other'/in_budget=0 bug found in enrich_transactions.py is suspected but not confirmed. |
| Next Step | Upload/review enrich_apple_csv.py directly — do not assume the same bug exists without seeing the code, per project debugging protocol. |

---

### ISSUE-026 — 14 unclassified merchants from the ISSUE-019 backlog awaiting identification
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-03 |
| Priority | Low |
| Description | Bizjtix ($2,700, CHASE), WISCONSINGOV ($318.93, ASSOCIATED_PERSONAL), Retail/RETAIL (x2, CHASE), Bayshore D (ASSOCIATED_PERSONAL), Musa I (x2, ASSOCIATED_PERSONAL), Shorewood (ASSOCIATED_PERSONAL), TEMPORARY FUNDS HOLD (ASSOCIATED_PERSONAL — possibly a bank-side pending/hold artifact, not real spend), SHEBOYGAN (ASSOCIATED_PERSONAL — Plaid's RENT_AND_UTILITIES_OTHER_UTILITIES code suggests a utility, but no matching "Other" subcategory currently exists under Bills & Utilities), Google Cloud ($0.06, AMEX). Not blocking — these sit under Plaid's genuinely broad catch-all codes where a blanket category_map mapping would risk misclassifying unrelated future merchants. "Center" and "Product" from the original 16-row list were identified and resolved in Session 14 (see DecisionLog 2026-08-03). Railway ($5.00, AMEX) identified and resolved 2026-09-01 (see DecisionLog) — removed from this list. |
| Next Step | Tom to identify remaining merchants; Claude drafts final merchant_patterns/manual-correction SQL once identified. |

---

### ISSUE-032 — Azure SQL auto-pause collides with the automated monthly timer trigger
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-09-01 |
| Priority | Medium |
| Description | FinanceDB (Free tier Serverless) can auto-pause between runs. The 2026-09-01 03:00 UTC automated monthly_sync/timer_sync run hit Azure SQL error 40613 ("Database not currently available") because the DB was paused, then failed all retries with "Login timeout expired." Both functions logged "Succeeded" in Application Insights despite the internal SQL work failing entirely — function-level status does not reflect actual sync success. This occurrence was resolved manually (DB resumed, endpoints retriggered by hand), but the underlying collision risk remains for every future monthly run. |
| Next Step | Add retry-with-backoff to db.py's SQL connection logic for error 40613, so a close collision with the auto-pause window can self-heal without manual intervention. Discussed but not implemented this session. |

---

## Resolved Issues (archive — recent)

### RESOLVED — ISSUE-023 — BANK_FEES/LOAN_PAYMENTS sign convention regression
- Resolved: 2026-09-01
- The Session 14 fix (remove BANK_FEES/LOAN_PAYMENTS from plaid_sync.py's INFLOW_CATEGORIES) had been drafted but never deployed — same deploy-gap pattern as ISSUE-019 below. Confirmed still live via a Power BI screenshot showing Rocket Mortgage/University Club Collection posting positive. Scope grew from Session 14's ~25-row estimate to 32 rows (2026-02-26 through 2026-09-01) since the bug kept running an extra month. Fix deployed and verified via VS Code's "Files (Read-only)" remote view; historical correction (amount = -amount) run on all 32 rows, verified 0 remaining. Live verification against a new real transaction still pending — no qualifying transaction has occurred since deploy; follow up against the next Amex/Chase autopay or mortgage payment. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-027 — Apple/Associated Personal merchant-name-drift + pending/settled duplicate
- Resolved: 2026-09-01
- Associated Bank's feed sometimes truncates "Apple Card" autopay descriptions to a bare "Apple," which fell through to the generic %APPLE% catch-all and miscategorized as Subscriptions/Apps & Software instead of Payment/Apple Card (June/July/August 2026 rows; March/April/May had been separately hand-corrected). Also found a pending/settled duplicate pair for the same underlying transaction. Fixed: deleted the stale pending row, added an exact-match (verified via direct code read, not assumption) merchant_patterns entry for bare "Apple," corrected the 3 miscategorized rows. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-028 — HSA transaction history duplicated 2x–4x (unstable transaction_id hashing)
- Resolved: 2026-09-01
- import_hsa_transactions.py hashed raw, unparsed CSV text for transaction_id; Bank of America's export-formatting drift between two monthly exports changed every row's hash, re-inserting the full account history under new IDs on each affected reimport instead of no-op'ing via MERGE (~400+ duplicate groups). Fixed to hash normalized values, verified idempotent via a real two-run test. Cleanup script caught and corrected mid-review to avoid deleting 157 of 159 orphaned rows Session 13 had explicitly preserved. Result: 835 rows deleted, 461 canonical rows remain, 0 duplicate groups. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-029 — UnicodeEncodeError (cp1252 vs UTF-8) across 5 monthly import/enrich scripts
- Resolved: 2026-09-01
- Box-drawing summary print output crashed on Windows terminals defaulting to cp1252. Crash occurred after DB commit, so no data was affected — confirmed by checking the DB directly. Same sys.stdout.reconfigure(encoding="utf-8") fix applied and verified against real runs in import_hsa_transactions.py, enrich_hsa_csv.py, import_hsa_holdings.py, import_apple_csv.py, enrich_apple_csv.py. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-030 — import_apple_csv.py MERGE unconditionally overwrote enrichment metadata on every re-run
- Resolved: 2026-09-01
- Any re-run of import_apple_csv.py against already-enriched data reset category_source/category_confidence to hardcoded import-time defaults and could flip in_budget back to 1 — confirmed as a real production risk (not just a test artifact) discovered when a same-session test re-run corrupted 131 rows. category/subcategory themselves were unaffected. First fix attempt (two WHEN MATCHED clauses) was invalid T-SQL, caught via a real test before deploying. Corrected fix uses a single WHEN MATCHED with CASE expressions, protecting already-enriched rows (including category_source='manual') while still updating never-enriched rows normally. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-031 — manual category corrections left type/in_budget as stale inherited values
- Resolved: 2026-09-01
- The 3 manually-corrected Apple/Associated Personal rows (ISSUE-027) and 1 legacy Session 13 HSA row set category/subcategory correctly but left type/in_budget unchanged from before the correction, causing the wrong Power BI Expense/Income/Other bucket despite the correct category label — caught via a Power BI pivot review, not anticipated in advance. Fixed all 4 rows using category_map's correct type/in_budget values for their category. New BestMethods lesson added: manual corrections must always set type/in_budget together with category/subcategory. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-019 — Power BI Monthly Spend showed Apple-only data for June/July 2026
- Resolved: 2026-08-03
- Root cause: plaid_sync.py stored the full personal_finance_category JSON object in plaid_category_raw instead of a clean category string, so category_map's exact-string match silently never worked for Plaid-synced sources. Compounded by a separate bug where enrich_transactions.py's isna()-gated retry logic never actually reprocessed rows previously written by apply_fallback(). Both fixed (CC prompts deployed); 64 affected rows backfilled via script 59, re-enriched, and verified. 26 of the remaining 42 unmatched rows resolved via category_map/merchant_patterns additions (scripts 60/61). 14 rows remain — tracked separately as ISSUE-026, not blocking. Power BI visual reconfirmation (manual refresh + Monthly Spend page check) still pending as of session end. See DecisionLog 2026-08-03 for full diagnostic trail.
- **RECURRENCE (2026-09-01):** the Session 14 code fix was correctly written but never actually deployed to Finance-ingest-Tom-v6 — the bug silently kept creating new JSON-blob rows for a full month (732 affected, not the original 64). "Resolved" had only ever been true for the historical backfill, not the ingestion path. Now genuinely deployed and verified (VS Code "Files (Read-only)" remote view confirmed the live INFLOW_CATEGORIES/extraction logic), 732 rows backfilled, 0 remaining. See DecisionLog 2026-09-01. New BestMethods lesson added: a "Resolved" status describes a fix being drafted/committed, not necessarily deployed — any Function-App-deployed file needs an explicit deploy-and-verify step before an issue is genuinely closed.

### RESOLVED — ISSUE-015 — Deployed Function App silently crash-looped for 6+ weeks (dotenv)
- Resolved: 2026-08-01
- python-dotenv was added to db.py's dependencies during the July session but never added to requirements.txt. Deployed app crashed on every cold start (ModuleNotFoundError), causing the trigger indexer to find 0 functions — not a portal display quirk, a genuine failure. All automated syncs (daily transactions, monthly balance/NWM sync) were silently dead for at least 6 weeks. Fixed by adding python-dotenv to requirements.txt and redeploying; confirmed via Application Insights ("5 functions loaded", clean host start).

### RESOLVED — ISSUE-009 — Principal Financial 401k Token Pending
- Resolved: 2026-08-01
- First Plaid connection completed via get_plaid_tokens.py. Real account name confirmed: "BAIRD PROFIT SHARING AND SAVINGS PLAN" (Prft Shr 401(K) Def Thrift), owner Amy. New principal_sync.py script created to pull holdings via /investments/holdings/get — confirmed live: 1 account, 11 securities, 11 holdings, $2,096,195.86 total value. Not yet wired into the automated pipeline (standalone local script only).

### RESOLVED — ISSUE-013 — Chase/Amex/Associated transaction verification needed
- Resolved: 2026-08-01
- Verified 2026-08-01: Chase and Associated were current (07-01/08-01), but AMEX had not synced since 2026-05-27 — root cause was NOT a query/data issue but the full production outage above (ISSUE-015) plus a separate expired Amex credential. Both fixed; Amex now current through 07-25. The underlying question of why nobody noticed the outage is tracked separately (see ISSUE-016).

### RESOLVED — ISSUE-018 — MAIN - Brokerage vs MAIN - BKG naming drift
- Resolved: 2026-08-01
- August 2026 Baird export used "MAIN - Brokerage" instead of canonical "MAIN - BKG" for all security holdings. Corrected via SQL (script 50) after confirming no symbol overlap between the two names (clean rename, not duplicated data). Root habit not addressed — worth double-checking next month's export uses the correct name from the start.

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
