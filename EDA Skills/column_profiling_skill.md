---
name: column-profiling-eda
description: 'Stage 2 of the modular EDA suite: read-only, column-level profiling of every table and view in any connected database or data platform (SQL Server, Azure SQL, PostgreSQL, MySQL, Oracle, Snowflake, Databricks, BigQuery, Fabric, or an MCP data connection). Produces each column''s data type, nullability, declared key role, null count and percentage, distinct count and cardinality class, string length shape, and length utilisation for variable-length columns, and verifies unenforced or candidate keys. Value-free: returns counts and lengths only. Use this whenever the user asks to profile columns or fields, check data types, nulls, completeness, cardinality, distinct counts, uniqueness, candidate or duplicate keys, over-sized or right-sized varchar columns, or column sizing for a migration or modernization, even if they do not say "column profiling".'
argument-hint: 'Optional: schema(s) or objects to include/exclude, candidate keys to test, approximate-distinct row threshold, length-utilisation thresholds'
user-invocable: true
---

# EDA Stage 2: Column-Level Profiling

Produce one record per column for every table and view in scope, describing its type, completeness, cardinality, and string length shape, and verify any keys that table profiling could not confirm. Every value must trace to platform metadata or an aggregate query that was actually run.

This skill is platform-agnostic. Use whatever the connected platform exposes. If `table-profiling-eda/references/catalog-sources.md` is available, read the section for the connected platform for catalog and key-enforcement specifics.

## Scope of this skill

This skill owns: column metadata, declared key role, null counts, distinct counts and cardinality, string length shape, length utilisation, and key verification.

It does not produce, and must not drift into:
- Min/max values, numeric or date ranges, distributions, category frequencies → data-patterns skill
- Empty-string and whitespace-only checks, cross-field rules, defect rules → data-quality skill
- Timestamp/watermark analysis and growth → volume-growth skill
- PHI/PII classification → phi-review skill
- Foreign-key mapping and ER diagrams → relationships skill

When an observation matters to another stage, record it in `Notes` as a handoff instead of investigating it.

It does not decide the output format. If an output-format skill is available, follow it. Otherwise return results as plain tables in chat and state that formatting is pending.

## Safety: value-free by design

- Read-only. Discovery, catalog, and aggregate operations only. Never create, alter, or delete objects, and never execute procedures or functions.
- Return **counts and lengths only, never values**. No min/max values, no samples, no most-frequent values, no example duplicates. A minimum date of birth or a maximum account number is a value leak, not a statistic. Because the output contains no values, this skill is safe to run before the PHI review.
- Duplicate checks report group and row counts, never the duplicated key values.
- If a check is not supported by the connection, mark it `Unavailable` with the reason. Never approximate by sampling and present the result as a profile.

## Step 0: Inputs and run header

Reuse the `table-profiling-eda` inventory when it exists in this run or is supplied: object list, object type, Rows Count, declared keys, key enforcement, and visibility. If it does not exist, run table profiling first or gather the minimum (objects, row counts, declared keys) and say so.

Record in the run header:
- Platform, database/catalog, visibility (`Full catalog` / `Permission-limited catalog` / `Exposed entities only`), scope, run timestamp (UTC)
- Approximate-distinct row threshold (default **10,000,000 rows**, user-configurable)
- Cardinality thresholds (defaults below)
- Length-utilisation thresholds (defaults below)
- Length function and unit used for the platform

## Step 1: Column metadata

From metadata only, record for every column of every table and view in scope:

| Field | Content |
|---|---|
| Schema | |
| Object | |
| Object type | From table profiling |
| Column | |
| Ordinal | Column position |
| Data type | Readable declared type, e.g. `nvarchar(50)`, `decimal(19,4)`, `VARCHAR2(50 BYTE)` |
| Type family | `String` / `Numeric` / `Date-time` / `Boolean` / `Binary` / `Semi-structured` / `Spatial` / `Other` |
| Length | Declared length and unit (characters or bytes), `Unbounded`, or `N/A` |
| Fixed/Variable | For string and binary types |
| Precision / Scale | For numeric and date-time types where defined |
| Nullable | `Yes` / `No` as declared |
| Declared PK | Position within the PK, or blank |
| In unique constraint | Constraint name(s), or blank |
| Declared FK | `Yes` / `No` only; mapping belongs to the relationships skill |
| Key enforced | `Yes` / `No (informational)` / `Unknown` / `N/A` |

