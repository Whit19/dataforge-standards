# AFAS Project — Best Methods
**Hard-won lessons. Add entries as they are learned. Never delete.**
Last updated: 2026-09-21

---

## How to Use This File
Read before writing any new code or SQL for AFAS. Each entry captures a real mistake or
discovered constraint — not theoretical guidance. If a lesson is relevant to what you are
building, treat it as a hard rule.

---

## Python — Plaid SDK

### plaid-python ≥ 24.0.0: pass request objects directly, call .to_dict() on the response
The SDK changed its API surface in v24. Methods like `accounts_balance_get()` now expect
the request object itself — not `.to_dict()`. The response must be converted with `.to_dict()`
on the result, not on the way in.

```python
# CORRECT
request = AccountsBalanceGetRequest(access_token=token)
response = client.accounts_balance_get(request)
data = response.to_dict()

# WRONG — causes 400/422 errors with no obvious message
response = client.accounts_balance_get(request.to_dict())
```
*Source: ISSUE-010 / Session 10 — plaid_client.py fix*

---

### Never use `cursor` as a variable name in pyodbc execute() calls
`cursor` is a reserved name in pyodbc contexts. Using it as a local variable in the same
scope as a DB cursor will cause a silent collision or runtime error.

```python
# WRONG — collides with pyodbc cursor object
cursor = "some_plaid_cursor_value"
db_cursor.execute("INSERT INTO run_log (cursor) VALUES (?)", cursor)

# CORRECT — use a distinct name
sync_cursor = "some_plaid_cursor_value"
db_cursor.execute("INSERT INTO run_log (cursor) VALUES (?)", sync_cursor)
```
*Source: Session 10 — balance_sync.py run_log INSERT fix*

---

### error_log requires a manually generated error_id — no identity column
The `error_log` table uses a `varchar` PK with no `IDENTITY`. Any insert path (happy path
or error path) must generate the PK explicitly.

```python
import uuid
error_id = str(uuid.uuid4())
cursor.execute("INSERT INTO error_log (error_id, ...) VALUES (?, ...)", error_id, ...)
```
Missing this causes a 515 NOT NULL constraint violation that only surfaces on error paths —
easy to miss in testing.
*Source: ISSUE-011 / Session 9*

---

### Plaid sign convention — negate at ingestion, not at query time
Plaid sends debits as positive. The schema convention is positive = credit, negative = debit.
The negation must happen in `plaid_sync.py` at ingest time, not in SQL views or reports.

```python
# In plaid_sync.py — enforced sign rules
if pfc_primary in INFLOW_CATEGORIES:      # INCOME, TRANSFER_IN
    amount = abs(tx["amount"])
elif pfc_primary in OUTFLOW_CATEGORIES:   # LOAN_PAYMENTS, BANK_FEES
    amount = -abs(tx["amount"])
else:                                     # All other debits
    amount = -tx["amount"]
```
If this logic is ever changed, re-audit all historical rows — the 2026-06-02 audit
corrected 235 sign-flipped rows across CHASE, ASSOCIATED_PERSONAL, AMEX.
*Source: Session 7 / Session 6 — sign convention fix*

---

### INFLOW_CATEGORIES and OUTFLOW_CATEGORIES must be explicit sets
Do not rely on Plaid category strings matching expectations at runtime. Define both sets
explicitly in `plaid_sync.py` and test against them with `in` checks. LOAN_PAYMENTS
and BANK_FEES are outflows (negative), not inflows.
*Source: Session 6 — sign convention design*

---

### When a value looks wrong, diff the deployed code against this file's own documented convention before assuming a data problem
plaid_sync.py's INFLOW_CATEGORIES set was found to include "BANK_FEES" and
"LOAN_PAYMENTS" — both already documented in this exact file (see "Plaid
sign convention" and "INFLOW_CATEGORIES and OUTFLOW_CATEGORIES must be
explicit sets," above) as outflows, not inflows. The deployed code had
drifted from its own spec, most likely from a since-unremembered fix for a
rare BANK_FEES refund case that widened the set too broadly and was never
caught by a full sign audit afterward. BestMethods entries describe intent;
they don't guarantee the deployed code still matches that intent. When a
transaction's sign or category looks wrong, check whether the running code
still matches what this file says it should do — don't assume the bug is
new or the data is bad before ruling out code/doc drift.
*Source: ISSUE-023 / Session 14 — found while investigating a positive-signed BANK_FEES row*

---

## Python — Enrichment Pipeline

### enrichment.py retry path must match the happy path parameter count exactly
The retry execute() call must pass the same number of parameters as the happy-path call.
When columns are added or removed from the schema, both code paths must be updated.
A mismatch causes silent failures only on retry — not on the first pass.

```python
# If you add or remove a column, update BOTH:
cursor.execute(UPSERT_SQL, (p1, p2, ..., pN))           # happy path
cursor.execute(UPSERT_SQL, (p1, p2, ..., pN))           # retry path — must match
```
*Source: ISSUE-005 / Session 8 — category_normalized removal left stale retry path*

---

### category_source = 'manual' is the pipeline's only hard stop
The enrichment engine checks `category_source` before overwriting. Any row with
`category_source = 'manual'` is skipped entirely — no pattern match, no historical
carry-forward. This is intentional and must never be removed.

When correcting data manually via SQL, always set `category_source = 'manual'` last.
*Source: Core design — all sessions*

---

### Dual-use merchants must never be mapped to Work-Expense in merchant_patterns
Hotels, restaurants, airlines, and any merchant that serves both personal and business
purposes must NOT be assigned to Work-Expense via `merchant_patterns`. Work travel is
corrected manually after the fact with `category_source = 'manual'`.

Mapping a dual-use merchant to Work-Expense in patterns will misclassify every personal
transaction from that merchant.
*Source: SessionStarter key policies*

---

### Subcategory must never mirror the parent category name
Use `General` instead. Mirrored names (e.g. subcategory `Clothing` under category `Clothing`)
look valid in queries but are a taxonomy violation that causes display and grouping issues
in Power BI. Over 100 rows were corrected across 6 categories in Session 4.
*Source: Session 4 — subcategory violation cleanup*

---

### merchant_patterns priority: more specific patterns must fire first (lower number)
Pattern priority is sorted ASC — lower number = higher priority. A broad pattern like
`%UNITED%` at priority 5 will never match `%UNITED WAY%` if `%UNITED WAY%` is at
priority 10. Specific patterns must always have a lower priority number than broad ones.

The United Way / United Airlines collision is the canonical example — `%UNITED WAY%`
must be at priority 3 or lower, before `%UNITED%` at priority 5.
*Source: Session 4 — United Way pattern fix*

---

### plaid_category_raw must store a single extracted category string, never the raw JSON object
Plaid's personal_finance_category field is a dict
({"primary": ..., "detailed": ..., "confidence_level": ..., "version": ...}).
category_map's plaid_category_raw column is matched with an exact string
comparison — storing the full dict via json.dumps() guarantees this match
can never succeed, since every stored value is also subtly unique
row-to-row (confidence_level varies even for identical categorizations).
Always extract a single field (detailed, falling back to primary) before
writing to any column a downstream exact-match lookup depends on.

```python
# WRONG — category_map can never match this
plaid_category_raw = json.dumps(pfc_dict)

# CORRECT
plaid_category_raw = pfc_dict.get("detailed") or pfc_dict.get("primary") or ""
```
*Source: ISSUE-019 root cause / Session 14 — plaid_sync.py fix*

---

### A non-NULL fallback sentinel silently breaks isna()-gated retry logic
enrich_transactions.py's apply_fallback() sets category='Uncategorized' (a
real string) for unmatched rows — but every enrichment step, including
apply_fallback() itself, decides whether a row still needs processing by
checking category.isna(). Once a row has been through fallback, isna() is
permanently False for it, so --unenriched-only's load filter
(category_source = 'unmatched') pulls the row in but every processing step
then silently skips it as "already matched." The fix: when loading rows
specifically because they were previously unmatched, explicitly reset
their enrichment fields to NULL before running the pipeline, so the
isna()-gated logic genuinely re-evaluates them.

```python
if args.unenriched_only:
    reset_mask = transactions["category_source"] == "unmatched"
    reset_cols = ["category", "subcategory", "in_budget", "type",
                  "category_source", "category_confidence"]
    transactions.loc[reset_mask, reset_cols] = None
```
Confirmed via a real test: re-running --unenriched-only on 64 known
-unmatched rows produced "category_map filled: 0" and "Fallback applied to
0 rows" — a clean log with no errors that nonetheless did nothing at all.
A successful-looking run is not proof of a successful re-enrichment; check
the actual before/after category_source distribution.
*Source: ISSUE-019 follow-on / Session 14 — enrich_transactions.py fix*

---

### pandas: OR-ing a correction term onto `!=` cannot fix a NaN-vs-NaN false positive

