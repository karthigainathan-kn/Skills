---
name: eda-output-format
description: 'Output-format skill for the EDA suite (table-profiling-eda, column-profiling-eda, categorical-profiling-eda, data-patterns-eda, data-quality-eda, relationships-eda, data-model-eda, phi-review-eda). Defines how every EDA skill delivers its results: one Excel workbook with exactly one sheet per skill, plus an ER diagram (Mermaid source, PNG, and an ER Diagram sheet in the workbook). Use this whenever any EDA skill finishes or the user asks for EDA results, findings, a report, an Excel sheet, a workbook, or an ER diagram, even if they do not name this skill.'
argument-hint: 'Optional: output folder (default reports/eda), database name for file names, which skills to include, whether to skip the PNG'
user-invocable: true
---

# EDA Output Format

The EDA skills decide **what** to report. This skill decides **how it is delivered**: one Excel workbook, one sheet per skill, and an ER diagram. When an EDA skill says "the output-format skill decides layout and file type", this is that skill. Follow it instead of returning plain tables in chat.

This skill never queries the database. It only formats results that the other skills already produced. If a skill has not been run, its sheet still exists and says so (see Missing results).

## Output files

Write everything to `reports/eda/` (create it if missing, or use the folder the user names). Use one run timestamp for every file in the run: `YYYYMMDD-HHMMSS-UTC`. `<db>` is the database or catalog name.

| File | Contents |
|---|---|
| `<db>-eda-findings-<timestamp>.xlsx` | The workbook described below |
| `<db>-er-diagram.mmd` | Mermaid ER source |
| `<db>-er-diagram.png` | Rendered diagram |
| `<db>-er-diagram.dbml` | DBML source, if `relationships-eda` produced it |

Keep one stable name for the diagram files and overwrite them only when the links change. The workbook name is new on every run so earlier runs are never overwritten.

## Workbook layout

Exactly these sheets, in this order. **One sheet per skill. Do not split a skill across several sheets and do not merge two skills into one.**

| # | Sheet name | Source skill |
|---|---|---|
| 1 | `Run Info` | all skills (header and limitations) |
| 2 | `1 Table Profiling` | table-profiling-eda |
| 3 | `2 PHI Review` | phi-review-eda |
| 4 | `3 Column Profiling` | column-profiling-eda |
| 5 | `4 Categorical` | categorical-profiling-eda |
| 6 | `5 Data Patterns` | data-patterns-eda |
| 7 | `6 Data Quality` | data-quality-eda |
| 8 | `7 Relationships` | relationships-eda |
| 9 | `8 Data Model` | data-model-eda |
| 10 | `ER Diagram` | relationships-eda (picture and source) |

Sheet names stay under 31 characters and use no `/ \ ? * [ ]`.

### Run Info sheet

Two blocks, top to bottom:

1. **Run header**: database, platform, server or endpoint (no credentials), access level, scope, run timestamp (UTC), and which skills ran.
2. **Limitations**: one row per limitation, with the skill it came from. Include every `Unavailable`, `Not available`, timeout and approximation noted by any skill.

### Skill sheets (sheets 2 to 9)

Every skill sheet follows the same pattern:

- **Row 1**: sheet title and the skill name.
- **Row 2**: one-line scope note (objects covered, thresholds used).
- **Row 4 onward**: the main result as one flat table with a header row. Use the exact field names and order the source skill defines. Do not rename, drop or reorder them.
- **Further sections on the same sheet**: the skill's secondary outputs (rankings, summaries, key checks, findings) as separate tables stacked below the main table, each with a bold section title and one blank row before it. Never put them on a new sheet.
- Freeze panes under the main table's header row, turn on the header row filter, bold the header with a fill, and size columns to the content with a sensible maximum width.
- Store numbers as numbers (not text), percentages as 0-100 values in a column headed `%`, and sizes in the unit the header names.

What goes on each skill sheet:

