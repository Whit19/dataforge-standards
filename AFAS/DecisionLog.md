# AFAS Project — Decision Log
**Append only. Never delete entries. Most recent session at top.**
Last updated: 2026-07-01

---

## 2026-07-01: enrich_transactions.py column-loading bug (critical)
load_transactions() in enrich_transactions.py never loaded category_source, category_confidence, subcategory, in_budget, or type from the DB before running — only category was loaded. Combined with a blind-initialization loop that set all uninitialized enrichment columns to None, any row that already had a category (i.e., most 2024+ rows) had those five columns silently blanked to NULL on write-back, since no enrichment step's mask logic ever touched them.
Impact: ~7,069 transactions had category_source/category_confidence/subcategory/in_budget/type wiped in a single run on 2026-07-01.
Recovery: Azure SQL point-in-time restore to 2026-07-01 22:15 UTC (pre-corruption), reconciled via a one-time script (scripts/restore_category_source.py) that used COALESCE-based updates to restore only NULL columns without touching any legitimately-set values. Full recovery confirmed — 0 data loss.
Fix: load_transactions() now loads all five columns from the DB; blind-initialization loop only applies to columns genuinely absent (category, category_reviewed).
Lesson: any script following the "load some enrichment columns, blind-init the rest" pattern needs every column it writes to also be loaded first — this is now a pattern to check for in any future enrichment script review.

---

## 2026-06-17 (Session 10 — Balance Pulls, NWM Sync, Baird Holdings Pipeline, Phase 4 Views)

| Date | Decision | Rows |
|------|----------|------|
| 2026-06-17 | ISSUE-010 resolved — Plaid Balance product approved. plaid_client.py fix: get_account_balances() now passes request object directly to accounts_balance_get() (not .to_dict()); response .to_dict() called on result. Fixed [cursor] reserved keyword in balance_sync.py run_log INSERT. Associated balance pull tested live: 7 accounts, checked=7, upserted=7, errors=0. | 7 |
| 2026-06-17 | nwm_sync.py created — sync_nwm_valuations() calls get_account_balances() for NWM Tom + Amy tokens, maps fixed account_ids to insurance_ids (NWM-TOM, NWM-AMY), upserts insurance_asset_valuations. valuation_source column added to insurance_asset_valuations (ALTER TABLE); existing rows backfilled as 'manual'. NWM-TOM=$94,825.93, NWM-AMY=$55,058.52 confirmed live. | 2 |
| 2026-06-17 | monthly_sync.py created — monthly timer trigger (1st of month, 03:00 UTC) runs balance_sync + nwm_sync. Separate from daily timer_sync.py (transactions only). Registered in function_app.py. Deployed to Finance-ingest-Tom-v6. | — |
| 2026-06-17 | http_ingest.py updated — added http_nwm_sync route (/api/nwm_sync). All 5 functions confirmed registered: http_balance_ingest, http_ingest, http_nwm_sync, monthly_sync, timer_sync. | — |
| 2026-06-17 | baird_holdings + security_sectors tables created (36_baird_holdings_tables.sql). symbol column widened to varchar(50) on both tables to handle options notation (e.g. 'AAPL CALL 235.00 2025/12/19'). | — |
| 2026-06-17 | security_sectors seeded with 115 tickers across equities, ETFs, mutual funds, fixed income, options, private equity. BRK'B apostrophe requires escaped single quote in SQL ('BRK''B'). CASH symbol stored as uppercase to match CSV. asset_classification 'Cash' normalized to 'Cash and Cash Equivalents' in baird_holdings for consistency with Baird export format. | 115 |
| 2026-06-17 | import_baird_holdings.py created — reads holdings_*.csv from imports\baird\, upserts baird_holdings with lot-level detail. holding_id = account_name + "_" + symbol + "_" + snapshot_date + "_lot{N}" where N is a per-file lot counter per (account, symbol, date). Blank Security ID treated as symbol='CASH' (not skipped). Total summary rows skipped. currency parser handles $, commas, parentheses for negatives. | — |
| 2026-06-17 | Baird historical data imported: 4,585 rows (2024-01 to 2026-05, BKG+PIM combined). June 2026 data imported for all 11 accounts: 529-ALEX, 529-BROOKE, 529-WHIT, BAIRD Capital, BAIRD Stock, IRA-Baird Stock, IRA-TOM, IRA ROTH-Baird Stock, IRA Roth-TOM, MAIN-BKG, MAIN-PIM. Total 5,509 rows. All account totals verified to match Baird CSV pivot (Grand Total $7,231,246). | 5509 |
| 2026-06-17 | physical_asset_valuations seeded: HOUSE-WFB-BELLE $1,290,000 (Homes.com AVM), CAR-TSLA-NEBULA $72,000 (2024 Model X LR, KBB), CAR-TSLA-STORM $24,500 (2020 Model Y LR, KBB), CAR-TSLA-TRINITY $21,000 (2017 Model X 100D, KBB). All valuation_source='manual', update monthly. | 4 |
| 2026-06-17 | Phase 4 Power BI views created (40_phase4_powerbi_views.sql): vw_holdings_summary (1,801 rows), vw_asset_allocation (18 rows), vw_net_worth (16 rows), vw_liability_summary (2 rows), vw_budget_vs_actual (1,425 rows — actual only until budget_targets seeded). GO batch separators required between CREATE VIEW statements in mssql extension. | 5 views |
| 2026-06-17 | Net worth confirmed: Investment $7,231,246 + Physical $1,407,500 + Insurance $149,884 + Cash $150,085 - Liabilities $667,356 = ~$8,271,359. Excludes Principal 401k (pending) and full Baird Plaid data (ISSUE-008 still blocked). | — |

