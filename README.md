# 🛒 SQL Code Review — Instacart Business Analytics Project

**Reviewer:** Antigravity AI  
**Files reviewed:** `InstacartSQL/01` through `InstacartSQL/09` (9 scripts, ~59 KB total)  
**Scope:** T-SQL correctness, schema design, ETL patterns, data quality, query logic, and readiness for a portfolio audience

---

## Overall Impression

This is a strong, well-above-average portfolio project. The author clearly understands **Kimball dimensional modelling**, writes defensive SQL (transactional ETL with rollback, dual validation layers, NULLIF guards), and documents design decisions in-file rather than leaving the reader to guess. Most issues below are refinements, not blockers. Where genuine bugs exist, they are flagged clearly.

---

## Summary Scorecard

| Area | Rating | Notes |
|---|:---:|---|
| Schema design | ⭐⭐⭐⭐⭐ | Textbook Kimball, surrogate keys, FKs, natural key UNIQUEs |
| ETL correctness | ⭐⭐⭐⭐☆ | Transactional with rollback; `PRINT` block placement is a bug |
| Data quality harness | ⭐⭐⭐⭐⭐ | Thorough — dual-layer, structured result table, future-proofed |
| Business queries | ⭐⭐⭐⭐☆ | Clean and well-commented; a few missed analytical angles |
| Portability / setup UX | ⭐⭐⭐☆☆ | Hard-coded path, double `.sql.sql` extension, no error for missing path |
| Code style | ⭐⭐⭐⭐⭐ | Consistent casing, aligned columns, excellent inline comments |

---

## File-by-File Review

---

### `01_database_setup.sql.sql` — Database & Schema Setup

**What it does:** Creates the `InstacartBA` database and `raw` / `dw` schemas with `IF NOT EXISTS` guards.

#### ✅ Strengths
- Idempotent guards (`IF DB_ID(...) IS NULL`, `IF NOT EXISTS`) mean the script can be re-run safely — a good habit.
- Concise and focused. Does exactly one job.

#### ⚠️ Minor Issues

**1. Missing `GO` between `IF` blocks**  
The `USE InstacartBA` statement on line 13 immediately follows `END GO`, which is fine, but schema creation commands (`EXEC('CREATE SCHEMA raw')`) run inside an `IF...BEGIN...END` which is fine syntactically. No bug here, but be aware that in some SSMS execution modes, batch separation matters.

**2. No collation specified for the database**  
```sql
-- Current
CREATE DATABASE InstacartBA;

-- Better: explicit collation avoids locale-dependent sorting surprises
CREATE DATABASE InstacartBA
    COLLATION Latin1_General_100_CI_AS_SC_UTF8;
```
The dataset contains UTF-8 encoded CSV files (`CODEPAGE = '65001'`). Specifying the collation at database creation ensures string comparisons and sorts behave predictably, especially for product names with special characters.

---

### `02_raw_tables.sql.sql` — Staging Tables

**What it does:** Creates the six raw landing tables with appropriate PKs and basic type constraints.

#### ✅ Strengths
- `DROP TABLE IF EXISTS` in reverse-dependency order (junction tables before parents) — correct and idempotent.
- Types are appropriately conservative: `TINYINT` for `order_dow` / `order_hour_of_day`, `DECIMAL(5,2)` for `days_since_prior_order`, `NULL` correctly allowed only where the dataset has known NULLs.

#### 🐛 Bug — Missing Comma in `raw.order_products_train`

```sql
-- Line 41-43 — MISSING COMMA before CONSTRAINT
CREATE TABLE raw.order_products_train (
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    add_to_cart_order INT NOT NULL,
    reordered BIT NOT NULL          -- ← missing comma here
    CONSTRAINT PK_raw_order_products_train PRIMARY KEY (order_id, product_id)
);
```

This is a **syntax error** that will cause `02_raw_tables.sql.sql` to fail at runtime. The `reordered` column definition is missing a trailing comma before the `CONSTRAINT` clause.

```sql
-- Fix
    reordered BIT NOT NULL,
    CONSTRAINT PK_raw_order_products_train PRIMARY KEY (order_id, product_id)
```