| Sheet | Main table | Stacked sections below |
|---|---|---|
| `1 Table Profiling` | Schema, Object, Object type, Rows Count, Business Domain, then the other fields the skill defines (no Size Basis, Size Method or Notes columns) | Top 10 by Rows Count, Top 10 by size, total size in MB and GB, objects by business domain, flags, handoffs |
| `2 PHI Review` | One row per sensitive base-table column: identifier class, standard, risk, recommended technique | **View exposure first** (views that expose PHI and the base columns they expose), then re-identification check, pseudonymization plan summary, script file names, human decision log |
| `3 Column Profiling` | One row per column with every field the skill defines | Key verification results, summary findings |
| `4 Categorical` | One row per column and category: Schema, Table, Column, Category type, Category, Row count, % | Per-column classification and reason, suppressed small groups, columns skipped and why |
| `5 Data Patterns` | One row per format shape (Schema, Table, Column, Format, Row count, %, Shortest, Longest, Rank) | Content check, email domains, number and date ranges, parent-child breakdowns, summary findings |
| `6 Data Quality` | One row per column: Schema, Table, Column, Fill rate %, Invalid %, Consistency %, Column score, Rating | Validity rules and results, consistency variants, duplicates by kind, table scores, summary findings |
| `7 Relationships` | One row per link: parent, child, columns, declared or data-proven, cardinality, optional or required, orphan count | Table roles, orphan summary, links not confirmed |
| `8 Data Model` | Fact and dimension candidates with grain, measures and evidence | Shared dimensions, hierarchies, MDM entities, match keys, blockers |

If the source skill defines its fields differently from this table, the source skill wins.

**PHI headline.** On the `2 PHI Review` sheet, row 2 must state how many views expose PHI, for example `PHI is exposed through 12 of 26 views`, before anything else. Row 3 stays blank, and the main table starts at row 4 as on every other skill sheet. The `View exposure` table is the first stacked section below the main table. Do not add a borderline-columns section to this sheet.

### ER Diagram sheet

- Cell A1: title. Cell A2: the diagram file name.
- Embed the PNG at the top-left, below the title, at a size that fits without scaling the text unreadably. If the diagram is very large, embed it at readable scale and note the full-size PNG file name beside it.
- Below the picture, a **Legend** table: line style, label (`1:1`, `1:M`, `M:N`), and what it means.
- Below the legend, the full Mermaid source, one line per row in a single column, so the diagram can be rebuilt from the workbook.
- If the PNG could not be rendered, say so in A3, still write the Mermaid source rows, and do not embed an empty image.

## ER diagram rules

Build the diagram from the `relationships-eda` results. Do not invent links.

- Show every table in scope with its key columns (PK, FK, and any column that forms a link). Add other columns only if the user asked for them.
- Mark each key column `PK` or `FK`.
- Label every line `1:1` or `1:M` (or `M:N` where a junction was proven). Use Mermaid crow's-foot notation, for example `Sales_Customer ||--o{ Sales_SalesOrderHeader : "1:M"`.
- Use a different line style or label suffix for links proven from data instead of declared keys, and explain it in the Legend.
- Group large databases by subject area (schema) so the picture stays readable. If one diagram would be unreadable, also write one diagram per schema using the same file-name stem plus `-<schema>`.
- Render the PNG with an available Mermaid renderer (for example `mmdc`). Only say a PNG exists if the file exists and is not empty. If no renderer is available, deliver the Mermaid source, say the PNG could not be produced, and ask before installing anything.

## Building the workbook

1. Collect the results each skill produced in this session. Do not recompute or re-query.
2. Check which Excel tooling is available (for example `python3 -c "import openpyxl"`). Use the `xlsx` skill if it is available. Ask before installing anything.
3. Write the workbook with a script saved in the scratchpad or a temporary folder, not in the project, unless the user asks to keep it.
4. Write each sheet from the tables in this skill's layout. Embed the PNG on the `ER Diagram` sheet.
5. **Verify before reporting**: reopen the saved file, confirm all ten sheets exist with the exact names above, that each skill sheet has data rows (or the missing-results note), and that the image is present on `ER Diagram`. Fix and re-save if any check fails.

## Missing results

If a skill was not run, or ran only partly:

- Keep its sheet. Put in row 2: `Not run in this session` or `Partial: <what is missing and why>`.
- Do not fill the table with guesses, defaults or old results.
- Add the same note to the Limitations block on `Run Info`.

## Data-safety rules

These carry over from the source skills and are not relaxed by formatting.

- Column profiling and PHI results contain **counts and lengths only, never values**. Do not add sample values, min/max values or example rows to any sheet.
- Categorical and data-pattern sheets show only the values their own skills allow. Keep small-group suppression exactly as those skills applied it: suppressed groups appear only as a count, never as a row.
- Never write credentials, connection strings, passwords or tokens to the workbook, including on `Run Info`.
- PHI scripts stay as separate files. The `2 PHI Review` sheet lists their names but does not paste their contents.

## Final response

After the files are written and verified, reply with:

1. The full path of the workbook and of each diagram file.
2. A one-line summary per sheet: what it holds and how many rows.
3. Any sheet marked not run or partial, and any limitation that affects how far the results can be trusted.

Do not paste the tables into chat. A short top-line finding per skill is fine.