---

## 2026-06-15 (Session 9 — Balance Pull Build & Plaid Product Request)

| Date | Decision | Rows |
|------|----------|------|
| 2026-06-15 | Baird (ISSUE-008): Baird Online support replied that a platform-migration-related Plaid disruption was expected to resolve by end of day 2026-06-14, retry 2026-06-15. Retried — still fails, now as generic Plaid Link "Couldn't connect to your institution" error. Rocket Money also still fails for Baird — confirms still Baird-side. Second email sent to Baird support. | — |
| 2026-06-15 | Pivoted to unblocked Phase 4 work (Option A) while awaiting Baird — selected Associated Bank balance pull (/accounts/balance/get) | — |
| 2026-06-15 | Project root path conflict resolved: C:\DEV_Projects\AFAS confirmed canonical. TechnicalArchitecture.md previously listed incorrect path C:\Dev_Projects\Azure-Finance-Functions — corrected. | — |
| 2026-06-15 | plaid_client.py created (new file) — did not previously exist despite TechnicalArchitecture.md listing it as "Ready". _make_plaid_client() duplicated verbatim from plaid_sync.py (intentional — avoids modifying plaid_sync.py); added get_account_balances() wrapping AccountsBalanceGetRequest (plaid-python >=24.0.0, dict-style response access, balances["limit"] field name). Prior doc note re: "plaid_category_raw fix" likely describes logic actually living in plaid_sync.py — flagged for verification, not resolved this session. | — |
| 2026-06-15 | balance_sync.py created (new file) — sync_account_balances(access_token, source_label): calls /accounts/balance/get with no account filter, matches returned accounts against accounts table by account_id (logs + skips unmatched), upserts account_balances (balance_id = account_id + "_" + snapshot_date, snapshot_date = current UTC date), logs run to run_log (transactions_inserted repurposed to hold balance-row-upsert count) and any errors to error_log | — |
| 2026-06-15 | http_balance_ingest route added to http_ingest.py — manual trigger at /api/balance_ingest, calls sync_account_balances with PLAID_ACCESS_TOKEN_ASSOCIATED_PERSONAL | — |
| 2026-06-15 | Balance pull scope decision: pull /accounts/balance/get with no account_ids filter (returns all accounts under the Item), upsert any account_id that already exists in accounts table — covers Money Market + 3 Kids Savings plus Associated Checking/Checking-Whit/DF Checking with one API call, no hardcoded account list | — |
| 2026-06-15 | ISSUE-010 opened: /accounts/balance/get returns 400 INVALID_PRODUCT for ins_116823 (Associated Bank - Personal) — Plaid Production client not authorized for "balance" product. Confirmed via Plaid Dashboard logs (request_id c082fc92f1b6a05) | — |
| 2026-06-15 | ISSUE-011 opened and resolved same session: balance_sync.py _insert_error_log() omitted error_id (varchar PK, no identity), causing 515 NOT NULL violation on any error path. Fixed via str(uuid.uuid4()), matching run_id pattern | — |
| 2026-06-15 | Plaid "Balance" product ($0.10/call) requested via Dashboard "Add products" flow — use case "Other", described as personal account aggregation/dashboard (not payments). Pending Plaid review. | — |

---

## 2026-06-03 (Session 8 — MD Sync & Bug Fix)

