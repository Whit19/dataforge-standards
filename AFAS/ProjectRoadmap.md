# AFAS Project — Roadmap
**Living document. Update status and tasks as work completes.**
Last updated: 2026-06-15

---

## Project Goal
Serverless financial data pipeline feeding AI agents that recommend:
- Spending habit adjustments (current and future spending needs)
- Portfolio rebalancing based on current market trends
- Holdings strategy based on current and evolving risk tolerance

---

## Phase Summary

| Phase | Description | Status |
|-------|-------------|--------|
| Phase 1 | Plaid ingestion → Azure SQL | ✅ Complete |
| Phase 2 | Enrichment, normalization, schema cleanup | ✅ Complete |
| Phase 3 | Automation, Power BI reporting | ✅ Complete |
| Phase 4 | Accounts/balances, investments/holdings | 🔄 In Progress |
| Phase 5 | AI agent layer | ⏳ Not Started |

---

## Phase 1 — Plaid Ingestion ✅ Complete

### 1. Finalize Plaid Transactions Ingestion
- ✅ Full transaction payloads (moved beyond count-only)
- ✅ Pagination via offset/count through total_transactions
- ✅ Basic metadata logging (date range, total, pages fetched)

### 2. Write Transactions to Azure SQL
- ✅ SQL connection via pyodbc
- ✅ Transactions table created (idempotent script)
- ✅ Primary key: Plaid transaction_id
- ✅ Upsert / insert-if-not-exists guard in place

### 3. Wire In CSV Reference Tables
- ✅ merchant_patterns table loaded from CSV
- ✅ category_map table loaded from CSV
- ✅ Keys and indexes confirmed

---

## Phase 2 — Enrichment & Normalization ✅ Complete

### 4. Merchant Normalization
- ✅ merchant_patterns table: SQL LIKE match on merchant_name_raw, ordered by priority ASC
- ✅ Fallback: raw merchant name preserved in merchant_name_raw
- ✅ merchant_normalized column populated
- ✅ 7 duplicate pattern groups removed
- ✅ 139 Dining Out patterns fixed
- ✅ 4 broad patterns tightened

### 5. Category Enrichment
- ✅ category and subcategory columns assigned via enrichment pipeline
- ✅ type column: Income / Expense / Other
- ✅ category_source: plaid / merchant_pattern / historical / manual / bonus_rule
- ✅ category_confidence: all rows set to HIGH
- ✅ category_version column in category_map for future taxonomy versioning
- ✅ Full taxonomy cleanup: 29 categories, 136 subcategories (see AFAS_Category_Taxonomy.md)
- ✅ AMEX / AMEX DF sign flip completed
- ✅ Positive Expense decision framework established
- ✅ Hard Rule #6: Credits / Refunds / Matched Pairs policy

---

## Phase 3 — Automation & Analytics ✅ Complete

### 6. Timer-Based Ingestion
- ✅ timer_sync.py: daily 02:00 UTC timer trigger — multi-institution loop
- ✅ http_ingest.py: manual HTTP trigger retained for testing
- ✅ plaid_sync.py: shared sync module — sign convention fix deployed 2026-06-02
- ✅ Production Plaid tokens live: Chase, Associated Personal, Amex, NWM Tom, NWM Amy
- ✅ Azure Function deployed to Finance-ingest-Tom-v6 (East US 2, Flex Consumption)
- ✅ First production sync: Chase 913 rows, Associated Personal 111 rows, Amex 10 rows
- ✅ Source labels consolidated: ASSOCIATED → ASSOCIATED_PERSONAL, AMEX DF → AMEX
- ✅ Apple Card CSV import script created (import_apple_csv.py in imports\ folder)
- ✅ Apple Card March + April + May 2026 imported (444 rows total)
- ✅ enrich_transactions.py: bonus rule >$10,000 threshold confirmed implemented (ISSUE-005 closed)
- ✅ enrich_transactions.py: retry path bug fixed — removed stale category_normalized parameter
- ✅ Manual review of 46 Uncategorized/VENMO-Review transactions complete (ISSUE-004 closed)
- ⏳ Review 8 intentionally parked Apple Uncategorized rows
- ⏳ Baird Plaid connection failed — Baird support replied 2026-06-15 (expected fix did not resolve issue); second email sent, awaiting response (ISSUE-008)

