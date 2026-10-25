---
name: data-model-eda
description: 'EDA suite design stage. Read-only. Uses profiling, quality and relationship results to assess what can be built from a database or data platform: star schema / OLAP readiness (fact and dimension candidates, grain, measures, hierarchies, shared dimensions, date needs, history tracking) and MDM readiness (master entities such as customer, patient, product, provider, location; the same entity across sources; match keys; best-source attributes; reference data; blockers). Use this whenever the user asks about data modelling, dimensional modelling, star schema, facts and dimensions, OLAP, semantic models, warehouse or lakehouse design, MDM, master data, golden records, entity resolution readiness, or "what can we build from this data", even if they do not say "data model".'
argument-hint: 'Optional: schemas or tables; business processes to model; master entities of interest; known source systems'
user-invocable: true
---

# EDA: Data Model Readiness (OLAP and MDM)

Assess what can be built from this data: a star schema for analytics (OLAP), and master data management (MDM). Everything here is a **suggestion with evidence**, built on facts from the other EDA skills. It is design input, not a finished design.

Default modelling approach: **Kimball star schema** (facts and dimensions).

## What this skill does and does not do

Does:
1. Star schema readiness: facts, dimensions, grain, measures, hierarchies, shared dimensions, dates, history
2. MDM readiness: master entities, same entity across sources, match keys, best-source attributes, reference data
3. Blockers and gaps for both

Does not do: build models, write DDL, design match-and-merge rules, or choose an MDM tool. Report layout → output-format skill (if none, plain tables in chat).

## Safety rules

- **Read-only, counts only, no values.**
- Respect the personal-data review: sensitive columns are named but never profiled for values. Flag where a match key or dimension attribute is personal data and point to the pseudonymization plan.

## Step 0: Inputs

This skill depends on earlier results. Use them; don't recompute:
- table profiling: row counts, row grain
- column profiling: types, distinct counts, key checks
- categorical profiling: category columns
- data patterns: hierarchies, date coverage, content check
- data quality: fill rates, validity, duplicates and duplicate kinds
- relationships: links, link types, table roles
- personal-data review: sensitive columns

If a needed result is missing, say which one and what can't be assessed without it. Run only light extra counts when needed.

Header: database, scope, run time (UTC), which earlier results were used, and what was missing.

Label every finding **Suggested** (from evidence) or **Confirmed** (the user or documentation says so).

## Part A: Star schema (OLAP) readiness

### A1. Fact or dimension

Classify each table, with evidence:

| Class | Typical evidence |
|---|---|
| **Fact candidate** | Many rows, child-only or middle role, several links out, number columns that can be added up, one or more date columns, grows over time |
| **Dimension candidate** | Fewer rows, parent role, descriptive text and category columns, referenced by fact candidates |
| **Bridge** | Mostly link columns, connects two tables many-to-many |
| **Already modelled** | Names and structure already follow a star (e.g. `Fact_`, `Dim_` prefixes, surrogate keys, a date table). Assess it as-is rather than redesigning it |
| **Unclear** | Mixed signals; say what is mixed |

### A2. Grain of each fact candidate

From table profiling, say what one row represents (e.g. "one row per order line"). Flag:
- **Mixed grain**: e.g. order-header amounts (freight, tax, total due) repeated on every order-line row. Summing them double-counts. Check by counting whether those columns repeat for every line of the same order.
- **Unclear grain**: no declared or proven unique key.

### A3. Measures

For number columns in fact candidates:

| Kind | Meaning | Examples |
|---|---|---|
| **Additive** | Can be summed across everything | Quantity, line amount |
| **Semi-additive** | Can be summed except over time | Balances, stock on hand |
| **Non-additive** | Should never be summed | Unit price, rates, percentages, ratios |
| **Header-level** | Belongs to a higher grain than the row | Order freight on a line table |

Number columns that are really codes or IDs are not measures.

### A4. Dimensions

For each dimension candidate:
- **Attributes**: descriptive and category columns, with fill rate from data quality.
- **Hierarchies**: from data patterns (e.g. Country > State > City, Category > Subcategory > Product), including whether each child belongs to only one parent.
- **Shared dimensions**: dimensions linked to more than one fact candidate (e.g. Customer used by sales and returns). These should be built once and shared.
- **Duplicate or overlapping dimensions**: two tables describing the same thing (e.g. customer details copied into a sales-person table). Flag for consolidation.
- **Key**: surrogate key (meaningless number) or business key (natural ID), and whether it is unique.
- **Junk dimension candidates**: several small yes/no or status columns on a fact.
- **Degenerate dimensions**: IDs left on the fact with no table of their own (e.g. order number). Normal; just list them.