| Date | Decision | Rows |
|------|----------|------|
| 2026-06-03 | enrich_transactions.py retry path bug fixed: removed extra sv(row, "category_normalized") parameter from write_results() retry execute call — category_normalized was removed from schema; retry path now matches 11-parameter happy path | — |
| 2026-06-03 | ISSUE-005 closed — bonus rule >$10,000 threshold confirmed already implemented in enrich_transactions.py apply_bonus_rule() | — |
| 2026-06-03 | Project root path confirmed: C:\DEV_Projects\AFAS | — |
| 2026-06-03 | get_plaid_tokens.py path confirmed: C:\DEV_Projects\AFAS\scripts\get_plaid_tokens.py | — |
| 2026-06-03 | All AFAS MD files synced to current state (Sessions 3–8) | — |

---

## 2026-06-02 (Session 7 — Token Acquisition)

| Date | Decision | Rows |
|------|----------|------|
| 2026-06-02 | get_plaid_tokens.py updated: institution_id removed from all entries and Plaid.create() call — now opens standard picker | — |
| 2026-06-02 | get_plaid_tokens.py updated: products now includes both transactions and investments | — |
| 2026-06-02 | get_plaid_tokens.py path confirmed: C:\DEV_Projects\AFAS\scripts\get_plaid_tokens.py | — |
| 2026-06-02 | NWM Tom + NWM Amy Plaid tokens acquired — stored in local.settings.json only (PLAID_ACCESS_TOKEN_NWM_TOM, PLAID_ACCESS_TOKEN_NWM_AMY) | — |
| 2026-06-02 | NWM accounts confirmed to return full cash value via Plaid — will feed insurance_asset_valuations via balance pull in Phase 4 | — |
| 2026-06-02 | Rocket Mortgage confirmed unsupported by Plaid — liability_balances will be updated manually | — |
| 2026-06-02 | Principal Financial 401k token acquisition deferred — to complete in next session (ISSUE-009) | — |

---

## 2026-06-02 (Session 6 — Transaction Audit & Sign Fixes)

| Date | Decision | Rows |
|------|----------|------|
| 2026-06-02 | ATM in_budget=0 fixed — 3 rows set to in_budget=1 to match taxonomy and category_map | 3 |
| 2026-06-02 | Uncategorized corrections: Anthropic/Claude.ai→Subscriptions/Apps & Software, Reunion+Washington House Inn+CLW INC sign flipped to negative expenses, Bush Center→Entertainment/Activities, Lavazza→Dining Out/Coffee | 9 |
| 2026-06-02 | merchant_patterns added: Anthropic, Claude.ai, Bush Center, Lavazza | 4 patterns |
| 2026-06-02 | 18 Venmo 2026 rows classified and sign-corrected; $600 Venmo→Travel/Lodging/Income; Chase catering Donation→General (4 rows) | 22 |
| 2026-06-02 | plaid_sync.py fixed: INCOME and TRANSFER_IN categories stored as positive (abs value); LOAN_PAYMENTS and BANK_FEES remain negative (outflows) | — |
| 2026-06-02 | plaid_sync.py deployed to Finance-ingest-Tom-v6 | — |
| 2026-06-02 | 12 Associated interest rows→Other Income/General/Income (sign flipped from negative) | 12 |
| 2026-06-02 | 1 Associated Transfer In→Transfer/General/Other (sign flipped, in_budget=0) | 1 |
| 2026-06-02 | 2 Chase Pellucid Travel→Travel/Flights/Expense (sign flipped from negative) | 2 |
| 2026-06-02 | Horse Shoe Farm refund→Travel/Lodging/Income +$739.64 (Chase, 2026-05-21) | 1 |
| 2026-06-02 | Zelle from Charles Driggars→Travel/Lodging/Income +$760.56 (Associated Personal, 2026-05-05) | 1 |
| 2026-06-02 | Green→Personal Care/Hair -$42.00; Superc→Groceries/General -$13.76 | 2 |
| 2026-06-02 | ISSUE-004 closed — all 46 Venmo/VENMO-Review rows resolved | — |

---

## 2026-06-01 (Session 5 — Phase 4 Tables, Merchant Patterns, View Fixes)

