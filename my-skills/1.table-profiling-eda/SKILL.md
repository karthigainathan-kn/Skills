---
name: table-profiling-eda
description: 'Stage 1 of the modular EDA suite: read-only, table-level profiling of any connected database or data platform (SQL Server, Azure SQL, PostgreSQL, MySQL, Oracle, Snowflake, Databricks, BigQuery, Fabric, or an MCP data connection). Inventories every object in every user schema and determines Schema, Object, Object type, Rows Count, Business Domain, Size (MB), Size (GB), and Row Grain, with the evidence behind each. Use this whenever the user asks to scan, inventory, or profile a database, schema, catalog, or lakehouse; list tables or views; get row counts, table sizes, storage footprint, or the largest tables; work out what one row represents; or start an EDA, discovery, or data platform assessment, even if they do not say "profiling".'
argument-hint: 'Optional: database/catalog, schema(s) to include or exclude (default: all user schemas), specific objects, known business domains or definitions'
user-invocable: true
---

# EDA Stage 1: Table-Level Profiling

Produce one record per database object across **every user schema in the database** (unless the user narrows the scope), plus Top 10 rankings by rows and by storage size, with eight headline fields: **Schema, Object, Object type, Rows Count, Business Domain, Size (MB), Size (GB), Row Grain**. Every value must trace to metadata the platform returned or a query that was actually run. Later EDA stages build on this inventory, so an error here propagates everywhere downstream.

This skill is platform-agnostic. Work out what the connected platform exposes and use that; do not assume any one engine's catalog. `references/catalog-sources.md` lists where common platforms keep the relevant metadata. Read it only when you need the specifics for the platform you are connected to.

## Mandatory core fields

Four fields are the backbone of this skill and must hold in every run, on every platform, whatever else fails:

**Schema · Object · Object type · Rows Count**

- Use exactly these names, in exactly this order, as the first four fields of every record.
- Every discovered object gets a record with all four populated. Never leave one blank, and never drop an object from the inventory because a later step (size, keys, grain) failed for it.
- Rows Count always holds a value: the count, `≥ N`, `N/A` (object stores no rows), or `Unavailable` (with the reason in `Count Method`). A blank cell is never acceptable, because downstream readers cannot tell "zero rows" from "not checked".
- A count of `0` means an empty object that was actually counted. Never use 0 as a placeholder.
- If time or access runs short, finish these four for every object before spending effort on size, keys, or grain. A complete inventory with partial enrichment beats a richly profiled half-inventory.

The remaining fields (business domain, size, keys, grain) enrich the record; they are produced when supported and marked `Unavailable` with a reason when not.

## Scope of this skill

This skill owns: object inventory, row counts, storage size, declared keys, and row grain.

It does not compute column statistics, null rates, distinct counts, date ranges, growth, PHI/PII flags, patterns, or relationships. Those belong to other EDA skills. When something observed here matters to another stage (no declared key, an unenforced constraint, a suspicious object name), record it in the run's **Handoffs** list (object, what was seen, which stage should look) rather than investigating it. There is no per-object Notes field.

It does not decide the output format. If an output-format skill is available, follow it. Otherwise return the inventory as a plain table in chat and state that formatting is pending.

## Safety

- Read-only. Use discovery, catalog, and aggregate operations only. Never create, alter, or delete objects, and never execute procedures or functions to see what they return.
- Do not read row values. Table-level profiling needs names, metadata, and counts. This keeps sensitive data out of scope by construction.
- If an object or capability is not exposed, say so. A silent gap reads as "this does not exist", which is worse than "not visible".

## Step 0: Identify the platform and what the connection can see

Record: platform/engine, database or catalog, server/workspace/account where available, run timestamp (UTC), included and excluded schemas, and any business definitions the user supplied.

Then classify the connection's **visibility**, because it decides whether the inventory can be called complete:

