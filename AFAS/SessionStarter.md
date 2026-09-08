# AFAS Project — Session Starter
> **Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
> Repo: github.com/Whit19/dataforge-standards
**Load this file at the start of every session. Update pick-up pointer before closing.**
Last updated: 2026-09-08 (Session 19 sync)

---

## Project in One Line
Serverless financial pipeline: Plaid → Azure Function (Python) → Azure SQL (FinanceDB) → Power BI → AI Agents

---

## Environment
| Item | Value |
|------|-------|
| Azure SQL Server | financeauto-sql-server.database.windows.net |
| Database | FinanceDB |
| Auth | SQL username/password (Managed Identity deferred) |
| Project Root (local) | C:\DEV_Projects\AFAS |
| get_plaid_tokens.py | C:\DEV_Projects\AFAS\scripts\get_plaid_tokens.py — now supports Plaid Link update mode (existing-token field per institution) as of 2026-08-01 |
| Amount convention | Positive = credit/income, Negative = debit/expense |
| Manual override rule | category_source = 'manual' — pipeline NEVER overwrites |
| Pending column | Always use ISNULL(pending,0) = 0 in all queries |
| Azure SQL auto-pause | Free tier Serverless — resume manually via Azure Portal before 1st-of-month imports. Can also collide with the automated monthly timer itself (ISSUE-032) — function status reports "Succeeded" even when this happens, so don't trust status alone |

---

## ⚠️ Critical Incident — 2026-09-01 (read before assuming a "Resolved" issue actually stopped)

Twice this session, an issue marked **Resolved** in a past session (ISSUE-019's
plaid_category_raw JSON bug, ISSUE-023's BANK_FEES/LOAN_PAYMENTS sign
regression) turned out to have a fix that was correctly written and
committed **locally**, but never actually deployed to Finance-ingest-Tom-v6.
Both bugs kept running silently for a full month after being marked
Resolved — the historical backfill fixed existing bad rows at the time, but
the ingestion-side bug that kept creating new ones was never stopped.

**Lesson: "Resolved" describes a fix being drafted/committed, not deployed.
Any CC prompt touching a Function-App-deployed file (plaid_sync.py,
timer_sync.py, monthly_sync.py, http_ingest.py, balance_sync.py,
nwm_sync.py, db.py, requirements.txt) needs an explicit deploy step AND a
verification step against the actual deployed file content — Kudu is not
available on this project's Flex Consumption plan; use VS Code's Azure
Functions extension "Files (Read-only)" remote view instead.** See
DecisionLog 2026-09-01 and BestMethods.md.

Both ISSUE-019 and ISSUE-023 are now genuinely deployed, fixed, and
historically corrected as of this session. ISSUE-023's live verification
against a real new transaction is still pending (no qualifying
LOAN_PAYMENTS/BANK_FEES transaction has occurred since deploy) — check
against the next Amex autopay (~9/25), Chase autopay (~9/26), or mortgage
payment.

---

## ISSUE-032: Azure SQL auto-pause vs the automated monthly timer — MANUAL WORKAROUND ADOPTED (real-world test 2026-09-03)

The db.py retry-with-backoff on Azure SQL error 40613 was tested for real
2026-09-03 (paused FinanceDB, triggered http_ingest via Portal). **It never
fired** — the actual failure is ODBC `HYT00` ("Login timeout expired"), a
client-side timeout that happens *before* Azure SQL can return 40613, because
the driver gives up waiting for the paused DB to wake. The 40613 retry works
exactly as designed; it just doesn't cover this failure shape, and catching
`HYT00`/`08001` would need the ODBC login timeout widened first.

**Decision: keep the manual monthly procedure as the standing solution** —
resume FinanceDB in the Portal, check freshness, manually retrigger
http_ingest / http_balance_ingest / http_nwm_sync if the automated run
failed. Not a broken fix — deliberately scoped. See ISSUE-032 (Resolved) and
DecisionLog 2026-09-03. **Now written up as the "Monthly DB Resume & Sync
Verification Procedure" below** (added Session 19).

**Lesson: function-level "Succeeded" status does not prove the internal SQL
work succeeded. Check Application Insights traces, not just the
top-level status, when verifying an automated sync actually ran.**

---

## Process notes — read before trusting taxonomy_audit.py output

