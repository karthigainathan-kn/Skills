---
name: categorical-profiling-eda
description: 'Stage 3 of the modular EDA suite: read-only frequency profiling of categorical fields in any connected database or data platform (SQL Server, Azure SQL, PostgreSQL, MySQL, Oracle, Snowflake, Databricks, BigQuery, Fabric, or an MCP data connection). Identifies categorical columns from evidence, classifies each as Nominal, Ordinal, Binary, or High-cardinality, and reports the row count and percentage for every category of every categorical column. Use this whenever the user asks for category counts, value frequencies, distinct values, breakdowns by country, state, city, region, product type, brand, gender, status, segment, channel, tier, rating, flag, or any other categorical or code field, even if they do not say "categorical".'
argument-hint: 'Optional: schema(s)/objects to include or exclude, columns to force in or out, codebooks or business definitions, small-cell suppression on/off and threshold, max categories per column'
user-invocable: true
---

# EDA Stage 3: Categorical Profiling

Find the categorical columns in every table and view in scope, decide what kind of categorical each one is, and report how rows are distributed across its categories. Nothing about which columns are categorical is hard-coded: Country, State, City, Brand, and Gender are examples of what you should recognise, not a list to look for. Decide from evidence while reading the metadata and the column-profiling results.

This skill is platform-agnostic. Use whatever the connected platform exposes.

## Scope of this skill

This skill owns: identifying categorical columns, classifying them, and per-category row counts and percentages.

It does not produce:
- Numeric and date distributions, ranges, outliers, and cross-tabs or hierarchy breakdowns (e.g. State within Country) → data-patterns skill
- Judgements that variants such as `TX` / `Tx` / `Texas` are defects → data-quality skill (flag them here as a handoff)
- PHI/PII classification → phi-review skill
- Null, distinct, and length statistics for all columns → column-profiling skill (reuse its results)

It does not decide the output format. If an output-format skill is available, follow it. Otherwise return plain tables in chat and state that formatting is pending.

## Safety: this skill returns values

Unlike column profiling, category values appear in the output. "Texas: 41,208" is a value. Apply these rules every time:

- Read-only. Discovery, catalog, and aggregate (`GROUP BY` / count) operations only. Never read row-level records.
- Run after `phi-review-eda` when its results exist, and never return values from columns it marks high-risk. If no PHI review exists, screen columns yourself from names, types, and descriptions, state that this screen is weaker, and when in doubt treat the column as sensitive.
- Never return values from identifiers, free-text columns, credentials, payment data, or direct personal identifiers. Report them as counts only.
- **Every category gets its own row count by default.** Single-column category counts are aggregates, and this skill does not cross-tabulate columns, which is where most re-identification risk comes from.
- **Small-cell suppression is targeted, not blanket.** Apply it only to columns that the PHI review (or your own screen) flags as sensitive or as quasi-identifiers in a sensitive context (e.g. ZIP code or birth year in health data), or to any column when the user turns suppression on. Default threshold when applied: fewer than **11** rows, configurable. Suppressed categories are folded into one `Suppressed small groups (N categories)` row and the column summary says suppression was applied and why. Record the suppression setting and the affected columns in the run header.
- Classification reasoning that needs to look at labels (e.g. deciding a column is ordinal) uses the aggregated category list only, never row samples, and never for columns screened as sensitive; those are classified from metadata alone.

## Step 0: Inputs and run header

Reuse, when available in this run or supplied: the table-profiling inventory (objects, Rows Count, keys) and the column-profiling results (data type, distinct count, cardinality class, null count, average length, key role). If column profiling is missing, gather the minimum (type, distinct count, null count, average length for strings) and say so.

Record in the run header: platform, database/catalog, visibility, scope, run timestamp (UTC), suppression setting and threshold, columns suppressed, max-categories limit, high-cardinality threshold, PHI review status (used / not available / self-screened), and any codebooks or business definitions supplied.

## Step 1: Identify candidate columns

Consider every column. A column is a categorical candidate when the evidence agrees; no single signal is enough.

Signals in favour:
- The user named it, or a codebook/business definition describes it as a set of categories.
- Type is string, boolean/bit, or a small integer or short code (e.g. `char(1)`, `char(2)`, `tinyint`).
- Column profiling class is `Constant`, `Low`, or a moderate `High` count of distinct values that is small relative to the row count.
- Short average string length (codes and labels, not sentences).
- The name and description suggest a category: things like a location (country, state, region, city, postal area), classification (type, category, class, segment, group, department), status or flag, channel or method, tier or level, rating or score band, brand or make, gender, unit of measure, currency, or language. Use these as recognition cues and reason about what the column means; do not treat them as a list to match.

Signals against:
- Declared PK, unique constraint, or `Near-unique` / `Unique (observed)` cardinality → identifier, not a category.
- Long or highly variable string length → free text.
- Continuous numeric (decimal amounts, measures), dates and timestamps, LOB, binary, semi-structured, spatial.
- Numeric codes where the evidence cannot tell a category from a quantity: mark `Unconfirmed` rather than guess.

Columns the user forces in or out override inference; record that.

## Step 2: Classify each categorical column

Assign exactly one **Categorical type**, with a **Type basis** and short **Evidence**.

