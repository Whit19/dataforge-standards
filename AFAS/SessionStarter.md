# AFAS Project — Session Starter
> **Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
> Repo: github.com/Whit19/dataforge-standards
**Load this file at the start of every session. Update pick-up pointer before closing.**
Last updated: 2026-08-01 (Session 13 sync)

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
| Azure SQL auto-pause | Free tier Serverless — resume manually via Azure Portal before 1st-of-month imports |

---

## ⚠️ Critical Incident — 2026-08-01 (read before assuming any automated sync is current)
The deployed Function App was silently crash-looping on every cold start for
an unknown but likely multi-week period (`ModuleNotFoundError: No module
named 'dotenv'` — python-dotenv was added to db.py's local dependencies in
the July session but never added to requirements.txt, so it worked locally
but was never vendored into the deployed package). This caused the trigger
indexer to find zero functions, meaning **no automated sync of any kind —
daily transactions, monthly balances, monthly NWM valuations — actually ran
for an extended period**, despite `plaid_sync_state` and stale run_log data
making things look superficially current. Fixed by adding python-dotenv to
requirements.txt and redeploying. Confirmed via Application Insights logs
(host now loads all 5 functions cleanly). See DecisionLog 2026-08-01 for
full diagnostic trail.

**Lesson: do not trust that automated syncs are running just because past
data looks current. Spot-check via manual /api/ingest calls and compare
MAX(date) against today, not just against the last known-good session.**

---

## ⚠️ Known Issue — Power BI Monthly Spend shows Apple-only data (2026-08-01)

The Monthly Spend report page currently shows no June/July 2026 data for
any source except Apple, despite Chase, Associated Personal, and Amex all
confirmed current and present in dbo.transactions for those months (see
vw_source_freshness — all three show max_date in late July/August 2026).
This is very likely a Power BI-side issue (stale refresh, a leftover
slicer/filter, or a Calendar relationship gap) rather than a data gap —
but not yet diagnosed. See ISSUE-019.

**Do not assume this means transactions are missing** — the SQL-level data
is confirmed present. Investigate the report/dataset side first.

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

1. **Investigate ISSUE-019** — Power BI Monthly Spend page showing
   Apple-only data for June/July 2026. Start with a manual dataset
   refresh and check for a stuck slicer/filter before assuming a deeper
   issue with vw_monthly_spend or the Calendar relationship.
2. **Run the ISSUE-022 diagnostic query** — confirm what the pre-existing
   2026-03-05/07 HSA merchant_patterns batch actually covers, to know
   whether this session's HSA categorization work filled real gaps or was
   mostly redundant with earlier (undocumented) work on this account.
3. **Investigate ISSUE-020** — category_confidence not populated for
   CHASE/ASSOCIATED_PERSONAL/AMEX (Plaid-synced) rows the way it is for
   APPLE/HSA (CSV-imported) rows. Check enrich_transactions.py directly
   rather than continuing to infer from the symptom.
4. **Wire principal_sync.py into the automated pipeline** (carried over
   from Session 12) — still a standalone local script only.
5. **ISSUE-012 — Category_Taxonomy audit** (carried over, not touched this
   session).
6. **ISSUE-014 — subcategory-mirror recurrence root cause** (carried
   over, not touched this session).
7. **ISSUE-016 — add run_log writes to plaid_sync.py's daily sync**
   (carried over, not touched this session).
8. **ISSUE-017 — HSA-Baird and other non-canonical Baird accounts never
   imported** (carried over — distinct from the new Bank of America HSA
   built this session; HSA-Baird is a Baird-administered account tracked
   via the Baird CSV pipeline, unrelated to HSA-BOFA-MANUAL).
9. **ISSUE-021 — fix import_baird_holdings.py's broken local.settings.json
   path** (low priority, currently harmless dead code).
10. **budget_targets table still not seeded** (carried over).
11. **Connect Phase 4 Power BI views** (Net Worth, Holdings Summary, Asset
    Allocation, Liability Summary) (carried over, still not done).
12. **Decide whether to fetch actual HSA holdings units/price monthly**
    from the dashboard screenshot, or continue tracking value-only from
    the Fund Summary export (current approach).

---