| Visibility | Typical access | Inventory claim |
|---|---|---|
| `Full catalog` | Read-only SQL against the platform's system catalog or `INFORMATION_SCHEMA` | Complete for the database/catalog, subject to the account's permissions |
| `Permission-limited catalog` | Catalog access, but the account cannot see all schemas/objects | Complete for what this account can see; say so |
| `Exposed entities only` | An API or MCP layer exposing configured entities (e.g. `describe_entities` / `aggregate_records`) | Covers the exposed entities only, not the database |

Never present an exposed-entities inventory as a database inventory.

## Step 1: Discover objects

Use the richest read-only discovery method available, in this order of preference:
1. The platform's system catalog or `INFORMATION_SCHEMA`.
2. Platform metadata commands (`SHOW` / `DESCRIBE` style).
3. MCP or API entity discovery. When there are many entities, list names first, then request details in manageable groups.

Exclude system and platform-internal objects. Record each object's schema, name, the raw type as returned, and the discovery method.

Normalize object types to this vocabulary, keeping the raw type in the run's Handoffs list only when it adds information:

`Table`, `View`, `Materialized View`, `External Table`, `Stored Procedure`, `Function`, `Table-Valued Function`, `Synonym/Alias`, `Stream`, `Other (<raw type>)`.

Use a more specific label (for example `Table (heap)`, `Indexed View`, `Delta Table`, `Dynamic Table`) only when the platform's metadata confirms it. Do not guess a storage format from a name.

## Step 2: Row counts

Row counts apply to tables, views, materialized views, and external tables. Procedures, functions, synonyms, and similar objects get `N/A` with the reason.

Use the most accurate method the platform supports and always record which one was used. Metadata counts and exact counts are different claims:

| Method | Label | When |
|---|---|---|
| `COUNT(*)` or an aggregate count tool | `Exact` | Default, when affordable |
| Platform metadata or statistics row count | `Metadata` | Large tables or when speed is requested. Note whether the platform documents it as exact or estimated, and when statistics were last refreshed if known |
| Bounded read | `Lower bound` | Only when no count is supported. Report as `≥ N` |
| Not possible | `Unavailable` | State why |

Guidance:
- Metadata counts vary widely in reliability across platforms. Some are maintained exactly; others are estimates from statistics that may be stale. Check `references/catalog-sources.md` and label accordingly.
- An exact count on a view executes the view. Run it, but record slowness or timeouts in the run's Limitations list; do not substitute a base-table count.
- External tables may scan remote storage when counted. Ask before counting large external objects if cost or time is a concern.
- Record the count timestamp. Counts on live systems drift.

## Step 2b: Storage size (MB and GB)

Size applies to objects that store data: tables, materialized/indexed views, and external tables where the platform reports it. Plain views, procedures, functions, and synonyms get `N/A`, because they hold no data of their own.

Size comes from platform metadata, never from reading rows. Platforms measure it differently, so record what the number includes:

- **Size (MB)**: total storage attributed to the object, rounded to 2 decimals. MB = 1,048,576 bytes.
- **Size (GB)**: the same figure in GB, rounded to 3 decimals so small objects do not all collapse to 0.00. GB = 1,073,741,824 bytes. Derive both from the same byte value, never from separately rounded numbers, so they always agree. State the binary (1,024-based) convention once in the run header; if the user's organisation reports decimal units (1 GB = 1,000,000,000 bytes), use that instead and state it.
- **Size basis and method are stated once, in the run header, not per object.** Say what the figure covers (e.g. `Data + indexes`, `Data only`, `Compressed physical`, `Logical (uncompressed)`, `Current version files only`, `Reserved (allocated)` vs `Used`) and the metadata source used. If the platform offers several bases, prefer total used storage including indexes, and mention the alternative in Limitations when the difference is material. If size is `Unavailable` for an object, the reason (permission denied, not exposed by the MCP layer, lake format without maintained statistics) goes in the Limitations list.

