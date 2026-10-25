# Catalog sources by platform

Where to find table-level metadata on common platforms. Read only the section for the platform you are connected to, and only when you need exact catalog names. Everything here is read-only metadata; none of it needs row values.

**How far to trust this file.** SQL Server was used and checked in this project. Every other section is written from general platform knowledge and has not been run here. Treat those as starting points: if a query fails or returns something odd, say so in the Limitations list instead of forcing it, and confirm against the platform's documentation for the connected version.

Each section answers the same six questions:

1. **Objects**: where to list them and how to leave out system objects
2. **Rows**: where the row count comes from, and whether it is exact or estimated
3. **Size**: where it comes from and what it includes
4. **Keys**: primary and unique keys, and whether they are enforced
5. **Links and descriptions**: foreign keys and comments (these feed the Business Domain step)
6. **Gotchas**

Use `COUNT(*)` for an exact row count wherever it is affordable. The metadata counts below are for large tables or when speed matters, and they must be labelled `Metadata`.

---

## SQL Server / Azure SQL (checked in this project)

1. **Objects**: `sys.tables`, `sys.views`, or `sys.objects` joined to `sys.schemas`. Leave out `is_ms_shipped = 1` and the `sys` and `INFORMATION_SCHEMA` schemas.
2. **Rows**: `sys.partitions.rows` or `sys.dm_db_partition_stats.row_count` (use index_id 0 for a heap or 1 for a clustered index, so rows are not counted once per index). Normally exact for rowstore tables, but label it `Metadata`. `COUNT_BIG(*)` is exact.
3. **Size**: `sys.allocation_units` joined through `sys.partitions` (`container_id` to `partition_id` for in-row and row-overflow data, and to `hobt_id` for LOB data). Use `used_pages * 8192` bytes for data plus indexes, or `total_pages` for reserved space. Say which. Do not call `sp_spaceused`; it is a procedure.
4. **Keys**: `sys.key_constraints` (type `PK` or `UQ`), `sys.indexes` (`is_primary_key`, `is_unique`, `has_filter`; skip filtered indexes for grain) and `sys.index_columns` for column order. Primary and unique keys are enforced. A foreign key can be disabled or not trusted (`sys.foreign_keys.is_disabled`, `is_not_trusted`); report that.
5. **Links and descriptions**: foreign keys in `sys.foreign_keys` and `sys.foreign_key_columns`. Descriptions in `sys.extended_properties` where `name = 'MS_Description'` and `class = 1` (`minor_id = 0` for the table, otherwise the column).
6. **Gotchas**
   - Seeing the catalog depends on permissions. Without `VIEW DEFINITION` some objects or properties are hidden; report visibility as permission-limited.
   - A `COUNT` on a view runs the view; log slow or failed ones.
   - Plain views have no storage: size is `N/A`. Indexed views do have storage.
   - Row counts for tables in columnstore or memory-optimized storage may come from different DMVs; label what you used.
   - Object names are case-insensitive under most collations; compare accordingly.

## PostgreSQL

1. **Objects**: `pg_class` joined to `pg_namespace`. `relkind`: `r` table, `p` partitioned table, `v` view, `m` materialized view, `f` foreign table. Leave out `pg_catalog`, `information_schema`, `pg_toast`.
2. **Rows**: `pg_class.reltuples` is an estimate (`-1` if never analyzed) and `pg_stat_user_tables.n_live_tup` is also an estimate. Label both `Metadata (estimated)`. `COUNT(*)` is exact.
3. **Size**: `pg_total_relation_size` (table plus indexes plus TOAST) or `pg_relation_size` (table only). These are built-in catalog functions that read metadata, so they are fine to call. State which one you used.
4. **Keys**: `pg_constraint` (`contype` = `p` primary, `u` unique, `f` foreign). All are enforced unless a foreign key is `NOT VALID`.
5. **Links and descriptions**: foreign keys in `pg_constraint` (`conrelid`, `confrelid`, `conkey`, `confkey`). Comments via `obj_description` or `pg_description`.
6. **Gotchas**: a partitioned parent table stores nothing itself; size and rows live in its partitions. Statistics go stale until `ANALYZE` or autovacuum runs.

