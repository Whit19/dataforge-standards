# AFAS Project — Technical Architecture
**Update this file when any component, connection, or configuration changes.**
Last updated: 2026-09-17 (Session 22: first live run-through of MonthlyProcedure.md — timer_sync/monthly_sync deregistered in favor of one manual http_monthly_ingest_all route; enrich_transactions.py's change-detection comparison fixed (was positional, not by transaction_id — flagged 9,604 untouched rows as changed); new vw_account_freshness (account-level freshness + latest_value, covers NWM/401k which vw_source_freshness never did); get_plaid_tokens.py pre-fills tokens; Tesla valuations switched to a flat monthly depreciation policy; the 61-row HSA ISSUE-043 backlog resolved via a previously-uncaptured Consumer Note CSV field)

---

## System Overview

AFAS (Automated Financial Analysis System) is a serverless pipeline that ingests financial data from Plaid, enriches and normalizes it in Azure, stores it in Azure SQL, and surfaces it through Power BI and AI agents.

```
Plaid API
    │
    ▼
Azure Function (Python)
  ├── Timer Trigger (monthly, 1st @ 03:00 UTC)
  └── HTTP Trigger (manual)
    │
    ├── Enrichment Engine (enrich_transactions.py — single enricher, all sources)
    │     ├── Merchant pattern matching (runs first)
    │     ├── Plaid category map (fallback)
    │     ├── Historical carry-forward
    │     ├── Bonus rule
    │     ├── Fallback → Uncategorized
    │     └── Manual override (never overwritten)
    │
    ▼
Azure SQL Database (FinanceDB)
  ├── Phase 1-3 (live)
  │     ├── transactions (primary)
  │     ├── merchant_patterns
  │     ├── category_map
  │     ├── accounts
  │     ├── holdings
  │     ├── institution_map
  │     ├── plaid_sync_state
  │     ├── run_log
  │     └── error_log
  │
  └── Phase 4 (created 2026-06-01)
        ├── securities
        ├── account_balances
        ├── liabilities
        ├── liability_balances
        ├── physical_assets
        ├── physical_asset_valuations
        ├── insurance_assets
        ├── insurance_asset_valuations
        ├── baird_holdings          ← ✅ Live (5,509 rows, 11 accounts, lot-level)
        ├── security_sectors        ← ✅ Live (115 tickers mapped)
        ├── budget_targets          ← ✅ Live, seeded 2026-09-15 (240 rows, 2026)
        └── net_worth_history       ← ✅ Live, added 2026-09-15 — 2011-2026 CSV backfill
    │
    ▼
External Valuation APIs (Phase 4 — not yet integrated)
  ├── Zillow API → physical_asset_valuations (real estate)
  └── KBB API   → physical_asset_valuations (vehicles)
    │
    ▼
Power BI (star schema)
  ├── Phase 3 (live — scheduled refresh 4:00 AM CT daily)
  │     ├── Calendar table (DAX hub)
  │     ├── vw_transactions_clean
  │     ├── vw_monthly_spend
  │     ├── vw_cash_flow
  │     ├── vw_top_merchants
  │     ├── vw_category_yoy
  │     └── vw_enrichment_quality
  │
  └── Phase 4 — Net Worth / Holdings / Asset Allocation pages built and
        verified 2026-09-14; Budget vs Actual data verified working
        2026-09-15 (page refinement still outstanding); Liability still planned
        ├── vw_net_worth              (thin wrapper over vw_holdings_all as of 2026-09-15 — Cash/Investment/Retirement/Physical/Liability)
        ├── vw_holdings_all           (built 2026-09-14, consolidated Baird + Plaid Investments; extended 2026-09-15 with l1_group..l5_source Power BI account hierarchy columns)
        ├── vw_net_worth_all_time     (built 2026-09-15 — full 2011-2026 monthly history, net_worth_history + vw_holdings_all, forward-filled)
        ├── vw_account_freshness      (added 2026-09-16 — account-level freshness + latest_value across every monthly sync, incl. NWM/401k which vw_source_freshness never covered)
        ├── vw_account_balances
        ├── vw_holdings_summary       (dead — Baird-only, superseded by vw_holdings_all, removed from PBI model)
        ├── vw_asset_allocation       (dead — Baird-only, superseded by vw_holdings_all, removed from PBI model)
        ├── vw_liability_summary      (planned — not yet built into a page)
        └── vw_budget_vs_actual       (data verified working 2026-09-15 — budget_targets seeded; needs no Power BI relationship, joins internally; page itself not yet refined)
    │
    ▼
AI Agent Layer (Phase 5)
  ├── Spending Intelligence Agent
  ├── Portfolio Rebalancing Agent
  └── Holdings Strategy Agent
```

---

## Components

### Plaid API
| Setting | Value |
|---------|-------|
| Environment | Production |
| Endpoints — Phase 3 | /transactions/sync |
| Endpoints — Phase 4 | /accounts/balance/get (blocked — ISSUE-010, Balance product not authorized in Production), /investments/holdings/get (✅ Live for Principal/Baird 401k as of 2026-08-01 via principal_sync.py. Still blocked for Baird's other accounts (ISSUE-008, CSV fallback in use)), /investments/transactions/get, /liabilities/get |
| Sync method | Incremental via cursor (/transactions/sync) |
| Tokens — Live | Chase (ins_56), Associated Personal (ins_116823), Amex (ins_10), NWM Tom, NWM Amy, Principal Financial 401k (connected 2026-08-01, real account = Baird Profit Sharing and Savings Plan (401k), owner Amy) |
| Tokens — Blocked | Baird (ins_117067) — INTERNAL_SERVER_ERROR, Baird-side issue (ISSUE-008) |
| Tokens — Unsupported | Rocket Mortgage — confirmed not supported by Plaid (manual liability updates only) |
| Tokens — Pending | None |
| Token storage | Environment variables in local.settings.json (local) and Azure Function App Settings (cloud) |

---

### Azure Function App
| Setting | Value |
|---------|-------|
| Runtime | Python |
| SDK | Azure Functions v2 |
| App name | Finance-ingest-Tom-v6 |
| Region | East US 2 (Flex Consumption) |
| Local folder | `C:\DEV_Projects\AFAS` |
| Deployment method | VS Code Azure Functions extension |
| Managed Identity | NOT configured — deferred to future phase |
| Auth (current) | SQL username/password via environment variables (local); Managed Identity in Azure (db.py updated) |

#### Triggers
| Trigger | File | Schedule / Endpoint |
|---------|------|-------------------|
| ~~Timer~~ | ~~timer_sync.py~~ | **Deregistered 2026-09-16** — removed from `function_app.py`. Ran the same `0 0 3 1 * *` schedule as `monthly_sync.py`; both fired while Azure SQL was still auto-paused (ISSUE-032), so the timer never actually completed a run. File left in place, inert. |
| HTTP | http_ingest.py | Manual triggers. New `http_monthly_ingest_all` route (added 2026-09-16) runs all 4 syncs (transactions, balances, NWM, Principal 401k) in one call — this is now the documented monthly entry point (see MonthlyProcedure.md). Individual routes (`http_ingest`, `http_balance_ingest`, `http_nwm_sync`, `http_principal_ingest`) remain for targeted retries. |

#### Python Files
| File | Purpose | Status |
|------|---------|--------|
| function_app.py | Azure Functions v2 entry point | ✅ Ready |
| plaid_sync.py | Shared sync module (de-duplicated from timer + HTTP) | ⚠️ Two issues found 2026-08-03: plaid_category_raw stores the full personal_finance_category JSON object via json.dumps() instead of an extracted category string (ISSUE-019 root cause — category_map's exact-string match has never worked for Plaid-synced sources as a result; fix drafted, not yet deployed). INFLOW_CATEGORIES incorrectly includes BANK_FEES and LOAN_PAYMENTS as inflows (ISSUE-023, sign regression, ~25 rows/~$136K affected; fix not yet drafted). description column confirmed dead code (ISSUE-024) — Plaid's transaction object has no field by that name, so tx.get("description", "") has always returned empty. |
| enrich_transactions.py | **The single enricher for every source** (Plaid CHASE/ASSOCIATED_PERSONAL/AMEX, Apple Card CSV, HSA CSV). Note: previously listed here as "enrichment.py" — corrected 2026-08-03. | ✅ 2026-09-16 (AFAS 3b6989d): fixed a false-positive "changed" bug in `main()`'s before/after comparison — `enrich_from_merchant_patterns()`/`enrich_from_history()` both reorder rows via `pd.concat(..., ignore_index=True)`, which broke the comparison's positional alignment against the pre-pipeline snapshot and misreported up to 9,604 of 16,754 untouched rows as changed (wasted writes only — verified live that no prior enrichment/`category_reviewed` data was actually overwritten). Comparison now keys off `transaction_id` instead of row position. 2026-09-08: `enrich_apple_csv.py` + `enrich_hsa_csv.py` retired — this file already had no source filter and is now the sole enricher (AFAS b1a3ef3). `load_transactions()` date cutoff (`date >= '2024-01-01'`) removed so a backdated pre-2024 CSV row isn't permanently skipped. `apply_fallback()` now only fills `in_budget`/`type` where NULL, preserving an importer's deliberate classification for an unmatched row. 2026-09-03: docstring order corrected; `enrich_from_history()` added between category_map and bonus_rule → `category_source = 'historical_carryforward'` / `MEDIUM` (AFAS 557cdd1 + 4f0c052). 2026-09-02: write-back scoped to changed rows (ISSUE-035). 2026-08-03: apply_fallback() defaults corrected; `--unenriched-only` retry fixed. |
| import_apple_csv.py | Apple Card CSV import script (manual monthly). Stores Apple's CSV "Category" column in `plaid_category_raw` (despite the column name — no Plaid involvement). | ✅ 2026-09-08: inserts `category_source = NULL` (was `'historical'`, which collided with the ~6k genuinely backfilled `'historical'` rows); MERGE re-default guard now treats only `NULL` / `'unmatched'` as "not yet enriched" (AFAS b1a3ef3). 2026-09-01: MERGE fixed to protect already-enriched rows via CASE. |
| plaid_client.py | Plaid SDK wrapper — created 2026-06-15 (did not previously exist despite this entry). _make_plaid_client() (duplicated from plaid_sync.py) + get_account_balances() | ✅ Ready — prior "plaid_category_raw fix" note likely describes logic actually in plaid_sync.py, needs verification |
| balance_sync.py | Phase 4 — sync_account_balances(): /accounts/balance/get → upsert account_balances; logs run_log/error_log. Created 2026-06-15. | ✅ Live — tested 2026-06-17 |
| nwm_sync.py | Phase 4 — sync_nwm_valuations(): NWM Tom + Amy cash values → insurance_asset_valuations. Fixed account_id→insurance_id map. Created 2026-06-17. | ✅ Live |
| monthly_sync.py | Monthly timer (1st of month 03:00 UTC) — runs balance_sync + nwm_sync. Created 2026-06-17. | ⚠️ Deregistered 2026-09-16 from `function_app.py` — superseded by the manual `http_monthly_ingest_all` route in http_ingest.py (both this and timer_sync.py fired on the same schedule while Azure SQL was still auto-paused, so neither ever completed). File left in place, inert. |
| import_baird_holdings.py | Phase 4 — Baird holdings CSV import → baird_holdings. Lot-level holding_id (account+symbol+date+lotN). Created 2026-06-17. | ✅ Ready |
| principal_sync.py | Phase 4 — Pulls Principal/Baird 401k holdings via Plaid Investments (/investments/holdings/get), upserts dbo.accounts/dbo.securities/dbo.holdings. Created 2026-08-01. | ✅ 2026-09-14 (ISSUE-009 closed, AFAS 0cee6b6): wired into `monthly_sync.py`'s timer + new `/api/principal_ingest` route in `http_ingest.py`; deployed to Finance-ingest-Tom-v6 and verified via Application Insights (real HTTP-triggered run). Also now captures Plaid's `sector`/`industry` per security into `dbo.securities` (AFAS 64e45cc) — that specific change is committed but not yet redeployed as of session end (the deployed function still has the pre-sector-capture version; existing data was backfilled by re-running locally). Confirmed via a direct raw-API check that Plaid returns `cost_basis: null` and empty `tax_lots` for all 11 holdings — genuinely unavailable from this institution, not a pipeline gap. |
| import_hsa_transactions.py | Imports Bank of America HSA cash-ledger CSV export into dbo.transactions. type/in_budget set deterministically at import (4 known non-spending description types whitelisted; everything else treated as real spending/income, typed by amount sign). category/subcategory left NULL for `enrich_transactions.py`. Created 2026-08-01. | ✅ Live — 461 rows |
| import_hsa_holdings.py | Imports Bank of America HSA "Fund Summary" CSV into dbo.holdings — value-only (no units/price in this export). Snapshot date parsed from filename. Created 2026-08-01. | ✅ Live — 2 holdings |
| db.py | DB connection — username/password local, Managed Identity in Azure | ✅ Ready |
| timer_sync.py | Monthly timer trigger (1st @ 03:00 UTC) | ⚠️ Deregistered 2026-09-16 — see Triggers table above. File left in place, inert. |
| http_ingest.py | Manual HTTP trigger — added http_balance_ingest route 2026-06-15 (/api/balance_ingest) | ✅ 2026-09-16: added `http_monthly_ingest_all` route (combines transaction/balance/NWM/Principal syncs into one manual call) + `_result_has_error` helper. Now the documented monthly entry point. |
| get_plaid_tokens.py | Local Flask tool for Plaid token acquisition | ✅ Ready — located at C:\DEV_Projects\AFAS\scripts — hardcoded PLAID_CLIENT_ID/PLAID_SECRET removed 2026-08-01, now loaded from local.settings.json at module scope. 2026-09-16: `index()` route now pre-fills each institution's existing-token text box from `os.environ`, preventing an accidental fresh Link session (instead of update/reauth mode) when the box was left blank. |
| scripts/taxonomy_audit.py | Read-only taxonomy-drift diagnostic. 4 checks: undocumented category/subcategory combos in merchant_patterns / category_map / live transactions, and same-priority pattern pairs where one pattern's text is a substring of another's (shadowing). Check 4 output split `[DIFFERENT DEST]` (real bugs) vs `[same dest]` (harmless). Exit 0 = clean, 1 = issues, 2 = error. DB creds via `.env` / `get_connection()`. AFAS 13b959e. | ✅ 2026-09-08: (a) ISSUE-041 (AFAS 6c652ae) — the hardcoded `CANONICAL_TAXONOMY` dict replaced with `load_canonical_taxonomy()`, which parses `Category_Taxonomy.md`'s `## Full Taxonomy` block at runtime, fresh in `main()` every run, exits 2 on a parse failure (no hardcoded fallback); (b) AFAS 1db78e4 — Check 4's `_would_match()` now mirrors the enricher's matcher, so a no-wildcard exact-match pattern (`ACT`, `IRS`) is only flagged as a shadow pair when the other pattern actually matches its literal text. Session-end state: Checks 1/2/3 = 0; Check 4 = 207 (8 benign `[DIFFERENT DEST]` + 199 cosmetic `[same dest]`). |
| scripts/plaid_transaction_name_check.py | **NEW 2026-09-03** — Read-only diagnostic. Queries Plaid /transactions/get for CHASE/ASSOCIATED_PERSONAL/AMEX over a date range and prints raw `name` / `merchant_name` / personal_finance_category (plus full JSON for the first 5). Used to check whether Plaid retains fuller merchant text than what's stored. No DB writes, no Plaid write endpoints, not wired into any pipeline. Config via local.settings.json Values block. AFAS 2ad1bf2. | ✅ Live — confirmed the Associated DDA truncation (Bayshore D / Musa I / Sheboygan) happens upstream of Plaid; processor-routed names (Toast) do carry more detail. |
| requirements.txt | Pinned dependencies | ✅ Ready |

#### Environment Variables (local.settings.json + App Settings)
| Variable | Purpose |
|----------|---------|
| DB_SERVER | financeauto-sql-server.database.windows.net |
| DB_NAME | FinanceDB |
| DB_USERNAME | SQL auth username |
| DB_PASSWORD | SQL auth password |
| PLAID_CLIENT_ID | Plaid client ID |
| PLAID_SECRET | Plaid secret (production) |
| PLAID_ENV | production |
| PLAID_ACCESS_TOKEN_CHASE | Chase production token |
| PLAID_ACCESS_TOKEN_ASSOCIATED_PERSONAL | Associated Bank Personal production token |
| PLAID_ACCESS_TOKEN_AMEX | American Express production token |
| PLAID_ACCESS_TOKEN_NWM_TOM | NW Mutual Tom whole life token (acquired 2026-06-02) |
| PLAID_ACCESS_TOKEN_NWM_AMY | NW Mutual Amy whole life token (acquired 2026-06-02) |
| PLAID_ACCESS_TOKEN_PRINCIPAL | Principal Financial 401k token (pending — ISSUE-009) |
| ZILLOW_API_KEY | Zillow API key for real estate valuations (Phase 4 — pending) |
| KBB_API_KEY | KBB API key for vehicle valuations (Phase 4 — pending) |

**Note:** Local scripts split configuration loading between `.env` (DB credentials, via `load_dotenv()`) and `local.settings.json` (Plaid tokens and other App-Settings-equivalent values, loaded manually in each script's `__main__` block). This split is a real source of confusion — a new script (`principal_sync.py`) was initially built using only `load_dotenv()` and failed to find a token that was correctly present in `local.settings.json`.

---

### External Valuation APIs (Phase 4 — Not Yet Integrated)

| API | Purpose | Trigger | Target Table |
|-----|---------|---------|--------------|
| Zillow | Real estate valuation for WFB-Belle | Weekly or monthly Azure Function trigger | `physical_asset_valuations` |
| KBB | Vehicle valuation for TSLA Nebula, Storm, Trinity | Weekly or monthly Azure Function trigger | `physical_asset_valuations` |

**Note:** Both APIs require feasibility scoping before integration — confirm rate limits, authentication method, and response format before building the ingestion function.

---

### Account Map — Plaid Coverage
| Account | Institution | Plaid Coverage | Phase |
|---------|-------------|----------------|-------|
| MAIN - Brokerage (BKG) | Baird | ⚠️ Plaid blocked (ISSUE-008) — CSV fallback operational via import_baird_holdings.py | 4 |
| MAIN - PIM | Baird | ⚠️ Blocked — INTERNAL_SERVER_ERROR (ISSUE-008) | 4 |
| HSA - Baird | Baird | ⚠️ Blocked — INTERNAL_SERVER_ERROR (ISSUE-008) | 4 |
| BAIRD Capital (PE) | Baird | ⚠️ Private equity — Plaid may not reach | 4 |
| BAIRD Stock | Baird | ⚠️ Blocked — INTERNAL_SERVER_ERROR (ISSUE-008) | 4 |
| 529 - Alex / Brooke / Whit | Baird | ⚠️ Blocked — INTERNAL_SERVER_ERROR (ISSUE-008) | 4 |
| 401k Baird Profit Share | Baird | ⚠️ Blocked — INTERNAL_SERVER_ERROR (ISSUE-008) | 4 |
| IRA - Baird Stock / Tom | Baird | ⚠️ Blocked — INTERNAL_SERVER_ERROR (ISSUE-008) | 4 |
| IRA ROTH - Baird Stock / Tom | Baird | ⚠️ Blocked — INTERNAL_SERVER_ERROR (ISSUE-008) | 4 |
| Vanguard (money market in PIM/Brokerage) | Baird | ⚠️ Blocked — shows as cash position when Baird resolves | 4 |
| NWM - Tom (whole life cash value) | NW Mutual | ✅ Token acquired 2026-06-02 — cash value via balance pull | 4 |
| NWM - Amy (whole life cash value) | NW Mutual | ✅ Token acquired 2026-06-02 — cash value via balance pull | 4 |
| Principal Financial 401k | Principal | ⏳ Token pending (ISSUE-009) | 4 |
| Associated - Money Market | Associated | ⚠️ Code ready, blocked — ISSUE-010 (Balance product not authorized) | 4 |
| Associated - Savings (Alex/Brooke/Whit) | Associated | ⚠️ Code ready, blocked — ISSUE-010 (Balance product not authorized) | 4 |
| Associated - Checking / Checking-Whit | Associated | ✅ Live (transactions) | 3 |
| DF Checking | Associated | ✅ Live (transactions) | 3 |
| Chase | Chase | ✅ Live (transactions) | 3 |
| American Express | Amex | ✅ Live (transactions) | 3 |
| Apple Card | Apple | ✅ Monthly CSV import (no Plaid support) | 3 |
| House — WFB-Belle | N/A | ❌ Not on Plaid — Zillow API (Phase 4) | 4 |
| TSLA Nebula / Storm / Trinity | N/A | ❌ Not on Plaid — KBB API (Phase 4) | 4 |
| HSA (Bank of America) | Bank of America | ❌ Plaid confirmed unsupported for this institution ("Connectivity not supported") — permanent CSV-only pipeline, same model as Baird | 4 |
| Rocket Mortgage | Rocket | ❌ Confirmed unsupported by Plaid — manual updates only | 4 |
| US Bank LOC (Baird Stock Loan) | US Bank | ✅ Liabilities endpoint | 4 |

---

### Azure SQL Database
| Setting | Value |
|---------|-------|
| Server | financeauto-sql-server.database.windows.net |
| Database | FinanceDB |
| Auth | SQL username/password (Managed Identity deferred) |
| Total rows | ~15,090+ transactions as of 2026-06-02 |
| Date range | 2020-01-01 → 2026-05-31 |

#### Connection String Pattern
```python
# pyodbc — used in db.py
conn_str = (
    f"DRIVER={{ODBC Driver 18 for SQL Server}};"
    f"SERVER={os.environ['DB_SERVER']};"
    f"DATABASE={os.environ['DB_NAME']};"
    f"UID={os.environ['DB_USERNAME']};"
    f"PWD={os.environ['DB_PASSWORD']};"
    "Encrypt=yes;TrustServerCertificate=no;"
)
```

**Note:** Azure SQL Free tier Serverless auto-pause is a known constraint. FinanceDB must be manually resumed via Azure Portal on the first of each month before importing the Apple Card CSV or Power BI refresh will fail.

---

### Power BI
| Setting | Value |
|---------|-------|
| Data source | Azure SQL (Import mode) |
| Refresh | Scheduled — daily 4:00 AM CT (live since 2026-05-27) |
| Data model | Star schema — Calendar table as hub |
| Calendar table | DAX-calculated |
| Published to | Power BI Service — My Workspace |

#### Data Model Relationships
All relationships: One-to-Many, Single cross-filter direction, Calendar as hub.

| View | Connects to Calendar via | Notes |
|------|--------------------------|-------|
| vw_transactions_clean | date | Base view — COALESCE merchant fix applied |
| vw_monthly_spend | month_start_date | |
| vw_cash_flow | month_start_date | |
| vw_top_merchants | — | Standalone — no Calendar relationship; COALESCE merchant fix applied |
| vw_category_yoy | year_start_date | |
| vw_enrichment_quality | — | Standalone |

#### Report Pages
| Page | Status |
|------|--------|
| Monthly Spend | ✅ Complete |
| Cash Flow | ✅ Complete |
| Year Over Year | ✅ Complete |
| Top Merchants | ✅ Complete |
| Transaction Review | ✅ Complete (added 2026-06-01) — Source/Year/Month slicers |
| Data Health | ✅ Complete (added Session 13) — built on vw_source_freshness + vw_category_health |
| Needs Review | ✅ Complete (added Session 17) — built on `vw_needs_review` (all `category_reviewed = 0` rows, with computed `suggested_category`/`suggested_subcategory` and a `review_priority` 1–4 tier). Backlog cleared 421 → 0 on 2026-09-03. See SessionStarter "Data Health (Power BI)". Table visuals on this page **must** include `transaction_id` (hidden) with numerics set to "Don't summarize" — see BestMethods. |
| Net Worth | ✅ Complete (added Session 20, 2026-09-14; categories reworked Session 21, 2026-09-15) — built on `vw_net_worth`, across Cash/Investment/Retirement/Physical/Liability (category set changed this session — see vw_net_worth below). |
| Holdings / Asset Allocation | ✅ Complete (added Session 20, 2026-09-14; hierarchy columns added Session 21) — built on `vw_holdings_all` (not `vw_holdings_summary`/`vw_asset_allocation`, both now dead — see below). Grouped by `account_name`, `sector`, `asset_classification`, `asset_type`, or the new `l1_group`..`l5_source` hierarchy. |
| Net Worth History (full 2011-2026 trend) | ✅ Data ready, page not yet built (Session 21, 2026-09-15) — `vw_net_worth_all_time`, one row per account per calendar month, forward-filled. |
| Liability Summary | ⏳ Designed with chat, not yet built/verified in Power BI — would use `vw_liability_summary` (already exists, unused). |
| Budget vs Actual | 🟡 Data verified working (Session 21, 2026-09-15) — `budget_targets` seeded for 2026, `vw_budget_vs_actual` confirmed producing correct actual/budget/variance/pct_of_budget numbers (no Power BI relationship needed, the view joins internally). Page design/refinement is explicitly the next session's focus. |

#### vw_holdings_all (added 2026-09-14, script 95; extended 2026-09-15, scripts 99-103)
Consolidates `dbo.baird_holdings` (CSV, flat) and `dbo.holdings` + `dbo.accounts` + `dbo.securities` (Plaid Investments — Principal 401k, HSA, any future connection) into one shape. Full history (every snapshot date), not just latest — built for equity-performance trending, not just a point-in-time number; downstream consumers filter to `MAX(snapshot_date)` per account themselves. Columns: `source`, `institution_id`, `account_key`, `holding_id`, `account_name`, `owner`, `account_subtype`, `snapshot_date`, `symbol`, `description`, `asset_type`, `asset_classification`, `sector`, `quantity`, `price`, `value`, `cost_basis`, `unit_cost`, `unrealized_gl`, `unrealized_gl_pct`, `term`, `est_annual_income`, `date_acquired`, `currency`, `include_in_net_worth`, `lender_name`, `net_worth_category`, `l1_group`..`l5_source`. Fields that only exist on one side (`sector`/`asset_classification`/`term`/`est_annual_income`/`date_acquired` are Baird-only) are `NULL` on the other; `unrealized_gl`/`unrealized_gl_pct`/`unit_cost` are computed from `cost_basis` for the Plaid side to keep parity (NULL when Plaid doesn't return `cost_basis`, e.g. the entire Principal 401k). `sector` for any fund-type Plaid holding (mutual fund/ETF, or Plaid's generic `'Miscellaneous'`) is normalized to `'Diversified'`, matching Baird's own convention (script 96). `asset_classification` for the Plaid side is derived from `security_type` (script 97) since Plaid has no equivalent field — mapped onto Baird's own bucket names (Equities/Fixed Income/Alternatives/Cash and Cash Equivalents). Supersedes `vw_holdings_summary` and `vw_asset_allocation` as the source of truth for holdings — both are dead (still `FROM dbo.baird_holdings` only, same gap `vw_net_worth` had before Session 20's fix), removed from the Power BI model, not deleted from SQL.

As of Session 21 (2026-09-15), also carries the non-holding net-worth sources (bank cash, insurance, physical, liabilities) as pseudo-holding rows (symbol/quantity/price/cost NULL, `value` = the balance/valuation) — this view is now the single source of truth for all net worth data, not just holdings. `account_key` is a real unique-per-account grouping key (Plaid `account_id` for Plaid-backed rows, the source's own PK for insurance/physical/liability, `'BAIRD-' + account_name` for Baird since it has no account table) — required because `dbo.accounts` had 3 rows literally named "Kids Savings Account" with different `account_id`s (since renamed, see DecisionLog), and grouping "latest snapshot per account" by `account_name` alone would silently collapse or drop rows for any account sharing a display name.

`net_worth_category` (Cash/Investment/Retirement/Physical/Liability) is computed **per row**, not per account — a cash-equivalent holding inside an Investment or Retirement account (e.g. a money market fund inside `MAIN - BKG`) shows under Cash, not its account's usual category. Insurance rolls into Cash. This is a separate, flatter field from the `l1_group`..`l5_source` Power BI account hierarchy (script 102) built to match Tom's target Account Type tree (Assets/Liabilities → Financial/Physical/Cash → Investments/HSA/PE & Baird/Education 529/Retirement/Insurance/Bank → asset-type or account-specific subgroup → friendly institution label) — both are maintained independently; see DecisionLog for the specific derivation rules and documented deviations from Tom's original sketch.

#### vw_net_worth (updated 2026-09-14 scripts 94+98; rebuilt as a thin wrapper 2026-09-15, script 99)
Now just a latest-snapshot-per-account wrapper over `vw_holdings_all`'s `net_worth_category`/`account_key` — `SELECT account_name, net_worth_category AS asset_category, snapshot_date, SUM(value) FROM vw_holdings_all ... WHERE include_in_net_worth = 1`, grouped to the latest date per `account_key`. Kept so existing Power BI relationships built on it keep working; new Power BI work should point at `vw_holdings_all` directly (one table, full history, the l1-l5 hierarchy available) and this view can eventually be retired the same way `vw_holdings_summary`/`vw_asset_allocation` were. Categories are now Cash/Investment/Retirement/Physical/Liability (Insurance folded into Cash, script 103; Retirement split out of Investment, script 100) — different from the Investment/Cash/Insurance/Physical/Liability set as of Session 20.

#### vw_net_worth_all_time (added 2026-09-15, script 105)
Unions `dbo.net_worth_history` (2011-2026 CSV backfill + interpolated gap-fill — see `net_worth_history` in Phase 4 Tables) with `vw_holdings_all` for one continuous account-level **monthly** series. Forward-fills each account's last known value across every calendar month with no snapshot — a plain "latest date in month" approach breaks badly for the pre-2020 years, where most accounts in Tom's CSV were only recorded annually (January): without forward-fill, April-December of those years would collapse to whatever one or two accounts happened to have an off-cycle update, swinging the "monthly total" from millions to near-zero. Doesn't fabricate anything past an account's latest known snapshot (a physical asset or liability whose last sync was last month simply carries that value forward, same forward-fill logic — not a gap, just not re-measured yet). Columns: `account_key`, `account_name`, `net_worth_category`, `month_start`, `as_of_date` (the actual snapshot date the month's value came from), `value`.

**Power BI modeling note:** don't relate `vw_net_worth_all_time` and `vw_holdings_all` to each other directly — they're two different grains of the same underlying data (one is literally built by aggregating the other) and relating two fact tables causes fan-out. Use whichever table fits the visual; relate each independently to the Calendar table instead. See BestMethods for the Calendar-table-date-range lesson this surfaced.

#### vw_account_freshness (added 2026-09-16, scripts 106-107)
Account-level freshness across every source `vw_holdings_all` covers (Baird, Plaid Investments, bank cash, insurance, physical, liability) plus `dbo.transactions`, in one place. Fills a gap `vw_source_freshness` never covered: that view only ever queried `dbo.transactions`, so it had no visibility into non-transactional monthly syncs like NWM (insurance) or the Principal 401k — an account could go stale there with nothing surfacing it. Adds `latest_value` (script 107) so a freshness check also shows what the last-known balance/valuation was, not just the date. Used in Step 2 of MonthlyProcedure.md as the pre-ingest freshness check, and referenced from the SessionStarter Data Health section.

#### Calendar Table Sort Orders
| Column | Sort By |
|--------|---------|
| MonthName | Month |
| MonthShort | Month |
| YearMonth | YearMonthSort |
| YearQuarter | YearQuarterSort |

---

### Enrichment Pipeline — Logic Order
Enrichment runs in strict priority order. Higher steps win; manual never gets overwritten.
Corrected 2026-08-03 — this table previously documented merchant_pattern and
plaid_category in the reverse of their real execution order, matching a stale
docstring inside enrich_transactions.py itself
rather than what the code actually executes (confirmed by reading the code
and its own inline comment: "merchant_patterns runs FIRST — specific
merchant match beats generic Plaid category / category_map runs SECOND —
Plaid category as fallback only").

**Corrected again 2026-09-03 (Session 17):** the "Historical carry-forward"
step had been in this table for 6+ months but **never existed in the
code** — the module docstring documented 4 steps and `main()` called 4
steps (see BestMethods "A documented pipeline step ... is not evidence it
was ever built"). It is now genuinely implemented as
`enrich_from_history()`, wired into `main()` **between category_map and
bonus_rule** (not after bonus, where this table used to place it). It
carries forward from the most recent prior row for the same
`merchant_name_raw` whose own `category_source` is a real decision
(`manual` / `merchant_pattern` / `historical` / `category_map` /
`bonus_rule` — `'unmatched'` excluded), writes
`category_source = 'historical_carryforward'` (distinct from legacy
pre-2024 `'historical'` from historical_mapping.sql) and
`category_confidence = 'MEDIUM'`. AFAS commits 557cdd1 + 4f0c052.

**Session 19 (2026-09-08):** `enrich_transactions.py` is now the **single
enricher for every source** — `enrich_apple_csv.py` / `enrich_hsa_csv.py`
were retired (they were drifted duplicates; see BestMethods "Duplicate
per-source enrichment scripts..."). The pattern matcher is the existing
Python-side one, which treats **only leading/trailing `%` as wildcards**
— a `%` embedded mid-pattern is deleted and the pattern silently can't
match (31 such patterns cleaned up, script 86). `load_transactions()`
no longer filters by date. `apply_fallback()` now only fills
`in_budget`/`type` where NULL, so an importer's own classification on an
unmatched row (e.g. HSA Employer Contribution → Income / not-in-budget)
survives. AFAS b1a3ef3.

| Step | Source | Logic |
|------|--------|-------|
| 1 | Merchant pattern | merchant_name_raw LIKE match in merchant_patterns, ordered by priority ASC then pattern ASC — runs first; a specific merchant match beats a generic Plaid category. **Same-priority note:** first textual match wins, so a broad pattern that is a substring of a specific one shadows it — more-specific patterns must sit at a strictly lower priority number (see ISSUE-012 / BestMethods). |
| 2 | Plaid category | plaid_category_raw → lookup in category_map table — fallback only, applied to rows merchant_patterns did not match |
| 3 | Historical carry-forward | `enrich_from_history()` — most recent prior real categorization for the same merchant_name_raw; `category_source = 'historical_carryforward'`, `category_confidence = 'MEDIUM'`. Implemented 2026-09-03. |
| 4 | Bonus rule | Hardcoded: Baird payroll → Pay/Amy; large Baird deposits → Bonus/Amy (threshold >$10,000) |
| 5 | Manual | Human override — category_source = 'manual', never overwritten by pipeline (a protection, not a pipeline step) |
| 6 | Fallback | Unmatched rows → category='Uncategorized', subcategory='General', type='Expense', in_budget=1, category_source='unmatched', category_confidence='LOW' (corrected 2026-08-03 — previously set type='Other'/in_budget=0, contradicting Category_Taxonomy.md's documented default) |

**Sign convention (as documented — enforcement currently has a known bug, see ISSUE-023):**
- INCOME and TRANSFER_IN Plaid categories → stored as positive (abs value)
- LOAN_PAYMENTS and BANK_FEES → should be stored as negative (outflows) per this
  documented convention, but plaid_sync.py's INFLOW_CATEGORIES set currently
  includes both incorrectly, causing them to post positive. Not yet fixed
  as of 2026-08-03 — see ISSUE-023.
- All other debits → negated at ingestion (-tx["amount"])

---

### AI Agent Layer (Phase 5 — Architecture Planned)
Three agents, each reading from FinanceDB views and external inputs:

| Agent | Primary Data Sources | External Input | Output |
|-------|---------------------|----------------|--------|
| Spending Intelligence | vw_category_yoy, vw_monthly_spend, vw_cash_flow, vw_budget_vs_actual | User-defined budget targets (budget_targets table) | Spending adjustment recommendations |
| Portfolio Rebalancing | vw_holdings_summary, vw_asset_allocation, securities | Market data feed (TBD) | Rebalancing recommendations |
| Holdings Strategy | vw_holdings_summary, vw_net_worth, vw_liability_summary | Risk tolerance profile | Holdings add/reduce/reallocate recommendations |

**Risk tolerance profile:** To be defined as a structured input — either a document or a dedicated `risk_profile` table in FinanceDB with target allocations, time horizon, and liquidity needs.

**Net worth inputs for AI agents:**
- Financial assets: `vw_holdings_summary` + `vw_account_balances`
- Physical assets: `physical_asset_valuations` (latest per asset)
- Insurance assets: `insurance_asset_valuations` (latest per policy) — NWM Tom + Amy seeded
- Liabilities: `vw_liability_summary` — Rocket Mortgage + US Bank LOC seeded

---

## Security Notes
| Item | Current State | Target State |
|------|--------------|--------------|
| SQL Auth | Username/password via env vars | Managed Identity (deferred) |
| Plaid tokens | Production — Chase, Associated Personal, Amex, NWM Tom, NWM Amy live | Baird pending (ISSUE-008); Principal pending (ISSUE-009) |
| Plaid product authorization | Balance product requested 2026-06-15 (Production, use case: Other / account aggregation) | Pending Plaid approval (ISSUE-010) |
| Function auth | Default (function key) | Review before Phase 5 |
| Power BI | Scheduled refresh live (4:00 AM CT) | Service principal auth (future) |

---

## Incident History

### 2026-08-03 — plaid_category_raw JSON-vs-string mismatch (ISSUE-019 root cause)
plaid_sync.py's _sync_institution() has been storing the entire
personal_finance_category JSON object (via json.dumps()) in
dbo.transactions.plaid_category_raw since Plaid's v2 personal finance
category API was adopted, rather than a single clean category string.
DB_Schema.md documents this column as "Plaid's raw category string"
(singular value) and enrich_transactions.py's category_map matching does
an exact string comparison — meaning this lookup has effectively never
succeeded for any Plaid-synced transaction (CHASE/ASSOCIATED_PERSONAL/AMEX),
not just the June/July 2026 backlog that originally surfaced the issue.
Scoping confirmed impact contained to 2026 (64 rows) — the JSON blob
formatting appears to postdate whatever Plaid API version bump introduced
personal_finance_category in its current form. Fix: extract
personal_finance_category.detailed (fallback .primary, then the legacy v1
category list) at ingestion. Historical rows backfilled via script 59.
**Lesson: a documented column description ("raw category string") is not
the same as a verified one — the actual stored format should be spot
-checked directly against what a downstream consumer (category_map) expects
to match, not assumed from the column's name or comment.**

---

### 2026-08-01 — Silent production outage (dotenv)
python-dotenv was added to db.py's local dependencies during the July session but never added to requirements.txt. The deployed Function App crashed on every cold start (ModuleNotFoundError: No module named 'dotenv'), causing the trigger indexer to find 0 functions — not a portal display quirk, a genuine failure. This silently killed all automated syncing (daily transactions, monthly balance/NWM sync) for at least 6 weeks, discovered only when investigating an apparently-isolated Amex sync failure. Fixed by adding python-dotenv to requirements.txt, reinstalling into .python_packages\lib\site-packages, and redeploying.
**Lesson: local script success does not guarantee deployed Function App success — .env and local.settings.json can mask a dependency gap that only surfaces in the deployed environment.**

### 2026-08-01 — Hardcoded production credentials in get_plaid_tokens.py

get_plaid_tokens.py hardcoded PLAID_CLIENT_ID and PLAID_SECRET as plain-
text literals rather than loading them from local.settings.json, violating
Master Protocol section 6. Found and fixed same session it was discovered
— TechnicalArchitecture.md's own Environment Variables table already
documented these two keys as expected in local.settings.json, so the fix
was a straightforward swap to the existing established values rather than
a new credential to generate. **Lesson: standalone local-only scripts
(never deployed to Azure) can accumulate security debt that deployment-
focused reviews won't catch, since they're outside the Function App's own
audit surface.**

---

## Key Constraints & Decisions
| Constraint | Detail |
|------------|--------|
| No category_normalized column | Removed — was 100% null. Do not re-add. |
| Upsert only | Never INSERT without upsert guard. PK = transaction_id. |
| pending column | NULL for all existing rows — always use ISNULL(pending,0)=0 in queries |
| amount sign convention | Positive = credit/income, Negative = debit/expense |
| type column | Income / Expense / Other — not free text |
| Manual overrides | category_source = 'manual' — pipeline never overwrites |
| Azure SQL auto-pause | Free tier Serverless pauses monthly — resume manually via Azure Portal before 1st-of-month Apple CSV import |
| Rocket Mortgage | Confirmed unsupported by Plaid — liability_balances updated manually |
| Work-Expense merchant_patterns | Dual-use merchants (hotels, restaurants, airlines) must NEVER be mapped to Work-Expense in merchant_patterns — work trips corrected manually with category_source = 'manual' |
| Subcategory naming | Subcategory must never mirror parent category name — use 'General' instead |
| merchant_patterns priority | More specific patterns must fire at higher priority (lower number) than broad ones |