Guidance:
- Sizes across platforms are not directly comparable. Compressed columnar storage can be a small fraction of the same data in a row store. Do not rank or compare sizes across platforms without saying so.
- For lake formats (Delta, Iceberg, Parquet), file size may exclude or include old versions and time-travel history. Say which.
- For external tables, size usually describes remote files, not anything the database manages. Label it.
- Do not compute size as row count × average row width. That is an estimate, not metadata; if the user asks for one, label it `Estimated` explicitly.

## Step 3: Declared keys and their enforcement

Collect from metadata only, per table/view:
- Primary key: name and ordered column list (composite keys in key order).
- Unique constraints and unique indexes: each as an ordered column list. Ignore filtered or partial unique indexes for grain purposes, since they do not cover every row.
- **Enforcement**: whether the platform actually enforces the constraint. Many analytical platforms let you declare primary and unique keys as informational only, without enforcing them. A declared-but-unenforced key describes intent, not fact.

Do not infer a key from a column named `ID`, from uniqueness in a sample, or from naming conventions. Testing whether a candidate or unenforced key holds in the data belongs to the column-profiling stage; record it as a handoff.

## Step 4: Row grain

Row grain answers "what does one row represent?". Split it into two parts, because they carry different levels of certainty:

1. **Structural grain** — what makes a row unique, from declared keys.
   - Single-column key → `One row per CustomerID`
   - Composite key → `One row per (OrderID, LineNumber)`
   - No PK but a unique constraint/index → `One row per (<cols>) [unique constraint]`
   - No declared key → `Undetermined: no declared key`
2. **Semantic grain** — the business meaning, e.g. "one order line". Unless the user supplied a business definition, this is an interpretation drawn from names and must be labelled as one.

Assign a **Grain Status**:

| Status | Meaning |
|---|---|
| `Confirmed` | Enforced declared key **and** semantic meaning supplied by the user or documentation |
| `Key-based` | Enforced declared key; semantic meaning interpreted from names |
| `Declared, unenforced` | Grain comes from an informational key the platform does not enforce; uniqueness unverified |
| `Interpreted` | No declared key; grain suggested from names or structure only |
| `Undetermined` | Not enough evidence to propose a grain |

Rules:
- Record the evidence for every grain (key name, columns, enforcement), or the reason none exists, in the Grain Evidence field.
- Give concrete examples where the structure supports them: one customer vs. many order headers; one order header vs. many order lines; one person vs. many phone or email rows. Only describe the "many" side as structural when a declared key or relationship makes it so; otherwise phrase it as an interpretation.
- For views, use a declared key only if the platform supports keys on that view type. Otherwise the grain is `Undetermined`, unless a readable definition makes it unambiguous, in which case status is at most `Interpreted`.
- Call out bridge/link objects (a composite key made of two foreign keys). They are the usual source of fan-out in downstream joins.
- Record unresolved ambiguity in the Handoffs list, e.g. "surrogate PK; business uniqueness of (OrderNumber, LineNumber) unverified → column profiling".

## Step 5: Business domain

Assign every object one **Business Domain**: the business area it belongs to (for example `Sales & Orders`, `Customers`, `Products & Catalog`, `Purchasing & Vendors`, `Human Resources`, `Finance & Currency`, `Reference Data`, `System & Audit`). It lets a reader group the inventory by business area instead of by technical schema.

- Decide from metadata only: the schema name, the object name, the names of its key and foreign-key columns, the tables it links to by declared foreign keys, and any description the platform holds (for example SQL Server extended properties). Never read row values.
- A business definition the user supplies wins over anything inferred.
- Use short labels and reuse the same label for the same area across the whole run, so that grouping works. Prefer a small set of domains (roughly 6 to 12 for a database) over a new label per schema. A schema does not have to map to exactly one domain, and one domain can span schemas.
- Bridge and link objects take the domain of the business process they serve, not the domain of one side only.
- This is an **interpretation**, not a fact in the metadata. If the evidence is thin or conflicting, use `Unclassified` instead of guessing. List every `Unclassified` object in the Handoffs list.
- State once, in the run header, that domains are interpreted from names, keys, links and descriptions, and list the domains used with a one-line meaning each.
- Do not add a separate column for the evidence or confidence. If an assignment is debatable, mention it in Handoffs.