1. **taxonomy_audit.py now parses Category_Taxonomy.md live** (ISSUE-041,
   fixed 2026-09-08 — AFAS 6c652ae). No more hardcoded canonical dict; it
   reads the doc's own `## Full Taxonomy` block on every run and exits 2 if
   that can't be parsed. So its "undocumented combo" flags are trustworthy
   again — still re-run the audit fresh at session start rather than
   trusting a prior session's count.
2. **Azure SQL's default collation is case-insensitive.** A taxonomy_audit.py
   flag like `Payment/AMEX` vs. canonical `Payment/Amex` can be a pure
   Python-string-comparison artifact, not a real distinct value in the
   database (confirmed for this exact case). Check with `COLLATE
   Latin1_General_CS_AS` before treating a "cosmetic casing" flag as real.
3. **The pattern matcher only treats leading/trailing `%` as wildcards.**
   A `%` embedded mid-pattern (`%CASK%ALE%`) is deleted by
   `pattern.replace("%","")` and the pattern silently can't match. 31 such
   patterns were cleaned up 2026-09-08 (script 86); if you add a pattern,
   keep `%` at the ends only.

---

## Phase Status
| Phase | Description | Status |
|-------|-------------|--------|
| Phase 1 | Plaid ingestion → Azure SQL | ✅ Complete |
| Phase 2 | Enrichment, normalization, schema cleanup | ✅ Complete |
| Phase 3 | Automation, Power BI reporting | ✅ Complete |
| Phase 4 | Accounts/balances, investments/holdings | 🔄 In Progress |
| Phase 5 | AI agent layer | ⏳ Not Started |

---

## Pick Up Here — Next Session

1. **Assign categories to the 31 APPLE Uncategorized transactions (ISSUE-043).**
   $3,586.58 total, all real identifiable merchants (Starlink, Jostens,
   Bay Books, several liquor stores, Blinktechus, Magbak, My Martinizing,
   bbcmkids.org, …). Assign via `category_source = 'manual'` (set
   `type`/`in_budget`/`category_reviewed = 1` per BestMethods) or add
   `merchant_patterns` rows for recurring ones (Starlink → Bills &
   Utilities/Internet is obvious). Then re-run `taxonomy_audit.py` + the
   Power BI Needs Review page. Separately decide what to do with the 61 HSA
   "Normal Distribution" rows ($9,885.16) — the CSV doesn't say what each
   distribution paid for.
2. **Watch the next real Apple Card / HSA CSV import closely.** The Session
   19 enrichment consolidation (single enricher, date cutoff removed,
   `apply_fallback` change) was verified by inspection + unit tests but
   **not by a live prod run** (the harness blocked it). Confirm the first
   real `import_apple_csv.py` → `enrich_transactions.py --unenriched-only`
   cycle behaves correctly — this project's history (ISSUE-019/023/018) is
   "verified by inspection, still had a hole."
3. **Continue the `taxonomy_audit.py` backlog (ISSUE-012)** — re-run fresh
   at session start. Still open (see IssuesTracker ISSUE-012 for the full
   list): `%UBER CASH%`/`%UBER%` shadow (`%BP%` deliberately left as-is per
   Tom); `Travel / "Travel activities"` lowercase casing (~44 patterns);
   several add-vs-normalize subcategory decisions (Children Birthday
   Party/Photos/Boating, Other Income Dividends/Pension/Tax Refund,
   Work-Expense Education/Professional Dev); easy category_map renames
   (Children/School → School Lunch, Taxes/Federal Tax → Federal, etc.);
   Car/Rideshare → Travel/Transportation; the remaining `[same dest]`
   Check 4 pairs (cosmetic) and the lower-dollar `[DIFFERENT DEST]` ones;
   add Check 5 for subcategory==category mirrors (ISSUE-014).
4. **Add `vw_potential_duplicates` as a Power BI Data Health tile** — the
   view exists and works (0 legacy_vs_plaid, catches pending_vs_settled);
   Tom wants it visible on the existing Data Health page.
5. **Session 17's ad-hoc SQL is still not reconstructed as numbered scripts**
   — the two views (`vw_needs_review`, `vw_potential_duplicates`) and
   Session 17's correction/pattern-rename SQL remain uncommitted. Sessions
   18-19 (scripts 72-87) ARE committed. Decide whether to reconstruct
   Session 17's or accept the gap.
