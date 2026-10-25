---
name: data-quality-eda
description: 'EDA suite quality stage. Read-only. Judges whether the data is right in any connected database or data platform (SQL Server, Azure SQL, PostgreSQL, MySQL, Oracle, Snowflake, Databricks, BigQuery, Fabric, or an MCP data connection): completeness (fill rate), validity against formats, ranges and dates, consistency of values within a table, duplicates and their kinds (exact, same key, same business key, old versions kept), and a quality score per column and per table. Counts only, never values. Use this whenever the user asks about data quality, invalid or bad data, missing values, completeness, duplicates, duplicate customers, inconsistent values, dirty data, quality scores, or "is this data any good", even if they do not say "quality". It judges; describing what the data looks like is done by data-patterns-eda.'
argument-hint: 'Optional: schemas, tables or columns to include or exclude; business rules or valid code lists; business keys to test for duplicates; thresholds for Good / Fair / Poor; dummy date values used by the source'
user-invocable: true
---

# EDA Stage 5: Data Quality

This skill **judges** the data. It answers "is it right?", not "what does it look like?". Describing formats, ranges and contents is the job of `data-patterns-eda`; this skill uses those results and decides what is valid, what is missing, and what is duplicated.

Example of the difference:
- Pattern (other skill): "82% of phone numbers look like `(999) 999-9999`."
- Quality (this skill): "11% of phone numbers are invalid."

Works on any database. Use whatever the connected platform supports. If something cannot be calculated on this platform, say **Not available** and why. Never guess a number.

## What this skill does and does not do

Does:
1. Completeness: how much of each column is filled in
2. Validity: how many values break a format, range, date or code-list rule
3. Consistency: the same thing written in different ways
4. Duplicates: how many rows repeat, and what kind of repeat
5. Quality scores per column and per table

Does not do (other skills handle these):
- Formats, ranges, domains, date coverage as description → data-patterns-eda
- Blank and distinct counts as raw statistics → column-profiling-eda (reuse its results)
- Category counts → categorical-profiling-eda
- Missing parent or child links (orphans) → relationships-eda; only quote its orphan counts here
- Personal and health data review → phi-review-eda
- Fixing data, writing cleanup scripts or choosing match-and-merge rules → out of scope
- Report layout → eda-output-format

## Safety rules

- **Read-only.** Only run counts and aggregates. Never change anything. Never pull individual rows.
- **Counts only, never values.** Report how many rows are invalid or duplicated, not which values. No example bad values, no example duplicate rows, no samples.
- **Sensitive columns** (flagged by the personal-data review, or clearly personal like date of birth, SSN, card or account numbers): completeness and duplicate counts only. No validity detail that would reveal a value, such as "the invalid values are 0000". If no personal-data review exists, judge it yourself, say your check is basic, and when unsure, treat the column as sensitive.
- **Hide tiny groups.** Any breakdown group with fewer than **11** rows (changeable) is not shown on its own; add it to one line: `Small groups hidden (N)`. A duplicate group of size 2 is reported only inside the totals, never as its own row about a person.
- **Rules come from evidence.** A validity rule must be backed by a declared type or constraint, a pattern result, a user-supplied rule or code list, or an obvious logical limit (a quantity below zero, a date in the future for something that already happened). State the rule used for every validity result. If a rule is a judgement call, mark it `Assumed`.
- **A finding is not a verdict on the business.** Say "does not match the expected format", not "wrong". Real data can break a rule for a legitimate reason.

## Step 0: Before you start

Reuse earlier results if available: the table inventory and keys, column profiling (null and distinct counts, key roles), data-pattern results (dominant formats, ranges, dummy dates, name/content mismatches), the sensitive-column list, and orphan counts from relationships. If any are missing, collect the basics and say so.

Write a short header: database, what was included, run time (UTC), whether the personal-data review and data-pattern results were used, tiny-group limit, thresholds for Good / Fair / Poor, and anything this platform could not calculate.

Cost: aggregates over large tables can be expensive. Run all checks for a table in one pass where the platform allows. Profile tables before views, mark views that time out as `Unavailable – timeout/cost`, and continue.

## Step 1: Completeness

For every column:
- **Fill rate** = rows with a real value ÷ total rows.
- Treat as **missing** both true nulls and empty or whitespace-only text. Also count **placeholder values** that stand for missing (for example `N/A`, `NULL`, `unknown`, `0` in a text field, `1900-01-01`), but only placeholders you can show are used as placeholders: for example a value that makes up an unusually large share of a column that otherwise has varied values, or one that data-patterns flagged. Report placeholder counts separately from true nulls and say how you identified them.
- Flag **required** columns with any missing value: declared not-null columns cannot be missing, so a placeholder there is a finding; business-critical columns (keys, amounts, dates, names) deserve a note.
- Rate each column: `Good`, `Fair`, `Poor` by the thresholds in the header (default: Good ≥ 98%, Fair 90–98%, Poor < 90%).

