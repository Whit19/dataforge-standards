# AFAS Project — Session Starter
> **Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
> Repo: github.com/Whit19/dataforge-standards
**Load this file at the start of every session. Update pick-up pointer before closing.**
Last updated: 2026-06-17 (Session 10 sync)

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
| get_plaid_tokens.py | C:\DEV_Projects\AFAS\scripts\get_plaid_tokens.py |
| Total transactions | ~15,090+ (as of 2026-06-02) |
| Date range | 2020-01-01 → 2026-05-31 |
| Amount convention | Positive = credit/income, Negative = debit/expense |
| Manual override rule | category_source = 'manual' — pipeline NEVER overwrites |
| Pending column | Always use ISNULL(pending,0) = 0 in all queries |
| Azure SQL auto-pause | Free tier Serverless — resume manually via Azure Portal before 1st-of-month Apple CSV import |

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
**Phase 4: Power BI holdings pages, budget targets, Principal token, remaining cleanup**

1. **Full Category_Taxonomy.md audit** (ISSUE-012) — every category/subcategory/in_budget flag reviewed against live category_map/merchant_patterns
2. **Verify Chase/Amex/Associated transactions are actually present in dbo.transactions** (ISSUE-013)
3. **Investigate root cause of subcategory-mirror recurrence** (ISSUE-014)
4. **Baird activity/dividend/fee ingestion** (deferred earlier this session)
5. **HONA security_sectors entry** — confirm final classification is correct

---

## Active Data Issues
| Issue | Priority | Description | Next Step |
|-------|----------|-------------|-----------|
| ISSUE-008 | Medium | Baird Plaid connection fails (ins_117067). CSV pipeline now built as permanent fallback — manual monthly export covers all accounts. | Await Baird response. CSV fallback operational. |
| ISSUE-009 | Low | Principal Financial 401k Plaid token not yet acquired | Run get_plaid_tokens.py and complete connection |
| NOTE | — | 8 Apple Uncategorized rows intentionally parked — category_source = 'manual', in_budget = 1 | Owner review when ready |

---

## Key Policies & Rules (quick reference)
| Rule | Detail |
|------|--------|
| Manual overrides | category_source = 'manual' — never overwritten by pipeline |
| Dual-use merchants | Hotels, restaurants, airlines NEVER mapped to Work-Expense in merchant_patterns — correct work trips manually with category_source = 'manual' |
| merchant_patterns updates | Only when: new unrecognized merchant appears, or existing pattern consistently miscategorizes |
| merchant_patterns priority | More specific patterns must fire at lower priority number than broad ones (e.g. %UNITED WAY% at 3 before %UNITED% at 5) |
| Positive Expense | Cross-period refunds → keep as Expense (negative spike acceptable) |
| Hard Rule #6 | Credits/Refunds/Matched Pairs policy — see Decision Log 2026-03-10 |
| in_budget | Transfer, Payment, Taxes, HSA Deposit = 0; all other categories = 1 |
| Taxes excluded from budget | Baird dividend-driven — unpredictable, not budgetable |
| type column | Income / Expense / Other only — not free text |
| Subcategory naming | Must never mirror parent category name — use 'General' instead |
| Sign convention | plaid_sync.py: INCOME/TRANSFER_IN → positive (abs); LOAN_PAYMENTS/BANK_FEES → negative; all other debits → negated (-tx["amount"]) |
| Reimbursements | Always recorded as positive amounts in same category/subcategory as original expense |
| Work trips | Dual-use merchant charges corrected manually; category_source = 'manual' so pipeline never overwrites |
| Baird CSV holding_id | account_name + "_" + symbol + "_" + snapshot_date + "_lot{N}" — lot counter resets per file, guarantees uniqueness across multiple lots of same symbol |
| Baird CSV cash rows | Blank Security ID treated as symbol='CASH' — not skipped |
| SQL CREATE VIEW batches | Always use GO between view definitions — mssql extension requires each CREATE VIEW as its own batch |

---

