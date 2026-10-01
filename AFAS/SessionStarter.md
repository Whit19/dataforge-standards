# AFAS Project — Session Starter
> **Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
> Repo: github.com/Whit19/dataforge-standards
**Load this file at the start of every session. Update pick-up pointer before closing.**
Last updated: 2026-10-01 (Session 25 — October monthly close plus a major data-safety fix: discovered and fixed a structural bug where the HSA and Apple Card import scripts' MERGE statements silently reverted manually-reviewed categorization work on every re-sync, rewrote both to insert-only, and restored the specific reverted rows; onboarded the new Baird IRA - PE Millennium account; stopped storing pending Plaid transactions; fixed a raw financial data exposure in the deployed Function App package via a move-aside deploy workaround (now scripted); stopped tracking CSV/XLSX exports in git. SQL watermark 135)

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
| PLAID_ACCESS_TOKEN_CHASE | Chase (ins_56) | ✅ Live |
| PLAID_ACCESS_TOKEN_ASSOCIATED_PERSONAL | Associated Bank Personal (ins_116823) | ✅ Live |
| PLAID_ACCESS_TOKEN_AMEX | American Express | ✅ Live — reconnected via Link update mode 2026-10-01 (was ITEM_LOGIN_REQUIRED again — third occurrence for this institution, see DecisionLog 2026-09-01/2026-10-01); synced successfully same day (3 rows) |
| PLAID_ACCESS_TOKEN_NWM_TOM | NW Mutual Tom whole life | ✅ Live — reconnected via Link update mode 2026-08-01 (was ITEM_LOGIN_REQUIRED) |
| PLAID_ACCESS_TOKEN_NWM_AMY | NW Mutual Amy whole life | ✅ Live — reconnected via Link update mode 2026-08-01 (was ITEM_LOGIN_REQUIRED) |
| PLAID_ACCESS_TOKEN_PRINCIPAL | Baird Profit Sharing and Savings Plan (401k), owner Amy — recordkept via Principal | ✅ Live — first connection 2026-08-01 (ISSUE-009 resolved after being open since 2026-06-02). Real account name: "BAIRD PROFIT SHARING AND SAVINGS PLAN" / "Prft Shr 401(K) Def Thrift". This resolves the "Principal 401k" vs "401k Baird Profit Share" naming ambiguity — same account. |
| Baird (ins_117067) | All Baird accounts (transactions/investments via Plaid) | ⚠️ Still blocked — CSV fallback operational (unrelated to Principal 401k, which connects independently) |
| Rocket Mortgage | Mortgage | ❌ Confirmed unsupported — manual updates only |

---

## Monthly Procedure

See `MonthlyProcedure.md` (new file, same repo/folder) for the full
start-of-month checklist — DB resume, Apple/HSA imports, the shared
Uncategorized review, Baird holdings, and physical asset valuations, all in
one place and in run order.

---

## Python Scripts (key files)
| File | Purpose | Status |
|------|---------|--------|
| plaid_sync.py | Shared sync module | ✅ 2026-10-01 (commit `6f232b9`): `_sync_institution()` now skips any transaction Plaid reports as `pending` (added/modified) instead of storing it — eliminates the pending/settled duplicate-row class at the source; logged via `skipped_pending` in the returned dict. `removed` transaction_ids from Plaid are still only counted (`len(removed)`), never acted on — no DELETE is issued for them; this is by design, not a gap (confirmed by code read 2026-10-01). Earlier: both known bugs fixed AND deployed 2026-09-01 (confirmed via VS Code's "Files (Read-only)" remote view — both had been drafted-but-undeployed since Session 14): (1) plaid_category_raw now extracts a clean category string instead of storing full JSON; (2) INFLOW_CATEGORIES no longer includes BANK_FEES/LOAN_PAYMENTS (ISSUE-023). Still does not write to run_log (ISSUE-016, carried over). |
| enrich_transactions.py | **The single enricher for every source** (Plaid CHASE/ASSOCIATED_PERSONAL/AMEX, Apple Card CSV, HSA CSV) as of 2026-09-08 | ✅ 2026-09-16 (AFAS 3b6989d): fixed the "only write real changes" comparison — it compared rows by *position*, not `transaction_id`, so a full run flagged 9,604 of 16,754 rows as changed (most were untouched — `enrich_from_merchant_patterns`/`enrich_from_history` both reorder rows via `pd.concat(ignore_index=True)`, breaking positional alignment with the pre-pipeline snapshot). Not a data-safety bug (`write_results()` targets `transaction_id`) but destroyed `updated_at` as a "genuinely modified" signal. Verified: an immediate full re-run now reports "0 of 16,754 rows actually changed." Earlier: 2026-09-08 (AFAS b1a3ef3) `enrich_apple_csv.py` + `enrich_hsa_csv.py` retired — this file already had no source filter; `load_transactions()` date cutoff removed; `apply_fallback()` now only fills `in_budget`/`type` where NULL. **Matcher note:** only leading/trailing `%` are wildcards; a mid-pattern `%` is dead. Earlier still: `enrich_from_history()` carry-forward step (2026-09-03, AFAS 557cdd1 + 4f0c052); write-back scoped to changed rows (ISSUE-035); apply_fallback() defaults + `--unenriched-only` retry (2026-08-03). |
| scripts/taxonomy_audit.py | Read-only taxonomy-drift diagnostic (4 checks: undocumented category/subcategory combos in merchant_patterns / category_map / transactions; same-priority pattern shadowing). AFAS 13b959e. | ✅ 2026-09-08: (a) ISSUE-041 (AFAS 6c652ae) — parses `Category_Taxonomy.md`'s `## Full Taxonomy` block live every run, no hardcoded dict, exits 2 on parse failure; (b) AFAS 1db78e4 — Check 4 no longer false-flags no-wildcard exact-match patterns (`ACT`, `IRS`) as substring collisions. As of session end: Checks 1/2/3 = 0, Check 4 = 207 (8 benign `[DIFFERENT DEST]` + 199 cosmetic `[same dest]`). |
| scripts/plaid_transaction_name_check.py | **NEW 2026-09-03** — read-only Plaid /transactions/get diagnostic (CHASE/ASSOCIATED_PERSONAL/AMEX); prints raw name / merchant_name / PFC. No writes, not in any pipeline. AFAS 2ad1bf2. | ✅ Live |
| plaid_client.py | Plaid SDK wrapper | ✅ Ready |
| balance_sync.py | Associated balance pull | ✅ Live |
| nwm_sync.py | NWM Tom + Amy cash value sync | ✅ Live — both Items reconnected 2026-08-01 |
| principal_sync.py | Pulls Principal/Baird 401k holdings via Plaid Investments (/investments/holdings/get), upserts dbo.accounts/dbo.securities/dbo.holdings. Created 2026-08-01. | ✅ 2026-09-14 (ISSUE-009 closed, AFAS 0cee6b6): wired into the pipeline, now run via `http_monthly_ingest_all` (Session 22) rather than the deregistered `monthly_sync.py` timer. Also captures Plaid's `sector`/`industry` fields into `dbo.securities` (sql/96) — **confirmed live in production 2026-09-16**: queried `dbo.securities` directly after a real `http_monthly_ingest_all` run and found 10 of 11 current 401k holdings carrying real sector/industry data (`Miscellaneous` / `Investment Trusts or Mutual Funds`) with `updated_at` timestamped that same run. The 11th (MINGX) is a holding sold out of the account before August — its security row is a stale pre-fix leftover, not evidence the fix is missing; harmless since it's no longer an active holding. |
| import_hsa_transactions.py | Imports Bank of America HSA cash-ledger CSV (Run_Monthly/imports/HSA/HSA_Transactions_*.csv) into dbo.transactions. type/in_budget set deterministically at import (not enrichment-dependent) — 4 known non-spending description types whitelisted, everything else treated as real spending/income typed by amount sign. category/subcategory left NULL for `enrich_transactions.py`. Created 2026-08-01. transaction_id hashes normalized (parsed) date/amount — fixed 2026-09-01 after BofA export-formatting drift caused ~400 duplicate groups; watermark check warns on unexpectedly-new IDs. UTF-8 stdout fix applied. | ✅ **Rewritten insert-only 2026-10-01** (commit `b1fd9aa`) — the prior `MERGE ... WHEN MATCHED THEN UPDATE` unconditionally overwrote `merchant_name_raw`/`type`/`in_budget` on every re-sync, silently reverting manually-reviewed work (found when today's resync reverted script 118's corrections and 60 ISSUE-043 merchant-text restorations — restored via script 135). Now a plain INSERT, guarded by an in-memory existing-ID set; a new ID dated on/before the HSA watermark is logged for manual review instead of inserted. `merchant_name_raw` now falls back Merchant Name → Consumer Note → Description. Supports `--dry-run`. Also: the last 9 rows on the pre-2026-09-01 raw-text hash scheme re-keyed to the current normalized scheme (script 134) — 466/466 HSA rows now on one scheme. Live — 466 canonical rows |
| import_hsa_holdings.py | Imports Bank of America HSA "Fund Summary" CSV (value-only, no units/price available in this export) into dbo.holdings. Snapshot date parsed from filename. Created 2026-08-01. UTF-8 stdout fix applied 2026-09-01. | ✅ Live — 2 holdings, value confirmed reconciled to the CSV export. |
| import_baird_holdings.py | Baird holdings CSV → baird_holdings | ✅ Ready — Total-row detection bug fixed 2026-08-01 (was checking wrong column). **2026-10-01:** added `"IRA - PE Millennium"` (new CAIS-administered private-equity feeder account) to `CANONICAL_ACCOUNT_NAMES`; blank Cost/Unit Cost/Unrealized G/L cells on that account's holding confirmed to parse to NULL cleanly. |
| import_baird_activity.py | **NEW 2026-09-16** — Baird Activity CSV (buys/sells, fees, dividends/interest/cap gains) → `dbo.baird_activity`, categorized by `vw_baird_activity`. Pure review/visibility, does not feed Budget vs Actual. Reuses `import_baird_holdings.py`'s account-name normalization + currency parser. Handles two different column layouts Baird has already used across export vintages (auto-detected from the CSV header), plus a plain-signed vs accounting-style Amount/Price format difference between them. `activity_id` hashes parsed values + an occurrence counter (never raw CSV text) so Tom's normal overlapping monthly export window is idempotent on re-import. | ✅ Live — 673 rows backfilled (2026-01-01 through 2026-09-16, BKG/PIM history plus all 9 other Baird accounts from May onward); verified idempotent by reimporting both files with no row-count change. **2026-10-01:** added `Deposit-journal`/`Withdrawal-journal`/`Other-adjust` (new activity types from funding IRA - PE Millennium) to `KNOWN_ACTIVITY_TYPES`; `vw_baird_activity` maps all three to Cash Movement (script 133). |
| monthly_sync.py | Monthly timer (1st of month, 03:00 UTC) — balance_sync + nwm_sync + principal_sync | ⚠️ **Deregistered 2026-09-16** (AFAS bd3ce83) — no longer imported in `function_app.py`, so this timer does not fire. Deliberately disabled: it fired on the same schedule Azure SQL was still auto-paused (ISSUE-032), always needing a manual DB resume beforehand anyway. `http_monthly_ingest_all` (in `http_ingest.py`) now covers the same syncs, plus transactions, as one manual call. File left in place for reference, not deleted. |
| import_apple_csv.py | Apple Card monthly CSV import. Lives in Run_Monthly\, auto-discovers files, auto-moves to imported\. Stores Apple's CSV "Category" column in `plaid_category_raw` (no Plaid involvement — the column name is a legacy misnomer). | ✅ **Rewritten insert-only 2026-10-01** (commit `9516e44`), mirroring the same-day HSA importer fix — the prior MERGE's `WHEN MATCHED` partially protected `category_source`/`category_confidence`/`in_budget` but still unconditionally overwrote `merchant_name_raw`/`type` on every re-sync (same flaw class as the HSA bug). Now a plain INSERT with an existing-ID guard and a watermark check (new-but-old-dated rows logged for review, not inserted); supports `--dry-run`. `transaction_id` hashing itself (raw-text, not normalized) deliberately left unchanged — out of scope for this fix. Earlier: 2026-09-08 (AFAS b1a3ef3): inserts `category_source = NULL` (was `'historical'` — collided with the ~6k genuinely backfilled `'historical'` rows); MERGE re-default guard now treats only `NULL`/`'unmatched'` as "not yet enriched". Enrichment is now `enrich_transactions.py` (the Apple-specific enricher was retired). 2026-09-01: MERGE CASE protection for already-enriched rows; UTF-8 stdout fix. |
| timer_sync.py | Monthly timer trigger (1st @ 03:00 UTC) — Plaid transactions (Chase/Amex/Associated Personal) | ⚠️ **Deregistered 2026-09-16** (AFAS bd3ce83) — same reasoning as `monthly_sync.py` above; no longer imported in `function_app.py`. `http_monthly_ingest_all` covers this too. File left in place for reference. |
| http_ingest.py | Manual HTTP triggers — http_ingest, http_balance_ingest, http_nwm_sync, http_principal_ingest, and (2026-09-16, AFAS bd3ce83) **http_monthly_ingest_all** — runs all four in one call, the one-click replacement for the two now-deregistered timers. The 4 single-source routes stay for targeted retries (e.g. only one source hit `ITEM_LOGIN_REQUIRED`). | ✅ Live |
| get_plaid_tokens.py | Local Flask tool for Plaid token acquisition | ✅ 2026-09-16 (AFAS 602f0d3): each institution's existing-token box now pre-fills automatically from `local.settings.json` (the script already loaded these into the environment for `PLAID_CLIENT_ID`/`PLAID_SECRET` — no reason the reauth boxes couldn't too). Fixes a live failure: an empty token box launched a brand-new Link session instead of true update mode, and institution search matched the wrong entity ("Kabbage" instead of Amex). Earlier: added Plaid Link update-mode support (existing-token field per institution) and NW Mutual Tom/Amy cards, both 2026-08-01. |
| db.py | DB connection | ✅ Ready — retry-with-backoff on Azure SQL error 40613 (auto-pause collision) added 2026-09-02, confirmed present in live code. Largely moot as of 2026-09-16 — the automated timers that used to collide with auto-pause (ISSUE-032) are deregistered; every sync is now a manual trigger run after an explicit DB resume (Step 0 of MonthlyProcedure.md). |
| scripts/deploy_function_app.ps1 | **NEW 2026-10-01** — wraps VS Code's "Deploy to Function App" step. `.funcignore` is confirmed NOT honored by this app's Flex Consumption deploy path (verified by downloading and unzipping the actual deployed package twice), so this script physically moves non-runtime folders (`Run_Monthly\imports`, `scripts`, `sql`, `.claude`, `powerbi`, plus any stray `*.csv`/`*.xlsx`) to a location genuinely outside the repo before the manual VS Code deploy step, then restores everything afterward (`finally` block, always runs, verifies each path on restore). Self-relaunches from a temp copy first since it needs to move its own parent folder (`scripts\`). Warns loudly but does not move `local.settings.json`/`.env`. **Use this for every future Function App deploy** — a prior deploy had shipped raw financial CSVs and this repo's own scripts folder inside the deployed package. | ✅ Live — `-WhatIf` dry-run tested, one real deploy completed clean (verified via direct package download/unzip). |

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
deletions. **Gap accepted 2026-09-16** — Tom's call: this SQL is already
live in the DB; reconstructing it as numbered scripts now is paperwork
with no functional benefit. Not being reconstructed.

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

Session 19 cont. (2026-09-08):
                           88_issue043_apple_uncategorized_backlog.sql
                           89_issue012_checks1-3_undocumented_combos.sql
                           90_issue012_check4_shadow_collisions.sql
                           91_issue012_final_check1-2_items.sql
                           92_issue012_check4_hotel_summerfest_cluster.sql
                           93_issue012_check_pattern_and_rent_map_cleanup.sql

Session 20 (2026-09-14):
                           94_vw_net_worth_add_principal_401k.sql
                           95_vw_holdings_all_consolidated.sql
                           96_securities_sector_industry_diversified_fix.sql
                           97_securities_type_normalize_and_asset_classification.sql
                           98_vw_net_worth_include_hsa.sql

Session 21 (2026-09-15):
                           99_vw_holdings_all_single_net_worth_table.sql
                           100_vw_holdings_all_retirement_category.sql
                           101_df_checking_include_kids_rename.sql
                           102_vw_holdings_all_account_hierarchy.sql
                           103_net_worth_category_insurance_to_cash.sql
                           104_net_worth_history_table.sql
                           105_vw_net_worth_all_time_monthly.sql

Session 22 (2026-09-16):
                           106_vw_account_freshness.sql
                           107_vw_account_freshness_add_value.sql
                           108_tesla_monthly_depreciation_september2026.sql
                           109_house_valuation_september2026.sql
                           110_drop_dead_holdings_views.sql
                           111_baird_activity_table.sql
                           112_vw_baird_activity.sql
                           113_vw_baird_activity_add_asset_bought.sql
                           114_ira_tom_missing_pim_fees.sql
                           115_ira_roth_tom_missing_pim_fees.sql
                           116_backfill_2025_budget_sep_dec_from_2026.sql
                           117_property_tax_budget_2026.sql
                           118_hsa_income_type_and_in_budget_fixes.sql

Session 23 (2026-09-21):
                           119_fix_budget_vs_actual_refund_netting.sql
                           120_budget_vs_actual_signed_net_by_type.sql
                           121_rideshare_credit_to_travel_transportation.sql
                           122_credits_stored_as_charges_sign_fix.sql

Session 24 (2026-09-22):
                           123_liability_balances_september2026.sql
                           124_drop_retired_summary_views.sql

Session 25 (2026-10-01):
                           125_hsa_drift_duplicate_cleanup_october2026.sql
                           126_liability_balances_october2026.sql
                           127_physical_asset_valuations_october2026.sql
                           128_september2026_uncategorized_review.sql
                           129_september2026_needs_review_corrections.sql
                           130_vw_needs_review_pending_filter.sql
                           131_pending_transaction_cleanup.sql
                           132_security_sectors_cais_millennium.sql
                           133_vw_baird_activity_journal_adjust_types.sql
                           134_hsa_rekey_old_hash_scheme.sql
                           135_hsa_restore_reverted_review_work.sql

Current high watermark: **135** (confirmed live 2026-10-01 — re-confirm
live rather than trust this number next session too. Note: code changes
committed to AFAS `main` this session that are NOT numbered SQL scripts —
plaid_sync.py (pending-transaction skip, commit `6f232b9`),
import_hsa_transactions.py + import_apple_csv.py (rewritten insert-only,
commits `b1fd9aa`/`9516e44`), import_baird_holdings.py (IRA - PE
Millennium canonical name), import_baird_activity.py (new
KNOWN_ACTIVITY_TYPES), scripts/deploy_function_app.ps1 (new, move-aside
deploy workaround), .funcignore/.gitignore (financial-data exclusion —
`.funcignore` confirmed to have no effect on this app's deploy path;
`.gitignore` fix is the one that actually works), 23 financial CSV/XLSX
files untracked from git via `git rm --cached` (kept on disk, no history
rewrite). Session 17's ad-hoc SQL gap was formally accepted 2026-09-16,
not reconstructed — see DecisionLog.)

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
