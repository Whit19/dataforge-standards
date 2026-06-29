# AFAS Project — Database Schema Reference
**Update this file when any table, view, or column changes.**
Last updated: 2026-06-01 (Phase 4 tables created in Azure; accounts backfilled; liabilities seeded)

---

## Schema Version History
| Date | Change |
|------|--------|
| 2026-03-09 | `category_normalized` column removed from all tables (100% null) |
| 2026-03-09 | `type` column confirmed as transaction_class: Income / Expense / Other |
| 2026-03-09 | `category_source` values defined: plaid / merchant_pattern / historical / manual / bonus_rule |
| 2026-03-10 | All `category_confidence` set to HIGH across all 14,730 rows |
| 2026-03-10 | All 6 Power BI views created and validated (16_powerbi_views.sql) |
| 2026-03-12 | Phase 4 table definitions designed: securities, account_balances, liabilities, liability_balances, physical_assets, physical_asset_valuations, insurance_assets — **pending Azure creation** |

---

## Tables

### `transactions`
Primary table. One row per Plaid transaction.

| Column | Type | Notes |
|--------|------|-------|
| transaction_id | varchar(100) PK | Plaid transaction_id |
| account_id | varchar(100) | FK → accounts |
| institution_id | varchar(100) | FK → institution_map |
| source | varchar(50) | e.g. CHASE, ASSOCIATED, APPLE |
| run_id | varchar(100) | FK → run_log |
| amount | decimal | **Positive = credit, Negative = debit** |
| date | date | Transaction date |
| authorized_date | date | |
| merchant_name_raw | varchar(300) | Raw from Plaid |
| merchant_normalized | varchar(200) | After merchant_patterns match |
| plaid_category_raw | varchar(300) | Plaid's raw category string |
| category | varchar(200) | Our taxonomy category |
| subcategory | varchar(200) | Our taxonomy subcategory |
| iso_currency_code | varchar(10) | |
| pending | bit | NULL for all existing rows — always use `ISNULL(pending,0)=0` in queries |
| payment_channel | varchar(50) | |
| transaction_type | varchar(50) | |
| description | varchar(500) | |
| category_source | varchar(50) | plaid / merchant_pattern / historical / manual / bonus_rule |
| category_confidence | varchar(10) | HIGH / MEDIUM / LOW — currently all HIGH |
| category_reviewed | bit | |
| last_enriched_at | datetime | |
| in_budget | bit | |
| type | varchar(10) | Income / Expense / Other (transaction_class) |
| original_category | varchar(200) | Pre-enrichment snapshot |
| original_subcategory | varchar(200) | Pre-enrichment snapshot |
| inserted_at | datetime | |
| updated_at | datetime | |

---

### `merchant_patterns`
Pattern matching for merchant normalization and category assignment.

| Column | Type | Notes |
|--------|------|-------|
| pattern | varchar PK | SQL LIKE pattern matched against merchant_name_raw |
| merchant_normalized | varchar | Clean merchant name |
| category | varchar | |
| subcategory | varchar | |
| priority | int | Lower number = higher priority |
| active | bit | |
| created_at | datetime | |
| updated_at | datetime | |

**Version History**
| Date | Change |
|------|--------|
| 2026-03-09 | 7 duplicate pattern groups removed |
| 2026-03-09 | 139 Dining Out patterns fixed |
| 2026-03-09 | 4 broad patterns tightened |

---

### `category_map`
Maps Plaid raw categories to our taxonomy. Used in enrichment step 2.

| Column | Type | Notes |
|--------|------|-------|
| plaid_category_raw | varchar PK | |
| category | varchar | |
| subcategory | varchar | |
| created_at | datetime | |
| updated_at | datetime | |
| category_version | varchar | Future-proof versioning |
| in_budget | bit | |
| type | varchar | Income / Expense / Other |

---

### `accounts`
One row per Plaid account. Covers all bank, brokerage, and credit accounts connected via Plaid.