6. **Sunnyside 651 Guerrero St watch (~2027-01).** `%SUNNYSIDE%651%` was
   deactivated in script 86; the SF cannabis-dispensary charge (manually
   filed Personal Care/Health & Wellness) will now fall to the generic
   `%SUNNYSIDE%` (→ Dining Out/General) or to review. If it recurs, decide
   whether it needs its own (contiguous-text) pattern.

---

## Active Data Issues
| Issue | Priority | Description | Next Step |
|-------|----------|-------------|-----------|
| ISSUE-012 | Medium | Systemic taxonomy drift found by `taxonomy_audit.py`. Sessions 18-19 fixed a large chunk (Work-Expense audit, ~25 shadow/destination bugs, 31 dead mid-`%` patterns, ISSUE-040/042); real gaps remain (see IssuesTracker) | Work the backlog — Pick Up Here #3 |
| ISSUE-043 | Medium | 31 APPLE transactions ($3,586.58) at `category = 'Uncategorized'` — real merchants needing category assignment. Plus 61 HSA "Normal Distribution" rows ($9,885.16), lower priority | Assign categories / add patterns — Pick Up Here #1 |
| ISSUE-016 | Medium | run_log missing entries for all daily transaction syncs | Add run_log writes to plaid_sync.py |

---

## Production Plaid Tokens (live, verified 2026-09-01)
| Token Variable | Institution | Status |
|----------------|-------------|--------|
| PLAID_ACCESS_TOKEN_CHASE | Chase (ins_56) | ✅ Live |
| PLAID_ACCESS_TOKEN_ASSOCIATED_PERSONAL | Associated Bank Personal (ins_116823) | ✅ Live |
| PLAID_ACCESS_TOKEN_AMEX | American Express | ✅ Live — reconnected via Link update mode 2026-09-01 (was ITEM_LOGIN_REQUIRED, second occurrence for this institution — see DecisionLog 2026-09-01) |
| PLAID_ACCESS_TOKEN_NWM_TOM | NW Mutual Tom whole life | ✅ Live — reconnected via Link update mode 2026-08-01 (was ITEM_LOGIN_REQUIRED) |
| PLAID_ACCESS_TOKEN_NWM_AMY | NW Mutual Amy whole life | ✅ Live — reconnected via Link update mode 2026-08-01 (was ITEM_LOGIN_REQUIRED) |
| PLAID_ACCESS_TOKEN_PRINCIPAL | Baird Profit Sharing and Savings Plan (401k), owner Amy — recordkept via Principal | ✅ Live — first connection 2026-08-01 (ISSUE-009 resolved after being open since 2026-06-02). Real account name: "BAIRD PROFIT SHARING AND SAVINGS PLAN" / "Prft Shr 401(K) Def Thrift". This resolves the "Principal 401k" vs "401k Baird Profit Share" naming ambiguity — same account. |
| Baird (ins_117067) | All Baird accounts (transactions/investments via Plaid) | ⚠️ Still blocked — CSV fallback operational (unrelated to Principal 401k, which connects independently) |
| Rocket Mortgage | Mortgage | ❌ Confirmed unsupported — manual updates only |

---