## Step 2: Validity

For each column that has a rule you can justify, count the rows that break it:

| Rule type | Examples | Evidence needed |
|---|---|---|
| Format | Phone, email, ZIP, ID codes | The dominant format from data-patterns, or a user-supplied pattern |
| Range | Quantity, price, age, percentage; negative or zero where that makes no sense | Column meaning plus a logical limit |
| Date | Future dates for past events, end before start, dummy dates, impossible dates stored as text | Column meaning, paired date columns, dummy values from data-patterns |
| Code list | Status, country, unit of measure, flag columns | A user-supplied list, or a small reference table in the database |
| Type | Numbers or dates stored as text that do not parse | Declared type versus content check |
| Cross-column | Total ≠ quantity × price, discount larger than price, subtotal + tax + freight ≠ total | Column names and meaning, only when the relationship is clear |

For every result give: the rule, whether it is `Declared`, `Pattern-based`, `User-supplied` or `Assumed`, invalid row count, invalid %, and the rating. Do not check a rule you cannot state.

## Step 3: Consistency

Look for the same thing written more than one way, using counts only:
- Case or spacing variants of the same category (`TX`, `tx`, ` TX`).
- Different spellings or abbreviations for the same category, found from the category lists of categorical profiling and grouping on a normalized form (trim, lower case).
- The same code with more than one label, or the same label with more than one code.
- Mixed formats in one column, from data-patterns.
- The same value stored in different types or units across tables, when links are confirmed.

Report how many distinct raw values collapse to one normalized value, and how many rows are affected. Do not list the values themselves for sensitive columns.

## Step 4: Duplicates

Test duplicates at the grain of the table (from table profiling) and on the business keys the user names or the evidence suggests (for example name + date of birth, email, product code + size). Classify each duplicate group as exactly one **duplicate kind**:

| Duplicate kind | What it means |
|---|---|
| `Exact` | Every column is identical (ignoring system columns such as row id, timestamps and row GUIDs) |
| `Same key` | The declared or candidate key repeats, but other columns differ |
| `Same business key` | A business key repeats under different surrogate keys, and other columns may differ |
| `Old versions kept` | The key repeats because history is stored: valid-from / valid-to dates, current-row flags, or version numbers explain the repeats |

For each key tested report: key columns, groups with more than one row, rows involved, **excess rows** (rows beyond the first in each group), % of table, duplicate kind, and whether the key is enforced. Hide groups under the tiny-group limit; they stay inside the totals. Do not decide which row survives; that belongs to the data-model skill's MDM hints.

Use column-profiling results to skip keys already proven unique, and say so.

## Step 5: Quality score

Score each column and each table on the same transparent scale. Never invent a score without showing its parts.

- **Column score** = the average of the dimensions that apply to the column: completeness %, validity %, consistency % (rows not affected by variants). State which dimensions were included.
- **Table score** = the average of its column scores, weighted equally unless the user names key columns. Duplicate excess rows are shown beside the score and not blended into it.
- Rating: `Good`, `Fair`, `Poor` by the header thresholds.
- Show how many columns were scored, how many had no testable rule, and which were excluded (sensitive, cost).

## What to return

- **Completeness**: Schema · Table · Column · Rows · Missing (null) · Missing (empty) · Placeholder · Fill rate % · Required · Rating · Method
- **Validity**: Schema · Table · Column · Rule · Rule basis · Rows tested · Invalid rows · Invalid % · Rating · Method
- **Consistency**: Schema · Table · Column · Distinct raw values · Distinct normalized values · Variant groups · Rows affected · Rows affected % · Method
- **Duplicates**: Schema · Table · Key columns · Key basis · Key enforced · Duplicate kind · Groups · Rows involved · Excess rows · % of table · Small groups hidden
- **Scores**: Schema · Table · Column · Completeness % · Validity % · Consistency % · Column score · Rating, plus a table-level row with Table score · Columns scored · Columns unscored · Excess duplicate rows

Use `N/A` when a field does not apply and `Not available` (with the reason) when it could not be calculated. Never leave a blank that could be read as zero. Add the header and a list of anything that could not be checked. The output-format skill decides layout and file type.

## Summary for the user

1. Header: what was checked, rules used, and anything that could not be checked.
2. Columns with the lowest fill rates and required columns with missing values.
3. Columns with the highest invalid counts, with the rule and its basis.
4. Consistency problems and the rows they affect.
5. Duplicates by kind, with excess rows.
6. Tables with the lowest quality scores.
7. Handoffs: orphan counts to relationships, match-key and survivorship hints to data-model, sensitive findings to phi-review.

Judge, with the evidence shown. "11% of phone numbers do not match the expected format, based on the dominant pattern" belongs here. "82% use one format" belongs to data-patterns.