| Column | Type | Notes |
|--------|------|-------|
| account_id | varchar(100) PK | Plaid account_id |
| institution_id | varchar(100) | FK → institution_map |
| account_name | varchar(200) | Plaid account name |
| account_type | varchar(50) | Plaid type: depository / investment / credit / loan |
| account_subtype | varchar(50) | Plaid subtype: checking / savings / brokerage / 401k / ira / etc. |
| mask | varchar(10) | Last 4 digits |
| official_name | varchar(200) | Full official account name from institution |
| currency | varchar(10) | ISO currency code |
| asset_class | varchar(50) | Financial / Cash / Liability — maps to your financial structure |
| display_name | varchar(100) | Friendly name for Power BI (e.g. MAIN - Brokerage, 529 - Alex) |
| owner | varchar(50) | Tom / Amy / Joint / Alex / Brooke / Whit — for household reporting |
| include_in_net_worth | bit | 1 = include in net worth calc, 0 = exclude (e.g. credit card payments) |
| created_at | datetime | |
| updated_at | datetime | |

---

### `holdings`
Investment holdings snapshots from Plaid Investments endpoint. One row per account/security/date combination.

| Column | Type | Notes |
|--------|------|-------|
| holding_id | varchar(100) PK | Composite: account_id + security_id + date |
| account_id | varchar(100) | FK → accounts |
| security_id | varchar(100) | FK → securities |
| quantity | decimal(18,6) | Number of shares/units held |
| price | decimal(18,4) | Price per unit at snapshot date |
| value | decimal(18,2) | quantity × price |
| cost_basis | decimal(18,2) | Total cost basis — from Plaid if available |
| date | date | Snapshot date |
| created_at | datetime | |
| updated_at | datetime | |

---

### `institution_map`
Maps institution IDs to human-readable names and sources.

| Column | Type |
|--------|------|
| institution_id | varchar PK |
| institution_name | varchar |
| source | varchar |
| created_at | datetime |
| updated_at | datetime |

---

### `plaid_sync_state`
Tracks Plaid sync cursor state per account/source for incremental sync.

| Column | Type | Notes |
|--------|------|-------|
| state_key | varchar PK | |
| last_cursor | varchar | Plaid /transactions/sync cursor |
| last_sync | datetime | |
| last_error | varchar | |
| transaction_count_added | int | |
| created_at | datetime | |
| updated_at | datetime | |

---

### `run_log`
One row per Function execution run.

| Column | Type |
|--------|------|
| run_id | varchar PK |
| source | varchar |
| started_at | datetime |
| finished_at | datetime |
| status | varchar |
| cursor | varchar |
| transactions_inserted | int |
| error_count | int |

---

### `error_log`
Individual error records linked to a run.

| Column | Type |
|--------|------|
| error_id | varchar PK |
| run_id | varchar |
| source | varchar |
| payload | varchar |
| error_message | varchar |
| created_at | datetime |

---

## Phase 4 Tables ⏳ Designed — Not Yet Created in Azure

---

### `securities`
One row per unique security. Referenced by `holdings`. Populated via Plaid Investments endpoint.

| Column | Type | Notes |
|--------|------|-------|
| security_id | varchar(100) PK | Plaid security_id |
| ticker | varchar(20) | Ticker symbol (e.g. AAPL, VTI) — NULL for mutual funds/money market |
| name | varchar(300) | Full security name |
| security_type | varchar(50) | equity / mutual_fund / etf / fixed_income / cash / other |
| cusip | varchar(20) | CUSIP identifier if available from Plaid |
| isin | varchar(20) | ISIN identifier if available from Plaid |
| close_price | decimal(18,4) | Most recent close price from Plaid |
| close_price_as_of | date | Date of close_price |
| iso_currency_code | varchar(10) | |
| created_at | datetime | |
| updated_at | datetime | |

---

### `account_balances`
Daily balance snapshots per account. Populated via Plaid `/accounts/balance/get`.

| Column | Type | Notes |
|--------|------|-------|
| balance_id | varchar(100) PK | Composite: account_id + snapshot_date |
| account_id | varchar(100) | FK → accounts |
| snapshot_date | date | Date of balance snapshot |
| current_balance | decimal(18,2) | Current balance from Plaid |
| available_balance | decimal(18,2) | Available balance (NULL for investment accounts) |
| limit_amount | decimal(18,2) | Credit limit — populated for credit/LOC accounts |
| iso_currency_code | varchar(10) | |
| created_at | datetime | |