## MySQL / MariaDB

1. **Objects**: `information_schema.TABLES` (`TABLE_TYPE` is `BASE TABLE` or `VIEW`). Leave out `mysql`, `sys`, `information_schema`, `performance_schema`.
2. **Rows**: `TABLES.TABLE_ROWS` is an **estimate** for InnoDB; it can be far off. Label `Metadata (estimated)`. Use `COUNT(*)` for exact.
3. **Size**: `DATA_LENGTH + INDEX_LENGTH`. `DATA_FREE` is unused space and not part of the size.
4. **Keys**: `TABLE_CONSTRAINTS`, `KEY_COLUMN_USAGE`, `STATISTICS`. Enforced on InnoDB.
5. **Links and descriptions**: `REFERENTIAL_CONSTRAINTS` and `KEY_COLUMN_USAGE` for foreign keys. Descriptions in `TABLES.TABLE_COMMENT`.
6. **Gotchas**: in MySQL 8 these figures are cached (`information_schema_stats_expiry`), so they can lag. Foreign keys are enforced only by engines that support them (InnoDB), not MyISAM.

## Oracle

1. **Objects**: `ALL_OBJECTS` or `ALL_TABLES` / `ALL_VIEWS` (use `DBA_` versions if you have access). Leave out Oracle-maintained schemas (`ALL_USERS.ORACLE_MAINTAINED = 'Y'`).
2. **Rows**: `ALL_TABLES.NUM_ROWS` from optimizer statistics, with `LAST_ANALYZED`. It can be null or stale; label `Metadata (estimated)` and give the date. `COUNT(*)` is exact.
3. **Size**: `ALL_SEGMENTS.BYTES` (needs privilege; `DBA_SEGMENTS` otherwise). Sum the table's segments, including partitions. LOB and index segments are separate: say whether they are included.
4. **Keys**: `ALL_CONSTRAINTS` (`CONSTRAINT_TYPE` `P` primary, `U` unique, `R` foreign) and `ALL_CONS_COLUMNS` for column order. Check `STATUS` (`ENABLED`/`DISABLED`) and `VALIDATED`. A `RELY` constraint with `NOVALIDATE` is informational.
5. **Links and descriptions**: foreign keys via `R_CONSTRAINT_NAME`. Comments in `ALL_TAB_COMMENTS`.
6. **Gotchas**: you only see what your privileges allow. Partitioned tables need their partitions summed. Statistics may never have been gathered.

## Snowflake

1. **Objects**: `<database>.INFORMATION_SCHEMA.TABLES` (`TABLE_TYPE`: `BASE TABLE`, `VIEW`, `EXTERNAL TABLE`, and others). Leave out the `INFORMATION_SCHEMA` schema.
2. **Rows**: `ROW_COUNT` from table metadata (maintained for native tables; null for views). Label `Metadata`.
3. **Size**: `BYTES` is the compressed active storage. It excludes Time Travel and Fail-safe copies; `SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS` includes them but lags by hours.
4. **Keys**: `INFORMATION_SCHEMA.TABLE_CONSTRAINTS` or `SHOW PRIMARY KEYS`. Primary, unique and foreign keys are **declared but not enforced** (only `NOT NULL` is). Report `Key Enforced = No (informational)`.
5. **Links and descriptions**: `SHOW IMPORTED KEYS` for foreign keys. `COMMENT` column on `TABLES`.
6. **Gotchas**: size is compressed and not comparable with a row store. Declared keys describe intent, not fact, so the grain status is `Declared, unenforced`.

## Databricks / Delta Lake (Unity Catalog)