Explain types in plain words where useful: `nvarchar(50)` holds up to 50 Unicode characters; `decimal(19,4)` holds 19 digits in total, 4 after the decimal point. Label `rowversion`/`timestamp` (SQL Server) and similar as **binary change tokens, not dates**. Do not infer type details the metadata does not provide.

The first six fields (Schema · Object · Column · Ordinal · Data type · Nullable) are mandatory for every column, even when every later step fails for that object.

## Step 2: Completeness

For every column: **Null count** and **Null %** (of Rows Count).

- Use a non-null count aggregate (e.g. `COUNT(column)`) against the total row count, or an explicit null count. Record the method.
- Use the same row-count snapshot for the percentage as for the null count. If the table was counted in a different query or at a different time, note it, because live tables drift between queries.
- A declared `NOT NULL` column still gets counted once as a check, but may be reported as `0 (declared NOT NULL)` without a scan when the platform enforces nullability and cost matters. Say which was done.
- If the connection cannot count non-null values reliably, mark `Unavailable` rather than estimating.

## Step 3: Cardinality

For every column whose type supports it: **Distinct non-null count**, **Distinct %** (of non-null rows), **Distinct method**, and **Cardinality class**.

Method:
- Below the approximate-distinct threshold: exact distinct count, labelled `Exact`.
- Above the threshold: use the platform's approximate distinct function where it exists (e.g. `APPROX_COUNT_DISTINCT`, `approx_count_distinct`, HyperLogLog-based functions), labelled `Approximate`. If the platform has none, ask before running exact distinct counts on those tables.

Cardinality class (defaults, configurable, stated in the run header):

| Class | Rule |
|---|---|
| `All null` | No non-null values |
| `Constant` | Exactly 1 distinct value |
| `Low` | 2 to 50 distinct values |
| `High` | More than 50 and below the near-unique threshold |
| `Near-unique` | ≥ 99% of non-null rows distinct |
| `Unique (observed)` | 100% distinct and no nulls |

`Unique (observed)` is an observation about current data, not a key. Always say so. Never assign `Unique (observed)` from an approximate count; use `Near-unique (approximate)` instead.

Skip distinct counting for types where it is unsupported or meaningless (LOB/`max` types, binary, XML/JSON/variant, spatial). Mark them `N/A – <type>` and keep their null counts.

## Step 4: String length shape

For string columns only: **Min length**, **Avg length**, **Max length** of non-null values. These are lengths, never values.

- Record the length function and its unit. Platforms differ: some functions count characters, others bytes, and some ignore trailing spaces (e.g. SQL Server `LEN`). Use a character-count function where available, and note any trailing-space behaviour.
- For LOB/`max` types, compute lengths only if supported and affordable; otherwise mark `Unavailable` and say why.

## Step 5: Length utilisation (variable-length strings)

Applies to **variable-length string columns** only. Fixed-length `char`/`nchar` columns are excluded, since they are always full by definition.

| Field | Content |
|---|---|
| Declared max length | From Step 1, with unit |
| Observed max length | From Step 4, in the same unit |
| Utilisation % | Observed ÷ declared × 100, 1 decimal |
| Length class | See below |

Length class (default thresholds, configurable, stated in the run header):

| Class | Rule |
|---|---|
| `Over-sized` | Utilisation below 10% |
| `Right-sized` | 10% to below 90% |
| `Near limit` | 90% to below 100% |
| `At limit` | Exactly 100%: possible upstream truncation, flag for review |
| `Unbounded` | No meaningful declared limit |
| `All null` | No values to measure |
| `Unavailable` | Units cannot be matched, or lengths could not be computed |