## Production Plaid Tokens (live)
| Token Variable | Institution | Status |
|----------------|-------------|--------|
| PLAID_ACCESS_TOKEN_CHASE | Chase (ins_56) | ✅ Live |
| PLAID_ACCESS_TOKEN_ASSOCIATED_PERSONAL | Associated Bank Personal (ins_116823) | ✅ Live |
| PLAID_ACCESS_TOKEN_AMEX | American Express (ins_10) | ✅ Live |
| PLAID_ACCESS_TOKEN_NWM_TOM | NW Mutual Tom whole life | ✅ Live |
| PLAID_ACCESS_TOKEN_NWM_AMY | NW Mutual Amy whole life | ✅ Live |
| PLAID_ACCESS_TOKEN_PRINCIPAL | Principal Financial 401k | ⏳ Pending (ISSUE-009) |
| Baird (ins_117067) | All Baird accounts | ⚠️ Blocked — Baird-side issue (ISSUE-008). CSV fallback operational. |
| Rocket Mortgage | Mortgage | ❌ Confirmed unsupported — manual updates only |

---

## Python Scripts (key files)
| File | Purpose | Status |
|------|---------|--------|
| plaid_sync.py | Shared sync module — sign convention fix deployed 2026-06-02 | ✅ Ready |
| plaid_client.py | Plaid SDK wrapper — get_account_balances() passes request object directly (fix 2026-06-17) | ✅ Ready |
| balance_sync.py | Phase 4 — sync_account_balances(): /accounts/balance/get → upsert account_balances. Tested live 2026-06-17. | ✅ Live |
| nwm_sync.py | Phase 4 — sync_nwm_valuations(): NWM Tom + Amy cash values → insurance_asset_valuations. Created 2026-06-17. | ✅ Live |
| import_baird_holdings.py | Phase 4 — Baird holdings CSV → baird_holdings. Lot-level detail, holding_id = account+symbol+date+lotN. Created 2026-06-17. | ✅ Ready |
| monthly_sync.py | Monthly timer (1st of month, 03:00 UTC) — balance_sync + nwm_sync. Created 2026-06-17. | ✅ Deployed |
| enrich_transactions.py | Enrichment engine (Plaid transactions) | ✅ Ready |
| enrich_apple_csv.py | Enrichment for Apple Card CSV | ✅ Ready |
| import_apple_csv.py | Apple Card monthly CSV import (manual, ~1st of month) | ✅ Ready |
| timer_sync.py | Daily timer trigger (02:00 UTC) — transactions only | ✅ Ready |
| http_ingest.py | Manual HTTP triggers — http_ingest, http_balance_ingest, http_nwm_sync | ✅ Ready |
| get_plaid_tokens.py | Local Flask tool for Plaid token acquisition | ✅ Ready |
| db.py | DB connection — username/password local, Managed Identity in Azure | ✅ Ready |

---