1. **Objects**: `<catalog>.information_schema.tables` (`table_type`: `MANAGED`, `EXTERNAL`, `VIEW`, and others). Leave out `information_schema`.
2. **Rows**: Delta metadata does not always hold a reliable row count. Use `COUNT(*)` (exact) by default; file statistics may give an approximate figure. Label accordingly.
3. **Size**: `DESCRIBE DETAIL <table>` returns `sizeInBytes` and `numFiles` for the **current table version only**. It excludes old versions kept for time travel until vacuumed. Say so.
4. **Keys**: `information_schema.table_constraints`. Primary and foreign keys are **informational, not enforced**. Report `No (informational)`.
5. **Links and descriptions**: `information_schema.referential_constraints`. Comments in `information_schema.tables.comment`.
6. **Gotchas**: external tables describe remote files, not storage the database manages. `DESCRIBE DETAIL` is a metadata command, so it is allowed here.

## BigQuery

1. **Objects**: `<project>.<dataset>.INFORMATION_SCHEMA.TABLES` (`table_type`: `BASE TABLE`, `VIEW`, `MATERIALIZED VIEW`, `EXTERNAL`). Metadata is per dataset and per region.
2. **Rows**: `INFORMATION_SCHEMA.TABLE_STORAGE.total_rows` or the legacy `__TABLES__.row_count`. Label `Metadata`. Rows still in the streaming buffer are not counted.
3. **Size**: `TABLE_STORAGE.total_logical_bytes` or `total_physical_bytes`. State which one you use; they differ with compression and billing model.
4. **Keys**: `INFORMATION_SCHEMA.TABLE_CONSTRAINTS` and `KEY_COLUMN_USAGE`. Keys are **not enforced**. Report `No (informational)`.
5. **Links and descriptions**: `TABLE_OPTIONS` (option `description`) and `CONSTRAINT_COLUMN_USAGE`.
6. **Gotchas**: an `INFORMATION_SCHEMA` query only covers one dataset or region unless you use the region-wide views. Scanning a table to count rows can cost money; prefer metadata when asked.

## Microsoft Fabric / Azure Synapse

1. **Objects**: a Fabric Warehouse or Synapse dedicated SQL pool exposes a T-SQL surface, so the SQL Server catalog views (`sys.tables`, `sys.views`, `sys.schemas`) apply. A Lakehouse SQL analytics endpoint is read-only and exposes only the Delta tables.
2. **Rows**: `COUNT_BIG(*)` is exact. Partition or distribution metadata may give counts on some pools; label what you used.
3. **Size**: availability varies by product and SKU. If no size metadata is exposed, report `Unavailable` with the reason.
4. **Keys**: primary key and unique constraints can be declared only as `NOT ENFORCED`. Report `No (informational)`.
5. **Links and descriptions**: foreign keys are informational as well. Extended properties may not be supported.
6. **Gotchas**: this section is the least certain. Check which catalog views your Fabric item type actually exposes before relying on any of them.

## MCP or API entity layers (for example Data API Builder)

These layers expose configured entities, not the database. Visibility is `Exposed entities only`.

1. **Objects**: the layer's discovery tool (for example `describe_entities`). It lists only entities someone configured. Field lists can come back empty.
2. **Rows**: an aggregate tool (for example `aggregate_records` with count). Label `Exact`.
3. **Size**: normally **not available**. Report `Unavailable – not exposed by the MCP layer`.
4. **Keys**: only if the entity configuration exposes them. Otherwise `Unknown`.
5. **Links and descriptions**: usually not exposed. The Business Domain step then relies on entity and field names alone, and says so.
6. **Gotchas**: never present this inventory as a database inventory. The Data API Builder endpoint in this project (version 2.0.12) offers `describe_entities`, `read_records` and `aggregate_records`, with no system catalog. When a direct read-only SQL connection is available, prefer it for metadata.
