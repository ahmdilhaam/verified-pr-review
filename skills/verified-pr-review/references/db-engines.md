# Production queries per engine (read-only)

General rules, every engine:
- Read only: `SELECT` / metadata / execution plans. Open a read-only session if the client supports it.
- **Plan without executing** first. Variants that run the query (`EXPLAIN ANALYZE`, `ANALYZE`, `executionStats`, `STATISTICS PROFILE`) are only for queries already proven light.
- Set a per-query timeout if the engine supports it.
- Filter on indexed columns; wide aggregates across large tables can take minutes.
- Empty result → pair it with a control query using the same filter.

---

## MySQL / MariaDB / Percona

```sql
SELECT VERSION();                                  -- 8.x / 10.x; MariaDB reports "-MariaDB"
SHOW INDEX FROM <table>;                           -- Seq_in_index = column order in composite index
SELECT TABLE_NAME, TABLE_ROWS,                     -- TABLE_ROWS = InnoDB estimate
       ROUND(DATA_LENGTH/1024/1024)  AS data_mb,
       ROUND(INDEX_LENGTH/1024/1024) AS index_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME IN ('<t1>','<t2>');
EXPLAIN <query>;                                   -- does not execute; FORMAT=JSON for detail
```
- Reading `EXPLAIN`: `type=ALL` = full scan; `rows` = estimate (can be off by ~2×); `Using filesort` / `Using temporary` over hundreds of thousands of rows = slow.
- Timeout: MySQL 5.7.8+ `SELECT /*+ MAX_EXECUTION_TIME(30000) */ ...`; MariaDB 10.1+ `SET STATEMENT max_statement_time=30 FOR SELECT ...`.
- **Executes the query:** MySQL 8.0.18+ `EXPLAIN ANALYZE`, MariaDB `ANALYZE <query>`.
- Window functions & CTEs: MySQL 8+, MariaDB 10.2+. On older versions use derived subqueries.
- Namespace strings in polymorphic columns are written `'App\\Models\\X'`.
- phpMyAdmin shows the duration ("Query took N seconds") — ask the user to include it.

## PostgreSQL

```sql
SELECT version();
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = '<table>';
SELECT relname, n_live_tup,
       pg_size_pretty(pg_relation_size(relid))        AS data,
       pg_size_pretty(pg_indexes_size(relid))         AS indexes,
       pg_size_pretty(pg_total_relation_size(relid))  AS total
FROM pg_stat_user_tables WHERE relname IN ('<t1>','<t2>');
SELECT reltuples::bigint FROM pg_class WHERE relname = '<table>';   -- fast estimate
EXPLAIN <query>;                                   -- without ANALYZE = not executed
```
- Safe session: `BEGIN READ ONLY; SET LOCAL statement_timeout = '30s'; ...; ROLLBACK;`
- Reading `EXPLAIN`: `Seq Scan` on a large table, `Sort` with large `rows`, `Nested Loop` with an inner `Seq Scan` = trouble.
- Column selectivity without querying data: `SELECT attname, n_distinct, most_common_vals FROM pg_stats WHERE tablename='<table>';`
- Real slow queries (if the extension is enabled): `SELECT query, calls, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 20;`
- **Executes the query:** `EXPLAIN (ANALYZE, BUFFERS)`.

## SQLite

```sql
SELECT sqlite_version();
PRAGMA index_list('<table>');
PRAGMA index_info('<index_name>');
PRAGMA page_count; PRAGMA page_size;               -- file size = page_count × page_size
SELECT name, SUM(pgsize) FROM dbstat GROUP BY name ORDER BY 2 DESC;  -- if dbstat is compiled in
EXPLAIN QUERY PLAN <query>;                        -- does not execute
```
- Reading: `SCAN <table>` = full scan; `SEARCH ... USING INDEX` = good; `USE TEMP B-TREE FOR ORDER BY` = sort.
- Run against a copy of the DB file when possible (`sqlite3 copy.db`), not the file the app is using.
- No built-in timeout; limit with `LIMIT` and filters.

## SQL Server

```sql
SELECT @@VERSION;
SELECT i.name, i.type_desc, c.name AS col, ic.key_ordinal
FROM sys.indexes i
JOIN sys.index_columns ic ON ic.object_id = i.object_id AND ic.index_id = i.index_id
JOIN sys.columns c ON c.object_id = ic.object_id AND c.column_id = ic.column_id
WHERE i.object_id = OBJECT_ID('<schema.table>') ORDER BY i.name, ic.key_ordinal;
EXEC sp_spaceused '<schema.table>';
SET SHOWPLAN_XML ON;   -- estimated plan, query is NOT executed
GO
<query>
GO
SET SHOWPLAN_XML OFF;
```
- Reading: `Table Scan` / `Clustered Index Scan` on a large table, `Sort`, repeated `Key Lookup` = trouble.
- Timeout is set in the client (SSMS/Azure Data Studio: query timeout). Do not suggest `NOLOCK` as "safe" — results can be inconsistent.
- **Executes the query:** `SET STATISTICS IO, TIME ON` / "Include Actual Execution Plan".

## MongoDB

```js
db.version()
db.<collection>.getIndexes()
db.<collection>.estimatedDocumentCount()              // from metadata, fast
db.<collection>.aggregate([{ $collStats: { storageStats: {} } }])   // size & indexes
db.<collection>.find(<filter>).sort(<sort>).explain("queryPlanner")  // does not execute
```
- Reading: `COLLSCAN` = full scan; `IXSCAN` = uses an index; in-memory `SORT` on large data = trouble.
- Timeout: `.maxTimeMS(30000)` on `find`/`aggregate`.
- Avoid unfiltered `countDocuments({})` on large collections; use `estimatedDocumentCount`.
- **Executes the query:** `explain("executionStats")` / `explain("allPlansExecution")`.

## Other engines

Find the equivalent of four things: version, index list, table size from metadata, and an execution plan without executing. If there is no "without executing" equivalent, give a bounded query (`LIMIT`, one tenant) and tell the user the risk.
