# AFAS Project — Monthly Procedure
**Run this in order at the start of every month — covers every account/source
that needs monthly attention.** Consolidated 2026-09-16 from SessionStarter.md's
separate Baird/Apple/DB-Resume procedures. See DecisionLog.md for the history
and rationale behind individual steps; see BestMethods.md for the lessons
several of these steps encode.

---

## 0. Resume the Database (do this first — blocks everything else)
Azure SQL Free tier Serverless auto-pauses between uses. Every other step in
this procedure will fail (or silently produce a stale result) if FinanceDB is
paused.

1. **Resume FinanceDB in Azure Portal** — check first, before running
   anything else.
2. **Check source freshness** — query `vw_source_freshness` (or
   `dbo.plaid_sync_state` directly) for each source's `max_date` /
   `days_since_last_transaction`. Don't trust the automated timer's
   "Succeeded" status alone — function-level status does not reflect whether
   the internal SQL work completed. If a source looks stale, check
   **Application Insights traces** directly for the actual sync trace lines
   — that's ground truth, not the Portal Functions blade.
3. **If the automated run failed or a source is stale**, manually retrigger
   via Azure Portal Test/Run: `http_ingest`, `http_balance_ingest`,
   `http_nwm_sync` as needed.
4. This also covers verifying the automated monthly syncs that don't need a
   manual CSV step: **Associated Bank balances**, **NW Mutual (Tom + Amy)
   insurance cash values**, and **Principal 401k holdings** — all run via
   `monthly_sync.py`'s timer. Confirm each synced via Application Insights,
   not just "Succeeded" status.

---

## 1. Apple Card CSV Import
1. Export the Apple Card CSV for the month, drop into
   `Run_Monthly\imports\AppleCC\`.
2. Run `python import_apple_csv.py` from `Run_Monthly` (no filename needed —
   auto-discovers, auto-moves processed files to `imported\`).
3. Run `python enrich_transactions.py --unenriched-only` (the single
   enricher for every source as of 2026-09-08).
4. Continue to Step 3 below (the shared interactive Uncategorized review
   covers both Apple and HSA).

---

## 2. HSA (Bank of America) CSV Import
1. Export the Bank of America HSA cash-ledger CSV, drop into
   `Run_Monthly\imports\HSA\` (filename `HSA_Transactions_*.csv`).
2. Run `python import_hsa_transactions.py` — type/in_budget are set
   deterministically at import (4 known non-spending description types
   whitelisted; everything else treated as real spending/income, typed by
   amount sign). category/subcategory are left NULL for the enricher.
3. Run `python enrich_transactions.py --unenriched-only` (same single
   enricher as Apple — see Step 1).
4. If a new HSA Fund Summary export is available, also run
   `python import_hsa_holdings.py` (value-only import — no units/price in
   this export; snapshot date parsed from the filename).
5. Continue to Step 3 below.

---

## 3. Interactive Uncategorized Review (every month, both Apple and HSA together)
Pull **all** rows still at `category = 'Uncategorized'` — not just the
newly-imported ones — with `plaid_category_raw` (the source's raw
categorization field) **alongside** `merchant_name_raw`, not merchant name
alone. Work the list with Claude:
- confident merchant→category matches: a single yes/no batch confirmation
- genuinely ambiguous ones: an option card, best guess + alternatives
- if a recurring merchant turns up, add a `merchant_patterns` rule in the
  same pass so it doesn't reappear next month

Anything genuinely unclear stays `Uncategorized / General`
(`category_reviewed = 0`) — never invent a subcategory to fit it
(Category_Taxonomy.md's "closed list" rule).

**Dual-use merchants** (e.g. gas stations that also sell groceries/snacks)
are classified manually per-transaction, not given a blanket
`merchant_patterns` entry — same logic as the Work-Expense dual-use rule.
**Household has no gas-purchasing vehicles (all electric)** — a
gas-station-branded charge (Marathon, Raceway, ExxonMobil, …) is a
convenience-store purchase, not gasoline.

---

## 4. Baird Holdings CSV Import
1. Log into Baird Online, export the all-accounts holdings CSV.
2. Add an `Account Name` column — fill with the correct canonical account
   name per row (see the table below — **use "MAIN - BKG" for the main
   brokerage, not "MAIN - Brokerage"**; a wrong name has required a SQL
   correction more than once).
3. Add a `Date` column — fill all rows with the month-end date.
4. Add a manual cash sweep row if needed (symbol=CASH, Asset
   Classification=Cash and Cash Equivalents).
5. Drop the file into `C:\DEV_Projects\AFAS\Run_Monthly\imports\baird\` as
   `holdings_*.csv` (any name starting with "holdings_" works).
6. Run `python import_baird_holdings.py` from `Run_Monthly` — **this script
   does not auto-move processed files** (unlike `import_apple_csv.py`), so
   archive it manually afterward.

### Baird Account Name Conventions (canonical)
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

## 5. Physical Asset Valuations
Update if needed (Zillow for the house, KBB for the Teslas). **KBB "typical
mileage" figures can be significantly off for high-mileage vehicles** — get
actual VIN/mileage-specific values when possible rather than trusting a
general web search result.

---

## Notes
- Steps 1-3 (Apple/HSA/review) can run in either order relative to Step 4
  (Baird) — they don't depend on each other. Step 0 (DB resume) must always
  run first.
- See `BestMethods.md` for the lessons behind several of these steps
  (transaction_id normalization, MERGE overwrite protection, the taxonomy
  closed-list rule, etc.) and `IssuesTracker.md` for open issues these
  procedures touch (ISSUE-032, ISSUE-043, ISSUE-044).
- No dollar/currency figures belong in this file — see
  MASTER_CLAUDE_PROTOCOL.md Section 6a.