Rules:
- **Unbounded types** (`varchar(max)`, `nvarchar(max)`, `text`, unbounded `varchar`, `STRING` without a limit) get `Unbounded` with observed max length only, and no utilisation %. On platforms whose default string length equals the platform maximum (e.g. Snowflake `VARCHAR` = 16,777,216), treat a declared length equal to that maximum as `Unbounded`: no length was ever chosen.
- **Units must match.** Character vs byte semantics differ across platforms (e.g. Oracle `BYTE` vs `CHAR` semantics, multi-byte character sets, SQL Server `nvarchar` declared in characters but stored in bytes). If declared and observed lengths cannot be expressed in the same unit, use `Unavailable – unit mismatch`. Never divide bytes by characters.
- **Over-sized needs exact data.** If observed max length came from approximate or partial data, do not class the column `Over-sized`; the longest value may not have been seen.
- `At limit` is a prompt to investigate truncation, not proof of it. Label it that way.
- Length utilisation is descriptive. Do not recommend new column sizes as production-ready; resizing needs growth headroom, source-system limits, and downstream contracts that this skill cannot see.

## Step 6: Key verification

Test, as full column combinations (never column by column):
- every key that table profiling marked `Declared, unenforced`;
- every candidate key the user names;
- optionally, single-column `Unique (observed)` or `Near-unique` columns in tables with no declared key, when the user asks.

For each key report: key columns, rows tested, null rows in any key column, **duplicate group count**, **affected row count**, method, and result.

Result labels:
- `Holds (observed)`: no duplicates and no nulls in current data. Not a guarantee.
- `Fails – defect`: duplicates exist **and** uniqueness is documented or intended (declared key, supplied business rule).
- `Fails – candidate-key observation`: duplicates exist and uniqueness was only a hypothesis.
- `Unavailable`: the connection cannot run the check; say why.

Never return the duplicated values themselves.

## Execution and cost

- Profile tables first, then views. Every aggregate on a view re-executes it; if a view times out or is too costly, mark its columns `Unavailable – timeout/cost` and continue. Never drop a view silently.
- Batch all columns of an object into one or a few aggregate queries rather than one query per column. Split very wide objects into column groups.
- External and lake-format objects may scan remote files on every aggregate. Ask before profiling large external objects.
- Record the profiled timestamp per object.

## Result fields

Per column, in this order:

**Schema · Object · Object type · Column · Ordinal · Data type · Type family · Length · Fixed/Variable · Precision · Scale · Nullable · Declared PK · In unique constraint · Declared FK · Key enforced · Rows Count · Null count · Null % · Distinct count · Distinct % · Distinct method · Cardinality class · Min length · Avg length · Max length · Declared max length · Utilisation % · Length class · Profiling method · Profiled timestamp · Notes**

Fields that do not apply to a column hold `N/A`; fields that could not be computed hold `Unavailable` with the reason in `Notes`. Never leave a field blank where blank could be read as zero.

Plus a key-verification result per tested key (Step 6), the run header, and a limitations list. The output-format skill decides layout and file type.

## Summary for the user

Cover, in this order:
1. Run header: platform, visibility, scope, objects and columns profiled, thresholds used, methods used.
2. Columns that are 100% null.
3. Constant columns.
4. Top 10 columns by Null %.
5. Unique/near-unique candidate columns in tables with no declared key.
6. Key verification results, failures first.
7. **Top 10 most over-sized variable-length columns** (lowest utilisation, exact data only), showing Declared max length, Observed max length, and Utilisation %.
8. **Columns at limit**, flagged for truncation review.
9. Views and objects that timed out or were skipped for cost.
10. Limitations and blocked checks, stated plainly.

For every Top 10 list: rank on the unrounded value, let ties share a rank, show fewer than 10 when fewer qualify, and exclude `Unavailable` and approximate values with a stated count of exclusions.

Never describe an observed uniqueness, an approximate count, or an `At limit` flag in the same confident voice as a declared, enforced fact.