## Result fields

Produce these fields for every object. The first four are mandatory (see Mandatory core fields) and always appear first, in this order. Do **not** output `Size Basis`, `Size Method` or `Notes` fields: that information lives in the run header, the Limitations list and the Handoffs list. The output-format skill decides layout, file type, and naming; this skill defines the content.

| Field | Content |
|---|---|
| Schema | Schema, dataset, or namespace as the platform names it |
| Object | Object name |
| Object type | Normalized type from Step 1 |
| Rows Count | Integer, `≥ N`, `N/A`, or `Unavailable`. Never blank |
| Size (MB) | Decimal (2 places), or `N/A` / `Unavailable` |
| Size (GB) | Decimal (3 places), or `N/A` / `Unavailable` |
| Row Grain | Structural grain, then semantic grain, e.g. `One row per (OrderID, LineNumber) — one order line` |
| Grain Status | From the Step 4 table |
| Grain Evidence | Key name, columns, and enforcement, or the reason none exists |
| Declared PK | Ordered column list or `None declared` |
| Key Enforced | `Yes` / `No (informational)` / `Unknown` / `N/A` |
| Count Method | `Exact` / `Metadata` (with exact-or-estimated note) / `Lower bound` / `Unavailable` / `N/A – <reason>` |
| Count Timestamp | UTC |

## Ranked lists

Produce two ranked lists from the inventory, in addition to the per-object records.

**Top 10 tables by Rows Count** and **Top 10 tables by storage size**

Fields for both lists, in this order: **Rank · Schema · Object · Object type · Rows Count · Size (MB) · Size (GB)**. Showing both measures in each list makes size/row mismatches visible at a glance.

Rules:
- Rank tables only: all table variants (heap, Delta, external, etc.) plus materialized/indexed views, since they store data. Exclude plain views, procedures, functions, and synonyms, and say so under each list.
- Rank by Rows Count descending for the first list and by size in bytes descending for the second. Rank on the underlying value, not the rounded MB/GB figure.
- Exclude objects whose value is `Unavailable` or a `≥ N` lower bound from the ranking, and state how many were excluded and why. A lower bound cannot be ranked honestly.
- If counts or sizes come from mixed methods (some `Exact`, some `Metadata`, or different size bases), say so under the list. Rankings across different measurement methods are indicative, not precise.
- Ties share a rank (1, 2, 2, 4). If ties at position 10 push the list past 10 entries, include all tied entries and note it.
- If fewer than 10 tables qualify, list all of them and state the actual number. Do not pad.
- If the size list cannot be produced at all (size unavailable for every table), say so plainly instead of omitting the section.

Also produce a run header (platform, database/catalog, visibility, scope, run timestamp, count methods used, size basis and size method, MB/GB convention, and the business domains used), a **Handoffs** list (object, what was seen, which stage should look) and a list of limitations.

## Summary for the user

When reporting back, cover:
- Platform, visibility, scope, run time, and count methods used.
- Object counts by type, and by business domain.
- The Top 10 tables by Rows Count and Top 10 tables by storage size, plus total size of profiled objects in both MB and GB. Flag objects whose size and row count disagree sharply (large size with few rows, or the reverse) as a handoff; they often signal bloat, wide/LOB columns, or stale statistics.
- Grain status breakdown, listing every object that is `Interpreted`, `Undetermined`, or `Declared, unenforced`.
- Tables with no declared key.
- Limitations and blocked checks, stated plainly.

Never describe an `Interpreted` or unenforced grain in the same confident voice as a `Key-based` one.