### A5. Dates

- Every date or date-key column on fact candidates, and which date table it links to.
- **Role-playing dates**: one date table used several times (order, ship, due date).
- Whether a date table exists and covers the full range of fact dates (from data patterns).
- Dates stored as text or as numbers like `20240131` that need converting.

### A6. History (slowly changing dimensions)

- Does history exist? Look for valid-from / valid-to columns, current-row flags, version numbers, or the **Old versions kept** duplicate kind from data quality.
- Which attributes change over time (counts of distinct values per business key, where affordable).
- Suggest: **keep only latest** (Type 1) or **keep history** (Type 2) per dimension, with the reason. Final choice belongs to the business.

### A7. Star schema blockers

List anything that would stop or complicate the model: missing links, orphan rows, duplicate dimension keys, mixed grain, measures stored as text, low fill on important attributes, dates without a date table, personal data in dimension attributes.

### A8. Readiness rating

Per business area or fact candidate: **Ready**, **Ready with fixes** (list them), or **Not ready** (list blockers). Always say it is a suggestion.

## Part B: MDM readiness

### B1. Master entity candidates

Find tables that describe core business things, reasoning from content rather than a fixed list. Typical: customer, patient, member, product, item, provider, supplier, vendor, location, site, employee, account.

### B2. Same entity in several places

- Where does each entity appear (e.g. customer in CRM, billing and order tables, or customer details copied into another table)?
- For each place: row count, which attributes it holds, and fill rates.
- Show attribute overlap: which attributes appear in which sources.

### B3. Match keys

For each entity, rate columns that could identify the same real-world thing across sources:

| Rating | Meaning |
|---|---|
| **Strong** | Unique, well filled, valid (e.g. tax ID, email with high validity) |
| **Usable with cleanup** | Good after tidying (e.g. phone digits only, lower-cased email) |
| **Combination needed** | Only works combined (e.g. name + ZIP + birth date) |
| **Weak** | Low fill, low validity, or many duplicates |

Use fill rate, validity and duplicate results from data quality. Flag match keys that are personal data and point to the pseudonymization plan (a tokenized match key must use the same token in every source).

### B4. Best source per attribute

For each important attribute (name, address, phone, email, category, price), show which source has the best fill rate and validity. This gives **hints** for choosing which source wins (survivorship). It is not a rule set.

### B5. Reference data candidates

Small shared code lists that should be managed in one place: status codes, country, state, product type, unit of measure, payment method. Flag where the same list has different values in different tables (from data quality spelling variants).

### B6. MDM blockers

No reliable match key; conflicting duplicates; low fill on key attributes; the same entity with different IDs and no cross-reference; personal data on match keys without a pseudonymization plan.

### B7. Readiness rating

Per master entity: **Ready**, **Ready with fixes**, or **Not ready**, with reasons. Always a suggestion.

## What to return

- **Table classes**: Schema · Table · Class · Grain · Evidence · Suggested/Confirmed
- **Measures**: Schema · Table · Column · Kind · Evidence
- **Dimensions**: Schema · Table · Key type · Key unique · Attributes · Hierarchies · Shared by facts · Overlaps with · History found · History suggestion
- **Dates**: Fact table · Date column · Linked date table · Role · Date table covers range
- **Star blockers and readiness**: Area · Rating · Blockers · Fixes
- **Master entities**: Entity · Sources · Rows per source · Attribute overlap · Match keys and ratings · Best source per attribute · Reference data · Blockers · Rating
- Header and list of inputs that were missing

## Summary for the user

1. Header: scope and which earlier results were used or missing.
2. Fact and dimension candidates, with grain.
3. Mixed-grain and header-level measure issues.
4. Shared dimensions, overlaps, hierarchies, role-playing dates.
5. History found and suggestions.
6. Star schema readiness per area.
7. Master entities, where they live, and their match keys.
8. Reference data candidates.
9. MDM readiness per entity.
10. Blockers for both, and limits.

Everything here is a design suggestion backed by evidence. Never present it as an agreed model.
