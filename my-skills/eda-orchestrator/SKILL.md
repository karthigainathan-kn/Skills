---
name: eda-orchestrator
description: 'Runs the full EDA suite end to end as a human-in-the-loop agentic workflow: table profiling, PHI review (human checkpoint), column profiling, categorical profiling, data patterns, data quality, relationships, data model, then the Excel workbook and ER diagram. Coordinates one sub-agent per stage, passes results between stages, and pauses for the user at fixed checkpoints. Use this whenever the user asks to run the EDA, run the full EDA, profile the whole database end to end, run all the EDA skills, run the EDA workflow, or produce the EDA workbook from scratch, even if they do not name this skill.'
argument-hint: 'Optional: schemas or tables in scope (default: ask), data access method (DAB MCP or sqlcmd in Docker), stage to start from or stop at, output folder (default reports/eda)'
user-invocable: true
---

# EDA Orchestrator

Run the EDA suite in a fixed order, one stage at a time, with the user approving at checkpoints. This skill coordinates; the stage skills do the analysis and `eda-output-format` writes the files. The orchestrator never queries data itself except for the preflight checks.

The user may be new to this. At each step say, in one or two plain sentences, what is about to happen and why, and after each stage say what was found. Do not skip the explanations to save space.

## The flow

| # | Stage | Skill | Needs | Checkpoint after |
|---|---|---|---|---|
| 1 | Table profiling | `table-profiling-eda` | nothing | brief status |
| 2 | PHI review | `phi-review-eda` | stage 1 | **Checkpoint A: approve the sensitive-column list** |
| 3 | Column profiling | `column-profiling-eda` | stages 1, 2 | brief status |
| 4 | Categorical profiling | `categorical-profiling-eda` | stages 1, 2, 3 | brief status |
| 5 | Data patterns | `data-patterns-eda` | stages 1 to 4 | **Checkpoint B: new sensitive look-alikes, if any** |
| 6 | Data quality | `data-quality-eda` | stages 1 to 5 | brief status |
| 7 | Relationships | `relationships-eda` | stages 1 to 3, 5 | brief status |
| 8 | Data model | `data-model-eda` | stages 1 to 7 | **Checkpoint C: approve before writing files** |
| 9 | Output | `eda-output-format` | all results | final report |

Run the stages **sequentially in this order**. Each stage reuses the results of earlier ones, and the PHI list from stage 2 controls what later stages may show. Do not run stages in parallel unless the user asks, and tell them what the trade-off is (faster, but later stages lose earlier results and the checkpoints stop working).

## Step 0: Preflight (before any stage)

1. **Scope.** The default is **every user schema in the database**. Use a subset only if the user names one. Tell the user the scope you will use, and say the cost grows with the number of tables.
2. **Data access.** Find out what is available and tell the user which will be used and why:
   - The Data API Builder MCP endpoint (`http://localhost:5000/mcp`, tools `describe_entities`, `read_records`, `aggregate_records`). Read-only, but it exposes no system catalog, so sizes, declared keys and some metadata may be `Unavailable`.
   - `sqlcmd` inside the Docker `sqlserver` container. It can read the system catalog directly and is better for metadata (sizes, keys, foreign keys). Use only read-only queries.
   - Check each is reachable with a harmless request. If DAB is not running, say so and ask the user to start it (`dab start --config dab-config.json` in their own terminal) instead of starting it silently.
3. **Tooling.** Check for an Excel library (for example `python3 -c "import openpyxl"`) and a Mermaid renderer (`mmdc`). Report what is missing. Do not install anything without asking.
4. **Scratch folder.** Create a run folder in the session scratchpad for stage results, for example `<scratchpad>/eda-run-<timestamp>/`. Save each stage's results there as one JSON file (`01-table-profiling.json`, `02-phi-review.json`, and so on) so later stages and a resumed run can reuse them. Never save credentials or real data values there.
5. **Confirm the plan.** Summarise scope, access method, output folder and the three checkpoints in a few lines, and wait for the user to say go.

## Running a stage

For each stage, launch **one sub-agent** with the Agent tool and wait for it to finish. Brief it like a colleague who has none of this conversation:

- The skill to follow (by name) and the instruction to read its `SKILL.md` first.
- The scope and data access method, and the read-only rule.
- The paths of the earlier stage results to reuse.
- For stages 3 to 8: the **approved sensitive-column list** from Checkpoint A, plus the views that expose them (a view column that exposes a sensitive base column is sensitive too), and the rule that those columns get counts only, never values.
- Where to save its results (the scratch file) and what to return: a short summary, the path of the results file, and any `Unavailable` items with reasons.
- That it must not write to the project folder, change any data or run any PHI scripts.

After the sub-agent returns:

1. Check that the results file exists and is not empty. If it is missing or the stage reports an error, say so and stop to ask; never fill gaps with guesses.
2. Give the user a short plain-language status: what the stage did, headline findings, and anything unavailable.
3. Continue, or pause if a checkpoint follows.

If a stage is partial (for example a timeout on some views), record that in the scratch results and carry it to the `Run Info` limitations. Do not retry endlessly: one retry, then report.

## Checkpoint A: after PHI review (required)

Stop and wait. Show the user, as a compact table:

- **Views that expose PHI** first: how many of how many, which views, and which sensitive columns they expose.
- Sensitive columns found in the **base tables**: schema, table, column, group (identifier, health, other sensitive), confidence, and the planned handling. No borderline list is shown; unclear columns appear here as `Possible` so the user can reject them.
- That at this point the review is based on column names, types and descriptions only, and will be re-checked after the data-patterns stage.

Ask the user to reply with one of:
- `proceed`: use the list as shown.
- Changes: columns to add, remove or reclassify. Apply them, show the updated list, and ask again.
- `stop`: end the run and keep the files saved so far.

Record the approved list and the user's decisions in `02-phi-review.json` under `approved_sensitive_columns` and `decision_log`, and pass the approved list to every later stage. Do not continue without an explicit approval. If the review found **no** sensitive columns, still pause once and say so, because the user may know of columns it missed.

## Checkpoint B: after data patterns (only if something new appears)

The patterns stage reports counts of values that look sensitive (email, phone, SSN-like, card-like) and columns whose contents do not match their names. Compare those against the approved list.

- If nothing new: say so in one line and continue.
- If new columns turn up: show them, ask the user to approve adding them (or keep them out), update the approved list and the decision log, and re-run only the affected sections of `phi-review-eda` for those columns if the user wants a plan for them. Later stages use the updated list.

## Checkpoint C: before writing files (required)

After the data-model stage, show the user the plan for the output:

- The ten sheet names and the row count each will hold.
- The files to be written and their full paths (workbook, `.mmd`, `.png`, `.dbml`, any PHI script files).
- Any stage marked partial or not run, and the limitations that will appear on `Run Info`.
- Whether the PNG can be rendered with the tooling found in preflight.

Wait for `proceed`. Then follow `eda-output-format` to write and verify the files. Do not write anything to `reports/eda/` before this approval.

## Handling interruptions and changes

- **Resume.** If the user asks to continue a run, look for the scratch folder, list which stage files exist, say what is done and what remains, and restart from the first missing stage. Reuse approved decisions from `02-phi-review.json`.
- **Start from or stop at a stage.** If the user names one, run only that range. Stages that need earlier results use the saved files; if they do not exist, say which are missing and offer to run them first.
- **Scope change mid-run.** Stop, tell the user which stages are affected, and ask whether to restart them.
- **User questions at any point.** Answer them from the saved results without re-running anything.

## Safety rules

- Read-only throughout. No stage may change data, create objects in the database, or run the PHI scripts. Those scripts are saved as files for the user's team to review.
- Counts and aggregates only for sensitive columns, as the stage skills require. Never put sample values in a status message, a scratch file or a sheet.
- Never print or save credentials, connection strings or tokens. Read the DAB config only to find the endpoint, not to echo secrets.
- Do not install software, start or stop services, or write outside the scratch folder and the output folder without asking.
- Never claim a stage ran, a file exists or a diagram rendered unless it was checked.

## Final report

After `eda-output-format` finishes and verifies the files:

1. Full paths of the workbook and each diagram file.
2. One line per stage: what it found, in plain language, and the sheet it landed on.
3. The PHI decisions the user made and where they are recorded.
4. Stages that were partial or unavailable, and why, so the user knows how far to trust the results.
5. Suggested next steps, such as reviewing the PHI scripts or re-running a stage with a wider scope.

Do not paste the tables into chat.