### 7. Power BI Reporting Layer
- ✅ Azure SQL connection established in Power BI (Import mode)
- ✅ All 6 views created and validated (16_powerbi_views.sql)
- ✅ Calendar table (DAX) built with correct sort orders
- ✅ Star schema data model complete
- ✅ Monthly Spend page complete
- ✅ Cash Flow page complete
- ✅ Year Over Year page complete
- ✅ Top Merchants page complete — COALESCE merchant fix applied
- ✅ Transaction Review page complete (added 2026-06-01)
- ✅ Active data issues resolved: ~$27K gap closed, Sports/Clubs confirmed flat YoY, YoY review complete
- ✅ Report published to Power BI Service (My Workspace)
- ✅ Scheduled refresh live: daily 4:00 AM CT
- ✅ vw_top_merchants: COALESCE(merchant_normalized, merchant_name_raw, 'Unknown')
- ✅ vw_transactions_clean: same COALESCE applied

---

## Phase 4 — Accounts, Balances & Holdings 🔄 In Progress

### 8. Accounts & Balances
- ✅ Phase 4 tables created in Azure
- ✅ accounts table ALTERed: asset_class, display_name, owner, include_in_net_worth added
- ✅ physical_assets seeded: HOUSE-WFB-BELLE, CAR-TSLA-NEBULA, CAR-TSLA-STORM, CAR-TSLA-TRINITY
- ✅ physical_asset_valuations seeded 2026-06-17: house $1.29M, 3 Teslas (KBB estimates)
- ✅ insurance_assets + valuations seeded and live: NWM-TOM ($94,825.93), NWM-AMY ($55,058.52)
- ✅ NWM Plaid tokens live — cash value syncs monthly via monthly_sync.py
- ✅ liabilities seeded: Rocket Mortgage, US Bank LOC
- ✅ Rocket Mortgage confirmed unsupported by Plaid — liability_balances updated manually
- ✅ accounts backfilled: 10 known accounts inserted
- ✅ Associated balance pull live — 7 accounts, monthly sync via monthly_sync.py (ISSUE-010 resolved)
- ✅ NWM balance pull live → insurance_asset_valuations via monthly_sync.py
- ⏳ Principal Financial 401k Plaid token (ISSUE-009)
- ⚠️ Baird Plaid connection (ISSUE-008) — CSV fallback operational

### 9. Investments / Holdings
- ✅ baird_holdings table created with lot-level detail (holding_id = account+symbol+date+lotN)
- ✅ security_sectors table created and seeded (115 tickers)
- ✅ import_baird_holdings.py created — CSV import pipeline for all Baird accounts
- ✅ Historical data imported: 2024-01 through 2026-06-17 (5,509 rows, 11 accounts)
- ✅ Net worth verified: ~$8,271,359 (Investment + Physical + Insurance + Cash - Liabilities)
- ✅ Phase 4 Power BI views created: vw_net_worth, vw_holdings_summary, vw_asset_allocation, vw_liability_summary, vw_budget_vs_actual
- ⏳ Connect Phase 4 views to Power BI — add Net Worth, Holdings, Allocation report pages
- ⏳ Seed budget_targets table with initial annual targets
- ⏳ Principal Financial 401k Plaid Investments (when token acquired)
- ⏳ Plaid Investments for Baird (when ISSUE-008 resolves)

---

## Immediate Next Steps (Phase 4)
1. Connect Phase 4 Power BI views — add Net Worth, Holdings, Asset Allocation, Liability report pages
2. Seed budget_targets table with initial annual targets
3. Acquire Principal Financial 401k Plaid token (ISSUE-009)
4. Monthly Baird CSV procedure — export, add Account Name + Date, run import_baird_holdings.py
5. Update physical asset valuations monthly (Zillow + KBB)
6. Review 8 intentionally parked Apple Uncategorized rows

---

## Phase 5 — AI Agent Layer ⏳ Not Started

### 10. Spending Intelligence Agent
- ⏳ Agent reads vw_category_yoy, vw_monthly_spend, vw_cash_flow, vw_budget_vs_actual
- ⏳ Compares actual vs prior year / budget targets (budget_targets table)
- ⏳ Flags anomalies and generates spending adjustment recommendations

### 11. Portfolio Rebalancing Agent
- ⏳ Agent reads holdings data + external market data feed
- ⏳ Compares current allocation vs target allocation
- ⏳ Generates rebalancing recommendations based on market trends
- ⏳ Inputs: holdings snapshot + risk tolerance profile

### 12. Holdings Strategy Agent
- ⏳ Agent assesses current holdings vs risk tolerance (current and projected)
- ⏳ Recommends additions, reductions, or reallocation
- ⏳ Risk tolerance profile: structured table in FinanceDB (risk_profile)
- ⏳ Inputs: holdings, market data, risk profile

---

