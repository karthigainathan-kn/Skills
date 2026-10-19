---
name: phi-review-eda
description: 'EDA suite personal-data stage. Read-only. Finds personal and health data (PHI/PII) in any connected database or data platform (SQL Server, Azure SQL, PostgreSQL, MySQL, Oracle, Snowflake, Databricks, BigQuery, Fabric, or an MCP data connection), checks how easily people could be re-identified, and produces a pseudonymization plan plus review-ready scripts for each sensitive column, covering HIPAA Safe Harbor and GDPR. Never changes data and never runs the scripts. Use this whenever the user asks about PHI, PII, HIPAA, GDPR, sensitive or personal data, identifiers, re-identification risk, masking, tokenization, hashing, pseudonymization, de-identification, or making data safe to share or use for analytics or AI, even if they do not say "PHI".'
argument-hint: 'Optional: schemas, tables or columns; which standard (HIPAA, GDPR, both); confirmed sensitive columns; columns used to combine for the re-identification check; target platform for scripts'
user-invocable: true
---

# EDA: Personal Data Review and Pseudonymization Plan

Find the personal and health data, measure how easy it would be to identify people, and recommend how to pseudonymize each sensitive column, with scripts the team can review and run.

**This skill never changes data and never runs the scripts it writes.** It reads structure and counts, and it writes a plan.

Works on any database. Use what the platform supports. If something cannot be checked, say **Not available** and why.

## Three statements to make in every report

1. **This is a technical risk screen, not a HIPAA audit, a GDPR assessment, or legal advice.** Compliance also depends on access controls, encryption, retention, audit logs, consent, contracts and processes this skill cannot see.
2. **Pseudonymized is not de-identified.** If anyone holds the key, token table, or date-shift values, the data can be linked back to people. Under HIPAA it is still PHI unless Safe Harbor or Expert Determination is met; under GDPR it is still personal data. Never describe pseudonymized data as anonymous.
3. **Findings are based on names, types, descriptions, and counts.** A column can hold personal data without looking like it.

## What this skill does and does not do

Does:
1. Find sensitive columns and rate confidence
2. Check re-identification risk (counts only)
3. Recommend a pseudonymization method per column
4. Check that table links still work after pseudonymization
5. Write scripts for review (not run)
6. Pass the confirmed sensitive-column list to the other EDA skills

Does not do: change data, run scripts, scrub free text, judge legal compliance, or decide report layout (output-format skill; if none, show plain tables in chat).

## Safety rules

- **Read-only.** Only look at structure and run counts. Never change anything.
- **No real values in the output.** Not even samples to "confirm" a column is sensitive. Use names, types, descriptions, content-check counts from data-patterns, and aggregate counts.
- Do not read credentials, password hashes, card numbers, or keys at all. Report that they exist, from structure only.
- **Never put secret keys, salts, or date-shift values in scripts or reports.** Scripts refer to them by name from a secure store (key vault, secret manager) that the team configures.

## Step 0: Before you start

Reuse earlier results if available: table list and row counts, column types, category results, and the data-patterns content check (email, phone, SSN-like, card-like counts and name/content mismatches). If missing, collect the basics and say so.

**Which standard applies:**
- **HIPAA Safe Harbor**: US health data.
- **GDPR**: people in the EU/UK, or any client the user says GDPR applies to.
- **Both** when unsure. Use the user's answer if given; otherwise suggest one from evidence (e.g. country columns, health tables) and say it is a suggestion.

Header: database, what was included, run time (UTC), standard(s) applied, columns used for the re-identification check, target platform for scripts, and anything that could not be checked.

## Step 1: Find sensitive columns

Look at every column. Reason about what it means, using its name, type, description, table, and the content check. Don't rely on a fixed list of names.

**Groups**

| Group | Meaning | Examples |
|---|---|---|
| **Direct identifier** | Points to one person on its own | Name, SSN, medical record number, member/policy ID, account number, email, phone, fax, full street address, licence or certificate number, vehicle ID or plate, device ID or serial, web address, IP address, fingerprint/voice/face data, photo, any other unique ID or code |
| **Indirect identifier** | Can identify people when combined | Date of birth, age, ZIP/postal code, city, county, gender, admission/discharge/service/death dates, ethnicity, occupation, employer |
| **Health information** | About a person's health or care | Diagnosis, procedure, medication, lab result, allergy, provider, claim, treatment, disability |
| **GDPR special data** | Extra-protected under GDPR | Health, genetic, biometric, racial or ethnic origin, religion, political opinion, trade union membership, sex life or sexual orientation |
| **Other sensitive** | Not personal health data, still high risk | Passwords and password hashes, card numbers, bank details, API keys, tokens |
| **Free text** | Notes, comments, descriptions | Flag as **high risk**: names, dates and diagnoses hide in sentences. Not scrubbed by this skill |
| **Not sensitive** | None of the above | |

For HIPAA, map each identifier to the matching item of the **18 Safe Harbor identifiers** and show which ones were found and which were not.

**Confidence**

| Confidence | Meaning |
|---|---|
| **Confirmed** | A description, codebook, or the user says so |
| **Likely** | Name and content agree (e.g. column `Email` and 97% of values look like emails) |
| **Possible** | Name only, or content only |

Also flag hidden cases: content says personal but the name doesn't (e.g. `Field7` holding phone numbers), and IDs that are personal even though they look technical (e.g. patient GUIDs).

## Step 2: Re-identification risk (counts only)

For each table, combine the indirect identifiers it holds. **Default combination: ZIP/postal code, date of birth or age, gender, and city** (use whichever exist; the user can change the list). ZIP + birth date + gender alone identifies most people in the US.