Compare with the correctly written `raw.order_products_prior` on line 34, which has the comma. The inconsistency is likely a copy-paste oversight.

#### ⚠️ Minor Issues

**No `CHECK` constraints on known-range columns**  
The columns `order_dow` (0–6) and `order_hour_of_day` (0–23) have range rules that file 05 validates after the fact. Enforcing them at the table level would fail invalid rows at insert time rather than only at validation time:
```sql
order_dow          TINYINT NOT NULL CHECK (order_dow BETWEEN 0 AND 6),
order_hour_of_day  TINYINT NOT NULL CHECK (order_hour_of_day BETWEEN 0 AND 23),
```
This is a design choice (some prefer to land dirty data then validate), but it's worth documenting either way.

---

### `03_indexes_and_views.sql.sql` — Raw Indexes & Unified View

**What it does:** Adds non-clustered indexes on both `order_products` tables and creates the `raw.v_order_products_all` view that unions prior + train data.

#### ✅ Strengths
- `IF NOT EXISTS` guard on index creation — prevents duplicate-index errors on re-runs.
- `UNION ALL` is the correct choice (no deduplication needed; prior and train are disjoint sets).
- Joining `user_id` into the view from `raw.orders` is smart: the fact load in file 07 can look up `dim_user` directly from the view without an extra join.
- `source_table` flag ('prior' / 'train') is good practice for lineage tracing.

#### ⚠️ Minor Issues

**1. `CREATE OR ALTER VIEW` — not guarded like indexes**  
The index creation uses `IF NOT EXISTS` guards, but the view uses `CREATE OR ALTER`, which is fine for SQL Server 2016+. Just be consistent in comments — the file header says "Create raw indexes" but the view is being altered if it already exists. Minor documentation nit.

**2. Consider indexing `product_id` on both `order_products` tables**  
The current indexes are on `order_id`, which helps order-oriented joins. The view and ETL also join on `product_id` (to look up `dim_product`). Adding a second index on `product_id` would speed up the fact load considerably on a 32M-row prior table:
```sql
CREATE INDEX IX_raw_order_products_prior_product_id
ON raw.order_products_prior(product_id);
```

---

### `04_import_raw_data.sql.sql` — BULK INSERT

**What it does:** Truncates and reloads all six raw tables from CSV files via dynamic SQL + `BULK INSERT`.