## Active Data Issues
| Issue | Priority | Description | Next Step |
|-------|----------|-------------|-----------|
| NOTE | — | 8 Apple Uncategorized rows intentionally parked — category_source = 'manual', in_budget = 1 | Owner review when ready |
| NOTE | — | run_log missing entries for all daily transaction syncs (see Pick Up Here #3) | Add run_log writes to plaid_sync.py |

---

## Production Plaid Tokens (live, verified 2026-08-01)
| Token Variable | Institution | Status |
|----------------|-------------|--------|
| PLAID_ACCESS_TOKEN_CHASE | Chase (ins_56) | ✅ Live |
| PLAID_ACCESS_TOKEN_ASSOCIATED_PERSONAL | Associated Bank Personal (ins_116823) | ✅ Live |
| PLAID_ACCESS_TOKEN_AMEX | American Express | ✅ Live — reconnected via Link update mode 2026-08-01 (was ITEM_LOGIN_REQUIRED) |
| PLAID_ACCESS_TOKEN_NWM_TOM | NW Mutual Tom whole life | ✅ Live — reconnected via Link update mode 2026-08-01 (was ITEM_LOGIN_REQUIRED) |
| PLAID_ACCESS_TOKEN_NWM_AMY | NW Mutual Amy whole life | ✅ Live — reconnected via Link update mode 2026-08-01 (was ITEM_LOGIN_REQUIRED) |
| PLAID_ACCESS_TOKEN_PRINCIPAL | Baird Profit Sharing and Savings Plan (401k), owner Amy — recordkept via Principal | ✅ Live — first connection 2026-08-01 (ISSUE-009 resolved after being open since 2026-06-02). Real account name: "BAIRD PROFIT SHARING AND SAVINGS PLAN" / "Prft Shr 401(K) Def Thrift". This resolves the "Principal 401k" vs "401k Baird Profit Share" naming ambiguity — same account. |
| Baird (ins_117067) | All Baird accounts (transactions/investments via Plaid) | ⚠️ Still blocked — CSV fallback operational (unrelated to Principal 401k, which connects independently) |
| Rocket Mortgage | Mortgage | ❌ Confirmed unsupported — manual updates only |

---

## Python Scripts (key files)
| File | Purpose | Status |
|------|---------|--------|
| plaid_sync.py | Shared sync module | ✅ Live — does not write to run_log (see Pick Up Here #3) |
| plaid_client.py | Plaid SDK wrapper | ✅ Ready |
| balance_sync.py | Associated balance pull | ✅ Live |
| nwm_sync.py | NWM Tom + Amy cash value sync | ✅ Live — both Items reconnected 2026-08-01 |
| principal_sync.py | **NEW 2026-08-01** — Pulls Principal/Baird 401k holdings via Plaid Investments (/investments/holdings/get), upserts dbo.accounts/dbo.securities/dbo.holdings. Standalone local script only — not wired into http_ingest.py or any timer yet (see Pick Up Here #4). | ✅ Working — confirmed live: 1 account, 11 securities, 11 holdings, $2,096,195.86 total value |
| import_hsa_transactions.py | Imports Bank of America HSA cash-ledger CSV (Run_Monthly/imports/HSA/HSA_Transactions_*.csv) into dbo.transactions. type/in_budget set deterministically at import (not enrichment-dependent) — 4 known non-spending description types whitelisted, everything else treated as real spending/income typed by amount sign. Created 2026-08-01. | ✅ Live — 457 rows imported, 0 uncategorized, 0 untyped |
| enrich_hsa_csv.py | Enrichment for HSA CSV import — mirrors enrich_apple_csv.py's pattern/historical fallback chain, scoped to source='HSA'. Created 2026-08-01. | ✅ Live |
| import_hsa_holdings.py | Imports Bank of America HSA "Fund Summary" CSV (value-only, no units/price available in this export) into dbo.holdings. Snapshot date parsed from filename. Created 2026-08-01. | ✅ Live — 2 holdings, $24,163.69 total |
| import_baird_holdings.py | Baird holdings CSV → baird_holdings | ✅ Ready — Total-row detection bug fixed 2026-08-01 (was checking wrong column) |
| monthly_sync.py | Monthly timer (1st of month, 03:00 UTC) — balance_sync + nwm_sync | ✅ Deployed — did not actually run successfully for at least 6 weeks prior to 2026-08-01 due to the dotenv outage; now fixed |
| enrich_transactions.py | Enrichment engine (Plaid transactions) | ✅ Ready |
| enrich_apple_csv.py | Enrichment for Apple Card CSV | ✅ Ready |
| import_apple_csv.py | Apple Card monthly CSV import | ✅ Ready — now lives in Run_Monthly\, auto-discovers files, auto-moves to imported\ subfolder |
| timer_sync.py | Daily timer trigger | ✅ Ready — schedule confirmed correct after dotenv fix and redeploy (0 0 3 1 * * for monthly, matches monthly_sync cadence decision) |
| http_ingest.py | Manual HTTP triggers — http_ingest, http_balance_ingest, http_nwm_sync | ✅ Ready |
| get_plaid_tokens.py | Local Flask tool for Plaid token acquisition | ✅ Ready — added Plaid Link update-mode support (existing-token field per institution) and NW Mutual Tom/Amy cards, both 2026-08-01 |
| db.py | DB connection | ✅ Ready |

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

## Monthly Apple Card Import Procedure
1. Resume FinanceDB in Azure Portal (Free tier auto-pauses — check first)
2. Export Apple Card CSV for the month, drop into Run_Monthly\imports\AppleCC\
3. Run `python import_apple_csv.py` from Run_Monthly (no filename needed — auto-discovers)
4. Run `python enrich_apple_csv.py`
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

Current high watermark: **51**

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

New self-serve report page (added Session 13) built on two new views:
vw_source_freshness (row_count/max_date/min_date/days_since_last_transaction
per source) and vw_category_health (uncategorized/manual_override/
low_confidence/subcategory_mirror_violation counts per source). Both
standalone, no Calendar relationship. Conditional formatting (Rules-based
background color) flags days_since_last_transaction and
subcategory_mirror_violation_count. Check this page first before running
ad-hoc SQL to verify a data load — that's what it's for.

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
*Excludes HSA-Baird and any other non-canonical Baird accounts (never captured in any import — see Pick Up Here #2).*
