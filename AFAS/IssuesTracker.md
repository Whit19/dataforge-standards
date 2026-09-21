# AFAS Project — Data Issues Tracker
**Active issues only. Resolved items move to Decision Log with date closed.**
Last updated: 2026-09-21 (Session 23 — ISSUE-046 resolved and moved to DecisionLog; ISSUE-047 opened, fix deployed, awaiting live confirmation)

---

## How to Use
- **Open:** Issue identified, not yet resolved
- **In Progress:** Actively being investigated
- **Resolved:** Add resolution date and move ruling to Decision Log, then remove from this file

---

## Active Issues

### ISSUE-008 — Baird Plaid Connection Failure
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-06-01 |
| Priority | Medium |
| Description | Plaid connection to Robert Baird Online (ins_117067) fails. Originally INTERNAL_SERVER_ERROR / API_ERROR. Baird support advised a platform migration disrupted Plaid. Retried 2026-06-15 — still fails with generic "Couldn't connect to your institution" error. Second email sent. CSV fallback pipeline (import_baird_holdings.py) now operational as permanent workaround — all 11 Baird accounts covered via monthly manual CSV export. |
| Last Action | Second email sent to Baird Online Support. CSV fallback pipeline built and tested 2026-06-17. |
| Next Step | Await Baird response. If Plaid resolves, holdings will auto-populate via Plaid Investments endpoint. CSV pipeline remains as monthly fallback regardless. |

---

### ISSUE-047 — plaid_sync.py stores refunds and statement credits as charges
| Field | Value |
|-------|-------|
| Status | In Progress — fix deployed 2026-09-21 (deployed copy matched the local file), **awaiting live confirmation on the next sync** |
| Opened | 2026-09-21 |
| Priority | High |
| Description | `plaid_sync.py` stored every Plaid transaction outside `INCOME`/`TRANSFER_IN` as `-abs(amount)`, discarding the sign. Plaid sends a credit (refund, statement credit, return) as a negative amount, so it landed in `dbo.transactions` as a charge and was counted as spend. Verified 2026-09-21 against live Plaid data that the sign convention is identical for the Chase credit card and Associated checking/savings (positive = money out, negative = money in) — it is not account-type specific. Found while investigating a Chase rideshare statement credit stored with the wrong sign. |
| Last Action | Sign logic fixed in `plaid_sync.py` (merchant/spend categories now use `-amount`; `LOAN_PAYMENTS` transfers keep `-abs` as before). A replay of 825 live Chase + Associated transactions changed exactly 8, all credits. Eight already-stored rows corrected by script 122. Amex could not be checked — its Plaid item needs a Link update-mode re-login (`ITEM_LOGIN_REQUIRED`). |
| Next Step | Run the next sync (Portal `http_ingest`, or the monthly run) and confirm a Chase hotel credit already sitting in Plaid, dated 2026-09-18, lands in `dbo.transactions` as a positive amount — that is the live proof; then close this issue. Amex will error on that run until its Plaid item is re-logged-in. After the Amex re-login, re-run the sign comparison on the Amex credit card. A text search only catches credits with telltale wording, so other credits stored as charges before 2026-09-21 may remain — the Amazon return found here had no such wording. |

---

### ISSUE-014 (recurrence note on prior fix) — subcategory-mirror violations recurred
2026-06-01 taxonomy fix corrected subcategory=category mirror violations (Clothing, Dining Out, Gifts/Charity, Groceries, Payment). By 2026-07-01, 294 merchant_patterns rows and 11 category_map rows had the same violation again (Clothing, Groceries, Car, Property Tax, Payment, Dining Out, Interest). Root cause of the recurrence not yet investigated — worth checking whether a specific import/enrichment script is reintroducing these values, rather than treating each occurrence as an isolated one-off fix.

**2026-09-03 note:** Session 17's taxonomy-drift work (ISSUE-012) established the general mechanism for this class — a rename applied to transaction rows but never propagated to `merchant_patterns`/`category_map`, so the pattern table keeps re-creating the old value on every match. The mirror-violation recurrence is very likely the same shape (the retired mirror value still living in merchant_patterns).

