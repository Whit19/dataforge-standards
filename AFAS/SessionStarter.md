# AFAS Project — Session Starter
> **Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
> Repo: github.com/Whit19/dataforge-standards
**Load this file at the start of every session. Update pick-up pointer before closing.**
Last updated: 2026-11-04 (Session 30 — Holdings plan rebuilt as a rolling 8-year forecast; 401(k) fixed income shift made and recorded; Python tax estimate and the Estimated Tax agent added. SQL watermark 178)

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
| Machines | Desktop (original) and a Windows **ARM64** laptop (set up 2026-11-01): Python 3.12 arm64 + ODBC Driver 18 via winget, project venv at `.venv` (pyodbc, python-dotenv, pandas, openpyxl, flask, plaid-python). The laptop runs **pandas 3.x** — pandas 2.x has no win-arm64 wheel; enrichment verified clean on it. Each machine's public IP needs its own Azure SQL firewall rule. |
| get_plaid_tokens.py | C:\DEV_Projects\AFAS\scripts\get_plaid_tokens.py — now supports Plaid Link update mode (existing-token field per institution) as of 2026-08-01 |
| Amount convention | Positive = credit/income, Negative = debit/expense |
| Manual override rule | category_source = 'manual' — pipeline NEVER overwrites |
| Pending column | Always use ISNULL(pending,0) = 0 in all queries |
| Azure SQL auto-pause | Free tier Serverless — resume before 1st-of-month imports (the Portal may show no Resume button; a connection attempt wakes it — see MonthlyProcedure Step 0). Can also collide with the automated monthly timer itself (ISSUE-032) — function status reports "Succeeded" even when this happens, so don't trust status alone |

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
| Phase 4 | Accounts/balances, investments/holdings | ✅ Complete (Tom, 2026-11-02) |
| Phase 5 | AI agent layer | 🔄 In Progress — Spending Intelligence ✅, Holdings Strategy ✅, Portfolio Rebalancing ✅, Estimated Tax ✅ (open items below) |

---

## Pick Up Here — Next Session

2026-11-04 (Session 30 — rolling 8-year Holdings plan, 401(k) shift recorded, Python tax estimate, Estimated Tax agent). All agents are in AFAS `main` (latest `4b2b9fe`) and publish to Notion; each parent page needs the Notion connection added (a 404 means it was not).

**What changed (details in DecisionLog 2026-11-04 and TechnicalArchitecture):**
- **Holdings plan is a rolling 8-year forecast (2027-2034).** The plan runs one extra internal year so the last December column exists. Each year move `plan_first_year` / `plan_last_year` forward one and add that year's Marquette adjustment row. Amy retires at the end of 2034; retirement income and spending after that are NOT modelled yet.
- **Dividend:** actual December 2026 per-share amount, then a flat per-share planning amount (Tom: err on the safe side). Book value per share (price of new shares and the value of Baird stock) rises at a conservative planning rate; it is NOT derived from Tom's six percentages (those were dividend as a percent of book value). Each December: taxes, then cash for next year (gap + Baird Capital call + floor), then Baird shares (half the rest, rounded to 50 shares), then LOC paydown.
- **Cash floor is 3 months** of that year's budgeted spend (setting `core_floor_months`; it is a planning rule, not an outside source). Mortgage (principal and interest only) stops after the 9/1/2031 payment and is not grown. Baird Capital commitment is called evenly over five years from 2027, paid from cash.
- **Amy's pay model:** base pay from the paystub; bonus pre-tax is 1.4x base in 2027 and 1.0x after; after-tax bonus figures are separate settings; profit share = flat + 3.5% of base + bonus; pre-tax 401(k) additions (deferral at the limit, match, profit share) are separate from the after-tax 401(k), which is converted (Super-Roth) to Baird stock inside IRA ROTH - Baird Stock. No catch-up (Tom wants the cash flow). HSA uses the paystub amount plus the employer contribution. IRA Baird accounts reinvest their dividends.
- **401(k) fixed income:** Tom made the fund shift and the new contribution election on 2026-11-04. They are recorded in `pending_position_changes` (pro-forma until the next Principal import; it stops applying by itself once that account has a newer snapshot) and `contribution_elections`. Plan: the 401(k) alone carries overall fixed income to the 10% target (no moves in the other retirement accounts); the projection reaches the target around Dec 2031 if the match and profit share follow the election (Tom confirmed they do). Revisit the 401(k) allocations then.
- **Taxes:** `scripts/tax_engine.py` is a Python port of Tom's `Estimated Tax Calculation.xlsx` (federal incl. AMT, net investment income tax and additional Medicare tax; Wisconsin), verified to match the workbook to the dollar. The December 2026 tax bucket is the workbook's Q4 payment; later Decembers use the extra tax the dividend causes from the engine. New agent `scripts/tax_agent.py` publishes "Estimated Tax Reviews" (env NOTION_TAX_PAGE_ID) with the estimate, payments, safe-harbor test, payment options, dividend sensitivity, an extra-withholding idea, a workbook cross-check and the 8-year tax outlook.
- The Holdings review shows a milestones list, cash flow by year, holdings by year with where each change comes from, a fixed income path table and an estimated-tax-by-year table.

