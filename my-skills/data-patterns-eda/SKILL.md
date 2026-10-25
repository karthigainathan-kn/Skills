---
name: data-patterns-eda
description: 'Stage 4 of the EDA suite. Read-only. Describes what the data looks like in any connected database or data platform (SQL Server, Azure SQL, PostgreSQL, MySQL, Oracle, Snowflake, Databricks, BigQuery, Fabric, or an MCP data connection): the formats used in phone, email, ZIP and code columns, what each column really contains, email domains and the business vs personal email split, number ranges and averages, date ranges and gaps, and parent-child breakdowns such as Country > State > City. Use this whenever the user asks about formats, patterns, phone or email structure, email domains, B2B vs B2C, ranges, averages, outliers, date coverage, hierarchies, or "what does this data look like", even if they do not say "patterns". It only describes; judging right or wrong is done by data-quality-eda.'
argument-hint: 'Optional: schemas, tables or columns to include or exclude, code lists, tiny-group limit'
user-invocable: true
---

# EDA Stage 4: Data Patterns

This skill **describes** the data. It answers "what does it look like?", not "is it right?". Deciding right or wrong is the job of `data-quality-eda`, which uses this skill's results.

Example of the difference:
- Pattern (this skill): "82% of phone numbers look like `(999) 999-9999`."
- Quality (other skill): "11% of phone numbers are invalid."

Works on any database. Use whatever the connected platform supports. If something cannot be calculated on this platform, say **Not available** and why. Never guess a number.

## What this skill does and does not do

Does:
1. Format check: what formats values are written in
2. Content check: what each column really contains
3. Email domains
4. Number summaries
5. Date summaries
6. Parent-child breakdowns (e.g. State within Country)

Does not do (other skills handle these):
- Right/wrong judgements, invalid counts, scores → data-quality-eda
- Category counts → categorical-profiling-eda
- Blank and distinct counts → column-profiling-eda
- Missing parent/child links (orphans) → relationships skill
- Personal/health data review → phi-review-eda
- Growth over time → volume-growth skill
- Report layout → output-format skill (if none, show plain tables in chat)

## Safety rules

- **Read-only.** Only look at structure and run counts or summaries. Never change anything. Never pull individual rows.
- **No real values in the output.** Show formats, counts and summaries. The only exceptions are email domains and the parent-child labels in Step 6.
- **Sensitive columns** (flagged by the personal-data review, or clearly personal like date of birth or SSN): counts only. No ranges, no averages, no domains, no labels. If no personal-data review exists, judge it yourself, say your check is basic, and when unsure, treat the column as sensitive.
- **Hide tiny groups.** Any format, email domain, or parent-child pair with fewer than **11** rows (changeable) is not shown on its own. Add it to one line: `Small groups hidden (N)`. A rare format or a tiny company domain can point to one person.
- **Everything here is a finding to look at, not a confirmed rule.** Say "observed", not "confirmed". Never claim one thing causes another.

## Step 0: Before you start

Reuse earlier results if available: table list and row counts, column types, blank and distinct counts, category results, and any parent-child columns flagged by categorical profiling. If missing, collect the basics and say so.

Write a short header: database, what was included, run time (UTC), whether the personal-data review was used, tiny-group limit, and anything this platform could not calculate.

## Step 1: Format check

For text columns holding structured values (phone, ZIP, IDs, codes, numbers or dates stored as text), turn each value into a **format shape**:
- each digit becomes `9`
- each capital letter becomes `A`
- each small letter becomes `a`
- spaces and symbols stay as they are

Examples: `(713) 555-0142` → `(999) 999-9999`. `TX-77002` → `AA-99999`.

Do this inside the database query and group by the shape. Never pull values out to do it.

Report for each column:
- how many different formats it has
- row count and % for each format
- the most common format and what % of rows use it
- shortest and longest length per format

Skip long free text; formats mean nothing there. If a column has more than **50** formats, show the most common ones plus one line for the rest. Having that many formats is itself worth reporting.

## Step 2: Content check (what is really in the column)

Ignore the column name and look at what the values actually are. Count how many match each of these:

email, phone, website address, IP address, ZIP/postal code, country code, currency code, unique ID (GUID), date stored as text, number stored as text, yes/no stored as text.