**2026-09-16 note:** Tom closed ISSUE-012 without adding the proposed `taxonomy_audit.py` Check 5 (subcategory==category mirrors) — low priority, diminishing returns given Checks 1-3 are at zero. However, `enrich_transactions.py`'s new write-time taxonomy validation guard (ISSUE-044, resolved same day) means any *new* mirror-value write would now be blocked outright before it reaches `dbo.transactions`, since none of Category_Taxonomy.md's documented pairs are subcategory==category mirrors — this doesn't clean up any mirror value already sitting in `merchant_patterns`/`category_map`, but it does close off the recurrence mechanism going forward without a dedicated Check 5.

---

### ISSUE-016 — run_log missing entries for daily transaction syncs
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-01 |
| Priority | Medium |
| Description | plaid_sync.py's daily CHASE/AMEX/ASSOCIATED_PERSONAL transaction sync never writes to dbo.run_log, despite dbo.plaid_sync_state showing successful syncs. Only BAIRD_HOLDINGS, ASSOCIATED_PERSONAL_BALANCE, NWM_SYNC, and PRINCIPAL_401K currently log to run_log. This gap made the August 2026 outage harder to diagnose than it should have been. |
| Next Step | Add run_log INSERT calls to plaid_sync.py's _sync_institution(), matching the pattern already used in balance_sync.py/nwm_sync.py/principal_sync.py. |

---

### ISSUE-017 — HSA-Baird and other non-canonical Baird accounts never imported
| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-01 |
| Priority | Medium |
| Description | Confirmed via SELECT DISTINCT account_name FROM dbo.baird_holdings that HSA-Baird and "401k Baird Profit Share" (as a Baird-CSV-tracked account, separate from the now-connected Principal/Plaid version) have never appeared in a single monthly import. Not a Plaid gap — these accounts simply aren't part of what gets exported/tagged in the monthly CSV procedure. (Distinct from the new HSA-BOFA-MANUAL account built in Session 13 — that's a separate Bank of America custodial HSA with its own manual CSV pipeline; this issue is specifically about the Baird-administered HSA-Baird account.) |
| Next Step | Decide whether HSA-Baird needs to be added to the monthly export procedure, or confirm it's intentionally tracked elsewhere / excluded. |

---

### ISSUE-021 — import_baird_holdings.py has a broken local.settings.json path

| Field | Value |
|-------|-------|
| Status | Open |
| Opened | 2026-08-01 |
| Priority | Low |
| Description | Found while fixing get_plaid_tokens.py's hardcoded credentials: import_baird_holdings.py's local.settings.json path resolution points at Run_Monthly/local.settings.json, which doesn't exist. Currently harmless — that script never actually reads a PLAID_* value from local.settings.json — but would silently break if a future edit added one. |
| Next Step | Fix the path resolution to match the working pattern used elsewhere (same fix applied to get_plaid_tokens.py this session). Not urgent — dead code path today. |

---

---

## Resolved Issues (archive — recent)