### PRIORITY #1 — keep the tax and holdings inputs current
1. **Paystub:** add each new Amy paystub to `paystub_snapshots` (the tax projection is year-to-date plus remaining paychecks); update `tax_inputs` (estimated payments made, realized gains, other investment income) after each payment or 1099.
2. **When the December Baird dividend is declared:** update `baird_dividend_per_share_2026`, re-run `holdings_agent.py`, `rebalancing_agent.py` and `tax_agent.py`; the Q4 estimated payment is due 1/15; filing 4/15.
3. **After the next Principal and Baird holdings imports:** confirm `pending_position_changes` stopped applying, set `pending_baird_capital_call` to 0 and lower `baird_capital_unfunded_commitment` as calls are paid.
4. **Tax assumptions to confirm (ISSUE-053):** realized gains for later years, other investment income held at the 2026 estimate, bracket indexing, and Tom's workbook as the cross-check.
5. Remaining ISSUE-052 items (see IssuesTracker).

### Other open items
See ISSUE-052 (holdings/rebalancing assumptions) and ISSUE-053 (tax estimate inputs). The Anthropic API key expires 2027-02-03 (`ANTHROPIC_API_KEY` in `.env`). yfinance is free, delayed and unofficial: treat market context as indicative.

### Previous session — Session 29 (2026-11-03)
All three Phase 5 agents built and published; Baird stock reclassified as a private book-value alternative outside the allocation targets (80/10/3/7 ex-Baird); Baird Capital commitment; four-bucket December dividend; SQL 155-165. Superseded in part by Session 30 (flat dividend per share, rolling 8-year plan, 3-month floor, tax method).

### Previous session — Session 28 (2026-11-02)
Phase 4 declared complete; Spending Intelligence and Holdings Strategy
agents first built (`3a6da77`, `d21bbe4`, `54d3bab`), Notion publishing via
`scripts/notion_publish.py`, SQL 151-154. The 75/15/3/7 target and
"Baird counts as Equities" decision from that session are superseded by
Session 29 (DecisionLog 2026-11-03).

### Previous session — Session 27 (2026-11-01)
2026-11-01 (Session 27 — November monthly close): ran the full
MonthlyProcedure.md end to end, for the first time from the new ARM64
laptop (environment set up from scratch — see Environment above). Every
step closed: Plaid syncs current for all sources, Apple / Baird holdings
+ activity / HSA transactions + holdings imported, liabilities and
physical assets posted (scripts 147-148), all 29 Uncategorized rows
worked (script 149), Needs Review cleared to 0 (script 150), Power BI Data
Health confirmed clean by Tom.

Changes this session:
- **`import_apple_csv.py` watermark now compares Clearing Date against
  `MAX(authorized_date)`** instead of purchase date vs `MAX(date)`
  (commit `700e390`). The insert-only rewrite's old-dated hold fired on 5
  genuinely new rows on its first real month — 3 purchases dated the last
  day of September that cleared in October, and 2 Southwest refunds Apple
  back-dated to the original purchase date. A clearing-date watermark lets
  those through while still holding a drifted re-send. Dry run went
  169 + 5 held → 174 + 0 held.
- **Baird now exports `.xlsx`** (both Holdings and Activity). Converted to
  CSV with accounting-style negatives — `_parse_currency()` strips a bare
  minus sign and would have flipped 344 unrealized losses to gains.
  MonthlyProcedure Steps 4/4a updated.