`NaN != NaN` evaluates `True` in pandas — so a naive
`(df != original).any(axis=1)` change-detection mask flags every row with a
legitimately-still-NULL column (e.g. subcategory) as "changed" on every
single run, even when nothing actually changed. The instinctive fix —
OR-ing an `isna() != isna()` correction term onto the already-computed
mask — does not work: OR can only ever add `True` values to a mask, never
retract a `True` the `!=` comparison already produced. Compute
equality-or-both-NaN first, then invert, instead.

```python
# WRONG — cannot retract the NaN/NaN false positive != already produced
changed = (df != original).any(axis=1)
changed = changed | (df.isna() != original.isna()).any(axis=1)

# CORRECT — equality-or-both-NaN, computed first, then inverted
both_nan = df.isna() & original.isna()
values_equal = (df == original) | both_nan
changed = ~values_equal.all(axis=1)
```
Caught via a standalone pandas test before running the fix against real
data, not from inspection — the given (wrong) form looked plausible on
read-through.
*Source: ISSUE-035 / Session 16 — enrich_transactions.py write-back fix*

---

### Snapshot timing must respect what a retry path needs to detect

When comparing before/after state to decide what changed, the snapshot's
timing matters as much as its content. enrich_transactions.py's
`--unenriched-only` mode resets a previously-fallback'd row's enrichment
columns to NULL before re-running the pipeline (see the retry-reset lesson
above) — if the pre-enrichment snapshot for change-detection were taken
*before* that reset, a row that resets to NULL and then re-converges to
the exact same fallback values it started with would compare as
"unchanged" and get silently skipped, defeating the entire point of the
retry (refreshing `last_enriched_at` to prove the retry actually ran).
Snapshot after the reset, not before, whenever a reset-then-recompute path
exists upstream of a change-detection comparison.
*Source: ISSUE-035 / Session 16 — enrich_transactions.py write-back fix*

---

## SQL — Azure SQL / mssql Extension

### CREATE VIEW requires GO batch separator between each statement
The VS Code mssql extension requires each `CREATE VIEW` statement to be in its own batch.
Without `GO` between them, only the first view is created and the rest fail silently or
throw a misleading error.

```sql
CREATE VIEW dbo.vw_first AS
SELECT ...
GO

CREATE VIEW dbo.vw_second AS
SELECT ...
GO
```
*Source: Session 10 — 40_phase4_powerbi_views.sql*

---

### Write verification queries as separate standalone SELECTs
Never embed verification queries inside the same script as a CREATE or ALTER. Joining
the new table/view back to itself in the same batch causes ambiguous column name errors.
Run the DDL first, then run the verification SELECT separately.
*Source: Master protocol SQL scripting conventions*

---

### Filter by both `source` and `YEAR(date)` to avoid query timeouts
The `transactions` table has 15,000+ rows spanning 6 years across multiple sources.
Querying without both filters causes full scans that time out in VS Code mssql.

```sql
-- Always scope to source + year when doing data corrections
WHERE source = 'CHASE'
  AND YEAR(date) = 2026
  AND ISNULL(pending, 0) = 0
```
*Source: SessionStarter SQL workflow rules*

---

### Always use ISNULL(pending, 0) = 0 in transaction queries
The `pending` column is NULL for all historical rows, not 0 or 1. A filter of
`WHERE pending = 0` will exclude all historical rows. Always use `ISNULL(pending, 0) = 0`.
*Source: Core schema constraint — all sessions*

---

### BRK/B ticker requires escaped single quote in SQL
The apostrophe in `BRK'B` must be escaped as `BRK''B` in SQL string literals.

```sql
INSERT INTO security_sectors (symbol, ...) VALUES ('BRK''B', ...)
```
*Source: Session 10 — security_sectors seed*

---

### SQL scripts are numbered sequentially from current high watermark
Before creating a new SQL script, check the highest-numbered file in `sql\` and
increment. Current high watermark after Session 10: **42**.
*Source: SessionStarter SQL run order*

---

### Don't assume a documented table's column list is what's actually deployed
DB_Schema.md documented dbo.holdings with 10 columns including cost_basis and
updated_at; the live table only had 8 — both were simply never added when the
table was created (likely Phase 1-2, before updated_at became a project-wide
convention). Query INFORMATION_SCHEMA.COLUMNS directly against the live
database before writing code that assumes a documented schema is accurate,
especially for older tables that predate more recent conventions.
*Source: Session 12 — principal_sync.py / dbo.holdings*

---

## Baird Holdings CSV Pipeline

### Holdings CSV is lot-level — one row per purchase lot, not per position
Baird export format gives one row per purchase lot for the same security. The
`holding_id` must use a lot counter to prevent MERGE collisions.

```python
# holding_id scheme
holding_id = f"{account_name}_{symbol}_{snapshot_date}_lot{N}"
# N is a per-file counter that increments per (account, symbol, date) combination
# N resets to 1 for each new file import
```
Never use just `account_name + symbol + date` as the PK — multiple lots of the same
symbol will overwrite each other.
*Source: Session 10 — import_baird_holdings.py design*

---

### Blank Security ID in Baird CSV = cash row, not a skip
When the `Security ID` column is blank, the row represents a cash sweep position.
Treat the symbol as `CASH` (uppercase to match CSV conventions). Do not skip this row.
*Source: Session 10 — import_baird_holdings.py*

---

### The bottom Total row in Baird CSV must be skipped by description match
The last row of each Baird account section is a `Total` summary row, not a holding.
Skip it by matching on description, not by row count — the position of the Total row
may shift if Baird changes their export format.
*Source: Session 10 — import_baird_holdings.py*

---

### Currency parser must handle $, commas, and parentheses for negatives
Baird CSV values use standard accounting format: `$1,234.56` for positive,
`($1,234.56)` for negative. The parser must strip `$`, `,`, and convert `(X)` to `-X`.

```python
import re

def parse_currency(val):
    if val is None or str(val).strip() in ('', '-'):
        return None
    v = str(val).strip().replace('$', '').replace(',', '')
    if v.startswith('(') and v.endswith(')'):
        v = '-' + v[1:-1]
    return float(v)
```
*Source: Session 10 — import_baird_holdings.py*

---

### Baird account name conventions are canonical — use exact strings
The `Account Name` column added to each CSV export must use the exact canonical names
below. Variations will break the holding_id scheme and reporting.

| Account | Canonical Name |
|---------|---------------|
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

*Source: SessionStarter Baird account name conventions*

---

### asset_classification 'Cash' must be normalized to 'Cash and Cash Equivalents'
Baird CSV exports use `Cash` as the asset_classification for cash sweep rows.
The `baird_holdings` table uses `Cash and Cash Equivalents` for consistency.
Normalize on import.

```python
if row['Asset Classification'] == 'Cash':
    row['Asset Classification'] = 'Cash and Cash Equivalents'