---

### `liabilities`
One row per liability account. Covers mortgage and lines of credit via Plaid Liabilities endpoint.

| Column | Type | Notes |
|--------|------|-------|
| liability_id | varchar(100) PK | Plaid account_id for the liability |
| account_id | varchar(100) | FK → accounts |
| liability_type | varchar(50) | mortgage / line_of_credit / student / other |
| lender_name | varchar(200) | e.g. Rocket, US Bank |
| display_name | varchar(100) | Friendly name — e.g. WFB House Mortgage, Baird Stock Loan |
| origination_date | date | Loan origination date |
| origination_principal | decimal(18,2) | Original principal amount |
| interest_rate | decimal(8,4) | Annual interest rate |
| maturity_date | date | Loan maturity / payoff date |
| created_at | datetime | |
| updated_at | datetime | |

---

### `liability_balances`
Daily balance snapshots for liabilities. Mirrors `account_balances` pattern.

| Column | Type | Notes |
|--------|------|-------|
| balance_id | varchar(100) PK | Composite: liability_id + snapshot_date |
| liability_id | varchar(100) | FK → liabilities |
| snapshot_date | date | Date of balance snapshot |
| current_balance | decimal(18,2) | Outstanding principal balance |
| ytd_interest_paid | decimal(18,2) | Year-to-date interest paid — from Plaid if available |
| ytd_principal_paid | decimal(18,2) | Year-to-date principal paid — from Plaid if available |
| past_due_amount | decimal(18,2) | Past due amount if any |
| created_at | datetime | |

---

### `physical_assets`
One row per physical asset (house, vehicles). Manually maintained; valuations updated via API.

| Column | Type | Notes |
|--------|------|-------|
| asset_id | varchar(50) PK | Human-readable: e.g. HOUSE-WFB-BELLE, CAR-TSLA-NEBULA |
| asset_type | varchar(50) | real_estate / vehicle / other |
| display_name | varchar(100) | e.g. WFB Belle House, TSLA Nebula |
| owner | varchar(50) | Tom / Amy / Joint |
| purchase_date | date | Date of purchase |
| purchase_price | decimal(18,2) | Original purchase price |
| description | varchar(500) | e.g. 2022 Tesla Model Y, 4BR house at [address] |
| valuation_source | varchar(50) | zillow / kbb / manual |
| valuation_api_key | varchar(200) | API lookup key (Zillow zpid, KBB vehicle ID) — store in env vars, not here |
| active | bit | 1 = currently owned, 0 = sold/disposed |
| created_at | datetime | |
| updated_at | datetime | |

**Note:** `valuation_api_key` stores only the lookup identifier (e.g. Zillow zpid), not API credentials. Credentials stay in Azure Function environment variables.

---

### `physical_asset_valuations`
Valuation history for physical assets. Updated on each API refresh cycle.

| Column | Type | Notes |
|--------|------|-------|
| valuation_id | varchar(100) PK | Composite: asset_id + valuation_date |
| asset_id | varchar(50) | FK → physical_assets |
| valuation_date | date | Date of valuation estimate |
| estimated_value | decimal(18,2) | Estimated market value from API or manual entry |
| valuation_source | varchar(50) | zillow / kbb / manual |
| low_estimate | decimal(18,2) | Low end of range (Zillow/KBB provide ranges) |
| high_estimate | decimal(18,2) | High end of range |
| notes | varchar(500) | Manual override notes if applicable |
| created_at | datetime | |

---

### `insurance_assets`
Whole life insurance policies with trackable cash value. Manual entry — not on Plaid.

