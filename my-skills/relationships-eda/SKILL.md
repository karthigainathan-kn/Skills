---
name: relationships-eda
description: 'EDA suite links stage. Read-only. Finds how tables link in any connected database or data platform (SQL Server, Azure SQL, PostgreSQL, MySQL, Oracle, Snowflake, Databricks, BigQuery, Fabric, or an MCP data connection): declared foreign keys, links proven by checking the data when none are declared, one-to-one / one-to-many / many-to-many, optional vs required links, orphan rows, table roles, and draws a relationship diagram with tables, columns, keys, and 1:1 / 1:M labels on every line. Use this whenever the user asks about relationships, foreign keys, joins, links between tables, orphans, referential integrity, ER diagrams, ERDs, data model diagrams, star schema diagrams, or how tables connect, even if they do not say "relationships".'
argument-hint: 'Optional: schemas or tables; links to confirm or ignore; diagram scope (all, by subject area, by fact table); match threshold'
user-invocable: true
---

# EDA: Relationships

Find out how tables link, prove each link with evidence, measure the 1:1 / 1:M pattern, count orphan rows, and draw a relationship diagram where every line says what kind of link it is.

Works on any database. Use what the platform supports. If something can't be checked, say **Not available** and why.

## What this skill does and does not do

Does:
1. Declared links (foreign keys)
2. Links proven by checking the data
3. Link type: 1:1, 1:M, M:M, optional or required
4. Orphan rows
5. Table roles
6. Relationship diagram

Does not do:
- Fact/dimension design, star schema and MDM readiness → data-model-eda (uses this skill's results)
- Report layout → output-format skill (if none, show plain tables in chat)

## Safety rules

- **Read-only.** Only look at structure and run counts. Never change anything, never add keys.
- **No real values.** Link checks return counts and %, never the matching or orphan key values.
- Column and table names can appear in the diagram. Values never do.

## Step 0: Before you start

Reuse earlier results if available: table list and row counts, declared keys and whether they are enforced, column types, distinct counts, sensitive-column list. If missing, collect the basics and say so.

Header: database, what was included, run time (UTC), match threshold, and anything that couldn't be checked.

## Step 1: Declared links

Read foreign keys from the platform's catalog. For each link record: from schema · table · column(s) → to schema · table · column(s), the key name, and whether the platform enforces it (many cloud platforms let you declare keys they don't enforce).

Composite keys: **one row per column pair**, keeping the column order.

## Step 2: Links proven by checking the data

Many databases, especially warehouses, Snowflake, Databricks and lakehouses, declare few or no foreign keys. Find likely links, then **prove them with counts**. A link is never accepted on names alone.

**Find candidates.** Pair a column with a likely parent column when most of these agree:
- the parent column is a declared key or unique (from column profiling)
- the names match or follow a pattern (`CustomerID` → `Customer.CustomerID`, `ProductKey` → `Dim_Product.ProductKey`, `ShipToAddressKey` → `Dim_Address.AddressKey`)
- the data types are compatible
- the child column has many repeats, the parent column has none

**Check the values** (counts only):
- **Match %**: share of the child's non-null values that exist in the parent column.
- **Parent is unique**: the parent column has no duplicates.

**Evidence level**

| Level | Rule |
|---|---|
| **Declared** | A foreign key exists in the database |
| **Data-checked** | Not declared, parent is unique, and match % is **99% or more** (changeable) |
| **Data-checked, with orphans** | Not declared, parent is unique, match % between **90% and 99%** |
| **Name-only lead** | Names look related but the values weren't checked or match below 90%. Listed for follow-up, **never drawn as a link** |

Checking every possible pair is expensive on big databases. Check the most likely candidates first, and ask before checking very large tables.

## Step 3: Link type

For every Declared or Data-checked link, work out the type from counts:

| Type | How it is decided | Line label |
|---|---|---|
| **One-to-one** | Parent unique **and** each parent value appears at most once on the child side | `1:1` |
| **One-to-many** | Parent unique, child values repeat | `1:M` |
| **Many-to-many** | Two tables linked through a bridge table holding both keys | `M:M` (via bridge) |

Also record:
- **Child side required or optional**: does the child column have blanks? Blanks mean the link is optional (`0..1` parent).
- **Parents without children**: % of parent rows never used (e.g. products never sold). This makes the many side `0..M` rather than `1..M`.
- **Max children per parent**: the largest number of child rows for one parent (counts only).
- **Same parent, several roles**: one table linked several times through different columns (e.g. a fact with OrderDateKey, ShipDateKey and DueDateKey all pointing to the date table). Each is its own link, labelled with the column name.
- **Self-links**: a table linked to itself (employee → manager, category → parent category).

## Step 4: Orphan rows

For each link: child rows whose value has no match in the parent, as count and %.
- On a **Declared, enforced** link: orphans should be zero; any found is a **Problem**.
- On a **Declared, not enforced** link: orphans are a **Problem**.
- On a **Data-checked** link: orphans are **Worth checking**.