```
*Source: Session 10 — DecisionLog*

---

### symbol column must be varchar(50) to handle options notation
Options tickers in Baird export use long notation: `AAPL CALL 235.00 2025/12/19`.
Both `baird_holdings.symbol` and `security_sectors.symbol` must be `varchar(50)`.
Do not use `varchar(20)` — options will be silently truncated.
*Source: Session 10 — 36_baird_holdings_tables.sql*

---

### When a bug produces the right outcome for the wrong reason, it's still worth fixing
import_baird_holdings.py's Total-row detection checked the wrong CSV column
(Security Name instead of Account Name) for this month's export format. The
Total row still got excluded from the data — it fell through to the
CASH-with-unparseable-date path and was skipped there instead, purely by
coincidence of both cells being blank on that particular row. A future export
where the coincidence doesn't hold (e.g. the Total row has a value in the
Security Name column) would let a fake CASH holding slip through. Verify the
*mechanism* behind a correct-looking result, not just the result.
*Source: Session 12*

---

## Azure SQL Infrastructure

### Azure SQL Free tier Serverless auto-pauses — resume manually on the 1st
FinanceDB is on the Free tier Serverless plan. It auto-pauses after inactivity.
The Power BI scheduled refresh and the Apple Card CSV import will both fail if the
database is paused. Resume manually via Azure Portal before running either.

This is a monthly maintenance step, not an intermittent bug.
*Source: Session 4 — Power BI refresh failure root cause*

---

### Managed Identity is deferred — current auth is SQL username/password via env vars
`db.py` supports both: username/password locally, Managed Identity in Azure.
Do not configure Managed Identity until explicitly planned — it requires coordinated
changes to both the Function App and the SQL firewall rules.
*Source: TechnicalArchitecture security notes*

---

## Azure Functions Deployment

### requirements.txt must include every dependency actually used — local success does not guarantee deployed success
A dependency can be installed in your local Python environment (making local
script runs succeed) without ever being added to requirements.txt. On Azure
Functions, this means the deployed app silently fails to import at all —
the trigger indexer finds 0 functions with no error visible anywhere in the
Portal UI, only in Application Insights traces (ModuleNotFoundError).
This exact failure mode killed all automated syncing for 6+ weeks before
being discovered, because "0 functions" in the Portal Functions blade looks
identical to a display/caching bug, not a crash.

**Check:** after adding any new import to a script that runs in Azure,
immediately verify it's in requirements.txt — don't wait for a deploy to
surface the gap.
*Source: Session 12 — dotenv/requirements.txt outage*

---

### Application Insights traces are the ground truth for deployed function health — the Portal Functions blade can be misleading
On Flex Consumption plans especially, the Functions list in the Azure Portal
can appear empty even when functions are actually registered, and can also
genuinely be empty when a crash is silently killing every cold start. VS
Code's Azure extension queries a different API path and can also show a
stale/cached tree. When function registration is in doubt, query Application
Insights traces directly for "Found the following functions" or "X functions
loaded" log lines — that's the actual host state, not a UI's interpretation
of it.
*Source: Session 12*

---

### Config loading is split between .env and local.settings.json in this project — know which one a new script needs
DB credentials (DB_USERNAME, DB_PASSWORD) load via load_dotenv() reading
.env. Plaid access tokens and other App-Settings-equivalent values load via
each script's own __main__ block manually reading local.settings.json (see
import_baird_holdings.py for the reference pattern). A new script that only
calls load_dotenv() will fail to find any Plaid token even if it's correctly
present in local.settings.json — this happened when principal_sync.py was
first created and had to be fixed.
*Source: Session 12 — principal_sync.py config bug*

---

### Standalone local-only scripts can hide security debt that deployment reviews won't catch

get_plaid_tokens.py hardcoded production PLAID_CLIENT_ID/PLAID_SECRET as
plain-text literals for an unknown period, in direct violation of Master
Protocol section 6 — but because this script never gets deployed to
Azure, none of the deployment-focused review that caught the dotenv
outage would ever have surfaced it. Any script living outside the Function
App's deploy path needs its own periodic check against the project's
security rules; "it's never deployed" is not the same as "it's not a
risk."
*Source: Session 13 — get_plaid_tokens.py hardcoded credentials*

---

## Power BI

### COALESCE merchant_normalized with merchant_name_raw in all views
Several transactions have a NULL `merchant_normalized` (no pattern match). Views must
fall back to `merchant_name_raw`, then `'Unknown'`, to avoid `(Blank)` rows in reports.

```sql
COALESCE(merchant_normalized, merchant_name_raw, 'Unknown') AS merchant_display
```
Apply this to every view that surfaces merchant data.
*Source: Session 5 — vw_top_merchants and vw_transactions_clean COALESCE fix*

---

### Import mode + scheduled refresh at 4:00 AM CT — dataset must be refreshed after schema changes
Power BI uses Import mode, not DirectQuery. After any schema change (new view, column,
or table), the dataset must be manually refreshed in Power BI Service before the new
data appears in reports. Scheduled refresh runs at 4:00 AM CT daily.
*Source: TechnicalArchitecture Power BI settings*

---

### Phase 4 views connect to Calendar via snapshot_date, not date
Phase 4 views (`vw_net_worth`, `vw_holdings_summary`, `vw_asset_allocation`,
`vw_liability_summary`) use `snapshot_date` as the date key — not `date` or
`month_start_date`. Set up the Calendar relationship to `snapshot_date` in the
Power BI data model.
*Source: SessionStarter Power BI data model table*

---

### vw_top_merchants and vw_enrichment_quality have no Calendar relationship
These are standalone views — do not create a Calendar relationship for them in the
data model. They filter independently.
*Source: TechnicalArchitecture data model notes*

---

## Plaid API — Production Environment

### Plaid product authorization must be explicitly requested per product
Not all products are authorized by default in the Plaid Production environment.
`/accounts/balance/get` requires the "Balance" product ($0.10/call), which must be
requested separately via the Plaid Dashboard. Do not assume a product is available
in Production just because it works in Sandbox.
*Source: ISSUE-010 — Balance product not authorized*

---

### Baird (ins_117067) is a known problematic institution
Baird has caused repeated Plaid connection failures across multiple platforms
(Rocket Money also fails). Treat Baird Plaid connection as permanently unreliable
until confirmed stable. The CSV fallback pipeline (`import_baird_holdings.py`) is
the operational solution.
*Source: ISSUE-008 — Sessions 5, 9, 10*

---

### Rocket Mortgage is confirmed unsupported by Plaid
Do not spend time attempting to connect Rocket Mortgage via Plaid. It is not in
Plaid's supported institution list. Update `liability_balances` manually.
*Source: Session 7 — DecisionLog*

---

### ITEM_LOGIN_REQUIRED means the Item's credentials need Plaid Link update mode — not a token replacement
When a Plaid API call fails with error_code ITEM_LOGIN_REQUIRED, the fix is
re-authenticating through Plaid Link in **update mode** (passing the existing
access_token into LinkTokenCreateRequest, omitting products), not creating a
brand-new connection. A brand-new connection creates a second, separate
Plaid Item — since transaction_ids are Item-scoped, this causes the full
historical sync window to re-insert under new IDs alongside existing data,
double-counting months of transactions. get_plaid_tokens.py was extended
2026-08-01 to support this via an optional existing-token input per
institution card.
*Source: Session 12 — Amex, NWM Tom, NWM Amy reauths*

---

### Distinguish transient API errors from stable credential errors before treating either as "the" root cause
Not every failed sync attempt has the same cause even when it recurs. Chase
threw a one-time INTERNAL_SERVER_ERROR that resolved cleanly on retry with no
code change — a transient Plaid-side blip, not worth chasing. Amex's
ITEM_LOGIN_REQUIRED was identical and stable across multiple independent
retries — a real, fixable condition. Treat a single occurrence of an error as
provisional until confirmed by at least one retry; don't assume every error
in a batch shares one root cause.
*Source: Session 12*

---

## Plaid API — Data Quality

### Plaid's transaction object has no `description` field — verify a documented column mapping against the actual API response, not the column's name
plaid_sync.py has been writing tx.get("description", "") into
dbo.transactions.description since the ingestion script was first built.
Plaid's transaction object has no field called description — this line has
always evaluated to the empty-string default, meaning the column has never
once been populated. A column's documented purpose ("raw description text")
is not evidence that the code populating it is correct; check the actual
upstream API response keys directly, especially for any field whose name
is a plausible-but-unverified guess at what an external API might call it.
*Source: ISSUE-024 / Session 14 — found while investigating merchant name truncation*

---

### Plaid's merchant_name resolution for a given vendor can change year over year — a working merchant_pattern can silently stop matching with no code change on your end
A TradingView subscription charge from 2025 had merchant_name_raw
"TradingViewV*Product," matched an existing merchant_patterns rule, and
categorized correctly. The same recurring subscription in 2026 came
through with merchant_name_raw simply "Product" — Plaid's own
entity-resolution service re-resolved the identical merchant to a more
generic name, with no way to detect this from inside the project's own
pipeline. The existing pattern is not wrong and doesn't need to change for
2025 data; it just can't match 2026's differently-resolved name. When a
previously-categorized recurring merchant reappears as Uncategorized, check
whether Plaid's merchant_name actually changed before assuming the pattern
itself is broken or was never built. A raw fallback text field (see
ISSUE-024, description column) would make this kind of drift visible
without relying on manually recognizing a truncated name in a review query.
*Source: ISSUE-024 / Session 14 — TradingView "Product" investigation*

---

### `merchant_name_raw` can silently genericize over time, not just drift to a different specific string

Historical Venmo transactions carried identifying detail in
`merchant_name_raw` (e.g. `DDA PUR VENMO *MAR *2055...`) that let
merchant_patterns distinguish a recurring payment from a generic one;
current Venmo rows arrive as the bare string `"Venmo"` with no
distinguishing text at all. Same failure class as the Apple Card→bare-"Apple"
and TradingView→"Product" cases above (ISSUE-024/ISSUE-027), but total
rather than partial — there's currently no field left to pattern-match
against for these rows once the detail is gone. Worth re-checking once
`description` (ISSUE-024's fix — Plaid's raw, unresolved `name` field,
captured at ingestion regardless of merchant_name quality) has been
populating for a while, in case it retains identifying detail that
`merchant_name` no longer does.
*Source: Session 16 — Venmo/Check review backlog cleanup (script 70)*

---

## CC Prompt Delivery (Claude → Claude Code)

### Never combine multiple full-file rewrites into one CC prompt
Large multi-file prompts risk truncation before they can be copied from chat.
Generate one CC prompt MD file per target file, even for closely related changes.
*Source: MASTER_CLAUDE_PROTOCOL section 4*

---

### Use four backticks when CC prompt content contains triple-backtick code fences
If the CC prompt body includes SQL, Python, or other code blocks wrapped in ```,
the outer CC prompt wrapper must use ```` (4 backticks) to prevent the inner
closing fence from prematurely closing the outer one.
*Source: MASTER_CLAUDE_PROTOCOL section 4*

---

### All CC prompts are delivered as downloadable .md files
Never deliver CC prompts as inline chat text. Use `create_file` to write to
`/mnt/user-data/outputs/CC_[ShortDescription].md`, then `present_files` at the end.
*Source: MASTER_CLAUDE_PROTOCOL section 4*

---

## Data Integrity Rules

