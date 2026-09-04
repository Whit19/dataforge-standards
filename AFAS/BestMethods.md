# AFAS Project — Best Methods
**Hard-won lessons. Add entries as they are learned. Never delete.**
Last updated: 2026-09-04

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

### Reimbursements are positive amounts in the same category as the original expense
Work reimbursements are not recorded as Other Income. They are recorded as positive
amounts in the same category and subcategory as the original expense (e.g. positive
Work-Expense/Amy). This keeps net work expense tracking accurate.
*Source: SessionStarter key policies*

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