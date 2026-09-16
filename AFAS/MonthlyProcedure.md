# AFAS Project — Monthly Procedure
**Run this in order at the start of every month — covers every account/source
that needs monthly attention.** Rewritten 2026-09-17 to match Tom's actual
standing routine (superseded the 2026-09-16 first draft, which was extracted
from SessionStarter.md's older per-source procedures and had drifted from
real practice in a few places); Step 1 updated same day after the two
monthly timers were deregistered in favor of a single combined manual
trigger (`http_monthly_ingest_all`). See DecisionLog.md for the history and
rationale behind individual steps; see BestMethods.md for the lessons several
of these steps encode.

---

## 0. Resume the Database (do this first — blocks everything else)
Azure SQL Free tier Serverless auto-pauses between uses. Every other step in
this procedure will fail (or silently produce a stale result) if FinanceDB is
paused. Both sub-steps below are required, not alternatives to each other.

1. **Resume FinanceDB in Azure Portal** — SQL databases > FinanceDB > Resume.
   Do this before anything else in the monthly routine.
2. **In VS Code, reconnect the SQL Server extension** to FinanceDB (enter the
   Azure password when prompted).

---

## 1. Trigger the Plaid Syncs
There is no automated monthly timer anymore — `timer_sync` and
`monthly_sync` were deregistered 2026-09-17 (they fired on the same
schedule Azure SQL was still auto-paused, ISSUE-032, and always needed a
manual DB resume beforehand anyway). Every monthly sync is now a deliberate
manual trigger, run here after Step 0's DB resume.

**Normal case — one click:** open `http_monthly_ingest_all` in the Azure
Portal, click Run/Test, select **default (function key)** as the key, click
Run. This runs all four sources in one call: transactions (Chase/Amex/
Associated Bank), Associated Bank balances, NW Mutual (Tom + Amy) valuations,
and Principal 401k holdings. Check the JSON response — it reports
`"status": "success"` or `"status": "partial_failure"` with a per-source
breakdown.

**If it reports a partial failure**, retrigger just the source(s) that
failed, the same way (Run/Test → default (function key) → Run):
- `http_ingest` — Chase / Amex / Associated Bank transactions
- `http_balance_ingest` — Associated Bank balances
- `http_nwm_sync` — NW Mutual (Tom + Amy) insurance cash values
- `http_principal_ingest` — Principal 401k holdings

---

## 2. Re-verify
Run the same source-freshness query again (`vw_source_freshness` or
`dbo.plaid_sync_state`) — confirm all four sources above now show a current
`MAX(date)`. Also check the **JSON response from each endpoint** for errors
(e.g. `ITEM_LOGIN_REQUIRED`) before moving on — don't trust the Portal
Test/Run panel's "Succeeded" status alone; that doesn't reflect whether the
internal SQL work actually completed.

---

## 3. Apple Card CSV Import
1. Copy the CSV file to `Run_Monthly\imports\AppleCC\`.
2. Rename it with no spaces — use underscores.
3. Run `python import_apple_csv.py` (auto-discovers the file regardless of
   exact name, auto-moves it to `imported\` once processed).

---

## 4. Baird Holdings Export and Import
1. Baird Online: **Investments → Holdings → Unrealized Gain/Loss → Export.**
   Account names are already included natively in this export — no manual
   `Account Name` column needed (see the canonical-name reference table
   below only to spot-check the export, not to fill anything in by hand).
2. Rename the file to `holdings_YYYY-MM-DD_ALL`, save as CSV, and add a
   `Date` column (all rows, month-end date).
3. Move the file to `Run_Monthly\imports\baird\`.
4. Add cash sweep rows: **Holdings → Asset Class → Cash and Cash
   Equivalents → All Cash Sweeps** — add one row per cash sweep (symbol=CASH,
   Asset Classification=Cash and Cash Equivalents).
5. Run `python import_baird_holdings.py`.
6. Move the processed file to `imports\baird\archive\` — **this script does
   not auto-move files** (unlike `import_apple_csv.py`), so this is a manual
   step every time.

### Baird Account Name Conventions (reference — for spot-checking the export, not manual entry)
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

**Use "MAIN - BKG" for the main brokerage, not "MAIN - Brokerage"** — a wrong
name in a past export required a SQL correction after the fact.

---

## 5. HSA (Bank of America) Exports
**Account Activity (transactions):**
1. Export account activity from the BofA HSA portal.
2. Rename to `HSA_Transactions_YYYY-MM-DD`.
3. Run `python import_hsa_transactions.py`. type/in_budget are set
   deterministically at import (4 known non-spending description types
   whitelisted; everything else treated as real spending/income, typed by
   amount sign) — category/subcategory are left NULL for the enricher.

**Fund Summary (holdings):**
4. Export via **Accounts → Investment Summary → Fund Activity Details →
   Export.**
5. Rename to `HSA_Fund_Summary_YYYY-MM-DD`.
6. Run `python import_hsa_holdings.py` (value-only import — no units/price in
   this export; snapshot date parsed from the filename).

---

## 6. Update Liabilities
1. Update the US Bank LOC balance if it's changed.
2. Update the current mortgage amount from Rocket Money.

---

## 7. Physical Asset Valuations
Update if needed (Zillow for the house, KBB for the Teslas). **KBB "typical
mileage" figures can be significantly off for high-mileage vehicles** — get
actual VIN/mileage-specific values when possible rather than trusting a
general web search result.

---

## 8. Enrichment
Run `python enrich_transactions.py --unenriched-only` **once, after every CSV
import above is done** (Apple, Baird, HSA) — not after each one individually.
This is the single enricher for every source (Chase/Amex/Associated Plaid
transactions, Apple Card CSV, HSA CSV) as of 2026-09-08; `enrich_apple_csv.py`
and `enrich_hsa_csv.py` were retired the same day. `--unenriched-only`
processes whatever's currently unenriched regardless of which import it came
from, so there's no need to run it per-source — confirmed this is the only
enrichment script in the pipeline.

---

## 9. Audit New Transactions (Interactive Uncategorized Review)
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

## 10. Power BI Refresh
Manual refresh in Power BI Desktop to confirm everything loaded cleanly —
check the Data Health page first for any freshness or categorization-quality
flags before trusting the rest of the report pages.

---

## Notes
- Steps 3-5 (Apple/Baird/HSA imports) can run in any order relative to each
  other — they don't depend on one another. Steps 0-2 (DB resume, sync
  triggers, re-verify) must always run first; Step 8 (enrichment) must run
  after all three imports, not interspersed between them.
- See `BestMethods.md` for the lessons behind several of these steps
  (transaction_id normalization, MERGE overwrite protection, the taxonomy
  closed-list rule, etc.) and `IssuesTracker.md` for open issues these
  procedures touch (ISSUE-032, ISSUE-043, ISSUE-044).
- No dollar/currency figures belong in this file — see
  MASTER_CLAUDE_PROTOCOL.md Section 6a.