Report, using grouped counts only:
- number of rows
- number of different combinations
- **smallest group size** (how few rows share the rarest combination)
- **rows that are unique** (combination appears once) and %
- rows in groups smaller than **11** (changeable) and %

Then run the **same check after the planned changes** (e.g. ZIP → first 3 digits, birth date → birth year, age 90+ grouped), computed inside the query without changing data. Show **before vs after** so the plan's effect is visible.

Risk rating per table:
- **High**: many unique rows, or a direct identifier present
- **Medium**: some small groups remain
- **Low**: no direct identifiers and no small groups

Say that this rating only covers the columns combined, not outside data someone could link in.

## Step 3: Pseudonymization method per column

| Method | What it does | Use for |
|---|---|---|
| **Token** | Replaces the value with a consistent random ID kept in a secure lookup table; same input gives the same token in every table | IDs that must still join: MRN, member ID, SSN used as a key, email used as a key |
| **Keyed hash** | One-way scramble with a secret key; same input gives the same output | IDs that must match but never be reversed |
| **Generalize** | Makes values less exact | Birth date → birth year or age band; ZIP → first 3 digits; dates → month/year or year; city → region |
| **Date shift** | Moves all of one person's dates by the same random number of days | Visit, admission, service dates, keeping the gaps between them |
| **Partial mask** | Shows only part of the value | Last 4 digits for display; never for analytics output |
| **Remove** | Drop the column | Names, free text, anything not needed for the purpose |
| **Keep** | No change | Not sensitive, or needed and low risk after other changes |

Rules:
- **Never a plain hash** (unkeyed MD5/SHA). SSNs, phone numbers and dates have so few possible values that a plain hash can be reversed by trying them all. Always use a keyed hash or tokens.
- **Keep links working.** A column used to join tables gets the same token or keyed hash, with the same key, everywhere it appears.
- **One date-shift value per person**, used for all of that person's dates in every table.
- **HIPAA Safe Harbor rules to apply when HIPAA is in scope:**
  - Dates (except year) linked to a person → year only, or date shift (date shift is pseudonymization, not Safe Harbor).
  - Ages over 89 → `90+`, and matching birth years grouped too.
  - ZIP → first 3 digits, and `000` where that 3-digit area has 20,000 people or fewer. This needs census population data; if it isn't available, say the 3-digit ZIP rule could not be fully checked.
- **GDPR rules to apply when GDPR is in scope:** remove or reduce anything not needed for the stated purpose (data minimisation); keep keys and lookup tables separate with restricted access; special data gets the strongest method available.
- **Free text:** recommend **Remove** or **Exclude until scrubbed**. Proper scrubbing needs a text-scanning tool (NLP-based de-identification), not SQL. Say so.
- **Other sensitive** (passwords, cards, keys): **Remove** from any analytics copy. They are not pseudonymized; they should not be there.

For each column record: method, settings (e.g. "birth year", "first 3 digits"), whether joins are kept, what is lost for analysis (e.g. "exact age no longer available"), and the reason.

## Step 4: Check table links

- List every column that links tables (declared links, or confirmed by the relationships skill) and is being pseudonymized.
- Confirm both sides use the same method and key.
- Flag links that would break (e.g. one side tokenized, the other removed) and links using a personal value as the key (e.g. email as a join key).

## Step 5: Write scripts for review

Write scripts in the **dialect of the connected platform** (or the target the user names), to build a pseudonymized **copy** (views or new tables in a separate schema). Never write scripts that change the source tables.

Script rules:
- Header comment on every script: what it does, the source and target, "Generated for review – do not run without approval", and the three statements from the top of this skill.
- Secrets (keys, salts, date-shift seed) come from a named secure store, never written in the script.
- Token lookup tables and date-shift tables go in a separate, restricted schema, with a comment saying who should have access.
- One section per table, one line per column, with a comment naming the method and reason.
- Columns marked Remove are left out of the copy, with a comment listing them.
- If the platform lacks a needed function (e.g. keyed hashing), say so in the script and suggest the closest safe option. Never fall back to a plain hash.
- Save scripts as files. **Do not run them.** If the agent has a way to run SQL, it still must not run these.

## Step 6: Pass results on

Give the other EDA skills a list of sensitive columns with group and confidence. Categorical, patterns and quality use it instead of their own basic checks: counts only for these columns, no values, no ranges.

## What to return

- **Sensitive columns**: Schema · Table · Column · Data type · Group · HIPAA identifier # (if any) · GDPR special data (yes/no) · Confidence · Evidence · Method · Settings · Keeps joins · What is lost · Reason
- **Safe Harbor checklist**: the 18 identifiers · Found (yes/no) · Where · Planned method
- **Re-identification risk**: Schema · Table · Columns combined · Rows · Combinations · Smallest group · Unique rows (and %) · Rows in small groups (and %) · Same after plan · Risk before · Risk after
- **Link check**: From table and column · To table and column · Method on each side · Result (kept / broken / personal value used as key)
- **Scripts**: file names, target platform, what each covers
- Header and list of anything that could not be checked

## Summary for the user

1. The three statements.
2. Standard(s) applied and scope.
3. Direct identifiers found, by table, with confidence.
4. High-risk tables and their before/after re-identification results.
5. Free text and other sensitive data that should be removed.
6. Pseudonymization plan highlights and what each method costs the analysis.
7. Broken or risky table links.
8. Scripts written, where they are, and that they were **not run**.
9. Hidden cases (personal data in oddly named columns) and limits.

Never say a table is "compliant", "anonymous", or "de-identified". Say what was found, what is planned, and what is still needed.