### Money that offsets a prior expense nets against that expense's category — it is not separate income
Work reimbursements are recorded as **positive** amounts in the **same
category/subcategory as the original expense** (e.g. positive
Work-Expense/Amy), never as Other Income — so net spend stays accurate.
Session 19 extended the same principle to **tax refunds**: a refund nets
against `Taxes/Federal` (or `Taxes/State`), it does not post as
`Other Income/Tax Refund`. The rule generalizes: if an inflow exists only
because of a specific earlier outflow, book it against that outflow's
category as a positive, not as income.
*Source: SessionStarter key policies; extended to tax refunds Session 19 (script 89)*

---

### Household has no gas-purchasing vehicles (all electric) — a gas-station-branded charge is a convenience-store purchase, never gasoline
Marathon, Raceway, ExxonMobil — and any other gas-station brand not yet
seen — all resolve to convenience-store spending (`Groceries/General`) in
this household's real data, not `Car/Gas`. This was rediscovered as an
independent finding in Session 19 (Marathon/Raceway) and again in the
Session 19 continuation (ExxonMobil), so it's pinned here: when a
gas-station-branded `merchant_patterns` row points at `Car/Gas` or
`Car/Car Wash`, it is almost certainly wrong — repoint the generic to
`Groceries/General` and let it win (lower priority number) over any
store-number-specific Car/* variants.
*Source: Session 19 — Marathon/Raceway (script 81), ExxonMobil (script 90)*

---

### in_budget rules by category type
| in_budget | Categories |
|-----------|-----------|
| 0 (excluded) | Transfer, Payment, Taxes, HSA Deposit |
| 1 (included) | All other categories, including Large Purchases |

Taxes are excluded because they are Baird dividend-driven and unpredictable.
Payments are excluded because they double-count underlying spending.
*Source: Category taxonomy type assignments*

---

### True duplicate transactions: delete the pending version, keep the settled version
When a transaction appears twice (pending + settled), delete the pending row.
Log every deletion in the DecisionLog with date, merchant, amount, and reason.
Do not use `UPDATE`to merge — delete is cleaner and auditable.
*Source: Session 4 — Mariner North Resort + Phish Tickets duplicates*

---

### Positive Expense rows are a recurring data quality risk
Transactions that should be negative (debit/expense) sometimes come through as
positive due to Plaid sign convention mishandling. Run periodic sign audits filtered
by `amount > 0 AND type = 'Expense'` across all sources. The 2026-06-01 audit
caught and corrected 235 rows.
*Source: ISSUE-007 / Sessions 3, 4*

---

### KBB/Edmunds "typical mileage" valuations can be significantly wrong for high-mileage vehicles
Web-searched KBB/Edmunds figures are usually based on average mileage for a
vehicle's age, not the actual vehicle's mileage. For a 9-year-old car with
126,000+ miles (well above typical), these figures run meaningfully high.
Get actual VIN/mileage-specific values from kbb.com directly for any vehicle
whose mileage is notably above or below what's typical for its age, rather
than treating a general web search result as authoritative.
*Source: Session 12 — Tesla Trinity valuation*

---

### When building an import pipeline for an account with unknown history, default unrecognized activity to "real," not "needs review"

import_hsa_transactions.py's first version whitelisted a small set of
known transaction types and defaulted everything else to type=NULL,
"flag for manual review" — reasonable for an account believed to be
investment-only going forward. But the actual CSV export covered the
account's full history back to 2014, including years of genuine
medical-spending activity the whitelist had never seen. 194 of 456 rows
came back type=NULL on the first run. The fix: whitelist only the small,
truly fixed set of *non-spending* description types (contributions,
transfers, interest), and treat literally everything else as real
spending/income, typed by amount sign. When an account has any real
transactional history — not just a handful of recurring internal
transfers — assume unknown activity is real until proven otherwise, not
the reverse.
*Source: Session 13 — import_hsa_transactions.py _classify() redesign*

---

### A merchant_patterns insert that "conflicts" on a case-insensitive duplicate key is a no-op, not a failure — check what it collided with before assuming a gap exists

Two separate attempts to seed new HSA-related merchant_patterns rows this
session (Payroll Deduction, Employer Contribution) both hit the table's
case-insensitive PK collision with patterns already present from
2026-03-05/07 — from before this session's HSA reconnection work even
began. The inserts correctly did nothing (per the project's own
stop-and-flag rule), and the *pre-existing* rows kept doing their job.
Before writing a new merchant_patterns entry, it's worth checking whether
one already exists under a different case rather than assuming a gap —
this project's collation is case-insensitive on that table's PK, so
'%Foo%' and '%FOO%' are the same row.
*Source: Session 13 — scripts 56/57 collisions*

---

### "RESOLVED" means a fix was drafted and committed — not necessarily deployed or executed

Twice in one session (ISSUE-019's plaid_category_raw JSON bug, ISSUE-023's
BANK_FEES/LOAN_PAYMENTS sign regression), a fix that was correctly written and
committed to the local repo in a *past* session had never actually been
deployed to Finance-ingest-Tom-v6. Both bugs kept running silently for a full
month after being marked "Resolved" in IssuesTracker.md — the historical
backfill fixed existing bad rows at the time, but the ingestion-side bug that
kept creating new ones was never actually stopped. Any CC prompt that changes
a Function-App-deployed file (plaid_sync.py, timer_sync.py, monthly_sync.py,
http_ingest.py, balance_sync.py, nwm_sync.py, db.py, requirements.txt) must
include an explicit, mandatory deploy step AND a verification step that
checks the *deployed* file's actual content — e.g. via VS Code's Azure
Functions extension "Files (Read-only)" remote view, since Kudu/Advanced
Tools is not available on this project's Flex Consumption plan (see below).
Local correctness is not deployment.

**Extends to SQL scripts and local-only scripts too, not just Function-App
deploys** (Session 16): ISSUE-018 (Baird `MAIN - Brokerage` vs `MAIN - BKG`
naming drift) recurred a second time despite a "Resolved" log entry from
Session 12 — the SQL fix (script 50) that was logged as having run against
the affected data apparently never actually executed. Same failure shape as
the deploy gap above, one level down the stack: "committed" ≠ "deployed" for
Function-App code, and "drafted" ≠ "executed" for SQL scripts / local
scripts. Don't mark an issue Resolved in IssuesTracker until deployment (for
Function-App-deployed files) or execution (for SQL scripts / local scripts)
is independently verified against live data or live code — a drafted fix
and a confirmed-run fix are different states and must be logged differently.
*Source: Session 15 — ISSUE-019 and ISSUE-023 recurrence; Session 16 — ISSUE-018 recurrence*

---

### SQL Server MERGE permits only one WHEN MATCHED...UPDATE clause per statement

Conditional field-level protection (e.g. "only overwrite column X if the
existing row hasn't been enriched yet") needs a CASE expression inside a
single WHEN MATCHED clause, not two separate WHEN MATCHED clauses — SQL
Server rejects a second WHEN MATCHED...UPDATE outright ("An action of type
'WHEN MATCHED' cannot appear more than once in a 'UPDATE' clause"). Multiple
WHEN MATCHED clauses are only valid when one of them is a DELETE.

```sql
-- WRONG — SQL Server error 10714, every matched-row update silently fails
WHEN MATCHED AND t.category_source IS NULL THEN UPDATE SET ...
WHEN MATCHED THEN UPDATE SET ...

-- CORRECT — one WHEN MATCHED, CASE per protected column
WHEN MATCHED THEN
    UPDATE SET
        category_source = CASE WHEN t.category_source IS NULL
                                     OR t.category_source IN ('historical','unmatched')
                                THEN s.category_source ELSE t.category_source END,
        ...
```

A first draft using two WHEN MATCHED...UPDATE clauses was caught by a real
test run against SQL Server before being deployed — as drafted, it would
have made every future matched-row update silently no-op, a worse regression
than the bug it was meant to fix.
*Source: Session 15 — import_apple_csv.py MERGE enrichment-protection fix*

---

### transaction_id generation must hash normalized (parsed) values, never raw upstream text

import_hsa_transactions.py's `_make_transaction_id()` hashed the raw,
unparsed CSV date and amount strings. When Bank of America changed its
export formatting between two monthly exports (leading zeros added to
dates, amount field quoting/whitespace changed), the same real transaction
hashed to a different ID each time, silently defeating the MERGE's
idempotency and re-inserting the account's entire multi-year history on
every affected reimport (~400+ duplicate groups, 835 rows to clean up).
Any transaction_id scheme should hash values only after they've been parsed
into a stable, normalized form (`YYYY-MM-DD`, a fixed-precision decimal) —
never the raw text a CSV export happens to contain that day.
*Source: Session 15 — import_hsa_transactions.py transaction_id fix*

---

### An import script's MERGE that unconditionally overwrites enrichment metadata is not safe to re-run against already-processed data — even as a test

Only enrich_*.py scripts (gated on `category IS NULL`) are safe to re-run
idempotently. import_*.py scripts, before this session's fix, could
silently clobber real category_source/category_confidence/in_budget
provenance on any re-run against previously-imported data — confirmed as a
real production risk, not just a testing artifact, since the same MERGE
would fire from an innocuous overlapping CSV export (a month whose export
re-includes the tail end of the prior month), not only from deliberate
re-testing. Before re-running any import_*.py script against data it may
have already processed, check whether its MERGE protects existing
enrichment metadata — do not assume idempotency just because the script
uses MERGE.
*Source: Session 15 — import_apple_csv.py re-run corrupted 131 rows' category_source, flipped in_budget on 10*

---

### Manual category corrections must set type and in_budget together with category/subcategory — never leave them as inherited stale values

A manual correction that only updates category/subcategory can leave a row
in the wrong Power BI Expense/Income/Other bucket even though the category
label itself displays correctly — Power BI's grouping is driven by `type`,
not `category`. Pull the correct type/in_budget from category_map's own row
for that category whenever hand-correcting a transaction, not just the
category/subcategory pair.
*Source: Session 15 — 3 Apple/ASSOCIATED_PERSONAL rows + 1 legacy Session 13 HSA row found with stale type/in_budget after correction*

---

### Manual-correction scripts must also set `category_reviewed = 1`, not just `category_source = 'manual'`

`category_reviewed` is meant to mean "a human looked at this and confirmed
it" — but the column had been silently ignored by every correction script
to date, including several written earlier in this very session, which is
exactly how it ended up holding effectively-meaningless inherited values
from a past bulk import instead of a real reviewed/unreviewed signal. A
one-time reset (script 71, Session 16) reclassified the column honestly:
421 rows now genuinely need review, 16,286 are genuinely reviewed. Every
manual-correction script going forward must set `category_reviewed = 1`
alongside `category_source = 'manual'` — the same discipline the earlier
type/in_budget lesson above already established for that pair.
*Source: Session 16 — category_reviewed reset (script 71)*

---

### Kudu/Advanced Tools is not available in the Azure Portal nav for Flex Consumption plans

Confirmed this session: Finance-ingest-Tom-v6's "Development Tools" blade
only shows "Recommended services" — no Advanced Tools/Kudu link exists for
this plan type. VS Code's Azure Functions extension "Files (Read-only)"
remote view is the working alternative for inspecting actual deployed file
contents when verifying a deploy landed.
*Source: Session 15 — verifying the ISSUE-019/ISSUE-023 redeploys*

---

## Taxonomy & Enrichment

### Category_Taxonomy.md's Full Taxonomy block is a closed list — never write a value that isn't on it

No SQL script, enrichment step, `merchant_patterns` row, `category_map`
row, or manual correction may write a `category`/`subcategory` value that
isn't already in `Category_Taxonomy.md`'s **Full Taxonomy** block. A new
subcategory is added in one order only: Tom approves it → it's committed
to `Category_Taxonomy.md` (version history + Full Taxonomy block) → *then*
a script may use it. Never introduce a new value in the same script that
uses it. When a transaction's correct category is unclear, route it to
`Uncategorized / General` (`category_reviewed = 0`) for later review —
don't invent a subcategory to fit it. This session alone, CC prompts
proposed `Check - Review`, and earlier `Gifts` / `Dry Cleaning` slipped
in ahead of the doc; `taxonomy_audit.py` Checks 1–3 exist to catch
exactly this, and a clean audit is the compliance bar.
*Source: ISSUE-012 — recurring "new value invented mid-fix" pattern (Session 19)*

### A no-wildcard merchant_pattern is an exact-string rule, not a substring one — and taxonomy_audit.py Check 4 must respect that

`enrich_transactions.py`'s matcher (~L302–312) treats a pattern with no
leading `%` **and** no trailing `%` (`ACT`, `IRS`, `Apple`) as
`raw_upper == pattern_clean` — an exact string match. It cannot collide
with `%ACTIVATE%` even though `"ACT"` is a substring of `"ACTIVATE"`,
because neither rule matches a string the other does. `taxonomy_audit.py`
Check 4 originally stripped `%` from both patterns and did a plain
substring test, which false-flagged every no-wildcard pattern; fixed
Session 19 (AFAS 1db78e4) with a `_would_match()` helper that mirrors the
real matcher. If a future edit "simplifies" Check 4 back to a bare
substring test, this class of false positive returns — keep the
exact-match branch.
*Source: Session 19 — taxonomy_audit.py Check 4 fix (post-ISSUE-039 `ACT` pattern)*

### A documented pipeline step in TechnicalArchitecture.md is not evidence it was ever built

The "Enrichment Pipeline — Logic Order" table had listed a "Historical —
carry forward category from prior enriched record for same merchant" step
for 6+ months. It never existed in `enrich_transactions.py` — the module
docstring documented 4 steps, `main()` called 4 steps. It was a
documented-but-never-implemented feature, mistaken as working behavior.
Verify old "should already work" behavior against the actual code path
(the function bodies and the `main()` call sequence), not against the
architecture doc or a docstring. Docstrings drift too — the same file's
docstring also had `category_map` and `merchant_patterns` in the wrong
order relative to execution.
*Source: Session 17 — enrich_from_history() build (AFAS 557cdd1)*

### A documented taxonomy rename must also update merchant_patterns AND category_map — not just existing transaction rows

Every taxonomy rename in Category_Taxonomy.md's version history (U-club →
Club Dues, Airlines → Flights, Hotels → Lodging, Fitness → Health &
Wellness, ...) updated the transaction rows that existed at the time but
left the `merchant_patterns` / `category_map` entries pointing at the
retired name. Because enrichment writes the pattern's stored
category/subcategory onto every future match, the pattern table silently
re-creates the retired value indefinitely — one rename produced a slow
drip of "new" bad rows for months. When renaming a subcategory: update
`merchant_patterns`, `category_map`, AND transactions in the same change,
and re-run `taxonomy_audit.py` afterward to confirm nothing still writes
the old name.
*Source: Session 17 — U-club / Airlines / Hotels / Fitness drift (ISSUE-012)*

### Same-priority merchant_patterns: a broad pattern that is a substring of a specific one permanently shadows it

`enrich_transactions.py` tries patterns in `ORDER BY priority ASC, pattern
ASC` and takes the first match. Within one priority number, the
alphabetically-earlier pattern wins — so `%MARQUETTE UN%` (priority 10)
shadowed `%MARQUETTE UNIV H%` (priority 10) on every possible match since
the day it was created, misrouting a $17,027 tuition payment. A
more-specific pattern must sit at a strictly *lower* priority number than
any broader pattern whose text it contains — never the same number.
`taxonomy_audit.py` Check 4 surfaces these pairs (`[DIFFERENT DEST]` ones
are the real bugs).
*Source: Session 17 — Marquette shadowing (ISSUE-012, ISSUE-036 follow-on)*

### Duplicate detection on date + amount alone produces heavy false positives — require a second specific signal

The first draft of `vw_potential_duplicates`'s legacy-vs-Plaid join
matched any two rows sharing date + amount where one wasn't
`category_source = 'historical'` → 232 hits, mostly false (sequential
flight/hotel booking reference numbers, repeat same-day purchases at the
same merchant). Tightening it to require the *counterpart* row to carry a
genuine non-null `pending` value (i.e. it demonstrably came through the
real Plaid sync path) dropped it to 0 false positives while still
catching all 5 real instances. Any date+amount duplicate check needs an
extra structural signal before its output is safe to delete from.
*Source: Session 17 — vw_potential_duplicates build*

### A merchant_pattern that has never matched a real transaction is a landmine, not a no-op

A reviewing session found 36 of 64 active Work-Expense merchant_patterns
rows had NEVER matched a single transaction in the database's history —
not "haven't matched recently," literally zero matches, ever. These sit
silently until a real transaction happens to hit the exact string, at
which point they fire with zero review (category_source = 'merchant_
pattern', not 'manual'). A pattern with zero historical matches is not
evidence it's harmless — it's evidence nobody has been burned by it yet.
Worth a periodic sweep: `SELECT pattern, COUNT(t.transaction_id) AS
times_matched FROM merchant_patterns mp LEFT JOIN transactions t ON
t.merchant_name_raw LIKE mp.pattern GROUP BY pattern HAVING
COUNT(t.transaction_id) = 0` against any category prone to accumulating
speculative one-off patterns (Work-Expense, Large Purchases, anything
tied to travel).
*Source: Session 18 — Work-Expense pattern audit, script 72*

### A merchant_pattern that fired exactly once is not automatically a safe durable rule

Beyond the 36 zero-match patterns above, 9 more Work-Expense patterns had
matched exactly once, ever. Reviewing each individually (rather than
assuming "it matched, so it must be a real recurring vendor") found 3 of
the 9 were one-off transactions that should have stayed manual
corrections instead of being promoted to permanent categorization rules —
including one, Wynn Las Vegas, that was compounding with an invalid
subcategory ('Lodging' under Work - Expense, which isn't a valid
combination at all). A single real match is not proof of recurrence;
check whether the underlying transaction actually looks like it'll
happen again before trusting a pattern built from it.
*Source: Session 18 — Work-Expense pattern audit, script 72*

### Azure SQL's default collation is case-insensitive — don't assume a Python-side string comparison agrees with the database

taxonomy_audit.py flagged `Payment/AMEX` as an undocumented combo distinct
from the canonical `Payment/Amex`. It isn't, functionally: Azure SQL's
default collation (SQL_Latin1_General_CP1_CI_AS) treats 'AMEX' and 'Amex'
as identical, so every WHERE/JOIN/GROUP BY in the actual pipeline already
merges them correctly. The audit script's Python-side comparison against
its canonical dict is case-sensitive, so it flagged a distinction that
doesn't exist in the data. Before treating any taxonomy_audit.py "cosmetic
casing" flag as a real bug, check whether it's actually a database-level
mismatch or just a Python-string-equality artifact — use `WHERE
subcategory = 'X' COLLATE Latin1_General_CS_AS` to force a case-sensitive
check if you need to confirm which case a stored value is.
*Source: Session 18 — Bucket 1 taxonomy renames, script 77*

### taxonomy_audit.py's own canonical dict can go stale — treat its "undocumented" flags with the same skepticism as any other doc-vs-reality mismatch

The script's own code comment states its CANONICAL_TAXONOMY dict was
transcribed from a 2026-07-01 snapshot of Category_Taxonomy.md. It has
not been refreshed since. Confirmed in Session 18: ATM/Cash Spending/ATM,
Dining Out/Fast Food, and Pay/Whit are all formally documented in
Category_Taxonomy.md's version history (added Session 17, 2026-09-03) but
still flag as "undocumented" in every Check 1/2/3 run, because the
script's own reference copy predates them. This is the same "doc says one
thing, code/data says another" failure class this project has hit
repeatedly elsewhere (enrichment pipeline docstrings, TechnicalArchitecture
vs. actual deployed code) — the audit tool meant to catch drift is itself
subject to drift. Cross-check any audit flag against the live
Category_Taxonomy.md doc directly before assuming it's a real gap. See
ISSUE-041.
*Source: Session 18 — Check 1/2/3 review*

### A single merchant_patterns row cannot express sign-dependent categorization

`WEB FR DDA TO DDA CONFIRMATION` produces both inbound (+$100) and
outbound (-$500, -$120, etc.) real transactions under byte-identical
merchant text — the only signal distinguishing direction is the amount's
sign. The current architecture (one static category/subcategory per
pattern) has no way to express "route this differently depending on
sign." Historical rows were corrected with a one-time sign-based CASE
UPDATE; the pattern itself was set to the majority real-world direction
(Transfer Out, 6 of 7 real cases) as a default, accepting that the
minority direction will need manual correction going forward — the same
default-common-case/manual-exception convention already established for
dual-use merchants (hotels, Ascension). If this pattern (or a similar one)
starts generating enough manual corrections to be worth automating
properly, it would need a code-level change to enrichment logic (a sign
check alongside the LIKE match), not just a SQL data fix.
*Source: Session 18 — Bucket 1 taxonomy renames, script 77*

### A documented rename in Category_Taxonomy.md's Notable Decisions table is not evidence it fully propagated

Car/Wash → Car Wash and Car/Supercharger → Charging were both already
listed in Category_Taxonomy.md's Notable Decisions table as completed
renames — but live merchant_patterns rows were still using the retired
values ('Wash', 'Supercharger') as of Session 18. Same failure class as
the Session 17 Airlines/Hotels/Fitness drift (ISSUE-012), now confirmed
recurring in a different category. A rename documented in the doc's
decision history describes intent, not verified current state — the only
way to know a rename is actually complete is to query live
merchant_patterns/category_map/transactions directly, the same lesson
already learned the hard way for "Resolved" issue statuses and deployed
Function App code.
*Source: Session 18 — Bucket 1 taxonomy renames, script 77*

### A taxonomy rename can land in merchant_patterns but still miss category_map

Travel/Hotels → Lodging was fixed in merchant_patterns during Session 17
(29 patterns renamed) but category_map still carried the retired 'Hotels'
value on 2 rows, undetected until Session 18's Check 2 review. Renames
touch up to three separate tables (merchant_patterns, category_map,
transactions) and fixing one doesn't imply the others got the same
treatment — always check all three before considering a taxonomy rename
complete.
*Source: Session 18 — Bucket 1 taxonomy renames, script 77*

### A merchant_pattern with a `%` in the MIDDLE of its text is silently dead under the Python matcher

`enrich_transactions.py` translates a stored LIKE pattern to a Python
check by doing `pattern.replace("%", "")` and then treating only a
leading/trailing `%` as a wildcard (exact / startswith / endswith /
contains). An internal `%` is simply deleted — `%CASK%ALE%` becomes the
literal substring `"CASKALE"`, which can never match "CASK & ALE". These
patterns only ever worked via the old SQL-side `LIKE` matcher in
`enrich_apple_csv.py` / `enrich_hsa_csv.py`; once those were retired
(Session 19) and every source went through the Python matcher, all 31
such patterns were confirmed dead. 26 turned out to already have a
working same-destination replacement someone had added later without
removing the broken original — so the fix was mostly deactivation, not
rewriting. Find them with
`WHERE SUBSTRING(pattern, 2, LEN(pattern)-2) LIKE '%[%]%'`.
*Source: Session 19 — script 86, ISSUE-012*

### Confirm a merchant_pattern's exact live text before writing a fix that assumes it

ISSUE-039's IssuesTracker entry described the pattern as `%ACT%`. The
live `merchant_patterns` row was actually `% ACT %` (space-bounded) — an
earlier, undocumented partial narrowing that never made it into the
issue notes. A fix written against the assumed `%ACT%` text would have
updated zero rows. Caught by querying the live table for the pattern
text before running the `UPDATE`, not by trusting the description. Same
"doc/notes describe intent, not current state" lesson as "Resolved"
statuses, deployed Function App code, and documented renames — applies
to individual pattern rows too.
*Source: Session 19 — script 83, ISSUE-039*

### Duplicate per-source enrichment scripts accumulate their own independent bugs

`enrich_apple_csv.py` and `enrich_hsa_csv.py` were written as scoped
copies of `enrich_transactions.py`'s logic. Over time each drifted and
picked up bugs that never existed in the canonical implementation:
category_map matches mislabeled `category_source = 'plaid'`; no
`in_budget`/`type` derivation (Payment rows kept the import-time
`in_budget = 1`); historical carry-forward still writing the pre-rename
`'historical'` label and willing to carry forward an Uncategorized row;
a non-deterministic `ORDER BY priority` with no `pattern ASC` tiebreak.
`enrich_transactions.py` already had no source filter and was the more
complete implementation the whole time. Session 19 retired both copies —
one enricher for every source. Same drift-prevention lesson already
learned for `taxonomy_audit.py`'s hardcoded dict and for
TechnicalArchitecture.md vs. deployed code: a second copy of shared
logic is a liability, not a safety measure.
*Source: Session 19 — enrichment consolidation, ISSUE-038 (AFAS b1a3ef3)*

### "Compiles clean, 0 rows need backfill" is inference that a change is safe, not a live-run confirmation

The Session 19 enrichment consolidation (dropping `enrich_transactions.py`'s
date cutoff, changing `apply_fallback()`) could not be run end-to-end —
the execution harness in use blocks prod-writing script runs. Safety was
argued from inspection (0 `category IS NULL` rows, 0 rows needing
`in_budget`/`type` backfill → a run would be a no-op) plus unit tests of
the two logic changes. That's a real gap this project has been burned by
before (ISSUE-019, ISSUE-023, ISSUE-018 all "verified by inspection,
still had a hole"). When a change can't get a live run, say so
explicitly and flag it for observation on the next real trigger — don't
let inspection-only verification quietly become "done."
*Source: Session 19 — enrichment consolidation*

---

## Power BI (continued)

### A table visual with no unique key in the field well silently groups and SUMs numeric columns

The Needs Review page showed `review_priority` values of 6, 8, 12, 16, 28
instead of the real 1–4 scale: a table visual without `transaction_id` in
the field well implicitly grouped rows sharing date/merchant/category and
summed `review_priority` and `amount` across the group. Fix: add the
unique key (`transaction_id`) to the visual — hide it via Format >
Columns > Show toggle — and set every numeric column that should display
a raw value (not an aggregate) to "Don't summarize". This is a durable
trap, not a one-time glitch — check it on any table visual showing
per-row scores or flags.
*Source: Session 17 — Needs Review page build*

### A "latest snapshot" DAX filter must group by the entity that updates atomically, not by a row-level key that embeds the date

While building the Holdings page on `vw_holdings_all` (a full-history view
— every snapshot date, not just the latest, by design), a "latest
snapshot only" measure grouped by `holding_id` instead of `account_name`
(or `source` + `account_name`). Baird's `holding_id` is
`account+symbol+date+lotN` — the date is baked into the key — so "max
snapshot_date per holding_id" is trivially always true for every row
(each `holding_id` only ever has one date), and the filter became a
silent no-op: every historical snapshot summed together instead of just
the latest one. Total came out ~13x too high ($129M vs. an expected
~$9.6–10.8M) before the mismatch was caught. When filtering any table to
"latest per X," X must be the thing that actually gets a new row on each
update cycle (here: source + account) — never a key that already has the
date folded into it, or the filter can't distinguish "this row's date"
from "the max date for this row's group" because they're always the same
thing.
*Source: Session 20 — Power BI Holdings page, vw_holdings_all*

### Don't average a stored percentage column across a Power BI rollup — recompute it from the underlying sum ratio instead

`vw_holdings_all.unrealized_gl_pct` is a per-row percentage
(`unrealized_gl / cost_basis`). A DAX measure that does `AVERAGE()` (or an
implicit average aggregation) across that column at an account or
portfolio level treats every position as equally weighted regardless of
size — a $500 position at +40% and a $500,000 position at +2% would
average to +21%, nowhere near the actual blended return. The fix is
`DIVIDE(SUM(unrealized_gl), SUM(cost_basis))` computed fresh at whatever
level the visual is grouped to, not an aggregation of the pre-computed
per-row percentage — this self-corrects at every level of a matrix or
table (position, account, or total) instead of needing a different
formula per grain.
*Source: Session 20 — Power BI Holdings page, vw_holdings_all*

### A category assigned at the account level hides real cash sitting inside investment accounts — classify at the holding level instead

`vw_net_worth` categorized every row of a Baird/Plaid account as
`'Investment'` regardless of what it actually held. Confirmed live:
`MAIN - BKG` carried $294,141.73 in literal cash and a Vanguard Treasury
Money Market fund, and `IRA - TOM` carried $5,253.24 in the same fund —
both already tagged `asset_classification = 'Cash and Cash Equivalents'`
at the holding level, but reported as Investment because the whole
account was. The fix (script 99) computes `net_worth_category` **per
row**, not per account, so an account with both cash and real investments
correctly splits across two categories instead of one hiding the other.
Any "what type of account is this" categorization scheme should default
to holding-level classification when the data supports it — account-level
labels are a convenience that silently goes wrong the moment an account
holds a mix.
*Source: Session 21 — vw_net_worth/vw_holdings_all Cash reclassification*

### Grouping "latest snapshot per account" by display name alone breaks the moment two accounts share a name

`dbo.accounts` had 3 rows literally named "Kids Savings Account" with 3
different `account_id`s (presumably one per kid, distinguished only by
`account_id`, not `display_name`). A "latest snapshot" query grouped by
`account_name` would compute one `MAX(snapshot_date)` across all three
and silently drop the other two if their sync dates ever diverged. Fixed
by introducing a real `account_key` (the underlying PK — Plaid
`account_id`, or the source table's own PK for insurance/physical/
liability) for any grouping/latest-snapshot logic, and using
`account_name` only for display. Never assume a display name is unique
just because it looks like an identifier.
*Source: Session 21 — vw_holdings_all account_key introduction*

### When merging a new source into an existing table's history, dedupe-to-latest-in-period isn't enough if the new source updates less often than the period — forward-fill instead

Building `vw_net_worth_all_time` by keeping only the latest snapshot
actually dated within each calendar month (a literal read of "view by
month") broke badly for 2011-2019: most accounts in the newly-backfilled
CSV were only recorded once a year (January), so April-December of those
years collapsed to whatever one or two accounts happened to have an
off-cycle update (e.g. a car, updated monthly), making the "monthly
total" swing from ~$3.6M in January to ~$110K in April and back. The fix
was a full month-spine per account with `CROSS APPLY ... TOP 1 ...
ORDER BY snapshot_date DESC` — each account's last known value carried
forward into every silent month, which is what a coherent trend actually
needs. Whenever a new data source has a coarser update cadence than the
output grain, forward-fill, don't just dedupe.
*Source: Session 21 — vw_net_worth_all_time, 2011-2026 CSV backfill*

### A recursive CTE inside a view can't raise MAXRECURSION — use a non-recursive row generator for a long spine instead

SQL Server views can't carry query hints (`OPTION (MAXRECURSION n)`) —
they're rejected at `CREATE VIEW` time. A recursive CTE generating a
~190-month spine (2011 through today) would hit the default 100-level
cap the moment it's queried through a view. Used the standard
`CROSS JOIN sys.all_objects` row-number trick to generate the month
sequence non-recursively instead — works inside a view, no recursion
limit to hit.
*Source: Session 21 — vw_net_worth_all_time month spine*

### A historical source that simply stops mentioning a closed account (instead of recording a final $0) will make forward-fill carry a stale balance forever

Two accounts in the CSV backfill (an old 401k that rolled over, an old
HSA custodian) just stop appearing in the source data once closed,
rather than showing one last $0 row. Combined with the forward-fill
lesson above, this meant their last real balance ($13,850 and $345)
carried forward into every subsequent month indefinitely — a real
$14,195 discrepancy against the live net worth total, only caught by
reconciling the combined view's total against the live source of truth
rather than assuming the merge was correct. Any account known to be
closed needs an explicit $0 (or otherwise terminal) row before
forward-fill logic runs, or it silently keeps contributing forever.
*Source: Session 21 — vw_net_worth_all_time, Tom 401K / HSA-GW closure*

### Before seeding a table that feeds an existing view, read the view's actual join and filter logic — don't infer the table's expected shape from the column names alone

Seeded `dbo.budget_targets` with `is_default = 1` and a specific `month`
on every row, reasoning from the column name alone ("this is the default
target"). The pre-existing `vw_budget_vs_actual` view actually treats
`is_default = 1` as a flat *annual* figure (`month IS NULL`, divided by
12) reserved for categories with no month-specific data, and only
matches month-specific rows via a separate join branch requiring
`is_default = 0`. The seed matched none of the view's three join
branches — `budget_amount` would have shown NULL for every category
despite the table having 240 rows. Separately, the view computes
`actual_amount` as always-positive `SUM(ABS(amount))`, but the seeded
`target_amount` was signed negative (matching the source spreadsheet's
own presentation), making `variance`/`pct_of_budget` backwards. Both
were only caught by querying the view's `OBJECT_DEFINITION()` directly
and checking real output, not by assuming the table's schema was
self-explanatory.
*Source: Session 21 — budget_targets seed vs. vw_budget_vs_actual*

### A Power BI Calendar table's date range must cover the full span of every fact table it relates to — not just the narrowest one

A January-only stacked bar chart only showed data back to 2020 even
though the underlying table (`vw_net_worth_all_time`) had rows back to
2011. The Calendar table's date range had been sized to match
transaction data (which starts ~2020), so it simply had no rows for
2011-2019 for the relationship to match against — those years were
silently dropped from anything sliced by Calendar, with no error. Fix:
size the Calendar table's start date to the earliest date across *every*
fact table it relates to, not whichever table it was originally built
for. Also worth remembering: don't relate two fact tables (e.g.
`vw_net_worth_all_time` and `vw_holdings_all`) directly to each other —
they're different grains of the same data, and relating them causes
fan-out. Relate each independently to Calendar instead.
*Source: Session 21 — Power BI Calendar table date range gap*

## Python — pandas (continued)

### `pd.concat(..., ignore_index=True)` silently breaks a positional before/after comparison

`enrich_transactions.py`'s `main()` captured a snapshot of every row's
enrichment columns before running the pipeline, then compared it against
the same columns after, to decide which rows actually changed and needed
a DB write-back. Two of the pipeline's own functions
(`enrich_from_merchant_patterns`, `enrich_from_history`) internally
split the DataFrame into matched/unmatched, then reassembled it with
`pd.concat([already_matched, unmatched], ignore_index=True)` — which
resets the index and reorders rows. The before/after comparison, keyed
purely on row position, was then comparing row 500's "before" against a
completely different transaction's "after." This misreported up to
9,604 of 16,754 untouched rows as changed on a single run, triggering
large wasted write-backs (not data corruption — every affected row's
actual enrichment values were unchanged, confirmed live via spot-checks
of `category_reviewed`-flagged rows before any fix was applied). Fix:
key the comparison on `transaction_id`, never on row position, whenever
a DataFrame has passed through any `concat`/`sort`/`merge` step that
doesn't guarantee order preservation.
*Source: Session 22 — enrich_transactions.py change-detection bug, AFAS 3b6989d*

## Power BI (continued)

### A relationship auto-created on a coincidentally same-named column is a landmine, not a convenience

Power BI's relationship autodetect created two Many-to-One relationships
joining `vw_monthly_spend` and `vw_category_yoy` to `vw_enrichment_quality`
on `txn_count` — a column all three views happen to name the same thing,
but which is a `COUNT(*)` in each, never a unique key. This produced
"duplicate value" refresh errors the moment the underlying counts
changed. Diagnosed from the refresh error text alone before confirming
via the relationship editor which exact relationships were at fault.
Fix is Power BI-side only (delete the relationship) — there is nothing
wrong on the SQL side to chase. General rule: don't trust autodetect on
any column whose name describes a generic aggregate (`count`, `total`,
`amount`) rather than an actual identifier.
*Source: Session 22 — Monthly Procedure live test, Power BI refresh error*

### A CSV re-exported with full history can carry a field the importer never looked at — check every column before declaring data unavailable

The Bank of America HSA transactions CSV (re-exported fresh each time,
full history) was long assumed to have no merchant detail for older
"Normal Distribution" rows, because `import_hsa_transactions.py` only
ever reads `Description`/`Merchant Name`, and `Merchant Name` is blank
for that era. The same CSV has a `Consumer Note` column, never read by
the importer, that contains embedded merchant text for those exact rows
(e.g. "TERMINAL 426979 DAN FITZGERALD PHARMACY WHITEFISH WI"). Found by
re-examining the raw CSV columns directly at Tom's prompt rather than
trusting the prior assumption that the data simply wasn't captured
anywhere. Resolved as a one-off manual historical cleanup (61 rows,
2017-era) — scoped deliberately, not built into the importer, since the
newer-era rows don't need it.
*Source: Session 22 — HSA Consumer Note discovery, ISSUE-043*

## Power BI (continued) — DAX

### A calculated column can't be sorted by another column in the same table whose own formula references the sorted column — even when it's not a real infinite loop

Tried to sort an Account slicer by each account's total value using a
`SortValue` calculated column defined via
`LOOKUPVALUE(DimTable[TotalValue], DimTable[account_name],
FactTable[account_name])` on the fact table itself, then set "Sort by
Column" on `account_name` → `SortValue`. Power BI's dependency checker
flags this as circular (`SortValue` reads `account_name`; `account_name`
is sorted by `SortValue`) even though `SortValue`'s actual *value*
doesn't depend on `account_name`'s *sort order* — the checker works at
the formula-dependency level, not the semantic level. Fix: move the
value into a genuine one-row-per-key dimension table (already had one
handy from an earlier `AccountLatestValue` build), relate it to the fact
table normally, and set "Sort by Column" *within* that dimension table
instead — no self-reference, no circularity, and the slicer/relationship
still filters the fact table exactly the same way regardless of which
column in the related table is used for sort-by or display.
*Source: Session 22 — Baird Activity page, Account slicer sort*

### An `ADDCOLUMNS`-added column can't be referenced directly inside a nested `CALCULATE`'s filter argument

`ADDCOLUMNS(PerAccount, "TotalValue", CALCULATE(SUM(...), table[date] =
[LatestDate]))` — where `LatestDate` was itself added earlier in the
same `ADDCOLUMNS` chain — fails with "Column 'LatestDate' cannot be
found." `CALCULATE`'s filter-argument parser doesn't resolve a bare
`[ColumnName]` reference to an ad-hoc `ADDCOLUMNS` column the same way
plain row-context expressions do. Fix: capture the value into a `VAR`
*before* the `CALCULATE` call (ordinary row-context evaluation, no
ambiguity), then reference the `VAR` inside `CALCULATE`'s filter
argument instead of the raw column name.
*Source: Session 22 — AccountLatestValue calculated table*

### Inside a calculated column, `CALCULATE` context-transitions on the *entire* current row, not just the column you care about — `ALLEXCEPT` is required to undo that

A calculated column meant to sum a category's budget across all 12
months (`CALCULATE(SUM(budget_amount), YEAR(month_start_date) =
YEAR(TODAY()))`) silently returned only that row's own single month,
not the full year. Reason: `CALCULATE` evaluated inside a calculated
column's row context performs a full context transition — it filters
the table to match *every* column of the current row (including
`month_start_date`), not just the column being read. The explicit
`YEAR(month_start_date) = YEAR(TODAY())` filter gets added on top of
that implicit per-row filter, not instead of it. Fix:
`ALLEXCEPT(table, table[category], table[type])` inside the `CALCULATE`
strips the implicit row-context filter back down to just the columns
that should stay fixed, before the explicit year/date filters are
applied.
*Source: Session 22 — CategoryWithOverspend/CategoryWithIncomeAhead calculated columns*

### A card visual's "K/M" abbreviation is a separate setting from currency/decimal formatting, and lives under a different Format-pane tab

Setting a measure's format to Currency with 0 decimals (both at the
model level and in the visual's General → Data format section) had no
effect on a Card visual still showing the abbreviated "K" form instead of the full number. The
abbreviation is controlled by **Display Units**, found under the
**Visual** tab's "Callout value" section — a completely different part
of the Format pane from the General tab's Data format section, and not
overridden by anything set there. Set Display Units to `None` to show
the full number.
*Source: Session 22 — Baird Activity Income Minus Net Fees card*

### A chart's "Apply settings to" dropdown defaulting to a specific category means conditional formatting is being bound per-category, not to the plotted measure

Tried to conditionally color a column chart's bars by the sign of the
plotted measure (red for negative, green for positive) via Format pane
→ Columns → Colors, but the dialog only offered per-category static
color pickers (one row per axis value) instead of the expected Rules
dialog. Cause: "Apply settings to" was set to an individual category
instead of `All` — with a specific category selected, Power BI assumes
you want a manual per-category color, not a value-based rule. Switching
"Apply settings to" back to `All` collapses it to a single color swatch
with an `fx` icon, which is the actual entry point to Rules-based
conditional formatting bound to a measure's value.
*Source: Session 22 — Budget vs Actual bar chart*

### A seed process built around "spread the annual figure evenly across 12 months" will silently omit any category that doesn't actually work that way

The original 20-category `budget_targets` seed assumed every Expense
category has a real monthly cadence. `Property Tax` was never included
at all — not underfunded, simply absent — because it's paid as one
lump sum every January (confirmed against 7 years of actuals,
2020-2026), not a recurring monthly bill like everything else that was
seeded. It only surfaced as a fully-red, no-budget bar on a Power BI
pace chart built much later. When seeding a budget/target table from a
category list, check each category's actual payment cadence first
(lump-sum vs. recurring) rather than assuming a uniform monthly split —
a category that doesn't fit the assumed shape won't throw an error, it
will just be silently missing.
*Source: Session 22 — Property Tax budget gap, script 117*

## SQL — Aggregation

### `SUM(ABS(x))` and `ABS(SUM(x))` are not the same thing — the first silently turns refunds into additional spend

`vw_budget_vs_actual` computed actual spend as `SUM(ABS(amount))` — take
the absolute value of each row, then sum. That's fine only if every row
in the group shares the same sign. An Expense-typed category also holds
positive-signed rows (returns, refunds, credits against an earlier
purchase, manual reimbursements), and `ABS()` per row flips each of those
to positive spend, so they get *added* to the total instead of netted
against the purchase they reverse. `ABS(SUM(amount))` sums the signed
amounts first (so a refund cancels against its purchase), then takes the
absolute value of the net. The gap showed up as a per-month overstatement
that was exactly twice the sum of that month's positive-signed rows, and
it was invisible for Income because Income rows are already virtually all
positive-signed — which is exactly why a side-by-side comparison showed
Income matching and only Expense drifting. When a magnitude is needed
for display, apply `ABS()` to the group's net total, not to each row
inside it.
*Source: Session 23 — ISSUE-046, script 119*

### Never `abs()` a signed source amount at ingestion — it turns refunds and credits into spend

`plaid_sync.py` stored every Plaid transaction outside two categories as
`-abs(amount)`. Plaid sends a refund, return or statement credit as a
*negative* amount, so `abs()` discarded exactly the information that
distinguishes a credit from a charge, and every credit landed as spend. It
went unnoticed for months because most rows are ordinary charges, the
effect only shows in months with a refund, and a text search for
"refund"/"credit" catches only credits that happen to say so — an Amazon
return had no such wording and was found only by comparing the stored sign
against the raw source. Negate a signed amount (`-amount`), don't take its
absolute value, and if a category genuinely needs `abs` (transfers whose
direction is meaningless) special-case that category explicitly and say why.
Related: a sign convention that "differs by account type" is a hypothesis to
test against the raw source per account, not an assumption to build
around — here it was uniform across the credit card and the bank accounts.
*Source: Session 23 — ISSUE-047, scripts 121-122*

### Rebuild pages on one base view with measures instead of many pre-summed views

Four spending pages each sat on their own pre-aggregated SQL view
(`vw_monthly_spend`, `vw_cash_flow`, `vw_category_yoy`, `vw_top_merchants`).
Each re-implemented its own aggregation, and three of them summed
`ABS(amount)` per row, so one aggregation mistake was copied four times. One
of them (`vw_top_merchants`) had no date column at all, so its Year slicer
never filtered anything — the page silently showed all-time totals under a
single-year label. A single base view (`vw_transactions_clean`) with a small
set of shared DAX measures has one aggregation rule to get right, makes
every slicer work because the Calendar relationship is on the fact table,
and gives transaction-level drill for free. Pre-summed views are worth it
only for a genuine performance problem, not as the default.
*Source: Session 23 — Power BI page review*

### A Power BI slicer can't be sorted by a measure — sort a small dimension table by column and slice on that

A slicer sorts only by its own field. To order a category slicer by spend,
build a one-row-per-category calculated table that carries the spend value,
relate it to the fact table, set `Sort by column` *inside that table*, and
slice on the table's category column. The order is a snapshot taken at
model refresh, not responsive to other slicers; a small table visual
(category plus the measure, sorted by the measure, click to filter) is the
alternative when it must follow the Year slicer.
*Source: Session 23 — Top Merchants category slicer*