## Python Scripts (key files)
| File | Purpose | Status |
|------|---------|--------|
| plaid_sync.py | Shared sync module | ✅ Both known bugs fixed AND deployed 2026-09-01 (confirmed via VS Code's "Files (Read-only)" remote view — both had been drafted-but-undeployed since Session 14): (1) plaid_category_raw now extracts a clean category string instead of storing full JSON; (2) INFLOW_CATEGORIES no longer includes BANK_FEES/LOAN_PAYMENTS (ISSUE-023). Still does not write to run_log (ISSUE-016, carried over). |
| enrich_transactions.py | **The single enricher for every source** (Plaid CHASE/ASSOCIATED_PERSONAL/AMEX, Apple Card CSV, HSA CSV) as of 2026-09-08 | ✅ 2026-09-08 (AFAS b1a3ef3): `enrich_apple_csv.py` + `enrich_hsa_csv.py` retired — this file already had no source filter. `load_transactions()` date cutoff removed. `apply_fallback()` now only fills `in_budget`/`type` where NULL. **Matcher note:** only leading/trailing `%` are wildcards; a mid-pattern `%` is dead. Earlier: `enrich_from_history()` carry-forward step (2026-09-03, AFAS 557cdd1 + 4f0c052); write-back scoped to changed rows (ISSUE-035); apply_fallback() defaults + `--unenriched-only` retry (2026-08-03). |
| scripts/taxonomy_audit.py | Read-only taxonomy-drift diagnostic (4 checks: undocumented category/subcategory combos in merchant_patterns / category_map / transactions; same-priority pattern shadowing). AFAS 13b959e. | ✅ 2026-09-08 (ISSUE-041, AFAS 6c652ae): now parses `Category_Taxonomy.md`'s `## Full Taxonomy` block live every run — no hardcoded dict, exits 2 on parse failure. Backlog = ISSUE-012. |
| scripts/plaid_transaction_name_check.py | **NEW 2026-09-03** — read-only Plaid /transactions/get diagnostic (CHASE/ASSOCIATED_PERSONAL/AMEX); prints raw name / merchant_name / PFC. No writes, not in any pipeline. AFAS 2ad1bf2. | ✅ Live |
| plaid_client.py | Plaid SDK wrapper | ✅ Ready |
| balance_sync.py | Associated balance pull | ✅ Live |
| nwm_sync.py | NWM Tom + Amy cash value sync | ✅ Live — both Items reconnected 2026-08-01 |
| principal_sync.py | **NEW 2026-08-01** — Pulls Principal/Baird 401k holdings via Plaid Investments (/investments/holdings/get), upserts dbo.accounts/dbo.securities/dbo.holdings. Standalone local script only — not wired into http_ingest.py or any timer yet (carried-over item, not in this session's Pick Up Here top 5). | ✅ Working — confirmed live: 1 account, 11 securities, 11 holdings, $2,096,195.86 total value |
| import_hsa_transactions.py | Imports Bank of America HSA cash-ledger CSV (Run_Monthly/imports/HSA/HSA_Transactions_*.csv) into dbo.transactions. type/in_budget set deterministically at import (not enrichment-dependent) — 4 known non-spending description types whitelisted, everything else treated as real spending/income typed by amount sign. category/subcategory left NULL for `enrich_transactions.py`. Created 2026-08-01. transaction_id hashes normalized (parsed) date/amount — fixed 2026-09-01 after BofA export-formatting drift caused ~400 duplicate groups; watermark check warns on unexpectedly-new IDs. UTF-8 stdout fix applied. | ✅ Live — 461 canonical rows |
| import_hsa_holdings.py | Imports Bank of America HSA "Fund Summary" CSV (value-only, no units/price available in this export) into dbo.holdings. Snapshot date parsed from filename. Created 2026-08-01. UTF-8 stdout fix applied 2026-09-01. | ✅ Live — 2 holdings, $24,163.69 total |
| import_baird_holdings.py | Baird holdings CSV → baird_holdings | ✅ Ready — Total-row detection bug fixed 2026-08-01 (was checking wrong column) |
| monthly_sync.py | Monthly timer (1st of month, 03:00 UTC) — balance_sync + nwm_sync | ✅ Deployed — did not actually run successfully for at least 6 weeks prior to 2026-08-01 due to the dotenv outage; now fixed. Collides with Azure SQL auto-pause if the DB isn't warm at trigger time (ISSUE-032, occurred 2026-09-01) |
| import_apple_csv.py | Apple Card monthly CSV import. Lives in Run_Monthly\, auto-discovers files, auto-moves to imported\. Stores Apple's CSV "Category" column in `plaid_category_raw` (no Plaid involvement — the column name is a legacy misnomer). | ✅ 2026-09-08 (AFAS b1a3ef3): inserts `category_source = NULL` (was `'historical'` — collided with the ~6k genuinely backfilled `'historical'` rows); MERGE re-default guard now treats only `NULL`/`'unmatched'` as "not yet enriched". Enrichment is now `enrich_transactions.py` (the Apple-specific enricher was retired). 2026-09-01: MERGE CASE protection for already-enriched rows; UTF-8 stdout fix. |
| timer_sync.py | Monthly timer trigger (1st @ 03:00 UTC) | ✅ Ready — confirmed 2026-09-01 this is the correct, deliberate design (not a stale "Daily" leftover); TechnicalArchitecture.md corrected to match |
| http_ingest.py | Manual HTTP triggers — http_ingest, http_balance_ingest, http_nwm_sync | ✅ Ready |
| get_plaid_tokens.py | Local Flask tool for Plaid token acquisition | ✅ Ready — added Plaid Link update-mode support (existing-token field per institution) and NW Mutual Tom/Amy cards, both 2026-08-01 |
| db.py | DB connection | ✅ Ready — retry-with-backoff on Azure SQL error 40613 (auto-pause collision) added 2026-09-02, confirmed present in live code (ISSUE-032). Not yet exercised against a real auto-pause event — see Pick Up Here #5. |

---

## Monthly Baird Holdings Import Procedure
1. Log into Baird Online, export all accounts holdings CSV
2. Add `Account Name` column — fill with correct canonical account name per row (**use "MAIN - BKG" for the main brokerage, not "MAIN - Brokerage" — August 2026 export used the wrong name and required a SQL correction**)
3. Add `Date` column — fill all rows with month-end date
4. Add manual cash sweep row if needed (symbol=CASH, Asset Classification=Cash and Cash Equivalents)
5. Drop file into `C:\DEV_Projects\AFAS\Run_Monthly\imports\baird\` as `holdings_*.csv` (any name starting with "holdings_" works)
6. Run `python import_baird_holdings.py` from Run_Monthly — note: this script does NOT auto-move processed files (unlike import_apple_csv.py), so archive it manually afterward
7. Update physical asset valuations if needed (Zillow for house, KBB for Teslas — note KBB "typical mileage" figures can be significantly off for high-mileage vehicles; get actual VIN/mileage-specific values when possible)

## Baird Account Name Conventions (canonical)
| Account | Account Name |
|---------|-------------|
| Main Brokerage | MAIN - BKG |
| Main PIM | MAIN - PIM |
| IRA Tom | IRA - TOM |
| IRA Roth Tom | IRA Roth - TOM |
| IRA Baird Stock | IRA - Baird Stock |
| IRA ROTH Baird Stock | IRA ROTH - Baird Stock |
| Baird Stock | BAIRD Stock |
| Baird Capital | BAIRD Capital |
| 529 Alex | 529 - ALEX |
| 529 Brooke | 529 - BROOKE |
| 529 Whit | 529 - WHIT |

---

## Monthly DB Resume & Sync Verification Procedure
(ISSUE-032 — the auto-pause/timer collision has no code fix; this is the standing manual solution.)

1. **Resume FinanceDB in Azure Portal** (Free tier Serverless auto-pauses — check first, before the 1st-of-month automated timer or any manual import runs).
2. **Check source freshness** — query `vw_source_freshness` (or `dbo.plaid_sync_state` directly) for each source's `max_date` / `days_since_last_transaction`. Do not trust the automated timer's "Succeeded" status alone — function-level status does not reflect whether the internal SQL work completed (ISSUE-032, 2026-09-01). If a source looks stale, check **Application Insights traces** directly for "X functions loaded" / actual sync trace lines — that's ground truth, not the Portal Functions blade.
3. **If the automated run failed or a source is stale**, manually retrigger via Azure Portal Test/Run: `http_ingest`, `http_balance_ingest`, `http_nwm_sync` as needed.
4. **After any CSV import (Apple Card, HSA) or manual retrigger**, run `python enrich_transactions.py --unenriched-only`. As of 2026-09-08 this is the **single enricher for all sources** — `enrich_apple_csv.py` / `enrich_hsa_csv.py` were retired and deleted (AFAS b1a3ef3); there is no separate per-source enrichment step.
5. **The db.py retry-with-backoff on Azure SQL error 40613 does not cover this failure** — the real failure mode is ODBC `HYT00`/`08001` (client-side login timeout), which occurs before Azure SQL can return 40613. No code-level fix without widening the ODBC login timeout first. Resuming the DB manually before the 1st of the month remains the solution.

---

## Monthly Apple Card Import Procedure
1. Resume FinanceDB in Azure Portal (Free tier auto-pauses — check first; see the DB Resume procedure above)
2. Export Apple Card CSV for the month, drop into Run_Monthly\imports\AppleCC\
3. Run `python import_apple_csv.py` from Run_Monthly (no filename needed — auto-discovers)
4. Run `python enrich_transactions.py --unenriched-only` (the single enricher — the Apple-specific script was retired 2026-09-08)
5. Review any unmatched merchants and classify via SQL — remember dual-use merchants (e.g. gas stations that also sell groceries/snacks) should be classified manually per-transaction, not given a blanket merchant_patterns entry, same logic as the Work-Expense dual-use rule

---

## SQL Scripts — Run Order Reference
Phase 1–4 through Session 11: 01–45_*.sql (see DecisionLog for details)

Session 12 (2026-08-01):
                           46_physical_asset_valuations_august2026.sql
                           47_us_bank_loc_balance_august2026.sql
                           48_mortgage_rate_correction_and_backfill.sql
                           49_mortgage_balance_august2026.sql
                           50_baird_main_brokerage_rename_fix.sql
                           51_holdings_table_add_missing_columns.sql

Session 13 (2026-08-01):   52 through 58 (see DecisionLog 2026-08-01 Session 13 for detail)

Session 14 (2026-08-03):
                           59_backfill_plaid_category_raw_json_fix.sql
                           60_category_map_generic_fallbacks_backlog.sql
                           61_merchant_patterns_recognized_vendors_backlog.sql

Session 15 (2026-09-01):
                           55_hsa_ticker_update.sql (run 2026-08-01, file itself
                             not committed until this session — repo gap fixed)
                           62_hsa_transaction_duplicate_cleanup.sql
                           63_august_uncategorized_backlog.sql
                           64_issue023_sign_correction.sql
                           65_apple_merchant_pattern_and_duplicate.sql

Session 16 (2026-09-02):
                           66_backfill_missing_account_id_pre_fe.sql
                           67_baird_main_brokerage_rename_fix_ju.sql
                           68_hsa_orphaned_null_account_id_dupli.sql
                           69_marquette_payroll_misclassificatio.sql
                           70_venmo_check_review_backlog_classif.sql
                           71_category_reviewed_reset.sql

Session 17 (2026-09-03): **no numbered scripts committed.** All SQL this
session was run ad-hoc: `CREATE VIEW dbo.vw_needs_review` and
`dbo.vw_potential_duplicates`; the 421-row Needs Review backlog cleanup;
taxonomy-drift corrections (U-club / Airlines / Hotels / Fitness pattern
renames, `%MARQUETTE UN%` priority → 40); 5 new merchant_patterns
(WISCONSINGOV, Dave's Hot Chicken, ATM W D U S BANK, 1-800-FLOWERS,
INDULGENCE CHOCOLAT); ~40 manual transaction corrections; 10 duplicate-row
deletions. **Gap still open** — see Pick Up Here #9: decide whether to
reconstruct these as scripts, or accept the gap.

Session 18 (2026-09-04):
                           72_work_expense_pattern_cleanup.sql
                           73_shadow_pattern_priority_fixes.sql
                           74_ascension_donation_pattern_removal.sql
                           75_shadow_pattern_batch2.sql
                           76_issue040_gifts_subcategory_normali.sql
                           77_bucket1_taxonomy_renames.sql
                           78_transfer_savings_pattern_fix.sql

Session 19 (2026-09-08):
                           79_marquette_sixt_taxonomy_cleanup_session19.sql
                           80_shadow_pair_priority_fixes_session19.sql
                           81_pattern_destination_fixes_session19.sql
                           82_issue042_books_gaming_manual_recategorize.sql
                           83_issue039_act_pattern_exact_match.sql
                           84_issue038_apple_enrichment_cleanup.sql
                           85_issue038_irs_pattern_exact_match.sql
                           86_python_matcher_broken_pattern_cleanup_session19.sql
                           87_issue038_shopping_category_map_remap.sql

Current high watermark: **87** (confirmed live 2026-09-08 — re-confirm
live rather than trust this number next session too. Note: `taxonomy_audit.py`'s
ISSUE-041 fix is a code change committed to AFAS `main` as 6c652ae, not a
numbered SQL script. Session 17's ad-hoc SQL is still not reflected in any
numbered file.)

---

## Phase 4 Tables
| Table | Status | Notes |
|-------|--------|-------|
| securities | ✅ Live | Now populated via principal_sync.py (11 securities as of 2026-08-01) |
| holdings | ✅ Live | Was missing cost_basis and updated_at columns despite being documented — added 2026-08-01 (script 51). Populated via principal_sync.py (11 rows, Principal 401k) and import_hsa_holdings.py (2 rows, HSA Bank of America, value-only — no units/price in that export). |
| account_balances | ✅ Live | Associated 7 accounts |
| liabilities | ✅ Live | Rocket Mortgage (rate corrected to 2.625%, origination_principal backfilled), US Bank LOC |
| liability_balances | ✅ Live | Mortgage: $162,472 as of 2026-08-01 (YTD interest $2,636.93, YTD principal $16,998.91). US Bank LOC: $500,000, confirmed unchanged from 2026-06-01. |
| physical_assets | ✅ Live | WFB-Belle, TSLA Nebula/Storm/Trinity |
| physical_asset_valuations | ✅ Live | Updated 2026-08-01: House $1,490,400 (Zillow), Nebula $79,000, Storm $23,000, Trinity $20,000 (KBB, mileage-specific) |
| insurance_assets | ✅ Live | NWM-TOM, NWM-AMY |
| insurance_asset_valuations | ✅ Live | Both reconnected 2026-08-01. Tom $95,211.97, Amy $55,281.18 |
| baird_holdings | ✅ Live | 737 rows as of 2026-08-01 snapshot ($7,364,285.34 total, reconciled to Baird's own CSV total + manual cash row to the penny). MAIN - Brokerage naming corrected to MAIN - BKG. |
| security_sectors | ✅ Live | 115 tickers mapped |
| budget_targets | ✅ Created | ⏳ Still not seeded |

---

## Data Health (Power BI)

Self-serve report page (added Session 13) built on:
- **vw_source_freshness** — row_count / max_date / min_date /
  days_since_last_transaction per source
- **vw_category_health** — uncategorized / manual_override / low_confidence /
  subcategory_mirror_violation counts per source
- **vw_needs_review** (added Session 17) — every `category_reviewed = 0` row,
  with a computed `suggested_category` / `suggested_subcategory` (from
  merchant_patterns + historical carry-forward, `'unmatched'` sources
  excluded) and a `review_priority` tier: 1 = no suggestion + currently
  Uncategorized; 2 = no suggestion + has a value; 3 = suggestion disagrees
  with current; 4 = suggestion matches current. Drives the **Needs Review**
  page. Backlog was cleared 421 → 0 on 2026-09-03; the view stays live for
  future rows. **Any table visual on this page must include `transaction_id`
  (hidden) with numeric columns set to "Don't summarize"** — without the key,
  the visual silently groups + SUMs `review_priority`/`amount` (see
  BestMethods).
- **vw_potential_duplicates** (added Session 17) — flags two shapes:
  `legacy_vs_plaid` (a legacy bulk-import row sharing date/amount with a row
  that has a genuine non-null `pending` flag) and `pending_vs_settled` (the
  known pending/settled pattern). As of 2026-09-03: 0 legacy_vs_plaid (all 5
  real ones cleaned), 0 pending_vs_settled (4 cleaned). **Not yet a Power BI
  tile** — planned follow-up.

All standalone, no Calendar relationship. Check this page first before
running ad-hoc SQL to verify a data load — that's what it's for.

---

## Net Worth Summary (as of 2026-08-01 — first fully verified figure; previous $8.27M figure never included Principal 401k)
| Category | Value | Confidence |
|----------|-------|------------|
| Investment — Baird direct | $7,364,285.34 | Verified — reconciled to CSV |
| Investment — Principal/Baird 401k | $2,096,195.86 | Verified — first connection, live data |
| Physical Assets | $1,612,400.00 | Verified — fresh Zillow/KBB values |
| Insurance (NWM Tom + Amy) | $150,493.15 | Verified — both reconnected and synced |
| Cash (Associated — 7 accounts) | ~$150,085 (2026-06-17 figure) | **NOT reconfirmed this session** — balance sync ran successfully today (7/7 upserted) but total not re-queried. Query dbo.account_balances for exact current figure before reporting. |
| Liabilities | -$662,472.00 (Mortgage $162,472 + LOC $500,000) | Verified |
| **Total Net Worth (approx, pending cash confirmation)** | **~$10.71M** | Up from $8.27M — increase is almost entirely the newly-discovered Principal 401k account, not market movement |
*Excludes HSA-Baird and any other non-canonical Baird accounts (never captured in any import — see ISSUE-017).*