## Monthly Baird Holdings Import Procedure
1. Log into Baird Online, export all accounts holdings CSV
2. Add `Account Name` column — fill with correct account name per row
3. Add `Date` column — fill all rows with month-end date (e.g. 6/30/2026)
4. Add manual cash sweep row if needed (symbol=CASH, Asset Classification=Cash and Cash Equivalents)
5. Drop file into `C:\DEV_Projects\AFAS\imports\baird\` as `holdings_YYYY-MM-DD_ALL.csv`
6. Run `python import_baird_holdings.py`
7. Move processed file to `imports\baird\archive\`
8. Update physical asset valuations if needed (Zillow for house, KBB for Teslas)

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
2. Export Apple Card CSV for the month
3. Run `import_apple_csv.py`
4. Run `enrich_apple_csv.py`
5. Review any unmatched merchants and classify via SQL
6. Add new merchant_patterns for recurring merchants

---

## SQL Scripts — Run Order Reference

Phase 1–2 (complete):      01–15_*.sql

Phase 3 (complete):        16_powerbi_views.sql through 26_*.sql

Phase 4 (complete 2026-06-01):
                           21_phase4_tables.sql
                           22_phase4_seed_data.sql
                           23_phase4_accounts_backfill.sql

Phase 4 (complete 2026-06-17):
                           35_insurance_valuations_add_source.sql
                           36_baird_holdings_tables.sql
                           37_security_sectors_seed.sql (61 rows)
                           38_security_sectors_additions.sql (34 rows — run ALTER + INSERT separately)
                           39_physical_asset_valuations_seed.sql
                           40_phase4_powerbi_views.sql (5 views, use GO between each)
                           41_security_sectors_additions2.sql (34 rows)
                           42_baird_holdings_holding_id_fix.sql (TRUNCATE — re-import after)

Phase 4 (pending):         budget_targets seed
                           Power BI Phase 4 report pages

---

## Power BI Data Model (star schema)
| View | Calendar Relationship | Status | Notes |
|------|-----------------------|--------|-------|
| vw_transactions_clean | date | ✅ Live | |
| vw_monthly_spend | month_start_date | ✅ Live | |
| vw_cash_flow | month_start_date | ✅ Live | |
| vw_top_merchants | **None** | ✅ Live | Standalone |
| vw_category_yoy | year_start_date | ✅ Live | |
| vw_enrichment_quality | **None** | ✅ Live | Standalone |
| vw_net_worth | snapshot_date | ✅ Created | Phase 4 — connect to Power BI next session |
| vw_holdings_summary | snapshot_date | ✅ Created | Phase 4 — connect to Power BI next session |
| vw_asset_allocation | snapshot_date | ✅ Created | Phase 4 — connect to Power BI next session |
| vw_liability_summary | snapshot_date | ✅ Created | Phase 4 — connect to Power BI next session |
| vw_budget_vs_actual | month_start_date | ✅ Created | Phase 4 — empty until budget_targets seeded |

### Report Pages
| Page | Status |
|------|--------|
| Monthly Spend | ✅ Complete |
| Cash Flow | ✅ Complete |
| Year Over Year | ✅ Complete |
| Top Merchants | ✅ Complete |
| Transaction Review | ✅ Complete |
| Net Worth | ⏳ Next session |
| Holdings | ⏳ Next session |
| Asset Allocation | ⏳ Next session |

---

## Phase 4 Tables
| Table | Status | Notes |
|-------|--------|-------|
| securities | ✅ Created | For Plaid Investments when Baird/Principal unblock |
| account_balances | ✅ Live | Associated 7 accounts — monthly sync via monthly_sync.py |
| liabilities | ✅ Created + seeded | Rocket Mortgage, US Bank LOC |
| liability_balances | ✅ Created + seeded | Current balance snapshots |
| physical_assets | ✅ Created + seeded | WFB-Belle, TSLA Nebula/Storm/Trinity |
| physical_asset_valuations | ✅ Seeded 2026-06-17 | House $1,290K, Nebula $72K, Storm $24.5K, Trinity $21K |
| insurance_assets | ✅ Created + seeded | NWM-TOM, NWM-AMY |
| insurance_asset_valuations | ✅ Live | Monthly sync via monthly_sync.py. Tom $94,825.93, Amy $55,058.52 |
| baird_holdings | ✅ Live | 5,509 rows — 2024-01 through 2026-06-17, all 11 accounts |
| security_sectors | ✅ Live | 115 tickers mapped (equities, ETFs, mutual funds, fixed income, options, PE) |
| budget_targets | ✅ Created | ⏳ Not yet seeded |

---

## Net Worth Summary (as of 2026-06-17)
| Category | Value |
|----------|-------|
| Investment (Baird — 11 accounts) | $7,231,246 |
| Physical Assets | $1,407,500 |
| Insurance (NWM Tom + Amy) | $149,884 |
| Cash (Associated — 7 accounts) | $150,085 |
| Liabilities | -$667,356 |
| **Total Net Worth** | **~$8,271,359** |
*Note: Excludes Principal 401k (token pending) and any accounts added when Baird Plaid resolves.*
