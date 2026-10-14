# AFAS Project — Session Starter
> **Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
> Repo: github.com/Whit19/dataforge-standards
**Load this file at the start of every session. Update pick-up pointer before closing.**
Last updated: 2026-10-01 (Session 25 — October monthly close plus a MERGE-statement data-safety fix; see Pick Up Here for full detail. SQL watermark 135)

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

2026-10-01 (Session 25 — October monthly close): ran the full
MonthlyProcedure.md end to end, and along the way found and fixed a
structural data-safety bug — see DecisionLog for the complete writeup.
Short version: `import_hsa_transactions.py` and `import_apple_csv.py`
both used a `MERGE ... WHEN MATCHED THEN UPDATE` that unconditionally
overwrote `merchant_name_raw`/`type`/`in_budget` (HSA) or
`merchant_name_raw`/`type` (Apple) on every re-import, silently
reverting manually-reviewed work whenever a routine monthly re-sync
happened to re-match an already-fixed row. Found because today's HSA
resync reverted script 118's two corrections and all 60 ISSUE-043
Consumer Note merchant-text restorations from 2026-09-16. Both
importers rewritten to insert-only (an existing `transaction_id` is
never touched, full stop — scripts/commits `b1fd9aa`/`9516e44`); the
specific reverted rows restored (script 135). A CSV row that produces a
genuinely new `transaction_id` but is dated on/before the source's
current watermark is no longer auto-inserted either — it's logged for
manual review instead, since an insert-only importer never gets a
second chance to reconcile it. **This changes the insert-only importers'
monthly-review workflow slightly — watch the "old-dated needs review"
warning list on the next Apple/HSA import.**

Also this session: re-keyed the last 9 HSA rows still on the
pre-2026-09-01 raw-text hash scheme to the current normalized scheme
(script 134 — closes out the recurring-duplicate root cause for good,
confirmed 466/466 HSA rows now on one scheme); onboarded the new Baird
**IRA - PE Millennium** account (a CAIS-administered private-equity
feeder fund) into `import_baird_holdings.py`'s canonical account list
and mapped its one unmapped security (script 132); stopped storing
pending Plaid transactions at ingest at all (commit `6f232b9`) and
deleted the 40 already stored (script 131 — Tom explicitly waived the
usual "don't delete manually-reviewed rows" guard here, since every
downstream view already excludes pending rows and none of that
categorization work was ever visible or load-bearing); found and fixed
raw financial CSVs/scripts appearing in the deployed Function App's
package (`.funcignore` is confirmed NOT honored by this app's deploy
path — worked around via a physical move-aside-before-deploy technique,
now scripted as `scripts/deploy_function_app.ps1`); stopped tracking
the 23 financial CSV/XLSX files in git (`git rm --cached`, files kept on
disk, no history rewrite).

**Needs action next session or next month's regular downloads:**
1. **Confirm the `plaid_sync.py` sign fix (ISSUE-047) live.** Checked
   today (2026-10-01) — no Chase or Amex credit/refund has posted since
   the 2026-09-21 fix deployed, so there is still nothing to verify
   against. Keep watching for the next statement credit or return.
2. **Amex needed a second Link update-mode re-login today**
   (`ITEM_LOGIN_REQUIRED` recurred since 2026-09-22) and is reconnected
   and syncing again as of 2026-10-01 — but the batch it pulled (3 rows,
   all charges) had no credit in it, so the Amex-specific sign
   comparison from ISSUE-047 is still unverified. Re-run it once a real
   Amex credit shows up.
3. **`baird_holdings.sector` is still NULL for the new IRA - PE
   Millennium holding** (symbol 3063340) even though `security_sectors`
   is now mapped (script 132) — `import_baird_holdings.py` only
   consults `security_sectors` at import time, and October's import
   already ran before the mapping existed. Needs either a Baird import
   re-run or a one-line SQL backfill. Tracked as ISSUE-048.
4. The United Way Gmwc pending donation deleted today (script 131) had
   posted in Tom's Chase app on 9/30 but hadn't reached Plaid as of this
   morning's sync — watch for it on the next sync and make sure it's not
   missed now that it's no longer sitting as a pending row anywhere.
5. Optional, not urgent: decide whether to scrub the 23 financial
   CSV/XLSX files out of git history entirely (they're untracked going
   forward as of today, but still present in past commits); investigate
   why `.funcignore` has no effect on this Function App's deploy path
   (worked around, not root-caused).