| Categorical type | What it means | How to recognise it | Examples |
|---|---|---|---|
| `Binary` | Exactly two possible categories | Boolean/bit types, or exactly 2 distinct non-null values. If only 2 values are observed but the domain could hold more, use `Binary (observed)` | Active/Inactive, Yes/No, Fraud/Not fraud, 0/1 flags |
| `Ordinal` | Categories with a meaningful order | The labels themselves carry an order you can explain: levels, sizes, tiers, ranks, rating scales, bands, ordered stages | Low/Medium/High, Bronze/Silver/Gold, satisfaction 1–5, S/M/L/XL, age bands |
| `Nominal` | Categories without a natural order | A moderate set of labels with no inherent order | Country, product category, department, payment method, brand |
| `High-cardinality` | Many distinct categories | Distinct categories above the high-cardinality threshold (default **250**, configurable), but still a grouping rather than one value per entity | City, postal/ZIP code, product SKU, store |

Split `High-cardinality` into two sub-types, because they are handled differently:
- `High-cardinality – analytical`: a real grouping many rows share (City, ZIP, SKU, store). Profile every category, like any other categorical column.
- `High-cardinality – identifier-like`: values that act as identifiers (Customer ID, order number, account number), even when not declared as keys. Do not profile categories; report distinct count only and hand it to column profiling and PHI review.

Type basis:
- `Confirmed`: a codebook, business definition, declared domain (e.g. check constraint or enum type), or the user says so.
- `Inferred`: decided from type, cardinality, length, names, and labels.
- `Unconfirmed`: evidence conflicts or is too thin (e.g. integer codes 1–5 that could be ratings or arbitrary codes). Profile it as `Nominal` for display and say why.

Ordinal rules:
- Infer the order from the meaning of the labels, and state it, e.g. `Order (inferred): Low < Medium < High`. Never order alphabetically and call it ordinal.
- If labels are codes whose order is not self-evident, the column stays `Nominal` / `Unconfirmed` unless a codebook supplies the order.
- Do not assume the observed categories are the complete scale. A missing level (e.g. no "Medium") is a handoff to data quality, not a conclusion.

Code labels: when values are codes (e.g. `M`/`F`/`U`, `1`/`2`), add a **Readable label** only if a codebook, declared domain, or the user confirms it. Otherwise write the likely label marked `(unconfirmed)`, or leave it as `Unconfirmed`. `M` is not always Male.

## Step 3: Count categories

For each categorical column (except identifier-like), with grouped count aggregates:

- Total rows, non-null count, null count, number of distinct categories.
- Row count per category, % of non-null, % of total, and rank by frequency.
- Show nulls as a separate `(Null)` row so percentages of total reconcile to 100%.
- Count the categories exactly as stored. Do not trim, upper-case, or merge variants; if near-duplicates such as case or spacing variants appear, record them as a data-quality handoff.

Display by type. Every categorical type except identifier-like returns **one row per category with its own row count**. There is no `Other categories` roll-up in the results.

| Type | Display |
|---|---|
| `Binary` | Both categories plus `(Null)`. Add **minority share** (smaller category ÷ non-null) and flag `Imbalanced` when below 5% (configurable), e.g. fraud flags |
| `Ordinal` | Every category **in the inferred or confirmed order**, not frequency order, with cumulative % along that order. Also show frequency rank |
| `Nominal` | Every category in frequency order |
| `High-cardinality – analytical` | Every category in frequency order, with frequency rank, cumulative %, and the share covered by the top 20 (useful for spotting long tails) |
| `High-cardinality – identifier-like` | No categories shown. Distinct count only; one row per value would just be a list of identifiers |

Very large category lists:
- Above the max-categories limit (default **10,000** distinct categories per column, configurable), ask the user before returning the full list, because the result may be slow to produce and large to deliver. Record the decision.
- If the platform or MCP layer truncates grouped results, say so, report how many categories were returned versus the distinct count, and never present a truncated list as complete.
- The chat summary may show only the top categories of long lists for readability; the full per-category results still go to the output. Say so in the summary.

Where suppression applies to a column, apply it after ranking. Suppressed categories never appear individually.

Cost: group-by on very large or high-cardinality columns can be expensive. Batch columns per table where the platform allows, profile tables before views, mark views that time out as `Unavailable – timeout/cost`, and ask before grouping large external or lake-format objects.

## Result fields

**Column summary**, one row per categorical candidate (including excluded ones, with the reason):

Schema · Object · Object type · Column · Data type · Categorical type · Type basis · Evidence · Order (ordinal only) · Rows Count · Non-null count · Null count · Null % · Distinct categories · Display mode (`All categories` / `All categories, small groups suppressed` / `Counts only` / `Excluded`) · Top-1 category share · Top-20 coverage % (high-cardinality) · Minority share (binary) · Suppression applied (yes/no, reason) · Suppressed groups · Notes

**Category frequencies**, one row per displayed category:

Schema · Object · Column · Categorical type · Order position (ordinal) · Frequency rank · Category value · Readable label · Row count · % of non-null · % of total · Cumulative % · Notes

Plus the run header and a limitations list. The output-format skill decides layout and file type.

## Summary for the user

Cover, in this order:
1. Run header: scope, thresholds, PHI review status, suppression setting.
2. Categorical columns found, counted by type (Nominal, Ordinal, Binary, High-cardinality analytical, identifier-like), with any `Unconfirmed` classifications listed.
3. Columns excluded and why (identifiers, free text, sensitive, cost).
4. Dominant categories: one category above 95% of non-null rows (near-constant columns).
5. Imbalanced binary columns.
6. Ordinal columns with their order and type basis.
7. High-cardinality analytical columns with distinct counts and top-20 coverage (long-tail indicator).
8. Columns where suppression was applied, with the reason and number of suppressed groups.
9. Handoffs: suspected label variants (data quality), possible hierarchy columns such as Country/State/City (data patterns), sensitive-looking columns (PHI review).
10. Limitations and blocked checks.

Never present an inferred classification, an inferred order, or an unconfirmed code label in the same confident voice as one confirmed by a codebook or the user.