| Date | Decision | Rows |
|------|----------|------|
| 2026-06-01 | CHASE blank merchant_normalized diagnosed — all affected rows had category_source = 'manual', no data issue | — |
| 2026-06-01 | 16 merchant patterns added for recurring Chase merchants (Hilton, HP Instant Ink, Southwest Airlines, American Airlines, United Airlines, Illy Caffe, Jilly's Car Wash, Marcus Theatres, Northwestern Athletics, Bayview Bowl, GolfNow, Duke Dash, Business Journals) | 16 patterns |
| 2026-06-01 | merchant_normalized backfilled for 16 Chase merchant groups (existing NULL rows) | 25 rows |
| 2026-06-01 | State Farm (Chase) — confirmed blended Car Insurance / Home Insurance payment — left as Car Insurance (Option B, no split) | — |
| 2026-06-01 | vw_top_merchants updated: COALESCE(merchant_normalized, merchant_name_raw, 'Unknown') — eliminates (Blank) rows | — |
| 2026-06-01 | vw_transactions_clean updated: same COALESCE fix applied | — |
| 2026-06-01 | Baird Plaid connection retried — INTERNAL_SERVER_ERROR, API_ERROR, 0% success rate on ins_117067 — Baird-side issue confirmed | — |
| 2026-06-01 | Rocket Money also fails to connect to Baird — confirms infrastructure issue on Baird's end, not AFAS config | — |
| 2026-06-01 | Email drafted to Baird Online Support requesting status and alternative connection options | — |
| 2026-06-01 | Phase 4 tables created in Azure: securities, account_balances, liabilities, liability_balances, physical_assets, physical_asset_valuations, insurance_assets, insurance_asset_valuations, budget_targets | 9 tables |
| 2026-06-01 | accounts table ALTERed: added asset_class, display_name, owner, include_in_net_worth columns | 4 columns |
| 2026-06-01 | physical_assets seeded: HOUSE-WFB-BELLE, CAR-TSLA-NEBULA, CAR-TSLA-STORM, CAR-TSLA-TRINITY | 4 rows |
| 2026-06-01 | insurance_assets seeded: NWM-TOM (paid-up, CV $94,671.52), NWM-AMY (paid-up, CV $54,969.47) | 2 rows |
| 2026-06-01 | insurance_asset_valuations seeded: current cash values for both NWM policies | 2 rows |
| 2026-06-01 | liabilities seeded: Rocket Mortgage (2.65%, bal $167,356, maturity 2031-09-01), US Bank LOC/Baird Stock Loan (6%, bal $500,000) | 2 rows |
| 2026-06-01 | liability_balances seeded: current balance snapshots for both liabilities | 2 rows |
| 2026-06-01 | accounts backfilled: 10 accounts inserted (Associated Checking, Checking-Whit, Money Market, 3x Kids Savings, DF Checking, Chase, AMEX, Apple Card) | 10 rows |
| 2026-06-01 | SQL scripts numbered: 24_chase_merchant_patterns.sql, 25_vw_top_merchants_coalesce_fix.sql, 26_vw_transactions_clean_coalesce_fix.sql, 21_phase4_tables.sql, 22_phase4_seed_data.sql, 23_phase4_accounts_backfill.sql | — |

---

## 2026-06-01 (Session 4 — May CSV Import, Sign Audit, Data Quality)

| Date | Decision | Rows |
|------|----------|------|
| 2026-06-01 | Power BI scheduled refresh failed — root cause: Azure SQL Free tier auto-pause. DB resumed manually, refresh succeeded. Monthly manual resume procedure established. | — |
| 2026-06-01 | May 2026 Apple Card CSV imported (import_apple_csv.py) | 141 inserted, 6 updated |
| 2026-06-01 | May 2026 Apple enrichment: 122 matched by merchant_pattern, 19 unmatched | — |
| 2026-06-01 | enrich_apple_csv.py fixed: category_map lookup added as Step 2 — was missing entirely | — |
| 2026-06-01 | 19 unmatched May Apple merchants manually classified via SQL | 19 |
| 2026-06-01 | 12 new merchant_patterns added: Scarr's Pizza, The Lookout, Angel's Envy Distillery, Merle's Whiskey Kitchen, Classic Cinemas, Cluckers Charcoal, Jersey's Bar & Grill, Oakley, Online Trackman Store, Cozy Earth, Zoolora, Wauwatosa West Athletics | 12 patterns |
| 2026-06-01 | Liquor World added to category_map (Alcohol → Groceries/Alcohol) and merchant_patterns | — |
| 2026-06-01 | United Way / United Airlines pattern collision fixed — %UNITED WAY% pattern added at priority 3 | — |
| 2026-06-01 | United Way historical rows corrected: 3 rows reclassified from Travel/Flights → Gifts/Charity/Donation | 3 |
| 2026-06-01 | North Shore United merchant_normalized corrected to 'North Shore United' | 1 |
| 2026-06-01 | ISSUE-007 resolved: 235 positive expense rows flipped to negative across CHASE (91), ASSOCIATED_PERSONAL (54), AMEX (89), HSA (1) | 235 |
| 2026-06-01 | True duplicate deleted: Mariner North Resort (Chase, 2026-02-27, -$557.76) — pending→settled duplicate | 1 deleted |
| 2026-06-01 | True duplicate deleted: Phish Tickets (Chase, 2026-02-26, -$843.60) — pending→settled duplicate | 1 deleted |
| 2026-06-01 | 109 subcategory name violations fixed: ATM/Cash Spending (2), Clothing (27), Dining Out (1), Gifts/Charity (7), Groceries (59), Payment (13) | 109 |
| 2026-06-01 | Travel taxonomy stragglers fixed: Airlines → Flights (3), Hotels → Lodging (10) | 13 |
| 2026-06-01 | in_budget flags normalized: Large Purchases → 1 (1 row), HSA Deposit → 0 (60 rows), Payment → 0 (3 rows) | 64 |
| 2026-06-01 | category_map synced: Large Purchases in_budget → 1 (6 rows), HSA Deposit subcategory → EFT (2 rows) | 8 |
| 2026-06-01 | Transaction Review page added to Power BI report | — |
| 2026-06-01 | ISSUE-006 closed — May 2026 Apple CSV imported and fully categorized | — |
| 2026-06-01 | ISSUE-007 closed — full historical sign audit complete, all sources clean | — |

---

## 2026-05-27 (Session 3 continued — Enrichment & Sign Fixes)

| Date | Decision | Rows |
|------|----------|------|
| 2026-05-27 | enrich_transactions.py run with --unenriched-only: enriched 345 new Plaid rows | 345 |
| 2026-05-27 | enrich_apple_csv.py run: 241 enriched by pattern, 62 manually categorized via SQL | 303 |
| 2026-05-27 | 62 unmatched Apple merchants categorized manually via categorize_unmatched_apple.sql | 62 |
| 2026-05-27 | 3 Apple Card payment rows reclassified: Subscriptions → Payment/Apple Card/Other/in_budget=0 | 3 |
| 2026-05-27 | ASSOCIATED_PERSONAL Income sign fix: 9 negative income rows flipped to positive (2026 data) | 9 |
| 2026-05-27 | ASSOCIATED_PERSONAL Expense sign fix: 40 positive expense rows flipped to negative (2026 data) | 40 |
| 2026-05-27 | CHASE Expense sign fix: 145 positive expense rows flipped to negative (2026 data) | 145 |
| 2026-05-27 | AMEX Expense sign fix: 8 positive expense rows flipped to negative (2026 data) | 8 |
| 2026-05-27 | plaid_sync.py fixed: amount negated at ingestion time (-tx["amount"]) — Plaid debit=positive → schema debit=negative | — |
| 2026-05-27 | Finance-ingest-Tom-v6 redeployed with sign fix | — |
| 2026-05-27 | ISSUE-007 opened: historical sign audit needed for pre-2026 ASSOCIATED_PERSONAL, CHASE, AMEX rows | — |

---

## 2026-05-27 (Session 3 — Production Deployment)

| Date | Decision | Rows |
|------|----------|------|
| 2026-05-27 | Source label ASSOCIATED → ASSOCIATED_PERSONAL (consolidation) | 1,709 |
| 2026-05-27 | Source label AMEX DF → AMEX — already clean, 0 rows affected | 0 |
| 2026-05-27 | Production Plaid tokens acquired: Chase, Associated Personal, Amex | — |
| 2026-05-27 | Baird Plaid connection failed (ins_117067) — deferred to Phase 4 | — |
| 2026-05-27 | Azure Function deployed to Finance-ingest-Tom-v6 (East US 2, Flex Consumption) | — |
| 2026-05-27 | First production sync: Chase 913 rows, Associated Personal 111 rows, Amex 10 rows | 1,034 |
| 2026-05-27 | plaid_sync.py created as shared module — de-duplicated from timer_sync.py and http_ingest.py | — |
| 2026-05-27 | db.py updated: username/password auth locally, Managed Identity in Azure | — |
| 2026-05-27 | Apple Card CSV import script created (import_apple_csv.py) | — |
| 2026-05-27 | Apple Card March + April 2026 imported via CSV (303 rows inserted, 2 updated) | 305 |
| 2026-05-27 | Power BI report published to Power BI Service (My Workspace) | — |
| 2026-05-27 | Power BI scheduled refresh configured: daily 4:00 AM CT | — |
| 2026-05-27 | Total transactions as of session end: 15,090 | — |