- **Grand Geneva is Work - Expense / Amy** (Tom's standing rule): May and
  October 2026 rows recategorized, all 6 Grand Geneva patterns repointed,
  exact pattern added for the new `Grande Geneva` spelling. 2024-2025 rows
  deliberately left as they were.
- **ISSUE-048 resolved** (October sector backfilled in script 150;
  November's import picked the mapping up on its own).
- **ISSUE-047 confirmed live for Chase** — two real Chase credits since
  the fix (a refund and a Pay Yourself Back credit) both stored positive.
  Amex still unverified.
- Amex needed a 4th Link update-mode re-login (`ITEM_LOGIN_REQUIRED`);
  token unchanged, so no settings update needed.

**Session 26 (2026-10-06) was never synced to these docs** — no Pick Up
Here, DecisionLog or TimeLog entry, and no Notion daily note was found.
From the AFAS git log only: scripts 136-146 (Apple type/in_budget mapping,
category_map Cash Advance, October review follow-ups, illy coffee rule,
2025 Baird activity layout + Sold - Cancel, Future Projections tables /
view / account map), the shared `account_scrub.py` account-number masker
for Baird activity text, and the Future Projections tooling in `scripts/`
(assumptions workbook generator, loader, history seeder). See the SQL
run-order list in TechnicalArchitecture.md. The projections work means
the "Future work" item below is no longer "not started".

**Needs action next session or next month's regular downloads:**
1. **ISSUE-049 — Chase autopay stored positive.** Plaid has labeled the
   Chase card autopay `INCOME_SALARY` since August, so `plaid_sync.py`'s
   inflow branch stores it as `+abs` (June/July were `LOAN_PAYMENTS`,
   negative). Categorized Payment / Other (out of budget), but any view
   summing raw amounts sees it as an inflow. Likely fix: treat
   `AUTOMATIC PAYMENT - THANK` as a card payment regardless of Plaid's
   label, plus a backfill of the Aug-Oct rows. Function-App file — needs
   deploy + deployed-content verification.
2. **ISSUE-050 — Apple "Gas" → Car / Gas via category_map** conflicts with
   the no-gas-vehicles rule (one October Quick Mart row needed manual
   correction). Decide whether to repoint that category_map row to
   Groceries / General or to Uncategorized for per-transaction review.
3. **ISSUE-047 (Amex half):** still waiting on a real Amex credit.
4. Optional: teach the Baird importers to read `.xlsx` directly (would
   remove the manual conversion and the minus-sign trap).
5. Optional, carried over: scrub the 23 financial CSV/XLSX files from git
   history; root-cause why `.funcignore` has no effect on this Function
   App's deploy path.
6. Backfill Session 26 (2026-10-06) into DecisionLog/TimeLog if Tom
   remembers the reasoning behind it.

**Future work — started in Session 26, 2026-10-06 (Tom, 2026-09-21): a Power BI page that
projects future years.** Tom built the model in Excel and wants it in
Power BI. The Excel workbook is the spec; it has two halves:
1. *Cash flow by category by year, with the ability to change a category's
   assumption for any year.* Columns are historical years (2020 through
   last year, actuals) then estimate years (current year through five
   years out), each with an amount and a percent-change-vs-prior-year
   column. Rows: **Budgeted Expenses** (Housing, Groceries / Dining Out,
   Sports / Clubs, Travel, Personal Care / Clothing, Gifts / Charity,
   Children, Entertainment / Subscription, Car, Medical / Health,
   Work - Expense, ATM / Cash Spending) with a total; **Budgeted Income**
   (Pay, Bonus, Work - Reimb, Mobile Deposit) with a total; Budgeted Cash
   Flow; **Other Expenses** (One Time, Taxes, Fee, LOC Interest Payment)
   with a total; **Other Income** (Baird Stock Div, Dividend, HSA Deposit,
   Interest) with a total; Other Cash Flow; and All Annual Cash Flow.
2. *Overall holdings / net worth roll-forward, year-end by year.* Rows:
   Fixed Assets (Car, Home), Cash (Checking, DF Checking, Insurance,
   Savings, Equities), Equity (Baird Stock, Education 529, HSA, Equities),
   Retirement (401k/IRA, Roth IRA), Loans (LOC, Mortgage), Net Worth Year
   End; plus household age rows (parents, each child) and a per-year Notes
   row holding the life-event assumptions that drive the numbers (college
   start and finish, loan payoff, contribution changes, and similar).

Design notes for when this is picked up:
- The workbook's "Budgeted" vs "Other" split lines up with the
  `in_budget` flag, and its current-year estimate column lines up with
  `dbo.budget_targets` — both already exist in SQL. Its category names are
  the older combined ones (e.g. Groceries / Dining Out), not the current
  taxonomy, so a mapping is needed.
- Editable per-year assumptions need somewhere to live: Power BI cannot
  write back. Options are an assumptions table in SQL (like
  `budget_targets`, edited by script), what-if parameters, or a writeback
  tool — decide before building.
- The balance-sheet half can start from the latest `vw_holdings_all` /
  `vw_net_worth` snapshot plus a growth rate and contribution per account;
  `vw_net_worth_all_time` already has monthly history back to 2011.
- The workbook's later-year Total Liabilities and Net Worth cells show
  `#REF!` errors — compute them, don't copy them.
- The workbook itself is not in the repo; ask Tom for a copy when this
  starts.

## Active Data Issues
| Issue | Priority | Description | Next Step |
|-------|----------|-------------|-----------|
| ISSUE-047 | High | `plaid_sync.py` stored refunds/statement credits as charges (`-abs`); fix deployed 2026-09-21 | **Chase confirmed live 2026-11-01** (two real credits stored positive). Amex half still unverified — confirm on the first real Amex credit |
| ISSUE-049 | Medium | Chase card autopay labeled `INCOME_SALARY` by Plaid since August, so stored positive (inflow) instead of negative | Payment-text override in `plaid_sync.py` + Aug-Oct backfill; deploy + verify deployed content |
| ISSUE-050 | Low | Apple CSV "Gas" category → Car / Gas via `category_map`, conflicting with the no-gas-vehicles rule | Repoint the category_map row (Groceries / General, or Uncategorized for per-transaction review) |
| ISSUE-052 | Medium | Holdings / Rebalancing agents: remaining unconfirmed assumptions (narrowed Session 30) | Walk the list with Tom; store confirmed values in settings |
| ISSUE-053 | Medium | Estimated tax: typed inputs and later-year assumptions | Keep paystub and tax inputs current; re-run tax_agent.py after each change |
| ISSUE-016 | Medium | run_log missing entries for all daily transaction syncs | Add run_log writes to plaid_sync.py |

---

## Production Plaid Tokens (live, verified 2026-09-01)
| Token Variable | Institution | Status |
|----------------|-------------|--------|
| PLAID_ACCESS_TOKEN_CHASE | Checking/credit issuer A | ✅ Live |
| PLAID_ACCESS_TOKEN_ASSOCIATED_PERSONAL | Checking issuer B | ✅ Live |
| PLAID_ACCESS_TOKEN_AMEX | Credit card issuer C | ✅ Live — reconnected via Link update mode 2026-11-01 (`ITEM_LOGIN_REQUIRED` again — fourth occurrence for this institution; token unchanged after re-login, so no settings update needed); synced 4 rows same day |
| PLAID_ACCESS_TOKEN_NWM_TOM | Whole life insurer — Tom's policy | ✅ Live — reconnected via Link update mode 2026-08-01 (was ITEM_LOGIN_REQUIRED) |
| PLAID_ACCESS_TOKEN_NWM_AMY | Whole life insurer — Amy's policy | ✅ Live — reconnected via Link update mode 2026-08-01 (was ITEM_LOGIN_REQUIRED) |
| PLAID_ACCESS_TOKEN_PRINCIPAL | 401k provider, owner Amy — recordkept via a separate plan administrator | ✅ Live — first connection 2026-08-01 (ISSUE-009 resolved after being open since 2026-06-02). Resolves a prior naming ambiguity between two names for the same account — confirmed same account. |
| *(no Plaid token — CSV fallback)* | Brokerage institution (transactions/investments via Plaid) | ⚠️ Still blocked — CSV fallback operational (unrelated to the 401k row above, which connects independently) |
| *(no Plaid token — unsupported)* | Mortgage lender | ❌ Confirmed unsupported — manual updates only |

---

## Monthly Procedure

See `MonthlyProcedure.md` (new file, same repo/folder) for the full
start-of-month checklist — DB resume, Apple/HSA imports, the shared
Uncategorized review, Baird holdings, and physical asset valuations, all in
one place and in run order.

---

## Python Scripts & SQL Script Log

Moved to TechnicalArchitecture.md (Section 5c — keep SessionStarter lean): the per-script status table (**Python Scripts Inventory**) and the session-by-session **SQL Scripts — Run Order Reference**. Current SQL high watermark: **150** (2026-11-01 — re-confirm live rather than trust this number).

---

## Phase 4 Tables
| Table | Status | Notes |
|-------|--------|-------|
| securities | ✅ Live | Populated via principal_sync.py (14 rows as of 2026-09-14 — 12 Principal, 2 HSA). `sector`/`industry` columns added 2026-09-14 (script 96) — Plaid returns a generic 'Miscellaneous' sector for all 11 Principal holdings (no fund look-through); `vw_holdings_all` normalizes fund-type/Miscellaneous to 'Diversified' for display. `security_type` casing normalized 2026-09-14 (script 97) — was 'mutual fund' (Principal, live Plaid) vs 'mutual_fund' (HSA, hand-seeded Session 13), now one value. Cost basis is genuinely unavailable from Plaid for the 401k (confirmed via raw API check — `cost_basis: null`, `tax_lots: []` for all 11 holdings) — not a pipeline bug. |
| holdings | ✅ Live | Was missing cost_basis and updated_at columns despite being documented — added 2026-08-01 (script 51). Populated via principal_sync.py (11 rows, Principal 401k) and import_hsa_holdings.py (2 rows, HSA Bank of America, value-only — no units/price in that export). `vw_holdings_all` (script 95, 2026-09-14) is now the consolidated read path across this table and `baird_holdings` — see TechnicalArchitecture. |
| account_balances | ✅ Live | Associated 7 accounts |
| liabilities | ✅ Live | Rocket Mortgage (rate corrected to 2.625%, origination_principal backfilled), US Bank LOC |
| liability_balances | ✅ Live | Mortgage and US Bank LOC balances updated as part of the monthly procedure (MonthlyProcedure.md Step 6) — most recently 2026-11-01 for both (script 147); see the Power BI Net Worth page for current figures (no dollar totals kept in this file — Section 6a). |
| physical_assets | ✅ Live | WFB-Belle, TSLA Nebula/Storm/Trinity |
| physical_asset_valuations | ✅ Live | House valuation via Zillow Zestimate, pasted in monthly (Claude Code can't fetch Zillow directly — 403 to automated requests). Teslas switched 2026-09-16 to a flat 1%/month depreciation policy instead of a monthly KBB lookup — see MonthlyProcedure.md Step 7. |
| insurance_assets | ✅ Live | NWM-TOM, NWM-AMY |
| insurance_asset_valuations | ✅ Live | Both reconnected 2026-08-01, synced monthly via `http_monthly_ingest_all` (or `http_nwm_sync` individually). |
| baird_holdings | ✅ Live | 737 rows as of 2026-08-01 snapshot, reconciled to Baird's own CSV total + manual cash row to the penny. MAIN - Brokerage naming corrected to MAIN - BKG. |
| security_sectors | ✅ Live | 116 tickers mapped — CAIS Millennium (symbol 3063340, the IRA - PE Millennium account's holding) added 2026-10-01 (script 132); the October row's NULL `baird_holdings.sector` was backfilled 2026-11-01 (script 150, ISSUE-048 resolved) |
| budget_targets | ✅ Live | Seeded 2026-09-15 (seed_budget_targets.py) — 240 rows, 20 categories × 12 months, `budget_year = 2026`. Combined old-budget categories (Groceries/Dining Out, Personal Care/Clothing, Entertainment/Subscriptions, Housing blending Housing+Bills & Utilities) split using real trailing-12-month actual spend ratios from `dbo.transactions`, not guessed. `is_default = 0` (month-specific override — required by `vw_budget_vs_actual`'s join logic) and `target_amount` stored positive (view computes `actual_amount` as always-positive `SUM(ABS(amount))`) — both discovered as real bugs against the pre-existing view, not assumed. |
| net_worth_history | ✅ Live | New 2026-09-15 (script 104). 2011-2026 backfill from Tom's manually-tracked CSV, `account_key` matching `vw_holdings_all`'s own scheme so historical + live union cleanly. `source_detail` = `CSV_IMPORT` or `INTERPOLATED` (linear interpolation across the interior gap between each account's last CSV value and first live value). Loaded via scripts/load_net_worth_history.py + scripts/interpolate_net_worth_gaps.py — see DecisionLog for the account-lineage decisions (Tom 401k rollover, HSA custodian history, NWM Tom/Amy split, the Vanguard/MAIN-BKG carve-out). |

---

## Data Health (Power BI)

Self-serve report page (added Session 13) built on:
- **vw_source_freshness** — row_count / max_date / min_date /
  days_since_last_transaction per source. Only ever queries
  `dbo.transactions` — has no rows for NW Mutual or the Principal 401k
  (see `vw_account_freshness` below).
- **vw_account_freshness** (added 2026-09-16, script 106/107) —
  account-level freshness across *every* monthly sync, including the two
  `vw_source_freshness` misses. One row per (source, account), not per
  source, plus `latest_value` (the value at that account's own latest
  snapshot) so freshness and "does the number look right" check in one
  query. See TechnicalArchitecture for the full column list.
- **vw_category_health** — uncategorized / manual_override / low_confidence /
  subcategory_mirror_violation counts per source
- **vw_needs_review** (added Session 17) — every `category_reviewed = 0` row,
  with a computed `suggested_category` / `suggested_subcategory` (from
  merchant_patterns + historical carry-forward, `'unmatched'` sources
  excluded) and a `review_priority` tier: 1 = no suggestion + currently
  Uncategorized; 2 = no suggestion + has a value; 3 = suggestion disagrees
  with current; 4 = suggestion matches current. Drives the **Needs Review**
  page. Backlog was cleared 421 → 0 on 2026-09-03; a fresh batch (49 rows,
  all priority 4 — the auto-suggestion already agreed, just needed
  `category_reviewed` flipped) was bulk-confirmed 2026-09-16 as part of
  MonthlyProcedure.md's new Step 10 "Final Review" — do this every month
  rather than letting priority-4 rows accumulate. **Any table visual on
  this page must include `transaction_id` (hidden) with numeric columns
  set to "Don't summarize"** — without the key, the visual silently
  groups + SUMs `review_priority`/`amount` (see BestMethods).
- **vw_potential_duplicates** (added Session 17) — flags two shapes:
  `legacy_vs_plaid` (a legacy bulk-import row sharing date/amount with a row
  that has a genuine non-null `pending` flag) and `pending_vs_settled` (the
  known pending/settled pattern). As of 2026-09-03: 0 legacy_vs_plaid (all 5
  real ones cleaned), 0 pending_vs_settled (4 cleaned). **Not yet a Power BI
  tile** — planned follow-up.

All standalone, no Calendar relationship. Check this page first before
running ad-hoc SQL to verify a data load — that's what it's for.

---

## Net Worth Summary
Net worth is tracked live in `vw_net_worth` (current snapshot) and
`vw_net_worth_all_time` (full 2011-2026 monthly history) — see the Power BI
Net Worth page for current figures. This file intentionally no longer
hardcodes dollar totals (see MASTER_CLAUDE_PROTOCOL.md Section 6a) since a
specific net worth figure doesn't belong in a doc fetched via a public raw
GitHub URL.

As of the 2026-09-15 session, category structure is Cash / Investment /
Retirement / Physical / Liability (Retirement split out from Investment;
Insurance folded into Cash; DF Checking included; cash-equivalent holdings
inside Investment/Retirement accounts correctly counted under Cash). See
DecisionLog 2026-09-15 for the reclassification detail.

As of 2026-09-16, the Teslas' Physical valuation no longer comes from a
monthly KBB lookup — a flat 1%/month depreciation is applied to each
vehicle's own last recorded value instead (Tom's standing decision, not
meant to be exact). See MonthlyProcedure.md Step 7 and DecisionLog
2026-09-16.
*Excludes HSA-Baird and any other non-canonical Baird accounts (never
captured in any import — see ISSUE-017). This is a different account from
"HSA - Bank of America" above.*

Full 2011-2026 monthly history (not just the current snapshot) is available in `dbo.vw_net_worth_all_time` — see Phase 4 Tables (`net_worth_history`) and DecisionLog for how it's built.