| Column | Type | Notes |
|--------|------|-------|
| insurance_id | varchar(50) PK | e.g. NWM-TOM, NWM-AMY |
| display_name | varchar(100) | e.g. NWM Whole Life - Tom |
| insurer | varchar(100) | e.g. Northwest Mutual |
| owner | varchar(50) | Tom / Amy |
| policy_type | varchar(50) | whole_life / universal_life / other |
| face_value | decimal(18,2) | Death benefit / face value of policy |
| policy_start_date | date | Policy inception date |
| annual_premium | decimal(18,2) | Annual premium paid |
| active | bit | 1 = active policy |
| created_at | datetime | |
| updated_at | datetime | |

---

### `insurance_asset_valuations`
Cash value history for whole life policies. Updated manually each period.

| Column | Type | Notes |
|--------|------|-------|
| valuation_id | varchar(100) PK | Composite: insurance_id + valuation_date |
| insurance_id | varchar(50) | FK → insurance_assets |
| valuation_date | date | Date of valuation |
| cash_value | decimal(18,2) | Current cash surrender value |
| notes | varchar(500) | e.g. from annual statement |
| created_at | datetime | |

---

### `budget_targets`
Annual budget targets by category with optional monthly overrides for known spikes.
Covers both expense and income categories. Manually maintained.

**Design pattern:**
- `is_default = 1` rows are the baseline budget — no `budget_year`, apply to any year unless overridden
- One row per category per year = year-specific annual target, overrides the default for that year
- Additional rows per category + year + month = monthly override for that specific month
- Each January, only add rows for categories where targets changed — everything else falls through to default

**Priority hierarchy for effective monthly budget:**
1. Monthly override for that specific year → use as-is
2. Annual target for that specific year → divide by 12
3. Default annual target (`is_default = 1`) → divide by 12
4. No match → NULL (unbudgeted)

| Column | Type | Notes |
|--------|------|-------|
| budget_id | int PK | Identity / auto-increment |
| budget_year | int | NULL when is_default = 1; e.g. 2025 for year-specific rows |
| category | varchar(200) | Must match taxonomy category exactly |
| month | int | NULL = annual target; 1–12 = monthly override for that month |
| target_amount | decimal(18,2) | Always positive — expense or income target |
| type | varchar(10) | Income / Expense — matches transaction type for the category |
| is_default | bit | 1 = baseline budget (no year); 0 = year-specific or monthly override |
| notes | varchar(500) | e.g. "Travel heavy — Europe trip in June", "Baird bonus expected Q4" |
| created_at | datetime | |
| updated_at | datetime | |

**Example rows:**
```
budget_id  budget_year  category   month  target_amount  type     is_default  notes
1          NULL         Travel     NULL   40000.00       Expense  1           Default annual travel budget
2          2025         Travel     NULL   55000.00       Expense  0           2025 override — Europe trip year
3          2025         Travel     6      10000.00       Expense  0           Monthly override — June Europe trip
4          2025         Travel     7      8000.00        Expense  0           Monthly override — July Europe trip
5          NULL         Pay        NULL   86528.00       Income   1           Default — Amy salary 26 x $3,328
6          2025         Bonus      NULL   175000.00      Income   0           2025 — expected Baird dividend + bonus
```

**Query pattern — effective monthly budget for a given year/month:**
```sql
-- Returns effective monthly budget per category for a given year + month.
-- Priority: monthly override → year annual → default annual → NULL
SELECT
    cat.category,
    cat.type,
    COALESCE(
        mo.target_amount,                    -- 1. monthly override this year
        ya.target_amount / 12,               -- 2. year annual / 12
        da.target_amount / 12                -- 3. default annual / 12
    ) AS monthly_budget
FROM (SELECT DISTINCT category, type FROM budget_targets) cat
LEFT JOIN budget_targets mo                  -- monthly override
       ON mo.category    = cat.category
      AND mo.budget_year = 2025              -- substitute target year
      AND mo.month       = 6                 -- substitute target month
      AND mo.is_default  = 0
LEFT JOIN budget_targets ya                  -- year-specific annual
       ON ya.category    = cat.category
      AND ya.budget_year = 2025
      AND ya.month       IS NULL
      AND ya.is_default  = 0
LEFT JOIN budget_targets da                  -- default annual
       ON da.category    = cat.category
      AND da.is_default  = 1
      AND da.month       IS NULL
WHERE COALESCE(mo.target_amount, ya.target_amount, da.target_amount) IS NOT NULL
ORDER BY cat.type, cat.category;
```