Never list the orphan values.

## Step 5: Table roles (from the link structure)

| Role | Meaning |
|---|---|
| **Parent only** | Referenced by others, references nothing (lists like product, customer, date) |
| **Child only** | References others, nobody references it (detail rows, transactions) |
| **Middle** | Both references and is referenced |
| **Bridge** | Mostly made of two or more link columns; connects other tables many-to-many |
| **Self-linked** | References itself |
| **Isolated** | No links found at all. Worth a look: staging, logs, or missing links |

These are structural roles, not design advice. Fact vs dimension is decided in data-model-eda.

## Step 6: Relationship diagram

Draw a diagram in the style of a database modelling tool: one box per table, header with the table name, then the columns with their data types, and lines between linked columns showing the link type.

**Box contents**
- Table name as the header.
- Columns with data types. Mark the primary key `PK` and link columns `FK`.
- Up to **25 columns** per box (changeable). For wider tables show all key and link columns first, then the rest up to the limit, then one line: `… N more columns`.

**Lines**
- One line per link. A table linked several times to the same parent (e.g. several date keys) gets several lines, each labelled with its column.
- Every line is labelled with the **link type and the column**, e.g. `1:M · ProductKey`, `1:1 · CustomerKey`.
- The line ends show the same thing visually: a single bar for "one", a crow's foot for "many", a circle where the side is optional.
- **Declared** links: solid line. **Data-checked** links: dashed line, label ending `(data-checked)`.
- **Name-only leads are never drawn.**

**Format**
Produce **Mermaid** `erDiagram` source and render it to **PNG**. Mermaid notation to use:

| Link | Mermaid | Meaning |
|---|---|---|
| 1:1, both required | `A \|\|--\|\| B : "1:1 · Col"` | exactly one on each side |
| 1:M, many side optional | `A \|\|--o{ B : "1:M · Col"` | one parent, zero or more children |
| 1:M, many side required | `A \|\|--\|{ B : "1:M · Col"` | one parent, at least one child |
| Optional parent (child column has blanks) | `A \|o--o{ B : "1:M · Col"` | zero or one parent |
| Data-checked | use `..` instead of `--` | dashed line |

Columns go inside the entity block as `type Name PK` or `type Name FK`. Example:

```
erDiagram
    Dim_Product ||--o{ Fact_SalesOrderDetail : "1:M · ProductKey"
    Dim_Date ||--o{ Fact_SalesOrderDetail : "1:M · SalesOrderDateKey"
    Dim_Date ||--o{ Fact_SalesOrderDetail : "1:M · SalesOrderShipDateKey"
    Dim_Address ||..o{ Fact_SalesOrderDetail : "1:M · ShipToAddressKey (data-checked)"
    Dim_Product {
        int ProductKey PK
        varchar ProductName
        varchar Color
    }
    Fact_SalesOrderDetail {
        int SalesOrderDetailKey PK
        int ProductKey FK
        int SalesOrderDateKey FK
        int SalesOrderShipDateKey FK
        int ShipToAddressKey FK
        int Quantity
        decimal LineTotal
    }
```

Also produce a **DBML** file (for dbdiagram.io or similar tools) with the same tables, columns, and links, using `>` for many-to-one and `-` for one-to-one, and a note on each data-checked link. These tools draw the `1` and `*` markers at the line ends and lay tables out cleanly.

**Large models**
- More than about 20 tables in one diagram becomes unreadable. Split into one diagram per **subject area** or per **central table** (the table with the most links, plus everything linked to it, like a star). Always also give one **overview diagram** with table names only.
- Isolated tables are listed, not drawn.

**Rendering**
- Render the PNG with an available Mermaid renderer (e.g. `mmdc`). Only say a PNG was created if the file exists and is not empty.
- If no renderer is available, deliver the Mermaid and DBML sources, say the PNG could not be produced, and ask before installing anything.
- Keep one stable file name per diagram; re-render only when links change.

## What to return

- **Links**: From schema · From table · From column · To schema · To table · To column · Key name · Evidence level · Enforced · Match % · Link type · Child side required/optional · Parents without children % · Max children per parent · Role name (for repeated links) · Orphan rows · Orphan % · Result (`Pass` / `Problem` / `Worth checking`)
- **Name-only leads**: From · To · Why suggested · Why not proven
- **Table roles**: Schema · Table · Role · Links in · Links out
- **Diagrams**: Mermaid source, PNG, DBML, one per scope, plus the overview
- Header and a list of anything that couldn't be checked

## Summary for the user

1. Header: scope, threshold, what couldn't be checked.
2. Links found: declared vs data-checked, and how many name-only leads remain.
3. Link types: 1:1, 1:M, M:M, and repeated links to the same table (e.g. role-playing dates).
4. Orphan problems first, then orphans worth checking.
5. Isolated tables and bridge tables.
6. Diagrams produced, with file names, and anything not drawn.

Never draw or describe a data-checked link as if it were declared, and never draw a name-only lead at all.