### RESOLVED — ISSUE-038 — Apple Card CSV enrichment mislabels category_source (two distinct shapes)
- Resolved: 2026-09-08
- **Anomaly 1** (`category_source = 'plaid'` on 46 Apple rows): `import_apple_csv.py` stored Apple's own CSV "Category" column in the `plaid_category_raw` DB column; the now-retired `enrich_apple_csv.py`'s category_map step hardcoded `category_source = 'plaid'` (the main enricher labels the identical path `'category_map'`). No Plaid connection has ever existed for Apple Card. Script 84 relabeled the 46 rows and fixed 31 Apple `Uncategorized` rows (type/in_budget) + 2 Apple `Payment` rows + 3 category_map rows behind the problem. Script 87 remapped the coarse `Shopping` bucket (Large Purchases/General → Clothing/General, Tom's call). **Anomaly 2** (Playerfirst\*Nxtsports → Taxes/Federal): the `%IRS%` catch-all matched "IRS" inside "PLAYERF**IRS**T" — script 85 changed it to a bare exact-match `IRS`. Root cause of both: `enrich_apple_csv.py`/`enrich_hsa_csv.py` were drifted duplicates — both retired this session (AFAS b1a3ef3), `enrich_transactions.py` is now the single enricher. See DecisionLog 2026-09-08.

### RESOLVED — ISSUE-039 — `%ACT%` merchant_pattern is over-broad
- Resolved: 2026-09-08
- The live pattern was actually `% ACT %` (space-bounded, priority 30 — an earlier undocumented partial narrowing), not `%ACT%`. Script 83 changed it to a bare no-wildcard `ACT`, which `enrich_transactions.py`'s matcher resolves as an exact-string match (`raw_upper == 'ACT'`), not a LIKE substring — same mechanism as the ISSUE-027 "Apple" fix. Verified the real $112 ACT test charge still matches; "TRANSACTION FEE" no longer does. `%IRS  TREAS 310%`-style specific patterns untouched. See DecisionLog 2026-09-08.

### RESOLVED — ISSUE-041 — taxonomy_audit.py's own CANONICAL_TAXONOMY dict is stale
- Resolved: 2026-09-08
- The hardcoded dict (transcribed 2026-07-01) was replaced with `load_canonical_taxonomy(doc_path)`, which parses `Category_Taxonomy.md`'s own `## Full Taxonomy` fenced block at runtime and populates `CANONICAL_TAXONOMY`/`CANONICAL_PAIRS` fresh in `main()` every run. On a parse failure the script exits 2 — no hardcoded fallback, since a fallback is what drifts. Verified: 29 categories / 174 pairs parsed; ATM/Cash Spending/ATM, Dining Out/Fast Food, Pay/Whit no longer false-flag. AFAS 6c652ae. See DecisionLog 2026-09-08.

### RESOLVED — ISSUE-042 — Entertainment/Books & Audible and Subscriptions/Gaming appear swapped
- Resolved: 2026-09-08
- Not a swap and not a rule bug: **no** merchant_pattern or category_map row produces either combo — all 19 rows are `category_source = 'manual'`, `category_reviewed = 1` (hand-categorized directly). Tom's call after reviewing the full list: the 2 bookstore purchases → Entertainment/General; all 17 PlayStation charges → Entertainment/Gaming. Script 82 applied it by explicit `transaction_id` with a combo guard, preserving `category_source`/`category_reviewed`/`original_*` as an audit trail. `taxonomy_audit` Check 3 dropped 6 → 4. See DecisionLog 2026-09-08.

### RESOLVED — ISSUE-037 — enrich_hsa_csv.py unmatched-fallback gap (mirrors ISSUE-025)
- Resolved: 2026-09-08
- Made moot by the enrichment consolidation: `enrich_hsa_csv.py` was retired this session (AFAS b1a3ef3). HSA rows now fall through `enrich_transactions.py`'s `apply_fallback()`, which sets a complete Uncategorized/General row — and, as of this session, only fills `type`/`in_budget` where NULL, so an importer's deliberate classification (e.g. an unmatched Employer Contribution) is preserved. 0 HSA rows were ever actually in the broken state. See DecisionLog 2026-09-08.

### RESOLVED — ISSUE-040 — Gifts / Charity/Gifts subcategory: keep or normalize?
- Resolved: 2026-09-04
- Gifts / Charity/Gifts had been retired to General on 2026-03-12, but taxonomy_audit.py found it was still live on 93 active merchant_patterns, 1 category_map row, and 8 real transactions ($740.58). Tom's decision: normalize everything to 'General' rather than formally re-recognizing 'Gifts'. All three tables corrected (script 76); verified 0 remaining 'Gifts' subcategory anywhere afterward. See DecisionLog 2026-09-04.

### RESOLVED — ISSUE-026 — unclassified merchants from the ISSUE-019 backlog
- Resolved: 2026-09-03
- All remaining merchants identified and corrected this session: Bayshore D → Medical/Health/Dental (bank CSV + historical carry-forward from a prior manual categorization); WISCONSINGOV → Car/Registration (WI DOT vehicle registration — new merchant_pattern added); Musa I → Dining Out/General; SHEBOYGAN → Entertainment/General; Shorewood → Travel/Transportation (corrected from an invalid top-level "Transportation" category — it is a subcategory under Travel); Bizjtix ($2,700) → Work - Expense/Amy; Retail x2 ($1,028.64 on 6/9 + $142.42 on 7/10) → Sports / Clubs/Golf (Sand Valley Golf Resort — `plaid_transaction_name_check.py` confirmed no richer merchant text exists in Plaid's API for this row, so "Retail" is genuinely all Plaid has); Google Cloud ($0.06) → Work - Expense/DataForge (confirmed this AMEX connection is the DataForge business card); TEMPORARY FUNDS HOLD ($630.94, 8/1/2026) → deleted (transient bank-side hold artifact — never posted as a real charge). "Center"/"Product" were closed in Session 14; Railway in Session 15. See DecisionLog 2026-09-03.

### RESOLVED — ISSUE-036 — Marquette University payroll deposits misclassified via overly generic merchant_pattern
- Resolved: 2026-09-02
- Two payroll deposits misfired to Children/School Lunch via an overly generic `%MARQUETTE UN%` merchant_pattern (priority 10) that beat more specific sibling patterns on an alphabetical tie-break. `plaid_category_raw` correctly said `INCOME_SALARY` the whole time — the merchant_pattern step ran first (per this project's documented enrichment order) and preempted category_map's correct mapping before category_map ever got a chance to run. Corrected via script 69: historical fix on the 2 affected rows plus a new priority-5 pattern → Pay/Tom, matching the existing Godfrey/Heraeus payroll pattern convention. See DecisionLog 2026-09-02.

### RESOLVED — ISSUE-035 — enrich_transactions.py blanket-updated every loaded row regardless of whether anything changed
- Resolved: 2026-09-02
- write_results() unconditionally rewrote all 7,865 loaded 2024+ rows on every run, costing ~4 minutes per run and eroding `updated_at`'s reliability as a "was this genuinely touched" signal (used elsewhere in this project to confirm whether a SQL fix actually landed on specific rows). Fixed to write back only rows where at least one enrichment column actually changed. Tom caught two genuine bugs in the first drafted fix before it ran: a pandas NaN-comparison OR-logic error (`mask | (isna() != isna())` can't retract a false-positive `NaN != NaN`, must compute equality-or-both-NaN first, then invert) and a snapshot-timing bug that would have silently defeated the ISSUE-019 `--unenriched-only` retry path (must snapshot after the reset, not before). Both corrected and verified: `--unenriched-only` run showed 15/15 rows correctly flagged changed (matching Step 1/2/4's own match counts exactly); full run showed 0/7,865 changed, completing in 1.2s instead of ~4 minutes; `updated_at` confirmed byte-identical on a 10-row sample across both runs. See DecisionLog 2026-09-02.

### RESOLVED — ISSUE-034 — 159 orphaned HSA rows confirmed as genuine duplicates, deleted
- Resolved: 2026-09-02
- The 159 HSA rows with NULL `account_id` that Session 13 explicitly decided to leave alone (predating full visibility into the later full-history CSV import that happened in that same session) were confirmed this session to be genuinely redundant: 157 exact date+amount matches and 2 near-matches on manual review, all already covered by rows imported via the full-history CSV pull. Deleted via script 68 — reverses the Session 13 "leave as-is" decision now that the reason for it no longer applies. See DecisionLog 2026-09-02.

### RESOLVED — ISSUE-033 — 14,571-row account_id backfill gap (AMEX/APPLE/ASSOCIATED_PERSONAL/CHASE)
- Resolved: 2026-09-02
- 14,571 rows across AMEX/APPLE/ASSOCIATED_PERSONAL/CHASE had NULL `account_id`, predating account linkage in the pipeline (~Feb–Mar 2026 cutover per source). Backfilled via script 66. HSA's 159 pre-existing NULL rows deliberately excluded from this backfill — see ISSUE-034 (handled separately, as a deletion rather than a backfill, since those rows are duplicates rather than legitimately-orphaned). See DecisionLog 2026-09-02.

### RESOLVED — ISSUE-032 — Azure SQL auto-pause collides with the automated monthly timer trigger
- Resolved: 2026-09-02 (code); real-world test 2026-09-03 → **manual workaround adopted, not a code fix**
- FinanceDB (Free tier Serverless) can auto-pause between runs; the 2026-09-01 03:00 UTC automated monthly_sync/timer_sync run hit Azure SQL error 40613 ("Database not currently available") with no retry handling, failing silently while still logging "Succeeded" at the function level. Fix: retry-with-backoff on error 40613 added to db.py's get_sql_connection() (10/20/40/80/160s exponential backoff, ~5 min worst case; any other connection error still raises immediately, unretried).
- **2026-09-03 real-world test:** paused FinanceDB, triggered http_ingest via Azure Portal Test/Run. The 40613 retry-with-backoff **never fired**. The actual failure Application Insights logged was ODBC `HYT00` ("Login timeout expired") — a client-side connection timeout that happens *before* Azure SQL can return 40613, because the driver gives up waiting for the paused serverless DB to wake. Same failure mode confirmed locally (`enrich_transactions.py`, error `08001`).
- **Decision:** keep the existing manual monthly procedure (resume DB in the Portal, check freshness, manually retrigger http_ingest / http_balance_ingest / http_nwm_sync if the automated run failed) as the standing solution, rather than further engineering the retry to also catch `HYT00`/`08001` — which would require widening the ODBC login timeout itself first so a retry could ever run. The retry code works exactly as designed for error 40613; it simply does not cover this real-world failure shape. **Not broken or incomplete — deliberately scoped.** See DecisionLog 2026-09-03.
- **Follow-up (next session):** fold the manual DB-resume/verify/retrigger steps into a proper "Monthly DB Resume & Sync Verification" procedure in SessionStarter, matching the Monthly Baird Holdings / Monthly Apple Card procedure format.

### RESOLVED — ISSUE-025 — enrich_apple_csv.py's unmatched-fallback path left category/subcategory/type/in_budget NULL
- Resolved: 2026-09-02 (verified present in live code during doc sync)
- Confirmed as a real but so-far-dormant bug: `_UPDATE_UNMATCHED_SQL` only set `category_source = 'unmatched'`, leaving the other 4 enrichment columns NULL, which would exclude any genuinely unmatched Apple row from every Power BI view that filters on `type = 'Expense'` — the same visible symptom as ISSUE-019, for a different root cause. A live scoping query confirmed this had caused zero actual impact so far (0 APPLE rows currently unmatched) — this was a pre-emptive fix, not a backfill. Fix applies Category_Taxonomy.md's documented unmatched default (Uncategorized/General/Expense/In Budget=Yes/LOW confidence), matching enrich_transactions.py's own apply_fallback() exactly. Confirmed present in the live file during this doc sync; also confirmed via a synthetic-row test (not real data) that a row falling back this way is correctly excluded from re-processing on the next run, since `category` is no longer NULL. See DecisionLog 2026-09-02.

### RESOLVED — ISSUE-020 — category_confidence not tiered by merchant_pattern priority for Apple/HSA sources
- Resolved: 2026-09-02 (verified present in live code during doc sync)
- Root cause confirmed (previous "Open" description's hypothesis had the direction backwards): enrich_transactions.py already ties category_confidence to merchant_pattern priority (HIGH ≤10, MEDIUM ≤20, LOW otherwise); enrich_apple_csv.py and enrich_hsa_csv.py instead hardcoded category_confidence='HIGH' for every pattern match regardless of priority — so the same kind of low-specificity match reads LOW on Plaid-synced sources but HIGH on Apple/HSA, purely by which script processed it. Decision: standardize on the tiered model everywhere (see DecisionLog 2026-09-02). Both scripts updated to select `priority` and compute the same tiered confidence; historical carry-forward path (no priority signal available) stays HIGH in both, unchanged. Confirmed present in both live files during this doc sync, and confirmed via synthetic-row tests (not real data) against real merchant_patterns rows spanning all 3 tiers plus the historical path, in both scripts.

### RESOLVED — ISSUE-024 — plaid_sync.py's description column was dead code; no fallback against Plaid merchant_name drift
- Resolved: 2026-09-02
- tx.get("description", "") always returned empty — Plaid's transaction object has no field called description. Repurposed to capture Plaid's raw, unresolved `name` field at ingestion, giving a fallback text source for diagnosing future merchant-name drift (see ISSUE-024's original discovery, and ISSUE-027) without waiting for a human to notice a truncated/generic merchant_name after the fact. Deployed to Finance-ingest-Tom-v6 and verified against live production sync after Tom manually triggered http_ingest: 4 new rows confirmed with description populated, including a real-world validating case (merchant_name_raw='Apple', description='Apple Card').

### RESOLVED — ISSUE-022 — Extent of pre-existing March 2026 HSA merchant_patterns batch confirmed
- Resolved: 2026-09-02
- Diagnostic query run: confirmed 2 of the 4 HSA merchant_patterns collisions found in Session 13 (Payroll Deduction, Employer Contribution) were exact duplicates of the pre-existing March 2026 batch of ~2,866 general-purpose patterns; the other 2 were genuinely new. No surprises found, no further action needed.

### RESOLVED — ISSUE-023 — BANK_FEES/LOAN_PAYMENTS sign convention regression
- Resolved: 2026-09-01
- The Session 14 fix (remove BANK_FEES/LOAN_PAYMENTS from plaid_sync.py's INFLOW_CATEGORIES) had been drafted but never deployed — same deploy-gap pattern as ISSUE-019 below. Confirmed still live via a Power BI screenshot showing Rocket Mortgage/University Club Collection posting positive. Scope grew from Session 14's ~25-row estimate to 32 rows (2026-02-26 through 2026-09-01) since the bug kept running an extra month. Fix deployed and verified via VS Code's "Files (Read-only)" remote view; historical correction (amount = -amount) run on all 32 rows, verified 0 remaining. Live verification against a new real transaction still pending — no qualifying transaction has occurred since deploy; follow up against the next Amex/Chase autopay or mortgage payment. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-027 — Apple/Associated Personal merchant-name-drift + pending/settled duplicate
- Resolved: 2026-09-01
- Associated Bank's feed sometimes truncates "Apple Card" autopay descriptions to a bare "Apple," which fell through to the generic %APPLE% catch-all and miscategorized as Subscriptions/Apps & Software instead of Payment/Apple Card (June/July/August 2026 rows; March/April/May had been separately hand-corrected). Also found a pending/settled duplicate pair for the same underlying transaction. Fixed: deleted the stale pending row, added an exact-match (verified via direct code read, not assumption) merchant_patterns entry for bare "Apple," corrected the 3 miscategorized rows. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-028 — HSA transaction history duplicated 2x–4x (unstable transaction_id hashing)
- Resolved: 2026-09-01
- import_hsa_transactions.py hashed raw, unparsed CSV text for transaction_id; Bank of America's export-formatting drift between two monthly exports changed every row's hash, re-inserting the full account history under new IDs on each affected reimport instead of no-op'ing via MERGE (~400+ duplicate groups). Fixed to hash normalized values, verified idempotent via a real two-run test. Cleanup script caught and corrected mid-review to avoid deleting 157 of 159 orphaned rows Session 13 had explicitly preserved. Result: 835 rows deleted, 461 canonical rows remain, 0 duplicate groups. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-029 — UnicodeEncodeError (cp1252 vs UTF-8) across 5 monthly import/enrich scripts
- Resolved: 2026-09-01
- Box-drawing summary print output crashed on Windows terminals defaulting to cp1252. Crash occurred after DB commit, so no data was affected — confirmed by checking the DB directly. Same sys.stdout.reconfigure(encoding="utf-8") fix applied and verified against real runs in import_hsa_transactions.py, enrich_hsa_csv.py, import_hsa_holdings.py, import_apple_csv.py, enrich_apple_csv.py. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-030 — import_apple_csv.py MERGE unconditionally overwrote enrichment metadata on every re-run
- Resolved: 2026-09-01
- Any re-run of import_apple_csv.py against already-enriched data reset category_source/category_confidence to hardcoded import-time defaults and could flip in_budget back to 1 — confirmed as a real production risk (not just a test artifact) discovered when a same-session test re-run corrupted 131 rows. category/subcategory themselves were unaffected. First fix attempt (two WHEN MATCHED clauses) was invalid T-SQL, caught via a real test before deploying. Corrected fix uses a single WHEN MATCHED with CASE expressions, protecting already-enriched rows (including category_source='manual') while still updating never-enriched rows normally. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-031 — manual category corrections left type/in_budget as stale inherited values
- Resolved: 2026-09-01
- The 3 manually-corrected Apple/Associated Personal rows (ISSUE-027) and 1 legacy Session 13 HSA row set category/subcategory correctly but left type/in_budget unchanged from before the correction, causing the wrong Power BI Expense/Income/Other bucket despite the correct category label — caught via a Power BI pivot review, not anticipated in advance. Fixed all 4 rows using category_map's correct type/in_budget values for their category. New BestMethods lesson added: manual corrections must always set type/in_budget together with category/subcategory. See DecisionLog 2026-09-01.

### RESOLVED — ISSUE-019 — Power BI Monthly Spend showed Apple-only data for June/July 2026
- Resolved: 2026-08-03
- Root cause: plaid_sync.py stored the full personal_finance_category JSON object in plaid_category_raw instead of a clean category string, so category_map's exact-string match silently never worked for Plaid-synced sources. Compounded by a separate bug where enrich_transactions.py's isna()-gated retry logic never actually reprocessed rows previously written by apply_fallback(). Both fixed (CC prompts deployed); 64 affected rows backfilled via script 59, re-enriched, and verified. 26 of the remaining 42 unmatched rows resolved via category_map/merchant_patterns additions (scripts 60/61). 14 rows remain — tracked separately as ISSUE-026, not blocking. Power BI visual reconfirmation (manual refresh + Monthly Spend page check) still pending as of session end. See DecisionLog 2026-08-03 for full diagnostic trail.
- **RECURRENCE (2026-09-01):** the Session 14 code fix was correctly written but never actually deployed to Finance-ingest-Tom-v6 — the bug silently kept creating new JSON-blob rows for a full month (732 affected, not the original 64). "Resolved" had only ever been true for the historical backfill, not the ingestion path. Now genuinely deployed and verified (VS Code "Files (Read-only)" remote view confirmed the live INFLOW_CATEGORIES/extraction logic), 732 rows backfilled, 0 remaining. See DecisionLog 2026-09-01. New BestMethods lesson added: a "Resolved" status describes a fix being drafted/committed, not necessarily deployed — any Function-App-deployed file needs an explicit deploy-and-verify step before an issue is genuinely closed.

### RESOLVED — ISSUE-015 — Deployed Function App silently crash-looped for 6+ weeks (dotenv)
- Resolved: 2026-08-01
- python-dotenv was added to db.py's dependencies during the July session but never added to requirements.txt. Deployed app crashed on every cold start (ModuleNotFoundError), causing the trigger indexer to find 0 functions — not a portal display quirk, a genuine failure. All automated syncs (daily transactions, monthly balance/NWM sync) were silently dead for at least 6 weeks. Fixed by adding python-dotenv to requirements.txt and redeploying; confirmed via Application Insights ("5 functions loaded", clean host start).

### RESOLVED — ISSUE-009 — Principal Financial 401k Token Pending
- Resolved: 2026-08-01 (connection); **genuinely closed 2026-09-14** (see follow-up)
- First Plaid connection completed via get_plaid_tokens.py. Real account name confirmed: "BAIRD PROFIT SHARING AND SAVINGS PLAN" (Prft Shr 401(K) Def Thrift), owner Amy. New principal_sync.py script created to pull holdings via /investments/holdings/get — confirmed live: 1 account, 11 securities, 11 holdings, $2,096,195.86 total value. Not yet wired into the automated pipeline (standalone local script only).
- **Follow-up (2026-09-14):** the 2026-08-01 "Resolved" only ever covered the standalone script working locally — same "resolved ≠ deployed" pattern this project has hit before (ISSUE-019/023/018). Wired into `monthly_sync.py`'s timer + a new `/api/principal_ingest` manual route (AFAS 0cee6b6), **deployed** to Finance-ingest-Tom-v6, and verified via the Application Insights log of a real HTTP-triggered run — not just a local test. Separately found and fixed a second bug that had nothing to do with deployment: `vw_net_worth`'s Investment category was hardcoded to `dbo.baird_holdings` only, so even a fully working pipeline never surfaced the 401k on the Power BI net-worth page (sql/94). See DecisionLog 2026-09-14 for the full chain, including the follow-on `vw_holdings_all` consolidation and sector/classification cleanup this surfaced.

### RESOLVED — ISSUE-013 — Chase/Amex/Associated transaction verification needed
- Resolved: 2026-08-01
- Verified 2026-08-01: Chase and Associated were current (07-01/08-01), but AMEX had not synced since 2026-05-27 — root cause was NOT a query/data issue but the full production outage above (ISSUE-015) plus a separate expired Amex credential. Both fixed; Amex now current through 07-25. The underlying question of why nobody noticed the outage is tracked separately (see ISSUE-016).

### RESOLVED — ISSUE-018 — MAIN - Brokerage vs MAIN - BKG naming drift
- Resolved: 2026-08-01
- August 2026 Baird export used "MAIN - Brokerage" instead of canonical "MAIN - BKG" for all security holdings. Corrected via SQL (script 50) after confirming no symbol overlap between the two names (clean rename, not duplicated data). Root habit not addressed — worth double-checking next month's export uses the correct name from the start.
- **RECURRENCE (2026-09-02):** the same drift happened a second time on the July 2026 export (23 rows) — the Session 12 "Resolved" status was only ever half-true, since script 50 apparently never actually landed on this data despite being logged as having fixed it (same "resolved-but-not-actually-run" pattern as ISSUE-019/ISSUE-023). Data corrected this session via script 67. A permanent code-level fix — `ACCOUNT_NAME_NORMALIZATION` dict + `normalize_account_name()` in import_baird_holdings.py, called on the Account Name column before `holding_id` construction, raising loudly on any unrecognized name instead of silently importing under a drifted one — confirmed present in the live file during this doc sync. This is a code-review confirmation, not a production test: the fix hasn't yet processed a real monthly Baird export. Keep watching the next monthly import to confirm it actually fires as intended (either passing a correctly-named export through cleanly, or raising loudly on a drifted one).

### RESOLVED — ISSUE-010 — Plaid Balance Product Not Authorized
- Resolved: 2026-06-17
- Balance product approved by Plaid. plaid_client.py fixed (request object passed directly, not .to_dict()). [cursor] reserved keyword fix in balance_sync.py run_log INSERT. Associated balance pull live: 7 accounts, all verified.

### RESOLVED — ISSUE-011 — balance_sync.py error_log NULL error_id
- Resolved: 2026-06-15
- Fixed via str(uuid.uuid4()) inline

### RESOLVED — ISSUE-005 — enrichment.py Bonus Rule Amount Threshold
- Resolved: 2026-06-03
- Already implemented; retry path bug also fixed

### RESOLVED — ISSUE-004 — Uncategorized/VENMO-Review (46 rows)
- Resolved: 2026-06-02
- All rows classified

### RESOLVED — ISSUE-006 — May 2026 Apple Card CSV
- Resolved: 2026-06-01

### RESOLVED — ISSUE-007 — Historical Amount Sign Audit
- Resolved: 2026-06-01
