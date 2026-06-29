# AFAS Project — Best Methods
**Hard-won lessons. Add entries as they are learned. Never delete.**
Last updated: 2026-06-29

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