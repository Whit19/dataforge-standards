# AFAS Project — Technical Architecture
**Update this file when any component, connection, or configuration changes.**
Last updated: 2026-09-01 (Session 15: corrected timer_sync.py's stale "Daily Trigger" documentation to Monthly — matches the actual deployed cron and the deliberate design confirmed this session)

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
    ├── Enrichment Engine (enrichment.py / enrich_apple_csv.py)
    │     ├── Bonus rules
    │     ├── Plaid category map
    │     ├── Merchant pattern matching
    │     ├── Historical carry-forward
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
        └── budget_targets
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
  └── Phase 4 (planned)
        ├── vw_net_worth
        ├── vw_account_balances
        ├── vw_holdings_summary
        ├── vw_asset_allocation
        ├── vw_liability_summary
        └── vw_budget_vs_actual
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
| Timer | timer_sync.py | Monthly, 1st @ 03:00 UTC (`0 0 3 1 * *`) — confirmed 2026-09-01: this is the correct, deliberate design, not a stale "Daily" leftover — balance_sync + nwm_sync |
| HTTP | http_ingest.py | Manual trigger for testing |

#### Python Files
| File | Purpose | Status |
|------|---------|--------|
| function_app.py | Azure Functions v2 entry point | ✅ Ready |
| plaid_sync.py | Shared sync module (de-duplicated from timer + HTTP) | ⚠️ Two issues found 2026-08-03: plaid_category_raw stores the full personal_finance_category JSON object via json.dumps() instead of an extracted category string (ISSUE-019 root cause — category_map's exact-string match has never worked for Plaid-synced sources as a result; fix drafted, not yet deployed). INFLOW_CATEGORIES incorrectly includes BANK_FEES and LOAN_PAYMENTS as inflows (ISSUE-023, sign regression, ~25 rows/~$136K affected; fix not yet drafted). description column confirmed dead code (ISSUE-024) — Plaid's transaction object has no field by that name, so tx.get("description", "") has always returned empty. |
| enrich_transactions.py | Enrichment engine for Plaid-synced transactions (CHASE/ASSOCIATED_PERSONAL/AMEX). Note: this file was previously listed here under the incorrect name "enrichment.py" — corrected 2026-08-03. | ⚠️ Two bugs fixed 2026-08-03: apply_fallback() type/in_budget defaults corrected to match Category_Taxonomy.md; --unenriched-only retry logic fixed (previously no-op'd on any row already processed by fallback once, since every step gates on category.isna() but apply_fallback() sets category='Uncategorized', not NULL). Module docstring still documents the wrong enrichment priority order (see Enrichment Pipeline section below) — not yet corrected. |
| enrich_apple_csv.py | Enrichment engine for Apple Card CSV imports | ✅ Ready — category_map lookup fixed 2026-06-01 |
| import_apple_csv.py | Apple Card CSV import script (manual monthly) | ✅ Ready |
| plaid_client.py | Plaid SDK wrapper — created 2026-06-15 (did not previously exist despite this entry). _make_plaid_client() (duplicated from plaid_sync.py) + get_account_balances() | ✅ Ready — prior "plaid_category_raw fix" note likely describes logic actually in plaid_sync.py, needs verification |
| balance_sync.py | Phase 4 — sync_account_balances(): /accounts/balance/get → upsert account_balances; logs run_log/error_log. Created 2026-06-15. | ✅ Live — tested 2026-06-17 |
| nwm_sync.py | Phase 4 — sync_nwm_valuations(): NWM Tom + Amy cash values → insurance_asset_valuations. Fixed account_id→insurance_id map. Created 2026-06-17. | ✅ Live |
| monthly_sync.py | Monthly timer (1st of month 03:00 UTC) — runs balance_sync + nwm_sync. Created 2026-06-17. | ✅ Deployed |
| import_baird_holdings.py | Phase 4 — Baird holdings CSV import → baird_holdings. Lot-level holding_id (account+symbol+date+lotN). Created 2026-06-17. | ✅ Ready |
| principal_sync.py | Phase 4 — Pulls Principal/Baird 401k holdings via Plaid Investments (/investments/holdings/get), upserts dbo.accounts/dbo.securities/dbo.holdings. Created 2026-08-01. Standalone local script — not wired into http_ingest.py or any timer trigger yet. | ✅ Working — confirmed live |
| import_hsa_transactions.py | Imports Bank of America HSA cash-ledger CSV export into dbo.transactions. type/in_budget set deterministically at import (4 known non-spending description types whitelisted; everything else treated as real spending/income, typed by amount sign). Created 2026-08-01. | ✅ Live — 457 rows |
| enrich_hsa_csv.py | Enrichment for HSA CSV import — mirrors enrich_apple_csv.py, scoped to source='HSA', category_map step removed (never applicable to this source). Created 2026-08-01. | ✅ Live |
| import_hsa_holdings.py | Imports Bank of America HSA "Fund Summary" CSV into dbo.holdings — value-only (no units/price in this export). Snapshot date parsed from filename. Created 2026-08-01. | ✅ Live — 2 holdings |
| db.py | DB connection — username/password local, Managed Identity in Azure | ✅ Ready |
| timer_sync.py | Monthly timer trigger (1st @ 03:00 UTC) | ✅ Ready |
| http_ingest.py | Manual HTTP trigger — added http_balance_ingest route 2026-06-15 (/api/balance_ingest) | ✅ Ready |
| get_plaid_tokens.py | Local Flask tool for Plaid token acquisition | ✅ Ready — located at C:\DEV_Projects\AFAS\scripts — hardcoded PLAID_CLIENT_ID/PLAID_SECRET removed 2026-08-01, now loaded from local.settings.json at module scope |
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
Corrected 2026-08-03 — this table previously documented the reverse of steps
2 and 3, matching a stale docstring inside enrich_transactions.py itself
rather than what the code actually executes (confirmed by reading the code
and its own inline comment: "merchant_patterns runs FIRST — specific
merchant match beats generic Plaid category / category_map runs SECOND —
Plaid category as fallback only").

| Step | Source | Logic |
|------|--------|-------|
| 1 | Merchant pattern | merchant_name_raw LIKE match in merchant_patterns, ordered by priority ASC — runs first; a specific merchant match beats a generic Plaid category |
| 2 | Plaid category | plaid_category_raw → lookup in category_map table — fallback only, applied to rows merchant_patterns did not match |
| 3 | Bonus rule | Hardcoded: Baird payroll → Pay/Amy; large Baird deposits → Bonus/Amy (threshold >$10,000) |
| 4 | Historical | Carry forward category from prior enriched record for same merchant |
| 5 | Manual | Human override — category_source = 'manual', never overwritten by pipeline |
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