#### ✅ Strengths
- Dynamic SQL via `sp_executesql` is the correct approach to parametrise `BULK INSERT` (the path can't be a variable directly).
- `KEEPNULLS` on `orders.csv` correctly preserves NULLs in `days_since_prior_order`.
- `FORMAT = 'CSV'` + `FIELDQUOTE` handles embedded commas in product names without requiring a format file.
- In-file comment about `ROWTERMINATOR` fallback (`0x0D0A` for Windows line endings) is genuinely helpful.

#### 🐛 Bug / UX Issue — Hard-coded Path to the Author's Machine

```sql
-- Line 13 — points to a specific machine
DECLARE @BasePath NVARCHAR(500) = N'C:\Users\Admin\Downloads\InstaCart Online Grocery Market Basket Analysis\';
```

Anyone cloning this project will hit an error immediately. The README says to update this line, but the comment in the file itself is too subtle. 

**Recommended fix:** Make the empty-path case fail loudly:
```sql
DECLARE @BasePath NVARCHAR(500) = N'';   -- ← Set this before running

IF @BasePath = N'' OR @BasePath IS NULL
    THROW 50000, 'Set @BasePath to the directory containing the Instacart CSV files before running this script.', 1;
```

#### ⚠️ Minor Issues

**1. No `TRY/CATCH` wrapping**  
If the third or fourth BULK INSERT fails (e.g. file not found), the first two tables are already loaded but the last ones are empty. The script has no rollback. Since BULK INSERT can't run inside a user transaction in the same batch, the practical options are: (a) wrap the dynamic SQL in individual TRY/CATCH blocks with a clear PRINT on failure, or (b) document this behaviour explicitly.

**2. File name case sensitivity**  
The CSV file `order_products__prior.csv` uses double underscore. The script correctly has it on line 84. But if someone downloads from Kaggle and the file is named `order_products_prior.csv` (single), the BULK INSERT will fail silently (SQL Server won't give a clear error about the path). Worth noting in the comment.

**3. `TRUNCATE` order**  
Lines 17-22 truncate the junction tables before the parent tables. This is only safe because there are no FK constraints on the raw schema. If FKs were ever added to raw, this truncate order would fail. A single comment noting "FK-free raw schema, order doesn't matter" would prevent future confusion.

---

### `05_raw_validation.sql.sql` — Raw Layer Validation

**What it does:** Runs row counts, TOP 10 spot checks, duplicate checks, NULL checks, range checks, orphan checks, eval set distribution — and creates a `raw.validation_log` table at the end.

#### ✅ Strengths
- Comprehensive coverage: the 5-category check structure (counts → duplicates → nulls → ranges → orphans) mirrors what a professional data quality framework would produce.
- Each `GO` statement isolates batches so a failure in one check doesn't abort the rest.
- Orphan checks cover all FK paths (prior → orders, train → orders, prior → products, train → products, products → aisles, products → departments).
- `eval_set` distribution query is a useful sanity check.

#### ⚠️ Issues

**1. `raw.validation_log` is created but never populated**  
The table is defined at the bottom of the file with `run_id`, `check_name`, `passed`, etc. — clearly intended for programmatic logging — but no `INSERT INTO raw.validation_log` statements exist anywhere in the project. The table definition sets up a good pattern but the pattern is incomplete.

Either populate it (turning the ad-hoc `SELECT` checks into structured inserts), or remove the table and note it as a "Future improvement" like the similar pattern in file 08. Leaving an empty table in the schema is confusing for reviewers.

**2. Duplicate checks only cover dimension tables, not fact tables**  
The PK on `raw.order_products_prior` is `(order_id, product_id)`. A composite-PK duplicate check isn't included:
```sql
-- Worth adding
SELECT order_id, product_id, COUNT(*) AS cnt
FROM raw.order_products_prior
GROUP BY order_id, product_id
HAVING COUNT(*) > 1;
```
This matters because BULK INSERT won't enforce the PK if the constraint is NOCHECK or was dropped.

**3. `SELECT TOP 10 *` on tables this large**  
`raw.order_products_prior` has ~32M rows. `SELECT TOP 10 *` without an `ORDER BY` returns arbitrary rows and isn't reproducible. Fine for development, but consider replacing with deterministic spot-check queries (`WHERE order_id = <known ID>`) in a production-ready script.

**4. No expected counts documented**  
The row count query produces numbers but gives the reader no pass/fail target. The README mentions the Kaggle dataset sizes. Adding expected counts as comments would make validation self-contained:
```sql
-- Expected: departments=21, aisles=134, products=49688,
--           orders=3421083, prior=32434489, train=1384617
SELECT 'raw.departments' AS table_name, COUNT(*) AS row_count FROM raw.departments
...
```

---

### `06_dw_tables.sql.sql` — Star Schema DDL

**What it does:** Creates the full Kimball star schema with 5 dimensions and 1 fact table, plus supporting indexes.

#### ✅ Strengths
- Design notes at the top are excellent — they explain every non-obvious decision (surrogate vs natural keys, degenerate dimension, factless fact, time attributes on dim_order).
- Drop order respects FK dependencies (fact → child dims → parent dims).
- `UQ_*_natural` constraints enforce natural key uniqueness on every dimension — a critical pattern that many beginners skip.
- `WITH CHECK` is used when re-adding FKs in file 07, which is correct (validates existing data, not just new inserts).
- `BIGINT` on the fact PK is appropriate for a 33M+ row table.
- `source_table VARCHAR(10)` on the fact is a good lineage column.

#### ⚠️ Minor Issues

**1. `user_key` is redundant on `fact_order_item`**  
The fact already joins to `dim_order`, and `dim_order` has `user_key`. Storing `user_key` directly on the fact table is a denormalisation. It saves one join in user-filtered queries but violates strict Kimball grain — the fact row already implies a user via the order. The choice is defensible, but it's undocumented in the design notes (unlike the other deliberate trade-offs).

**2. No `is_first_order` or `order_sequence` flag on `dim_order`**  
The most important business finding (loyalty inflection at order 5) is derived from `order_number`. A pre-computed `is_first_order BIT` flag on `dim_order` would simplify several business queries and is a natural fit for a dimension attribute.

**3. Index on `dw.fact_order_item(reordered)`**  
A single-column index on a BIT column with only two values (high cardinality: ~59% reordered) is generally not useful for a query optimizer. The index on `reordered` alone (line 210) will almost always be ignored in favour of a scan. Consider a composite covering index instead:
```sql
-- More useful: covers the common reorder-rate query pattern
CREATE INDEX IX_fact_order_item_reordered_cover
    ON dw.fact_order_item(reordered)
    INCLUDE (order_key, product_key);
```

---

### `07_load_dw.sql.sql` — Transactional ETL

**What it does:** Drops FKs, truncates all DW tables, loads all 6 tables in dependency order, re-adds FKs with `WITH CHECK`, then commits — all inside a single `BEGIN TRY / BEGIN TRANSACTION`.

#### ✅ Strengths
- Full rollback semantics: even the `TRUNCATE` is inside the transaction, so a failed load leaves the DW in a clean (empty) state rather than a half-loaded one.
- Load order correctly follows FK dependency chain (department → aisle → user → product → order → fact).
- INNER JOINs on dim lookups are deliberate and documented — a missing aisle/department will fail loudly, not silently drop rows.
- `WITH CHECK` when re-adding constraints validates all loaded data, not just future inserts.
- `COUNT_BIG(*)` on the fact avoids INT overflow on a 33M+ row table.
- The `do_` alias comment (reserved word avoidance) is a professional touch.

#### 🐛 Bug — `PRINT` Block is Outside the `TRY` Block

```sql
-- Line 190: COMMIT happens here
COMMIT TRANSACTION;

-- Lines 194-215: PRINT block is OUTSIDE the TRY block but BEFORE END TRY
DECLARE @cnt_dept INT, ...
SELECT @cnt_dept = COUNT(*) FROM dw.dim_department;
...
PRINT 'Load complete.';

END TRY  -- ← this closes the TRY
BEGIN CATCH
    ...
END CATCH;
```

The `DECLARE` and `PRINT` statements after `COMMIT TRANSACTION` (lines 194–215) are **inside the `TRY` block** syntactically (the `END TRY` is at line 217). This means:
- If a `SELECT COUNT(*)` fails for any reason after commit, the `CATCH` block would execute — but the transaction is already committed, so `ROLLBACK` would fail (`@@TRANCOUNT = 0`), and the error would surface confusingly.
- More practically: the `DECLARE` statements after `COMMIT` will generate a runtime warning in some SSMS versions because you are re-declaring variables inside the same batch after a commit point.

**Fix:** Move the PRINT block to after `END CATCH`, or wrap it in its own `BEGIN` block after the commit:
```sql
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
GO

-- Post-load summary (outside the transaction)
DECLARE @cnt_dept INT, ...
SELECT @cnt_dept = COUNT(*) FROM dw.dim_department;
...
PRINT 'Load complete.';
GO
```

#### ⚠️ Minor Issues

**1. FK drop/re-add pattern requires ALTER TABLE permissions**  
In shared or cloud environments (Azure SQL, Synapse), dropping FKs may require elevated permissions not always granted to an analyst role. Worth noting this dependency in the file header.

**2. No row-count assertion after load**  
The `PRINT` block reports counts but doesn't compare them to raw. Adding a simple assertion before the COMMIT would catch a silently short load:
```sql
IF (SELECT COUNT_BIG(*) FROM dw.fact_order_item) <
   (SELECT COUNT_BIG(*) FROM raw.v_order_products_all)
    THROW 50001, 'Fact row count is less than raw source. Load may be incomplete.', 1;
```

---

### `08_dw_validation.sql.sql` — DW Validation Harness

**What it does:** Runs 16 structured validation checks into a `#dw_validation_results` temp table, then outputs a consolidated pass/fail summary.

#### ✅ Strengths
- The structured temp table with a computed `status` column is the strongest design pattern in the project. It produces one clean pass/fail table instead of requiring the user to interpret multiple result panes.
- `ABS(@raw_count - @fact_count)` correctly handles the case where the fact has *more* rows than raw (which shouldn't happen but could with re-runs).
- Checks are grouped logically: counts → duplicates → surrogate null → FK integrity → range → value consistency → row parity → dense keys.
- The commented-out `THROW` block at the bottom is excellent forward-thinking — it shows exactly how to make this script pipeline-ready without enabling it prematurely.
- `CASE WHEN status = 'CHECK' THEN 1 ELSE 2 END` in the ORDER BY is a clever way to surface failures first.

#### ⚠️ Issues

**1. Dense surrogate key check duplicates Section 3**  
Section 3 (standalone `SELECT UNION ALL`) and Section 5.7 (INSERTs into `#dw_validation_results`) both check dense keys. The standalone Section 3 produces a separate, unstructured result pane that the reader must manually check — inconsistent with the goal of a single structured output. Consider removing Section 3 or merging it into Section 5.7.

**2. No check for `source_table` values in the fact**  
The `source_table` column should only contain `'prior'` or `'train'`. A simple check would catch data corruption:
```sql
INSERT INTO #dw_validation_results (check_name, issue_count, expected_result)
SELECT
    'Invalid source_table values in dw.fact_order_item',
    COUNT_BIG(*),
    '0 rows with values other than prior or train'
FROM dw.fact_order_item
WHERE source_table NOT IN ('prior', 'train');
```

**3. `days_since_prior_order` range check may over-flag**  
Section 5.4 (line 368–386) flags any `order_number > 1` row where `days_since_prior_order IS NULL OR < 0 OR > 30`. The Instacart dataset is known to cap `days_since_prior_order` at 30 in the `orders.csv` source (values > 30 are clipped). The range check is correct for this dataset, but the comment should note this is a dataset-specific rule, not a universal business rule.

---

### `09_business_queries.sql.sql` — 20 Business Queries

**What it does:** 20 analytical queries covering KPIs, timing patterns, basket analysis, customer segmentation, product rankings, and co-purchase pairs.

#### ✅ Strengths
- Queries use surrogate keys for joins and natural keys only as descriptive output — correct star-schema query pattern.
- `NULLIF` guards on all division operations prevent divide-by-zero errors.
- CTE usage in Q06 and Q13 eliminates repeated `CASE` logic and keeps queries readable.
- `HAVING COUNT(*) >= N` filters on Q09, Q11, Q15 prevent misleading rates from small samples.
- Q18 (department co-purchase pairs) uses `department_key < department_key` to avoid A-B / B-A duplicates — a subtlety many analysts miss.
- `DISTINCT` in the Q18 CTE correctly de-dupes department within order before self-joining.

#### ⚠️ Issues

**1. Q02 and Q03 count `dim_order` rows, not distinct orders**

```sql
-- Q02 — counts order-item rows if joined to fact, but counts dim_order rows here
SELECT o.order_dow, COUNT(*) AS total_orders
FROM dw.dim_order o
GROUP BY o.order_dow
```

`dim_order` has one row per order (grain = order), so `COUNT(*)` here equals `COUNT(DISTINCT order_id)`. This is correct. However, the column alias `total_orders` is slightly misleading because the query counts all orders including the `test` eval_set, which has no items in the fact. Consider filtering:
```sql
WHERE o.eval_set IN ('prior', 'train')
```
Or add a note that test orders are included.

**2. Q05 joins fact to dim_order but Q02/Q03 don't — inconsistency**  
Q02/Q03 query `dim_order` directly (correct, no fact join needed for order-level metrics), but Q05 joins the fact to `dim_order` to get basket size by weekday. This is also correct. However, the different approaches for related questions may confuse readers. A brief comment explaining why each approach is used would help.

**3. Q18 (department pairs) is O(N²) and can be very slow**  
The self-join on `order_department` with 3.4M orders is potentially a multi-minute query. Even with the `department_key < department_key` condition, this is a cross-product of departments per order. It would benefit from a `TOP 10000` on the CTE or a note that execution time may be several minutes.

**4. Missing: lift calculation for Q18**  
The README itself flags this as the most important next step, and the comments in Q18 mention raw co-occurrence overstates strength. Including a lift formula — even as a commented-out extension — would demonstrate analytical depth:
```sql
-- Lift = P(A∩B) / (P(A) * P(B))
-- To compute: divide pair_count / total_orders by
--   (dept1_count/total_orders) * (dept2_count/total_orders)
```

**5. Q12 mixes `prior` and `train` eval sets without filtering**  
Reorder behavior by `order_number` across all eval sets means customers with both prior and train orders appear twice in the aggregation. Since prior and train are different eval labels for the same customer's history, this may or may not be intentional. At minimum, add a comment clarifying the intent.

**6. Q19 — average gap by order number skips `order_number = 1`**  
The `WHERE days_since_prior_order IS NOT NULL` filter correctly excludes first orders (where the gap is undefined). This is right. However, the output starts at `order_number = 2`, which a reader may find surprising without a comment. Add:
```sql
-- Note: order_number = 1 is excluded because days_since_prior_order
-- is always NULL for a customer's first order (no prior order to measure from).
```

---

## Cross-Cutting Issues

### 1. Double `.sql.sql` File Extensions
All files have the extension `.sql.sql`. This is a GitHub upload artifact and not valid SQL file naming. While functionally harmless, it prevents SSMS from recognising files as SQL when opened from Explorer, and looks unprofessional on a portfolio. Rename to single `.sql`.

### 2. `raw.validation_log` is Orphaned
The table is created in file 05 but never used. Either implement the logging pattern or remove the table.

### 3. No `SET NOCOUNT ON`
Adding `SET NOCOUNT ON` at the top of each script suppresses the "X row(s) affected" messages that clutter the SSMS Messages pane during large batch operations (especially the fact table load).

### 4. No Execution Order Guard
There is no mechanism to prevent scripts from being run out of order (e.g. running `07_load_dw.sql.sql` before `06_dw_tables.sql.sql`). Consider adding an existence check at the top of each script:
```sql
-- 07_load_dw.sql — requires 06_dw_tables.sql to have been run first
IF OBJECT_ID('dw.fact_order_item') IS NULL
    THROW 50000, 'Run 06_dw_tables.sql before this script.', 1;
```

---

## Prioritised Recommendations

| Priority | Issue | File | Action |
|:---:|---|---|---|
| 🔴 | Missing comma in `raw.order_products_train` | `02` | Add comma after `reordered BIT NOT NULL` |
| 🔴 | Hard-coded `@BasePath` with no guard | `04` | Add empty-path `THROW` guard |
| 🔴 | `PRINT` block outside `TRY` (inside it syntactically, post-`COMMIT`) | `07` | Move post-`COMMIT` summary after `END CATCH` |
| 🟠 | `raw.validation_log` created but never populated | `05` | Implement inserts or remove the table |
| 🟠 | Dense key check duplicated in sections 3 and 5.7 | `08` | Remove standalone Section 3 |
| 🟠 | No `product_id` index on `order_products` tables | `03` | Add index to speed ETL join |
| 🟡 | Lift calculation missing from Q18 | `09` | Add commented-out lift formula |
| 🟡 | `index on reordered (BIT)` is low-selectivity | `06` | Replace with covering composite index |
| 🟡 | Duplicate fact-key check missing for composite PK tables | `05` | Add composite `(order_id, product_id)` dup check |
| 🟢 | Double `.sql.sql` file extensions | All | Rename on GitHub |
| 🟢 | Missing `SET NOCOUNT ON` | All | Add at top of each file |
| 🟢 | No execution order guards | `06`, `07`, `08`, `09` | Add `OBJECT_ID` pre-flight checks |
| 🟢 | Expected row counts not documented in file 05 | `05` | Add as comments |

---

## What's Already Portfolio-Ready

- **Transactional ETL with rollback** — This alone demonstrates production-grade thinking that most junior portfolios lack.
- **Dual-layer validation harness** — The structured `#dw_validation_results` temp table with computed `status` is exactly the pattern used in real data engineering QA pipelines.
- **Kimball star schema with `WITH CHECK` FKs** — Correctly implemented, documented, and aligned with the business queries.
- **20 business queries with NULLIF guards and CTE usage** — Clean, readable, and analytically sound.
- **In-file design documentation** — The long comment blocks in `06` and `07` show the author understands their own decisions, which is rare and impressive.