**Possibly sensitive look-alikes**: SSN-like, credit-card-like (also run the card checksum), national-ID-like, bank-account-like. Report **counts only** and pass them to the personal-data review.

Report for each column:
- what it mostly contains, and what % matches
- the second most common content, if it matters
- **name doesn't match content**: e.g. a `Phone` column where 30% of values look like emails
- how many sensitive look-alikes were found

Mark these as **observed**, not confirmed.

## Step 3: Email domains

For email columns:
- Take the part after `@`, make it lower case, and count rows per domain.
- Put each domain in a group, using judgement rather than a fixed list:
  - **Personal** (gmail.com, yahoo.com, outlook.com…)
  - **Business** (company domains)
  - **Education**, **Government**
  - **Throwaway** (temporary email sites)
  - **Test/fake** (test.com, example.com…)
  - **Not sure**
- Report the **business vs personal split** (B2B vs B2C), and say it is observed.
- Hide tiny groups.
- Values with no `@` or no domain go into one line: `No domain`. Whether they are invalid is for data quality to say.

## Step 4: Number summaries

For number columns (not IDs, codes, or sensitive columns):
- lowest, highest, average, middle value (median)
- spread points: 1%, 5%, 25%, 75%, 95%, 99%
- how spread out the values are (standard deviation)
- how many zeros and how many negatives, with %
- **unusually high or low values**: count values far outside the normal range (standard rule: more than 1.5 × the middle-50% range below the 25% point or above the 75% point). Call them "unusual", not "wrong".
- decimals that are always whole numbers

Some platforms only give approximate spread points on big tables. Say which kind you used.

## Step 5: Date summaries

For date columns, and dates stored as text (not sensitive ones like date of birth; those get counts only):
- earliest and latest date
- rows per year and per month
- **gaps**: months with no rows in between the first and last date
- **future dates**: how many are after today
- **dummy dates**: how many use placeholder dates like `1900-01-01`, `1753-01-01`, `0001-01-01`, `9999-12-31`, `1970-01-01`
- **midnight only**: % of date-times that are exactly 00:00, meaning only the date is really stored
- whether the column stores a time zone (don't assume one)

Load timing and growth questions go to the volume-growth skill.

## Step 6: Parent-child breakdowns

For column pairs flagged by categorical profiling (e.g. Country > State > City, Category > Product, Region > Territory):
- Count rows for each child within each parent (e.g. rows per State in each Country). Hide tiny groups.
- **Does each child belong to only one parent?** Count child values that appear under more than one parent, and the rows affected. Example: a City listed under two States.
  - Some are genuine (there is a Springfield in many states), so use the most specific columns available.
  - Report it as a finding. Deciding whether it is a problem belongs to data quality.
- Compare numbers across tables only through confirmed table links.

## What to return

- **Formats**: Schema · Table · Column · Format · Row count · % · Shortest · Longest · Rank
- **Content check**: Schema · Table · Column · Mostly contains · % matching · Second content · Name doesn't match content · Sensitive look-alikes · Method
- **Email domains**: Schema · Table · Column · Domain · Domain group · Row count · %
- **Numbers**: Schema · Table · Column · Lowest · 1% · 5% · 25% · Median · Average · 75% · 95% · 99% · Highest · Std dev · Zero % · Negative % · Unusual values · Exact or approximate
- **Dates**: Schema · Table · Column · Earliest · Latest · Future dates · Dummy dates · Gap months · Midnight % · Has time zone
- **Parent-child**: Schema · Table · Parent column · Child column · Parent value · Child value · Row count · Children with more than one parent

Use `N/A` when a field does not apply and `Not available` (with the reason) when it could not be calculated. Never leave a blank that could be read as zero. Add the header and a list of anything that could not be checked.

## Summary for the user

1. Header: what was checked and anything that could not be checked.
2. Columns with too many formats, and the % using the most common one.
3. What columns really contain, name/content mismatches, sensitive look-alikes passed on.
4. Email domain mix and business vs personal split.
5. Number columns with odd spreads, many unusual values, or unexpected negatives/zeros.
6. Date ranges, gaps, future dates, dummy dates.
7. Parent-child breakdowns and children with more than one parent.
8. Items passed to data quality, and limits.

Describe, don't judge. "82% use one format" belongs here. "18% are invalid" does not.