**Future work — not started (Tom, 2026-09-21): a Power BI page that
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
| ISSUE-047 | High | `plaid_sync.py` stored refunds/statement credits as charges (`-abs`); fix deployed 2026-09-21 | Still no Chase/Amex credit has posted since the fix (checked 2026-10-01) — confirm on the first one that does; Amex reconnected 2026-10-01 but its batch had no credit to check |
| ISSUE-048 | Low | `baird_holdings.sector` NULL for IRA - PE Millennium's CAIS Millennium holding (symbol 3063340) — `security_sectors` mapped 2026-10-01 (script 132) after that month's import already ran | Re-run `import_baird_holdings.py` for October, or a one-line SQL backfill |
| ISSUE-016 | Medium | run_log missing entries for all daily transaction syncs | Add run_log writes to plaid_sync.py |

---

## Production Plaid Tokens (live, verified 2026-09-01)
| Token Variable | Institution | Status |
|----------------|-------------|--------|
| PLAID_ACCESS_TOKEN_CHASE | Checking/credit issuer A | ✅ Live |
| PLAID_ACCESS_TOKEN_ASSOCIATED_PERSONAL | Checking issuer B | ✅ Live |
| PLAID_ACCESS_TOKEN_AMEX | Credit card issuer C | ✅ Live — reconnected via Link update mode 2026-10-01 (was ITEM_LOGIN_REQUIRED again — third occurrence for this institution, see DecisionLog 2026-09-01/2026-10-01); synced successfully same day (3 rows) |
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

Moved to TechnicalArchitecture.md (Section 5c — keep SessionStarter lean): the per-script status table (**Python Scripts Inventory**) and the session-by-session **SQL Scripts — Run Order Reference**. Current SQL high watermark: **135** (confirmed live 2026-10-01 — re-confirm live rather than trust this number).

---

## Phase 4 Tables
| Table | Status | Notes |
|-------|--------|-------|
| securities | ✅ Live | Populated via principal_sync.py (14 rows as of 2026-09-14 — 12 Principal, 2 HSA). `sector`/`industry` columns added 2026-09-14 (script 96) — Plaid returns a generic 'Miscellaneous' sector for all 11 Principal holdings (no fund look-through); `vw_holdings_all` normalizes fund-type/Miscellaneous to 'Diversified' for display. `security_type` casing normalized 2026-09-14 (script 97) — was 'mutual fund' (Principal, live Plaid) vs 'mutual_fund' (HSA, hand-seeded Session 13), now one value. Cost basis is genuinely unavailable from Plaid for the 401k (confirmed via raw API check — `cost_basis: null`, `tax_lots: []` for all 11 holdings) — not a pipeline bug. |
| holdings | ✅ Live | Was missing cost_basis and updated_at columns despite being documented — added 2026-08-01 (script 51). Populated via principal_sync.py (11 rows, Principal 401k) and import_hsa_holdings.py (2 rows, HSA Bank of America, value-only — no units/price in that export). `vw_holdings_all` (script 95, 2026-09-14) is now the consolidated read path across this table and `baird_holdings` — see TechnicalArchitecture. |
| account_balances | ✅ Live | Associated 7 accounts |
| liabilities | ✅ Live | Rocket Mortgage (rate corrected to 2.625%, origination_principal backfilled), US Bank LOC |
| liability_balances | ✅ Live | Mortgage and US Bank LOC balances updated as part of the monthly procedure (MonthlyProcedure.md Step 6) — most recently 2026-08-01 for both; see the Power BI Net Worth page for current figures (no dollar totals kept in this file — Section 6a). |
| physical_assets | ✅ Live | WFB-Belle, TSLA Nebula/Storm/Trinity |
| physical_asset_valuations | ✅ Live | House valuation via Zillow Zestimate, pasted in monthly (Claude Code can't fetch Zillow directly — 403 to automated requests). Teslas switched 2026-09-16 to a flat 1%/month depreciation policy instead of a monthly KBB lookup — see MonthlyProcedure.md Step 7. |
| insurance_assets | ✅ Live | NWM-TOM, NWM-AMY |
| insurance_asset_valuations | ✅ Live | Both reconnected 2026-08-01, synced monthly via `http_monthly_ingest_all` (or `http_nwm_sync` individually). |
| baird_holdings | ✅ Live | 737 rows as of 2026-08-01 snapshot, reconciled to Baird's own CSV total + manual cash row to the penny. MAIN - Brokerage naming corrected to MAIN - BKG. |
| security_sectors | ✅ Live | 116 tickers mapped — CAIS Millennium (symbol 3063340, the IRA - PE Millennium account's holding) added 2026-10-01 (script 132); `baird_holdings.sector` for that specific October row is still NULL since the import already ran before the mapping existed (ISSUE-048) |
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