---

## Views (all created 2026-03-10 via 16_powerbi_views.sql)

| View | Purpose | Key Date Column | Calendar Relationship |
|------|---------|----------------|----------------------|
| vw_transactions_clean | Base view, used by all reports | date | Yes |
| vw_monthly_spend | Monthly expenses by category | month_start_date | Yes |
| vw_cash_flow | Income vs expense by month | month_start_date | Yes |
| vw_top_merchants | Top merchants rolling 12mo | last_seen | **No — standalone** |
| vw_category_yoy | YoY spending by category | year_start_date | Yes |
| vw_enrichment_quality | Data quality dashboard | — | No |

### Phase 4 Views ⏳ Planned — Not Yet Created
| View | Purpose | Key Date Column |
|------|---------|----------------|
| vw_net_worth | Total assets minus liabilities by snapshot date | snapshot_date |
| vw_account_balances | Account balance history across all accounts | snapshot_date |
| vw_holdings_summary | Holdings value by account, security type, and date | date |
| vw_asset_allocation | Portfolio allocation by asset class / security type | date |
| vw_liability_summary | Liability balances and paydown progress | snapshot_date |
| vw_budget_vs_actual | Actual spend/income vs budget target by category and month | month_start_date |

### Power BI Data Model Notes
- Calendar table is DAX-calculated and acts as the hub of the star schema
- All relationships: One-to-Many, Single cross-filter direction
- `vw_top_merchants` has no Calendar relationship — filter independently

### Calendar Table Sort Orders
| Column | Sort By |
|--------|---------|
| MonthName | Month |
| MonthShort | Month |
| YearMonth | YearMonthSort |
| YearQuarter | YearQuarterSort |

---

## Database State Snapshot

### As of 2026-03-12 (post SQL-18 execution)
| Metric | Value |
|--------|-------|
| Total transactions | 14,730 |
| Date range | 2020-01-01 → 2026-02-28 |
| Categories | 29 |
| Subcategories | 142 |
| Type: Expense | 13,774 |
| Type: Other | 570 |
| Type: Income | 386 |
| category_confidence | All HIGH |

### Enrichment Quality (as of 2026-03-10)
| Source | Confidence | Count | % |
|--------|-----------|-------|---|
| historical | HIGH | 8,603 | 58.4% |
| merchant_pattern | HIGH | 6,008 | 40.8% |
| manual | HIGH | 116 | 0.8% |
| bonus_rule | HIGH | 3 | 0.0% |

---

## SQL Scripts — Complete Run Order
```
Phase 2 (complete 2026-03-10):
  01_remove_category_normalized.sql
  02 through 15_final_subcategory_cleanup.sql

Phase 3 (complete 2026-03-10):
  16_powerbi_views.sql              ← all 6 views, includes month_start_date fix

Phase 3 (complete 2026-03-12):
  18_amex_sign_flip_corrections.sql ← TX 10120, 2222, 8797 (3 rows)
  19_subcategory_renames.sql        ← 8 subcategory renames across 6 categories

Phase 3 (pending execution):
  20_set_in_budget_flags.sql        ← in_budget = 1 for 26 categories;
                                       0 for Transfer, Payment, Taxes

Phase 3 (complete 2026-06-01):
  24_chase_merchant_patterns.sql    ← 16 new patterns + backfill of existing NULL merchant_normalized rows
  25_vw_top_merchants_coalesce_fix.sql  ← COALESCE merchant_normalized with merchant_name_raw
  26_vw_transactions_clean_coalesce_fix.sql ← same COALESCE fix

Phase 4 (complete 2026-06-01):
  21_phase4_tables.sql              ← 9 new tables + 4 accounts columns (holdings already existed)
  22_phase4_seed_data.sql           ← physical_assets, insurance_assets, valuations, liabilities, liability_balances
  23_phase4_accounts_backfill.sql   ← 10 accounts inserted, US Bank LOC liability added