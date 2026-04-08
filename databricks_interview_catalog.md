# Databricks Senior/Staff Interview Q&A Catalog

> **Source**: Ansh Lamba YouTube videos (2025/2026), Databricks documentation, and senior-level engineering knowledge.
> **Videos Referenced**:
> - [3:23] Databricks X PySpark Interview Questions 2026 Guide (youtube.com/watch?v=__9tqYjEJhE)
> - [2:33] Azure Databricks Interview Questions 2025 (youtube.com/watch?v=njE89SwtPYI)
> - [3:12] Databricks Data Engineer Interview ONE SHOT 2025 (youtube.com/watch?v=7ganPN2kqSI)
> - [3:05] The ONLY Databricks Real Time Scenarios 2025 (youtube.com/watch?v=KnpUDrKHtGo)
> - [2:42] Unity Catalog Tutorial For Data Engineering (youtube.com/watch?v=GJa4YH4j-ic)

**Target**: Senior/Staff Data Engineer & Databricks Admin roles
**Format**: Conceptual + Scenario-based, organized by domain

---

## TABLE OF CONTENTS

1. [Unity Catalog & Governance](#1-unity-catalog--governance)
2. [Delta Lake Internals](#2-delta-lake-internals)
3. [Cluster & Compute Management](#3-cluster--compute-management)
4. [Workflows & Jobs](#4-workflows--jobs)
5. [Cost Optimization](#5-cost-optimization)
6. [Security & Access Control](#6-security--access-control)
7. [Performance Tuning & Spark Optimization](#7-performance-tuning--spark-optimization)
8. [Spark Internals](#8-spark-internals)
9. [Autoloader & Streaming](#9-autoloader--streaming)
10. [Scenario-Based Questions](#10-scenario-based-questions)

---

## 1. Unity Catalog & Governance

### Q1. What is Unity Catalog and why was it introduced?
**Answer:**
Unity Catalog (UC) is Databricks' unified governance solution for data and AI assets across all workspaces in an account. It was introduced to solve the fragmentation problem where each Databricks workspace had its own Hive metastore (local), leading to:
- No cross-workspace data sharing without manual duplication
- No centralized access control — permissions were workspace-local
- No fine-grained column/row-level security
- No lineage tracking across workspaces

**Key capabilities:**
- Centralized metastore shared across multiple workspaces
- Three-level namespace: `catalog.schema.table`
- Fine-grained access control (catalog, schema, table, column, row)
- Data lineage (automated column-level lineage)
- Audit logging via system tables
- Support for external locations, storage credentials, and Delta Sharing

---

### Q2. Explain the Unity Catalog object hierarchy.
**Answer:**
```
Account
  └── Metastore (one per region, linked to cloud storage)
        └── Catalog  (logical container, like a database cluster)
              └── Schema  (equivalent to a database/namespace)
                    ├── Table (managed or external)
                    ├── View
                    ├── Volume (files/unstructured data)
                    ├── Function (UDFs, Python/SQL)
                    └── Model (ML models via MLflow)
```

**Key points:**
- A **Metastore** is the top-level container. You get one per region per Databricks account.
- A **Catalog** is a logical grouping — use it to separate environments (dev, prod) or business domains (finance, marketing).
- **Managed tables** store data in the metastore's root storage; dropping the table drops the data.
- **External tables** point to storage you own; dropping the table does NOT delete the underlying data.
- **Volumes** are the UC equivalent of external locations for unstructured data (CSV, images, PDFs, etc.) — replacing `dbfs:/mnt/` patterns.

---

### Q3. What is a Metastore in Unity Catalog and how does it differ from the legacy Hive metastore?
**Answer:**

| Aspect | Legacy Hive Metastore | Unity Catalog Metastore |
|---|---|---|
| Scope | Per workspace | Per region, per Databricks account |
| Namespace | Two-level: `schema.table` | Three-level: `catalog.schema.table` |
| Access control | Workspace-level ACLs (TABLE ACL, incomplete) | Fine-grained: column, row, tag-based |
| Cross-workspace sharing | Not native | Native via metastore attachment |
| Lineage | None | Automated column-level lineage |
| Audit | Workspace audit logs only | System tables with full audit trail |
| Delta Sharing | Not supported | Built-in |
| Storage | Managed by Databricks per workspace | Customer-controlled root storage |

A single Unity Catalog metastore can be **attached to multiple workspaces** in the same region. This is a fundamental architectural shift.

---

### Q4. What are Storage Credentials and External Locations in Unity Catalog?
**Answer:**

**Storage Credential**: An object that wraps a cloud IAM identity (AWS IAM role, Azure managed identity, GCP service account) used by Unity Catalog to access cloud storage. You create it once and reference it from External Locations.

**External Location**: Maps a cloud storage path (e.g., `s3://my-bucket/my-path/`) to a Unity Catalog object using a Storage Credential. Once created, you grant access to users/groups via `GRANT READ FILES ON EXTERNAL LOCATION` rather than giving them direct cloud IAM permissions.

**Flow:**
```
User → External Location → Storage Credential → Cloud Storage
```

**Why this matters:**
- Users never need direct cloud IAM access — all storage access is mediated by UC
- You can grant/revoke storage access without touching IAM policies
- Path-level scoping: different teams can access different prefixes under the same bucket

**Admin scenario**: You have 3 teams needing access to `s3://datalake/team-a/`, `s3://datalake/team-b/`, `s3://datalake/team-c/`. You create one storage credential (IAM role with access to whole bucket), then three External Locations scoped to each team's prefix, then grant each team READ FILES on their External Location only.

---

### Q5. How does Unity Catalog handle fine-grained access control? Walk through the privilege model.
**Answer:**

UC uses a **DAC (Discretionary Access Control)** model with inheritance flowing down the hierarchy.

**Securable objects** (things you can grant privileges on):
- Metastore, Catalog, Schema, Table, View, Volume, External Location, Storage Credential, Connection, Share, Recipient

**Key privileges:**
| Object | Privilege | Effect |
|---|---|---|
| Catalog | `USE CATALOG` | Required to access anything inside |
| Schema | `USE SCHEMA` | Required to access tables/views inside |
| Table | `SELECT` | Read data |
| Table | `MODIFY` | Insert/update/delete |
| Table | `ALL PRIVILEGES` | Full control |
| External Location | `READ FILES` | Read files at that path |
| External Location | `WRITE FILES` | Write files at that path |
| Metastore | `CREATE CATALOG` | Create new catalogs |

**Inheritance**: Granting `SELECT` on a catalog grants it on ALL schemas and tables in that catalog — unless overridden.

**Row/Column level security**: Implemented via **Dynamic Views** in UC:
```sql
CREATE VIEW secure_view AS
SELECT * FROM raw_table
WHERE
  CASE WHEN is_account_group_member('finance') THEN TRUE
       ELSE customer_region = 'US'
  END;
```

**Column masking** (native UC feature): Apply masking policies directly to table columns, executed at query time.

---

### Q6. What are Delta Sharing and how does it integrate with Unity Catalog?
**Answer:**

**Delta Sharing** is an open protocol (Linux Foundation) for sharing live Delta Lake data with external recipients — across clouds, platforms, or organizations — **without copying data**.

**UC Integration:**
- In UC, you create a **Share** (a logical collection of tables/partitions/volumes)
- You create a **Recipient** (the external consumer, identified by a token or Unity Catalog identity)
- You grant the recipient access to the share

**Workflow:**
```
Provider side (your Databricks):
  1. CREATE SHARE finance_share
  2. ALTER SHARE finance_share ADD TABLE catalog.schema.sales_table
  3. CREATE RECIPIENT external_partner
  4. GRANT SELECT ON SHARE finance_share TO RECIPIENT external_partner

Recipient side:
  - Can be: Databricks, pandas (delta-sharing Python client), Spark (open source), PowerBI, etc.
  - Gets a credentials file with the sharing URL + token
  - No data is moved — they query directly from your storage via a sharing server
```

**Key benefits**: Live data (no export), cloud-agnostic, no vendor lock-in, access revocable instantly.

**Admin scenario**: Your org sells data products. Instead of S3 bucket exports, you share a Delta Sharing link. When the contract ends, you revoke the recipient — no data to clean up.

---

### Q7. Explain Unity Catalog lineage. What does it track and how is it captured?
**Answer:**

UC **automatically** captures lineage without any instrumentation — it hooks into the Spark execution engine.

**What it tracks:**
- **Table-level lineage**: Which tables were read/written in a query
- **Column-level lineage**: Which source columns contributed to which target columns (for SQL queries and Spark DataFrames)
- **Notebook/Job context**: Which notebook, job, or user triggered the lineage event

**How to access:**
1. UI: Table details → Lineage tab (shows upstream/downstream graph)
2. System tables: `system.access.table_lineage` and `system.access.column_lineage`

**Limitations:**
- Python UDFs break column-level lineage (column mapping can't be inferred)
- Lineage only captured when compute uses Unity Catalog (not legacy Hive metastore clusters)
- External tools writing directly to storage (not via Spark/SQL) don't generate lineage

**Senior-level nuance**: Lineage is eventual — it's written asynchronously. There can be a few minutes delay. For compliance purposes (GDPR right to erasure), lineage lets you quickly identify all tables containing a customer's data.

---

### Q8. How do you migrate from a Hive metastore to Unity Catalog?
**Answer:**

**Migration paths:**
1. **SYNC command** (HMS to UC sync) — `CREATE TABLE ... CLONE` for managed tables
2. **`UPGRADE TABLE` command** (in-place for Hive metastore external tables)
3. **Databricks UC Migration Tool** — automated bulk migration with mapping rules

**Step-by-step approach:**
```
1. Prerequisites:
   - Enable Unity Catalog for the account
   - Create a metastore and attach to workspace
   - Create root storage (ADLS Gen2 / S3 bucket for UC)
   - Create storage credentials and external locations mirroring old mounts

2. Assessment:
   - Inventory Hive metastore tables: databases, tables, views, permissions
   - Identify managed vs external tables
   - Identify mount points (dbfs:/mnt/) and map to External Locations

3. Migration:
   - Create target catalog + schemas in UC
   - For external tables: SYNC or UPGRADE TABLE
   - For managed tables: DEEP CLONE to new UC location
   - Recreate views pointing to new UC tables
   - Migrate table-level permissions to UC grants

4. Validation:
   - Row count checks
   - Schema validation
   - Permission smoke tests

5. Cutover:
   - Update notebooks/jobs to use 3-level namespace
   - Deprecate Hive metastore bindings
   - Remove legacy mount points
```

**Common pitfall**: `dbfs:/mnt/` paths become `Volumes` in UC. Any code referencing mount paths must be updated to use `/Volumes/catalog/schema/volume_name/` paths.

---

### Q9. What are Volumes in Unity Catalog and when would you use them over managed tables?
**Answer:**

**Volumes** are UC objects for managing access to **files** (non-tabular/unstructured data) stored in cloud object storage. They provide the same governance (access control, lineage, audit) as tables, but for raw files.

**Two types:**
- **Managed Volume**: UC manages the storage location under the metastore root storage
- **External Volume**: Points to a customer-managed cloud path via an External Location

**Use Volumes when:**
- Storing raw input files (CSV, JSON, images, PDFs) before transformation
- ML model artifacts, checkpoint files
- Config files, lookup files
- Replacing `dbfs:/mnt/` or `dbfs:/FileStore/` patterns

**Access syntax:**
```python
# Writing
df.write.csv("/Volumes/catalog/schema/my_volume/output/")

# Reading
df = spark.read.csv("/Volumes/catalog/schema/my_volume/input/data.csv")
```

**vs. Managed Tables:**
- Tables: structured, queryable with SQL, ACID guarantees via Delta
- Volumes: unstructured files, accessed via file APIs, no SQL query support

---

### Q10. A team in your organization wants to give a vendor read access to a specific table without exposing your entire data lake. How do you architect this in Unity Catalog?
**Answer (Scenario):**

```
Architecture:

1. Create a dedicated share catalog (optional, for isolation):
   CREATE CATALOG vendor_share_catalog;

2. Create a view exposing only approved columns (data minimization):
   CREATE VIEW vendor_share_catalog.default.approved_sales AS
   SELECT order_id, product_category, sale_date, region
   FROM prod.sales.transactions
   WHERE data_classification != 'PII';

3. Use Delta Sharing for external vendor access:
   CREATE SHARE vendor_acme_share;
   ALTER SHARE vendor_acme_share ADD TABLE vendor_share_catalog.default.approved_sales;
   CREATE RECIPIENT acme_corp;
   GRANT SELECT ON SHARE vendor_acme_share TO RECIPIENT acme_corp;

4. Audit:
   Monitor access via system.access.audit table
   Set expiration on the recipient credential

5. Revocation:
   DROP RECIPIENT acme_corp;  -- instant access termination
```

**Why this is better than S3 bucket access**: Revocable in seconds, no data duplication, vendor never touches your cloud IAM, full audit trail in UC system tables.

---


---

## 2. Delta Lake Internals

### Q11. What is Delta Lake and what problems does it solve over vanilla Parquet/Hive?
**Answer:**

Delta Lake is an open-source storage layer built on Parquet that adds **ACID transactions**, **schema enforcement**, **versioning**, and **unified batch+streaming** to data lakes.

**Problems with raw Parquet/Hive:**
| Problem | Delta Lake Solution |
|---|---|
| No ACID transactions — partial writes leave corrupt state | Write-ahead transaction log (delta log) ensures atomicity |
| No schema enforcement — bad data silently ingested | Schema enforcement on write + schema evolution support |
| No update/delete (immutable Parquet files) | Merge, Update, Delete operations via log-based mutations |
| Slow metadata operations at scale (listing millions of files) | Stats-based file skipping via transaction log |
| No versioning/time travel | Immutable log with snapshot isolation |
| Streaming and batch can't share same table easily | Unified table format — same table read by both |

---

### Q12. Explain the Delta Lake transaction log (_delta_log) in detail.
**Answer:**

The `_delta_log/` directory is the heart of Delta Lake. It contains a sequence of **JSON commit files** (and periodic **Parquet checkpoint files**) that record every operation on the table.

**Structure:**
```
my_delta_table/
├── _delta_log/
│   ├── 00000000000000000000.json   ← initial commit
│   ├── 00000000000000000001.json   ← second commit
│   ├── ...
│   ├── 00000000000000000010.checkpoint.parquet  ← checkpoint every 10 commits
│   └── _last_checkpoint              ← pointer to latest checkpoint
├── part-00000-xxx.parquet
├── part-00001-xxx.parquet
```

**Each commit JSON contains:**
- `add` actions: new Parquet files being added, with stats (min/max/nullCount per column)
- `remove` actions: files being logically deleted (not physically removed yet)
- `commitInfo`: metadata about the operation (type, timestamp, user, metrics)
- `protocol`: reader/writer version requirements
- `metaData`: schema, partition columns, configuration

**Checkpoint files**: Every 10 commits (by default), Delta consolidates all active `add` actions into a Parquet checkpoint. This avoids replaying thousands of JSON files on cold reads.

**Reading a Delta table:**
1. Find latest checkpoint (from `_last_checkpoint`)
2. Apply all JSON commits since the checkpoint
3. Result = current snapshot (list of active Parquet files)

---

### Q13. How does Delta Lake achieve ACID transactions? Explain optimistic concurrency control.
**Answer:**

Delta Lake uses **optimistic concurrency control (OCC)** — it assumes conflicts are rare and only checks for conflicts at commit time.

**Write protocol:**
```
1. Read current table version (snapshot N)
2. Perform transformation, write new Parquet files to staging area
3. Attempt to commit: write commit file N+1.json
4. If another writer already wrote N+1.json → conflict detected
5. Retry: re-read current snapshot, check if conflict is real:
   - Did the other writer touch the same data I read/wrote?
   - If no overlap → can safely retry commit as N+2
   - If overlap → abort and surface error to application
```

**Isolation levels:**
- **Serializable** (default for DML): strongest, no phantom reads
- **WriteSerializable**: weaker, allows concurrent appends — good for streaming ingestion

**Atomicity**: Either all Parquet files from a commit are visible (commit JSON written) or none are. If a job dies mid-write, the uncommitted Parquet files are simply orphaned and never referenced in the log — they get cleaned up by VACUUM.

---

### Q14. What is Z-Ordering and when should you use it? How does it differ from partitioning?
**Answer:**

**Z-Ordering** is a multi-dimensional clustering technique that co-locates related data within the same Parquet files to maximize data skipping efficiency.

```sql
OPTIMIZE delta.`/path/to/table` ZORDER BY (customer_id, event_date);
```

**How it works:**
- Takes a set of columns and uses a space-filling Z-curve to interleave their values
- Files are rewritten so that rows with similar values across ALL Z-order columns end up in the same files
- Delta stores per-file min/max statistics → queries with filters on Z-order columns skip most files

**Z-Order vs Partitioning:**
| Aspect | Partitioning | Z-Ordering |
|---|---|---|
| Mechanism | Physical directory split | File-level data clustering |
| Best for | Low-cardinality columns (date, region) | High-cardinality columns (user_id, device_id) |
| Risk | Small file problem if cardinality too high | None — files stay at target size |
| Predicate pushdown | Directory pruning | File-level min/max skipping |
| Columns | One dimension at a time | Multi-dimensional |

**When to use Z-Order:**
- Columns with high cardinality used in frequent WHERE filters
- Columns that would create too many small files if partitioned (user_id, session_id)
- Analytics queries that filter on 2-3 dimensions simultaneously

**Limitation**: Z-ordering is not cumulative — new data appended without OPTIMIZE doesn't benefit until the next OPTIMIZE run.

---

### Q15. Explain Time Travel in Delta Lake. What are its practical use cases and how does it work internally?
**Answer:**

**Time Travel** lets you query historical versions of a Delta table.

**Syntax:**
```sql
-- By version
SELECT * FROM my_table VERSION AS OF 5;

-- By timestamp
SELECT * FROM my_table TIMESTAMP AS OF '2025-01-15 10:00:00';

-- In Python
df = spark.read.format("delta").option("versionAsOf", 5).load("/path/to/table")
df = spark.read.format("delta").option("timestampAsOf", "2025-01-15").load("/path/to/table")
```

**Internals:**
- Each version = a specific set of `add` actions from the transaction log
- To read version N: take the checkpoint ≤ N, apply commits up to N, get file list
- The actual Parquet files are NOT deleted until VACUUM runs — so historical reads work as long as files exist

**Practical use cases:**
1. **Data audit/compliance**: "What did this customer's record look like on Jan 1st?"
2. **Bug recovery**: "A bad job ran at 3pm — restore to 2:59pm version"
3. **ML reproducibility**: Pin training datasets to a specific version
4. **Incremental processing**: Use version diff to get only changed rows since last run
5. **Debugging**: Compare current data vs previous version to understand what changed

**Retention**: Controlled by `delta.logRetentionDuration` (default 30 days) and `delta.deletedFileRetentionDuration` (default 7 days). VACUUM removes files older than the retention threshold.

---

### Q16. What is the OPTIMIZE command and when should you run it?
**Answer:**

`OPTIMIZE` rewrites small Delta Lake files into larger ones (default target size: 128 MB) to reduce the small-file problem and improve read performance.

```sql
OPTIMIZE my_table;
OPTIMIZE my_table WHERE date = '2025-01-01';  -- partition-targeted
OPTIMIZE my_table ZORDER BY (customer_id);    -- with clustering
```

**When to run:**
- After streaming ingestion (which creates many small files)
- After frequent MERGE operations (which produce many small files)
- Scheduled daily/weekly as maintenance

**Auto Optimize (Databricks feature):**
```sql
ALTER TABLE my_table SET TBLPROPERTIES (
  'delta.autoOptimize.optimizeWrite' = 'true',   -- coalesce on write
  'delta.autoOptimize.autoCompact' = 'true'       -- compact after write
);
```

- **Optimize Write**: Databricks automatically adjusts the number of output files during writes (reshuffles data to write ~128MB files). No explicit OPTIMIZE needed.
- **Auto Compact**: After a write, if enough small files exist, triggers a background OPTIMIZE.

**Liquid Clustering (new in DBR 13.3+)**: Replaces ZORDER + partitioning with a more flexible online clustering approach. Use `CLUSTER BY` instead of ZORDER — incrementally clusters only new/changed data.

---

### Q17. What is the MERGE operation in Delta Lake? Explain its internals and gotchas.
**Answer:**

MERGE (upsert) allows you to atomically insert, update, or delete rows based on a match condition.

```sql
MERGE INTO target t
USING source s ON t.id = s.id
WHEN MATCHED AND s.is_deleted = true THEN DELETE
WHEN MATCHED THEN UPDATE SET t.value = s.value, t.updated_at = s.updated_at
WHEN NOT MATCHED THEN INSERT (id, value, updated_at) VALUES (s.id, s.value, s.updated_at);
```

**Internals:**
1. Spark joins source and target on the merge condition
2. Matched rows in target: rewrite affected Parquet files with updates/deletes
3. Unmatched rows from source: write new Parquet files (inserts)
4. Atomic commit: log entry marks old files as `remove`, new files as `add`

**Performance gotchas:**
- MERGE is expensive — it rewrites affected files even if only 1 row changes
- On a large target table, the join can shuffle massive data
- **Optimization**: Use a partition filter in the MERGE condition to limit files scanned:
  ```sql
  MERGE INTO target t
  USING source s ON t.id = s.id AND t.date = s.date  -- limits file scanning
  ```
- **Photon**: Databricks Photon engine significantly accelerates MERGE via native vectorized execution
- Consider **insert-only append + periodic compaction** pattern if update rate is low

**WHEN NOT MATCHED BY SOURCE** (Delta 2.0+): Handles rows in target that have no match in source (useful for full-table syncs where deletions in source should delete from target).

---

### Q18. Explain VACUUM. What does it do, what are the risks, and what is the default retention period?
**Answer:**

`VACUUM` physically deletes Parquet files from storage that are no longer referenced in the Delta log and are older than the retention threshold.

```sql
VACUUM my_table;                     -- uses default retention (7 days)
VACUUM my_table RETAIN 168 HOURS;   -- retain 7 days explicitly
VACUUM my_table DRY RUN;             -- shows files that WOULD be deleted
```

**Default retention**: 7 days (`delta.deletedFileRetentionDuration = interval 7 days`)

**Risk — breaking Time Travel:**
- VACUUM permanently deletes files used by historical versions
- After VACUUM, you can only time travel within the remaining retention window
- **Never set retention below 7 days** — Delta internally needs some overlap for concurrent readers

**Risk — active streaming:**
- Streaming jobs may hold references to old files (via stream checkpoints)
- VACUUM can delete files a stream hasn't consumed yet → stream failure
- Mitigation: ensure `deletedFileRetentionDuration` >= your stream's maximum lag

**Risk — shallow clones:**
- Shallow clones reference the source table's files
- VACUUMing the source can break the clone
- Use DEEP CLONE for isolated copies

**Best practice**: Never set retention < 7 days. Use `DRY RUN` before first VACUUM on a table. Monitor table size vs. growth rate to schedule appropriately.

---

### Q19. What is Change Data Feed (CDF) in Delta Lake and how do you use it for incremental processing?
**Answer:**

**Change Data Feed** (CDF) records row-level changes (insert, update, delete) in a Delta table, making it easy to build incremental pipelines downstream.

**Enable:**
```sql
ALTER TABLE my_table SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
-- or at creation:
CREATE TABLE my_table (...) TBLPROPERTIES (delta.enableChangeDataFeed = true);
```

**Read changes:**
```python
# Read changes since version 5
changes = spark.read.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingVersion", 5) \
    .table("my_table")

# Or since a timestamp
changes = spark.read.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingTimestamp", "2025-01-01") \
    .table("my_table")
```

**Output columns added:**
- `_change_type`: `insert`, `update_preimage`, `update_postimage`, `delete`
- `_commit_version`: which version this change belongs to
- `_commit_timestamp`: when the change was committed

**Typical pattern:**
```python
# Streaming incremental pipeline: silver → gold
changes = spark.readStream.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingVersion", "latest") \
    .table("silver.events")

# Only process inserts and post-images of updates
filtered = changes.filter("_change_type IN ('insert', 'update_postimage')")
```

**Cost**: CDF adds ~2-3x storage overhead (keeps extra copies of changed rows). Only enable on tables with active downstream consumers.

---

### Q20. What is Liquid Clustering and how does it differ from traditional partitioning + Z-Order?
**Answer:**

**Liquid Clustering** (introduced in Databricks Runtime 13.3, GA in 14.x) is a next-generation data layout technique that replaces the combination of `PARTITIONED BY` + `ZORDER BY`.

```sql
CREATE TABLE my_table (id BIGINT, event_date DATE, region STRING)
CLUSTER BY (event_date, region);
```

**Key differences:**

| Feature | Partitioning + ZOrder | Liquid Clustering |
|---|---|---|
| When clustering happens | OPTIMIZE (manual/scheduled) | Incremental, online — each write clusters new data |
| Metadata | Directory structure | File-level statistics only |
| Flexibility | Partition columns fixed at creation | `CLUSTER BY` columns can be changed |
| Small file risk | High with high-cardinality partitions | Managed automatically |
| Read performance | Good after OPTIMIZE | Good incrementally, improves continuously |
| Writes | Can create small files | Auto-coalesces |

**When to choose Liquid Clustering:**
- You have multiple high-cardinality dimensions to filter on
- Your access patterns change over time (you can re-cluster without rewriting all data)
- You want to avoid the operational overhead of scheduling OPTIMIZE + ZORDER jobs
- Tables with frequent streaming appends

---

## 3. Cluster & Compute Management

### Q21. What are the different cluster types in Databricks and when do you use each?
**Answer:**

**1. All-Purpose Clusters (Interactive)**
- Started manually, persist until terminated
- Used for: notebooks, development, ad-hoc exploration, ML experiments
- Shared by multiple users simultaneously
- More expensive (billed while running even if idle)
- Support cluster access mode: Single User, Shared (UC), No Isolation Shared

**2. Job Clusters (Automated)**
- Spun up for a specific job run, terminated automatically when the job finishes
- Used for: production ETL pipelines, scheduled jobs, CI/CD
- Not shareable — dedicated to a single job run
- More cost-efficient (no idle time)
- Cannot be attached to a notebook manually

**3. SQL Warehouses (formerly SQL Endpoints)**
- Purpose-built for SQL analytics (Databricks SQL)
- Serverless or Classic; auto-start/stop; concurrent query execution
- Photon-accelerated by default
- Billed in DBU/hour based on warehouse size (2X-Small through 4X-Large)
- Used for: BI tools, dashboards, SQL analysts

**4. Serverless Compute** (GA in 2024)
- No cluster management — compute provisioned instantly by Databricks
- Available for: notebooks, jobs, SQL warehouses, Delta Live Tables
- Zero startup time (warm pool maintained by Databricks)
- Best for: bursty workloads, teams that don't want to manage clusters

---

### Q22. Explain Databricks cluster access modes. What is the difference between Single User, Shared, and No Isolation Shared?
**Answer:**

**Single User Mode**
- Cluster is assigned to exactly one user
- Full isolation — no other user can attach
- Required for: Python UDFs, arbitrary library installs, MLflow experiments with certain APIs
- Unity Catalog compatible
- Supports all languages (Python, Scala, R, SQL)

**Shared Mode (Unity Catalog)**
- Multiple users can share the cluster simultaneously
- User-level isolation via process separation — each user's code runs in a separate process
- Unity Catalog enforced — table permissions apply per user
- Restrictions: no arbitrary init scripts, no custom JARs, limited Spark configs per user
- Supported runtimes: DBR 11.3+
- Best for: cost-sharing across multiple data analysts/engineers in a governed environment

**No Isolation Shared** (legacy, not UC compatible)
- Multiple users share, NO process isolation
- Any user can see other users' variables, secrets (if leaked in code), data
- Not recommended for production; being deprecated
- Use only if you need legacy libraries that don't work in Shared mode

**Admin decision guide:**
- Production pipelines → Job cluster, Single User
- Team of analysts in UC environment → Shared mode cluster
- Individual power user / ML engineer → Single User
- Ad-hoc BI queries → SQL Warehouse

---

### Q23. How does Databricks autoscaling work? What are the limitations of autoscaling for streaming workloads?
**Answer:**

**How standard autoscaling works:**
- You set `min_workers` and `max_workers`
- Databricks monitors cluster utilization (CPU, shuffle, pending tasks)
- Scale-up: if tasks are queued for > 60 seconds, add workers (up to max)
- Scale-down: if a node has been idle for `spark.databricks.aggressiveWindowDownS` seconds (default 600s), remove it
- Scale-down is gradual — removes one node at a time to avoid disrupting running tasks

**Enhanced Autoscaling (for streaming)**
- Standard autoscaling is not ideal for streaming because:
  - Streaming jobs continuously process micro-batches — no natural "idle" periods
  - Adding nodes triggers a mini-rebalance and can cause processing delays
  - Removing a node can cause a lost executor and force task re-execution
- **Enhanced Autoscaling** (available for Structured Streaming on Databricks):
  - Looks at streaming lag metrics (backlog size) rather than task queuing
  - Scales up when lag is growing, scales down when lag is cleared
  - Uses **graceful decommission** — the node finishes its current micro-batch before leaving
  - Enable via: `spark.databricks.streaming.enhancedAutoscaling.enabled = true`

**Limitations of autoscaling:**
- Cold start: adding a node takes 1-3 minutes — not suitable for sub-second SLA workloads
- Scale-down can be slow (conservative to avoid data loss)
- Doesn't work well for single-node workloads (no room to scale down)
- For very large shuffles, scaling mid-job causes shuffle data re-read

---

### Q24. What is the difference between spot/preemptible instances and on-demand instances? How do you use them safely?
**Answer:**

**On-Demand instances:**
- Guaranteed availability, higher cost
- Never terminated by the cloud provider
- Use for: driver node, stateful workloads, production pipelines with strict SLAs

**Spot instances (AWS) / Preemptible VMs (GCP) / Spot VMs (Azure):**
- Up to 90% cheaper than on-demand
- Can be terminated by the cloud provider with 2-minute warning when capacity is needed
- Use for: worker nodes, stateless transformations, batch jobs with retry logic

**Recommended architecture: Spot workers + On-demand driver**
```
Cluster config:
  driver: on-demand (never lose the coordinator)
  workers: spot instances (cheap; if evicted, Spark re-runs the task on another worker)
  
  fallback_to_on_demand: true  ← Databricks handles automatic fallback if spot unavailable
```

**When NOT to use spot for workers:**
- Streaming jobs where losing a worker stalls the micro-batch
- Jobs with very long individual tasks (eviction = re-run the entire long task)
- Jobs writing to non-idempotent sinks (eviction + retry = duplicate writes)

**Instance pools**: Pre-allocate a pool of idle instances. Clusters draw from the pool instead of launching new VMs → eliminates cold start time. Critical for job clusters that need fast startup.

---

### Q25. What are Instance Pools? Why and when would an admin use them?
**Answer:**

**Instance Pools** maintain a set of idle, ready-to-use cloud VM instances. When a cluster is created from a pool, it uses pre-allocated VMs instead of requesting new ones from the cloud → **drastically reduces cluster start time** (from 3-5 minutes to 30-60 seconds).

**Why admins use them:**
1. **Job cluster start time**: Production jobs use job clusters that spin up per run. Without a pool, each run waits 3-5 minutes for VMs. With a pool: 30-60 seconds.
2. **Cost predictability**: Pre-allocate the instance type and quantity you know you'll need.
3. **Spot instance management**: Pool maintains a buffer of spot instances. If the cloud reclaims one, the pool replenishes it proactively.
4. **Governance**: Admins can restrict clusters to use only approved instance types via pools.

**Configuration:**
```json
{
  "instance_pool_name": "prod-job-pool",
  "node_type_id": "m5.xlarge",
  "min_idle_instances": 5,
  "max_capacity": 50,
  "idle_instance_autotermination_minutes": 60,
  "preloaded_spark_versions": ["14.3.x-scala2.12"]
}
```

**`preloaded_spark_versions`**: The DBR Docker image is pre-loaded onto pool instances → even faster cluster startup (eliminates image pull time).

**Cost**: Idle pool instances are billed at the VM rate (no DBU charge for idle instances).

---

### Q26. How do you configure and manage init scripts? What are the best practices?
**Answer:**

**Init scripts** run on each cluster node during startup, before Databricks Runtime is initialized. Used for: installing OS packages, custom libraries, modifying system configs.

**Types:**
| Type | Scope | Storage |
|---|---|---|
| Global init scripts | All clusters in workspace | Workspace admin UI |
| Cluster-scoped init scripts | Specific cluster | DBFS, Volumes, Workspace files |
| User-level (legacy) | Per-user clusters | DBFS only |

**Best practices:**
1. **Use Volumes (UC) instead of DBFS** for script storage — governed and audited
2. **Keep scripts idempotent** — script may run on each node restart
3. **Log output**: redirect to `/databricks/driver/logs/` so it appears in cluster event logs
4. **Fail fast**: If a required package fails to install, exit with non-zero code so the cluster fails to start (prevents silent misconfiguration)
5. **Avoid pip installs in init scripts** for large libraries — use pre-built Docker containers or instance pools with preloaded images instead (faster, more reproducible)
6. **Security**: Never embed secrets in init scripts; use Databricks Secrets API

```bash
#!/bin/bash
# Example: Install a system package
set -e  # exit on any error
apt-get install -y libgomp1 2>&1 | tee -a /tmp/init_script.log
echo "Init script completed successfully" >> /tmp/init_script.log
```

---

### Q27. A production job cluster is taking 8 minutes to start. How do you diagnose and reduce startup time?
**Answer (Scenario):**

**Step 1: Identify the bottleneck using Cluster Event Log:**
```
Workspace → Compute → [cluster] → Event Log
Look for timings between:
- "Starting" → "Pending" (cloud VM acquisition)
- "Pending" → "Running" (DBR image pull + init scripts)
```

**Common bottlenecks:**

| Bottleneck | Solution |
|---|---|
| VM acquisition (3-5 min) | Use Instance Pools |
| Docker image pull | Preload DBR version in pool config |
| Init script installs many packages | Move to Docker container cluster or pre-install on custom AMI/image |
| Init script installing large pip packages | Bundle into a custom Docker image; use `%pip install` in notebooks only for dev |
| Conda environment creation | Replace with pip + venv; Conda is much slower |
| EBS volume initialization (AWS) | Enable EBS optimized, or use local SSD instance types |

**Optimized setup:**
```
Instance Pool:
  - min_idle_instances: 3  (pre-warm for typical workload)
  - preloaded_spark_versions: [target DBR version]
  - instance_type: [standard worker type]

Job Cluster:
  - instance_pool_id: [pool above]
  - init_scripts: [] (moved all installs to Docker image)
  - docker_image: custom image with all dependencies
```

**Result**: Typical cold start goes from 8 minutes → 60-90 seconds.

---

---

## 4. Workflows & Jobs

### Q28. What is the difference between Databricks Workflows (Jobs) and Delta Live Tables Pipelines?
**Answer:**

| Aspect | Databricks Workflows (Jobs) | Delta Live Tables (DLT) |
|---|---|---|
| Orchestration model | Task-based DAG (sequential/parallel tasks) | Declarative pipeline (DLT manages execution order) |
| Code style | Imperative (you write the transformation logic and execution order) | Declarative (you define datasets, DLT infers dependencies) |
| Error handling | Retry policies per task; conditional branching | DLT handles retries and recovery automatically |
| Data quality | Manual assertions in code | Built-in `EXPECT` constraints with quarantine support |
| Table management | You manage Delta writes explicitly | DLT manages target tables automatically |
| Incremental loading | You implement with CDF, watermarks, or bookmarks | Built-in with `STREAMING TABLE` and `APPLY CHANGES` |
| Monitoring | Task-level run history | Pipeline event log, data quality metrics per dataset |
| Cost | Pay per cluster DBU | Pipeline-mode DBU pricing (slightly different) |
| Best for | Complex multi-step pipelines with diverse task types (Python, SQL, JAR, notebooks, dbt) | ETL-focused streaming/batch ingestion pipelines with data quality requirements |

**Recommendation**: Use DLT for ingestion pipelines (bronze → silver transformation with data quality). Use Workflows to orchestrate DLT pipelines alongside other tasks (ML training, notifications, exports).

---

### Q29. Explain Delta Live Tables (DLT). What are the key concepts: datasets, expectations, and pipeline modes?
**Answer:**

**DLT** is Databricks' declarative ETL framework built on top of Delta Lake and Spark Structured Streaming.

**Key concepts:**

**1. Datasets — three types:**
```python
import dlt
from pyspark.sql.functions import *

# STREAMING TABLE: incremental, append-only source
@dlt.table(name="bronze_events")
def bronze_events():
    return spark.readStream.format("cloudFiles") \
        .option("cloudFiles.format", "json") \
        .load("/Volumes/catalog/schema/raw/events/")

# MATERIALIZED VIEW (batch): re-computed on each pipeline run
@dlt.table(name="silver_events")
def silver_events():
    return dlt.read("bronze_events") \
        .filter(col("event_type").isNotNull()) \
        .withColumn("event_date", to_date("event_timestamp"))

# VIEW: not materialized, computed on-the-fly when referenced
@dlt.view(name="active_users_view")
def active_users_view():
    return dlt.read("silver_events").filter(col("is_active") == True)
```

**2. Expectations (Data Quality):**
```python
@dlt.table(
    name="silver_events",
    expect_all={"valid_event_type": "event_type IS NOT NULL",
                "valid_timestamp": "event_timestamp > '2020-01-01'"}
)
# or with actions:
@dlt.table(
    name="silver_events",
    expect_all_or_drop={"valid_id": "id IS NOT NULL"},        # drop bad rows
    expect_all_or_fail={"no_duplicates": "COUNT(*) = 1"}     # fail pipeline
)
```

**3. Pipeline modes:**
- **Triggered**: Runs once, processes all available data, then stops
- **Continuous**: Runs perpetually like a streaming job, processes as data arrives (near-real-time)

**4. APPLY CHANGES (CDC/SCD support):**
```sql
APPLY CHANGES INTO live.silver_customers
FROM stream(live.bronze_customers_cdc)
KEYS (customer_id)
SEQUENCE BY operation_timestamp
APPLY AS DELETE WHEN operation = 'DELETE';
```

---

### Q30. How do you implement conditional task execution and error branching in Databricks Workflows?
**Answer:**

Databricks Workflows supports conditional branching via **Task Dependencies** and **Run If conditions**.

**Run If conditions** (introduced in 2023):
```
Task A → Task B (run if: A succeeded)
       → Task C (run if: A failed)
       → Task D (run if: A was skipped OR succeeded)
```

**Available conditions:**
- `all_success`: All dependent tasks succeeded (default)
- `at_least_one_success`: At least one dependency succeeded
- `none_failed`: No dependency failed (skipped is OK)
- `all_done`: Run regardless of dependency status
- `at_least_one_failed`: Run if any dependency failed
- `all_failed`: Run only if all dependencies failed

**Practical pattern — alerting on failure:**
```
Extract Task → Transform Task (if Extract succeeded)
             → Failure Notification Task (if Extract failed)
             → Load Task (if Transform succeeded)
             → Success Notification Task (if Load succeeded)
             → Failure Notification Task (if Load failed)
```

**Retry policies:**
```json
{
  "max_retries": 3,
  "min_retry_interval_millis": 60000,
  "retry_on_timeout": true,
  "timeout_seconds": 3600
}
```

**For-each tasks** (Databricks 2024): Loop over an array of values and run a task for each — useful for processing partitions or customers in parallel:
```json
{
  "task_key": "process_regions",
  "for_each_task": {
    "inputs": "[\"US\", \"EU\", \"APAC\"]",
    "concurrency": 3,
    "task": {
      "task_key": "process_region",
      "notebook_task": {
        "notebook_path": "/Repos/team/process_region",
        "base_parameters": {"region": "{{input}}"}
      }
    }
  }
}
```

---

### Q31. How do you pass parameters between tasks in a Databricks Job?
**Answer:**

**Method 1: Task Values (recommended)**
```python
# Task A: set a value
from databricks.sdk.runtime import dbutils
dbutils.jobs.taskValues.set(key="record_count", value=df.count())

# Task B (downstream): get the value
count = dbutils.jobs.taskValues.get(
    taskKey="task_a_key",
    key="record_count",
    default=0,
    debugValue=100  # used in interactive notebook runs
)
```

**Method 2: Job Parameters (static at job definition)**
```python
# Defined in job UI or API as base_parameters
dbutils.widgets.get("start_date")
dbutils.widgets.get("environment")
```

**Method 3: Dynamic parameter references in job definition**
```json
{
  "base_parameters": {
    "date": "{{job.start_time.iso_date}}",
    "run_id": "{{job.run_id}}"
  }
}
```

**Available dynamic expressions:**
- `{{job.id}}`, `{{job.run_id}}`, `{{job.start_time.iso_date}}`
- `{{tasks.<task_key>.values.<key>}}` (reference task values in job config)

**Best practice**: Use Task Values for dynamic runtime data (record counts, file paths discovered at runtime). Use Job Parameters for static configuration (environment, date range).

---

### Q32. How do you monitor and alert on Databricks job failures at scale?
**Answer:**

**Native Databricks notifications:**
- Per-job email/webhook alerts on: start, success, failure, skipped
- Notification destinations: email, Slack (webhook), PagerDuty, Microsoft Teams

**System Tables (recommended for enterprise monitoring):**
```sql
-- Query job run history
SELECT 
  job_id, run_id, run_name, state, 
  start_time, end_time,
  DATEDIFF(second, start_time, end_time) AS duration_seconds
FROM system.lakeflow.job_runs
WHERE state = 'FAILED'
  AND start_time >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
ORDER BY start_time DESC;

-- Identify slowest jobs (SLA tracking)
SELECT job_id, AVG(DATEDIFF(second, start_time, end_time)) AS avg_duration
FROM system.lakeflow.job_runs
WHERE state = 'SUCCEEDED'
GROUP BY job_id ORDER BY avg_duration DESC;
```

**Integration with external monitoring:**
- Push metrics to Datadog/New Relic via webhook notifications
- Use Databricks REST API to poll run status and push to Prometheus
- Export system tables to external monitoring store

**Proactive alerting pattern:**
```python
# In a monitoring notebook (scheduled hourly):
long_running = spark.sql("""
  SELECT job_id, run_id, start_time
  FROM system.lakeflow.job_runs
  WHERE state = 'RUNNING'
    AND DATEDIFF(minute, start_time, CURRENT_TIMESTAMP()) > 120
""")

if long_running.count() > 0:
    # Send alert via webhook
    send_alert(long_running.collect())
```

---

## 5. Cost Optimization

### Q33. What are the main cost drivers in Databricks and how do you address each?
**Answer:**

**Cost = DBU × hourly rate + cloud VM cost**

**Main cost drivers and solutions:**

| Driver | Solution |
|---|---|
| Idle all-purpose clusters | Auto-termination (max 120 min idle); Cluster policies enforcing limits |
| Over-provisioned clusters | Right-size using Ganglia/Spark UI metrics; use autoscaling |
| Dev/test using production cluster types | Cluster policies forcing smaller instance types for dev |
| On-demand VMs for workers | Spot/preemptible instances for workers (60-90% savings) |
| SQL Warehouse left running | Auto-stop SQL warehouses (5-10 min idle timeout) |
| Unoptimized Spark jobs (extra shuffles, skew) | Performance tuning: broadcast joins, partition pruning, Adaptive Query Execution |
| Full table scans due to missing partitions/Z-order | OPTIMIZE ZORDER BY or Liquid Clustering on filter columns |
| VACUUM not run → growing storage | Regular VACUUM with appropriate retention |
| Too many small files → slow jobs → more compute time | Auto Optimize, OPTIMIZE scheduled jobs |
| Delta Lake bloat (CDF + frequent updates) | Monitor table size growth; tune CDF retention |

---

### Q34. What are Cluster Policies and how do you use them for cost governance?
**Answer:**

**Cluster Policies** restrict what cluster configurations users can create. Admins define allowed/default/forbidden values for cluster attributes.

**Use cases:**
1. Force auto-termination to prevent idle cluster costs
2. Limit maximum cluster size per team
3. Force spot instances for worker nodes
4. Prevent expensive GPU instances outside ML team
5. Standardize cluster tags for cost attribution

**Example policy:**
```json
{
  "autotermination_minutes": {
    "type": "fixed",
    "value": 60,
    "hidden": true
  },
  "num_workers": {
    "type": "range",
    "maxValue": 10
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["m5.large", "m5.xlarge", "m5.2xlarge"]
  },
  "aws_attributes.availability": {
    "type": "fixed",
    "value": "SPOT_WITH_FALLBACK"
  },
  "custom_tags.team": {
    "type": "fixed",
    "value": "{{user}}",
    "hidden": false
  }
}
```

**Policy families**: Databricks provides built-in policy families (Personal Compute, Power User, Job Compute) as starting points.

**Enforcement**: Policies are enforced at cluster creation time via workspace permissions. Users can only create clusters using policies they have access to.

---

### Q35. How do you use Databricks system tables for cost monitoring and chargeback?
**Answer:**

**Relevant system tables (in `system` catalog):**

```sql
-- DBU consumption by workspace/SKU
SELECT 
  workspace_id,
  sku_name,
  DATE(usage_start_time) AS usage_date,
  SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE usage_start_time >= DATEADD(month, -1, CURRENT_DATE())
GROUP BY workspace_id, sku_name, DATE(usage_start_time)
ORDER BY total_dbus DESC;

-- Cost by tag (chargeback to teams)
SELECT 
  custom_tags['team'] AS team,
  custom_tags['project'] AS project,
  SUM(usage_quantity) AS total_dbus,
  SUM(usage_quantity * list_price) AS estimated_cost_usd
FROM system.billing.usage
JOIN system.billing.list_prices USING (sku_name, cloud, usage_unit)
WHERE usage_start_time >= DATEADD(month, -1, CURRENT_DATE())
GROUP BY team, project
ORDER BY estimated_cost_usd DESC;

-- Identify expensive jobs
SELECT 
  j.run_name,
  SUM(b.usage_quantity) AS dbus,
  SUM(b.usage_quantity * p.list_price) AS cost
FROM system.billing.usage b
JOIN system.lakeflow.job_runs j ON b.usage_metadata.job_id = j.job_id
JOIN system.billing.list_prices p USING (sku_name)
GROUP BY j.run_name ORDER BY cost DESC LIMIT 20;
```

**Tagging strategy for chargeback:**
- Tag clusters with: `team`, `project`, `environment`, `cost_center`
- Enforce via Cluster Policies (fixed tag values)
- Build Databricks SQL dashboard on system tables for finance reporting

---

### Q36. What is Photon and when does it provide the most benefit?
**Answer:**

**Photon** is Databricks' native vectorized query engine written in C++ that replaces the JVM-based Spark execution engine for eligible operations.

**How it works:**
- Processes data in columnar batches (SIMD/vectorized operations)
- Avoids JVM overhead, GC pauses, and object serialization
- Tight integration with Delta Lake (reads Parquet natively in C++)

**When Photon helps most:**
| Workload | Speedup |
|---|---|
| Wide table scans with aggregations | 2-5x |
| Sort/merge joins | 2-4x |
| MERGE (upsert) operations | 3-5x |
| Delta Lake OPTIMIZE | 2-3x |
| Databricks SQL / BI queries | 2-8x |
| String operations, regex | 2-4x |

**When Photon doesn't help:**
- Custom Python UDFs (must go through JVM/Python bridge)
- Complex Scala UDFs
- Workloads bottlenecked on network I/O or storage

**Enabling**: Photon is available on DBR 9.1+ with Photon-enabled instance types. Enabled by default for SQL Warehouses. For clusters, select "Use Photon Acceleration" in cluster configuration.

**Cost note**: Photon clusters have a higher DBU rate (~2x for some SKUs) but typically deliver much better price/performance for SQL workloads.

---

### Q37. How do you right-size clusters? Walk through your approach.
**Answer (Scenario):**

**Step 1: Collect baseline metrics**
- Use Spark UI (Stages tab, Executors tab) during a representative run
- Monitor: CPU utilization, memory used/spilled, task duration distribution, shuffle read/write

**Step 2: Identify the bottleneck**
```
CPU-bound: tasks slow, executors busy → increase core count or add workers
Memory-bound: spill to disk visible in Spark UI → increase memory per executor
I/O-bound: tasks fast but waits between stages → may be metadata or small files issue
Under-utilized: executors idle → cluster is too large, reduce workers
Skewed: a few tasks take 10x longer → fix skew, not cluster size
```

**Step 3: Right-size formula**
```
Recommended workers = ceil(data_size_GB / target_GB_per_executor)
target_GB_per_executor ≈ 20-30 GB for typical transformations

Example: 500GB dataset
  → 500/25 = 20 executor cores needed
  → m5.xlarge (4 cores, 16GB) → 5 workers
  → Or m5.4xlarge (16 cores, 64GB) → 2 workers (fewer, larger)
```

**Step 4: Use Ganglia metrics**
- In cluster event log → Ganglia → Memory, CPU, Network IO over time
- If average CPU < 30% throughout job → cluster is too large

**Step 5: Enable autoscaling as a safety net**
- Set min = right-sized estimate, max = 2x
- Let autoscaling handle burst; analyze actual usage patterns over 1 week
- Reduce max based on 95th percentile actual usage

---

---

## 6. Security & Access Control

### Q38. What are the layers of security in a Databricks deployment?
**Answer:**

Databricks security operates at multiple layers:

```
1. Network Layer
   - VPC/VNET peering (customer-managed network)
   - Private Link (no public internet traversal)
   - IP access lists (whitelist specific IPs for workspace access)
   - Network Security Groups / Security Groups

2. Authentication Layer
   - SSO via SAML 2.0 / OIDC (Okta, Azure AD, Google)
   - SCIM provisioning for user/group sync
   - Service Principals for automated access (not personal tokens)
   - Personal Access Tokens (PATs) — should be rotated, scoped

3. Authorization Layer
   - Unity Catalog: catalog/schema/table/column/row-level permissions
   - Workspace-level ACLs: notebooks, folders, clusters, jobs
   - Cluster policies: restrict what compute users can create
   - Secret scopes: control access to credentials

4. Data Encryption Layer
   - At rest: cloud-native encryption (SSE-S3, ADE) or Customer-Managed Keys (CMK)
   - In transit: TLS 1.2+ everywhere
   - Databricks-managed encryption + optional BYOK

5. Secrets Management
   - Databricks Secret Scopes (backed by Databricks or Azure Key Vault)
   - Never store credentials in notebooks or code

6. Audit Layer
   - Workspace audit logs (all API calls)
   - Unity Catalog system tables (data access auditing)
   - Diagnostic log export to cloud logging (CloudTrail, Azure Monitor)
```

---

### Q39. Explain Databricks Secret Scopes. How do you use them and what are the two types?
**Answer:**

**Databricks-backed Secret Scope:**
- Secrets stored in Databricks internal encrypted storage
- Managed via Databricks CLI or API
- Access controlled via ACLs (READ, WRITE, MANAGE per user/group)

```bash
# Create scope
databricks secrets create-scope --scope my-scope

# Add secret
databricks secrets put --scope my-scope --key db_password

# Grant access
databricks secrets put-acl --scope my-scope --principal data-engineers --permission READ
```

**Azure Key Vault-backed Secret Scope:**
- Secrets stored in Azure Key Vault, not in Databricks
- Databricks reads from Key Vault at runtime
- Key Vault access policies control who can access what
- One-way sync: Key Vault is source of truth

```bash
# Create AKV-backed scope
databricks secrets create-scope \
  --scope akv-scope \
  --scope-backend-type AZURE_KEYVAULT \
  --resource-id /subscriptions/.../vaults/my-vault \
  --dns-name https://my-vault.vault.azure.net/
```

**Using secrets in notebooks:**
```python
# Read secret (returns string, never printed in output)
password = dbutils.secrets.get(scope="my-scope", key="db_password")

# Use in JDBC connection
jdbc_url = f"jdbc:postgresql://host:5432/db?user=admin&password={password}"
df = spark.read.jdbc(url=jdbc_url, table="my_table")
```

**Key security principle**: Databricks redacts secret values from notebook output — if you try to `print(password)`, it shows `[REDACTED]`.

---

### Q40. How do you implement network isolation for a Databricks workspace? Explain Private Link.
**Answer:**

**Standard deployment** (without network isolation):
- Control plane in Databricks-managed VPC
- Data plane in your VPC
- Traffic between control/data plane goes over public internet (encrypted)

**VPC/VNET injection (Customer-managed VPC):**
- You deploy the data plane (cluster VMs) into your own VPC
- Enables: VPC peering to on-premises, custom DNS, private subnet routing
- Databricks control plane still requires outbound internet access from data plane

**Private Link (recommended for high-security):**
- Eliminates public internet for control plane ↔ data plane communication
- Uses AWS PrivateLink / Azure Private Link service
- Two endpoints needed:
  1. **Front-end Private Link**: User browser/tools → workspace UI/API (no public IP needed)
  2. **Back-end Private Link**: Cluster nodes → Databricks control plane (no outbound internet)

**IP Access Lists:**
```
Workspace Settings → Security → IP Access Lists
Add allowed CIDR ranges: corporate network, VPN IPs, etc.
Effect: blocks all access from non-whitelisted IPs, even with valid credentials
```

**Firewall egress control** (with VPC injection):
```
Clusters allowed outbound to:
  - Databricks control plane endpoints (*.azuredatabricks.net)
  - Cloud storage (S3/ADLS/GCS)
  - Maven/PyPI for package installs (or use a private artifact repo)
  - Block all other egress
```

---

### Q41. What is a Service Principal and why should you use it instead of a Personal Access Token for automation?
**Answer:**

**Personal Access Token (PAT)**:
- Tied to a specific user account
- If user leaves the company → token revoked → all automation breaks
- No granular scope — full user permissions
- Hard to rotate at scale

**Service Principal**:
- A non-human identity (application identity in Azure AD / AWS IAM)
- Independent of any individual user — doesn't break when employees leave
- Can be granted exactly the permissions it needs (least privilege)
- Supports OAuth2 client credentials flow (no long-lived tokens)
- Can be audited separately from human users

**Creating a service principal for a job:**
```bash
# 1. Create SP in Azure AD / Databricks account
databricks service-principals create --display-name etl-pipeline-sp

# 2. Grant minimal permissions
# In Unity Catalog:
GRANT USE CATALOG ON CATALOG prod TO `sp-etl-pipeline`;
GRANT USE SCHEMA ON SCHEMA prod.sales TO `sp-etl-pipeline`;
GRANT SELECT, MODIFY ON TABLE prod.sales.transactions TO `sp-etl-pipeline`;

# In workspace (for job execution):
# Assign SP as owner/can-manage-run on the specific job only
```

**OAuth M2M (Machine-to-Machine):**
```python
# Modern approach — no PAT needed
from databricks.sdk import WorkspaceClient
w = WorkspaceClient(
    host="https://adb-xxx.azuredatabricks.net",
    client_id="sp-client-id",
    client_secret=dbutils.secrets.get("scope", "sp-secret")
)
```

---

### Q42. How do you implement row-level and column-level security in Unity Catalog without a third-party tool?
**Answer:**

**Column-Level Security — two approaches:**

*1. Column Masking (UC native, DBR 12.2+):*
```sql
-- Create a masking function
CREATE FUNCTION mask_email(email STRING)
  RETURNS STRING
  RETURN CASE WHEN is_account_group_member('pii_readers')
              THEN email
              ELSE CONCAT(LEFT(email, 2), '****@****.com')
         END;

-- Apply to column
ALTER TABLE customers
  ALTER COLUMN email SET MASK mask_email;
```

*2. Dynamic View (compatible with all UC versions):*
```sql
CREATE VIEW secure_customers AS
SELECT 
  customer_id,
  CASE WHEN is_account_group_member('pii_readers')
       THEN email
       ELSE '***MASKED***'
  END AS email,
  region,
  purchase_amount
FROM raw_customers;
-- Grant SELECT on view, not on raw table
```

**Row-Level Security:**
```sql
-- Row filter function (UC native)
CREATE FUNCTION filter_by_region(region_col STRING)
  RETURNS BOOLEAN
  RETURN CASE WHEN is_account_group_member('global_readers') THEN TRUE
              ELSE region_col = current_user_region()  -- custom lookup
         END;

-- Apply to table
ALTER TABLE sales_data
  SET ROW FILTER filter_by_region ON (region);

-- Combined approach with subquery
CREATE FUNCTION row_access_filter(team STRING)
  RETURNS BOOLEAN
  RETURN team IN (
    SELECT allowed_team FROM team_access_control
    WHERE user_email = CURRENT_USER()
  );
```

---

## 7. Performance Tuning & Spark Optimization

### Q43. How does Adaptive Query Execution (AQE) work in Spark 3.x and Databricks?
**Answer:**

**AQE** re-optimizes query plans at runtime using statistics collected during execution (not just at planning time).

**Three main features of AQE:**

**1. Dynamic Coalescing of Shuffle Partitions:**
```
Default: spark.sql.shuffle.partitions = 200
Problem: 200 partitions regardless of data size
AQE: After a shuffle, combines small partitions into larger ones
Result: avoid 1000s of tiny partition files → faster downstream stages
```

**2. Dynamic Switch to Broadcast Hash Join:**
```
Planning time: table B looks large (200 GB from metadata) → sort-merge join planned
Runtime: After filtering, table B is actually 5 MB → AQE switches to broadcast join
Result: eliminates a shuffle, massive speedup
```
Enable: `spark.sql.adaptive.localShuffleReader.enabled = true`

**3. Dynamic Skew Handling:**
```
Problem: partition 5 has 100M rows, others have 1M rows → task skew
AQE: detects skewed partition, splits it into sub-partitions
Result: more balanced task execution, no single task bottleneck
```
Enable: `spark.sql.adaptive.skewJoin.enabled = true`

**Enabling AQE:**
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")  # default True in Spark 3.2+
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

**Important**: AQE is enabled by default in Databricks Runtime 8.x+. In most cases, you don't need to manually enable it.

---

### Q44. What causes data skew and how do you handle it?
**Answer:**

**Data skew** = uneven distribution of data across partitions. One partition has dramatically more rows than others → one task takes 10x longer → entire stage waits for it.

**Common causes:**
- Joining on a column with NULL or heavily repeated values (e.g., `customer_id = 'UNKNOWN'` for all missing data)
- Grouping by a low-cardinality column (e.g., `country = 'US'` has 80% of rows)
- Hot key in streaming (same event type dominates)

**Detection:**
```python
# In Spark UI: Stages tab → Task Duration distribution
# Large gap between median and max task duration = skew

# In code:
df.groupBy("join_key").count().orderBy(desc("count")).show(20)
```

**Solutions:**

*1. Salting (for joins):*
```python
import random
from pyspark.sql.functions import col, lit, concat, rand, floor

# Add random salt to large table
salt_buckets = 10
large_df = large_df.withColumn("salted_key",
    concat(col("join_key"), lit("_"), (floor(rand() * salt_buckets)).cast("string")))

# Explode small table with all salt values
from pyspark.sql.functions import array, explode
small_df = small_df.withColumn("salt", array([lit(str(i)) for i in range(salt_buckets)]))
small_df = small_df.withColumn("salted_key", 
    concat(col("join_key"), lit("_"), explode(col("salt"))))

result = large_df.join(small_df, "salted_key")
```

*2. AQE Skew Join (automatic):*
```python
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256mb")
```

*3. Filter and handle hot keys separately:*
```python
# Process the dominant key separately, union results
hot_key_df = large_df.filter(col("customer_id") == "UNKNOWN")
normal_df = large_df.filter(col("customer_id") != "UNKNOWN")

result_normal = normal_df.join(small_df, "customer_id")
result_hot = hot_key_df.join(broadcast(small_df.filter(...)), ...)
result = result_normal.union(result_hot)
```

---

### Q45. Explain the different join strategies in Spark. When is each used?
**Answer:**

**1. Broadcast Hash Join (BHJ)**
- One table broadcast to all executors; built into in-memory hash table
- Other table's partitions probe the hash table locally — no shuffle needed
- Trigger: `spark.sql.autoBroadcastJoinThreshold` (default 10 MB); or manually: `broadcast(df)`
- Best for: large table joined with small/dimension table

**2. Sort-Merge Join (SMJ)**
- Both tables shuffled on join key, sorted, merged
- Works for any table size; most common for large-large joins
- Expensive: 2 full shuffles + sort
- Default when neither table fits in broadcast threshold

**3. Shuffle Hash Join (SHJ)**
- Both tables shuffled, but smaller side built into hash table (no sort)
- Faster than SMJ when one side is significantly smaller but too large to broadcast
- Less common; used when sort is more expensive than hash table build
- Enable: `spark.sql.join.preferSortMergeJoin = false`

**4. Broadcast Nested Loop Join (BNLJ)**
- Used for non-equi joins (range joins, cross joins)
- One side broadcast; nested loop comparison (expensive)
- Avoid on large tables

**5. Cartesian Product**
- Full cross join — every row × every row
- O(n²) — avoid unless explicitly needed and data is small

**Forcing a join strategy:**
```python
from pyspark.sql.functions import broadcast
df = large_df.join(broadcast(small_df), "key")  # force BHJ

# Hints in SQL:
spark.sql("SELECT /*+ BROADCAST(small_table) */ * FROM large_table JOIN small_table USING (id)")
spark.sql("SELECT /*+ MERGE(t1, t2) */ * FROM t1 JOIN t2 USING (id)")
```

---

### Q46. What is the difference between `repartition()` and `coalesce()`? When do you use each?
**Answer:**

**`repartition(n)`:**
- Full shuffle — redistributes data evenly across `n` partitions using round-robin or hash
- Output: `n` partitions, approximately equal size
- Expensive: triggers a shuffle stage
- Use when: increasing partitions, or you need even distribution before a join/aggregate

**`coalesce(n)`:**
- No shuffle (or minimal) — merges partitions on the same executor by combining them
- Can only REDUCE partition count (not increase)
- Output: `n` partitions, possibly uneven (depends on original distribution)
- Use when: reducing partitions before writing to storage (avoid many small files)

**`repartitionByRange(n, col)`:**
- Shuffle + sort — partitions data by value range of a column
- Useful for: range-based queries, windowed aggregations, writing sorted Parquet

**Practical write pattern:**
```python
# Before writing: reduce to match target file size
target_file_size_mb = 128
total_data_mb = df.count() * avg_row_bytes / 1024 / 1024
n_partitions = max(1, int(total_data_mb / target_file_size_mb))

df.coalesce(n_partitions).write.format("delta").save(path)
```

**Interview nuance**: `coalesce(1)` to write a single file is fine for small data but causes a full pull to driver-adjacent executor for large data — use `repartition(1)` only if you truly need 1 file and understand the cost.

---

### Q47. What is caching in Spark? When should you use `cache()` vs `persist()`?
**Answer:**

**`cache()`**: shorthand for `persist(StorageLevel.MEMORY_AND_DISK_DESER)` — stores in memory, spills to disk if needed.

**`persist(storageLevel)`**: explicit control over storage level.

**Storage levels:**
| Level | Memory | Disk | Serialized | Notes |
|---|---|---|---|---|
| `MEMORY_ONLY` | Yes | No | No | Fastest; evicts if no memory |
| `MEMORY_AND_DISK` | Yes | Yes | No | Safe default; spills to disk |
| `MEMORY_ONLY_SER` | Yes | No | Yes | Smaller footprint; CPU overhead |
| `MEMORY_AND_DISK_SER` | Yes | Yes | Yes | Balance of speed/size |
| `DISK_ONLY` | No | Yes | Yes | Slow; use for rarely-accessed data |
| `OFF_HEAP` | Yes (off-heap) | No | Yes | Avoids GC; requires off-heap config |

**When to cache:**
- DataFrame is reused in multiple downstream actions (join, filter, aggregate on same df)
- Iterative algorithms (ML training loops)
- Interactive notebook exploration (avoid re-reading from storage each cell)

**When NOT to cache:**
- DataFrame used only once → caching adds overhead with no benefit
- Very large DataFrame that exceeds available memory → spill is worse than re-read
- Fast sources (in-memory tables, small Parquet with predicate pushdown)

**Unpersist**: Always `df.unpersist()` when done — freed memory can be used by other operations:
```python
df_cached = df.cache()
result1 = df_cached.filter(...).count()
result2 = df_cached.groupBy(...).agg(...)
df_cached.unpersist()  # release memory
```

---

---

## 8. Spark Internals

### Q48. Explain the Spark execution model: Jobs, Stages, Tasks, and how they map to the DAG.
**Answer:**

**Spark execution hierarchy:**

```
Application
  └── Job (triggered by an action: .collect(), .count(), .write())
        └── Stage (separated by shuffle boundaries)
              └── Task (one per partition; runs on one executor core)
```

**DAG (Directed Acyclic Graph):**
- Spark builds a logical DAG of transformations (lazy evaluation)
- Physical plan breaks the DAG into stages at shuffle boundaries
- **Narrow transformations** (map, filter, select): stay in same stage — no shuffle
- **Wide transformations** (groupBy, join, repartition): require shuffle → new stage

**Example:**
```python
df = spark.read.parquet("s3://...")        # Stage 0: read
   .filter(col("age") > 18)               # Stage 0: narrow
   .groupBy("country").count()            # Stage 1: shuffle (groupBy)
   .filter(col("count") > 1000)           # Stage 1: narrow  
   .orderBy(desc("count"))                # Stage 2: shuffle (sort)
   .write.parquet("s3://output/")         # triggers job
```

**Stage execution:**
1. **Map stage**: each task reads one input partition, applies narrow transforms, writes shuffle output
2. **Reduce stage**: each task reads shuffle data from all map tasks for its partition key range

**Task execution:**
- Serialized closure (function + data) sent to executor
- Runs on one partition in one executor core
- Returns result or writes to shuffle/storage

**Key interview point**: 1 Task = 1 Partition = 1 Core. If you have 200 partitions and 50 cores → 4 waves of 50 parallel tasks.

---

### Q49. What is the Spark shuffle? Why is it expensive and how do you minimize it?
**Answer:**

**Shuffle** = redistribution of data across executors so that rows with the same key end up on the same executor. Required for: `groupBy`, `join` (non-broadcast), `repartition`, `distinct`, `orderBy`.

**Why it's expensive:**
1. **Serialization**: data serialized to bytes (CPU overhead)
2. **Disk writes**: shuffle output written to local disk on source executors
3. **Network transfer**: shuffle data transferred across executors (bounded by network bandwidth)
4. **Disk reads**: destination executors read shuffle data from disk/network
5. **Memory pressure**: shuffle buffers + read buffers compete with task memory

**Shuffle metrics to check in Spark UI:**
- Shuffle Read/Write bytes (Stages tab)
- Shuffle Spill (Memory) / Shuffle Spill (Disk)
- High spill = not enough memory → increase executor memory or reduce partition size

**How to minimize shuffles:**

1. **Broadcast small tables** (avoids join shuffle entirely)
2. **Filter before join** (reduce data volume before shuffle)
3. **Use partitioned tables** (co-located data can skip shuffle for partition-aligned joins)
4. **Bucket both tables** on join key (pre-partitioned files → no shuffle needed):
   ```python
   df.write.bucketBy(50, "customer_id").sortBy("customer_id") \
     .saveAsTable("bucketed_orders")
   ```
5. **AQE coalesce** (merges shuffle output partitions to reduce overhead)
6. **Avoid `distinct()` on large datasets** — replace with `dropDuplicates(['key_cols'])`

---

### Q50. What is the difference between the Spark driver and executors? What happens when the driver runs out of memory?
**Answer:**

**Driver:**
- Runs the main Spark application
- Hosts the SparkContext, SparkSession
- Builds and optimizes the logical/physical query plan
- Schedules tasks to executors via the cluster manager
- Collects results of `.collect()`, `.show()`, `.toPandas()` calls
- Memory: stores broadcast variables, accumulated results, task metadata

**Executor:**
- JVM processes running on worker nodes
- Execute tasks assigned by the driver
- Cache data (when `persist()` is called)
- Write shuffle output to local storage
- Report status to driver

**Driver OOM causes and fixes:**

| Cause | Fix |
|---|---|
| `.collect()` on large DataFrame | Never collect large data to driver; use `.write()` instead |
| Too many broadcast variables | Avoid broadcasting tables > 2GB |
| Large `toPandas()` | Use `spark_df.write.parquet()` + read with pandas in chunks |
| Excessive number of partitions (millions) | Reduce shuffle partitions; use AQE |
| Streaming stateful operations accumulating state | Set state TTL; use `flatMapGroupsWithState` |

**Driver memory settings:**
```python
# Cluster config
"spark.driver.memory": "8g"
"spark.driver.maxResultSize": "4g"  # max size for .collect() results
```

**Key rule**: The driver does not process data — it only coordinates. All data processing happens in executors. Collecting data to the driver defeats this architecture.

---

### Q51. Explain Spark's memory model. What is the difference between execution memory and storage memory?
**Answer:**

**Spark Unified Memory Manager (Spark 1.6+):**
```
JVM Heap
├── Reserved Memory (300 MB, system overhead)
├── User Memory  (spark.memory.fraction complement — default 40%)
│   └── User code objects, data structures outside Spark's control
└── Spark Memory (spark.memory.fraction = 0.6, i.e., 60% of heap minus reserved)
    ├── Execution Memory (dynamic share)
    │   └── Shuffle buffers, sort buffers, hash tables for joins
    └── Storage Memory (dynamic share)
        └── Cached RDDs/DataFrames (persist/cache)
```

**Key innovation — unified memory**: Execution and storage share the same pool. Either can borrow from the other:
- If no cached data → all Spark memory available for execution
- If lots of cached data → storage takes more; execution may spill to disk
- Execution can evict storage cache (LRU) if it needs more memory

**Config knobs:**
```python
spark.conf.set("spark.memory.fraction", "0.6")        # fraction of heap for Spark
spark.conf.set("spark.memory.storageFraction", "0.5") # storage's guaranteed minimum share
```

**Off-heap memory** (for JVM GC reduction):
```python
spark.conf.set("spark.memory.offHeap.enabled", "true")
spark.conf.set("spark.memory.offHeap.size", "4g")
```

**Spill to disk**: When execution memory is exhausted, Spark spills shuffle/sort data to local disk (slower but prevents OOM). Indicated in Spark UI as "Spill (Memory)" and "Spill (Disk)" — high spill = increase executor memory or reduce data per task.

---

### Q52. What is Tungsten execution engine? How does it improve Spark performance?
**Answer:**

**Tungsten** (introduced in Spark 1.5) is a set of execution engine optimizations that address JVM limitations in big data processing.

**Three key optimizations:**

**1. Unsafe (off-heap) binary memory format:**
- Stores data in binary compact format (not JVM objects)
- Avoids object header overhead (a JVM `Long` takes 16 bytes vs 8 bytes in binary)
- Eliminates GC pressure on large datasets
- Faster serialization/deserialization

**2. Cache-aware computation:**
- Data structures designed to fit in CPU L1/L2 cache
- Sort algorithms rewritten to maximize cache hits
- Reduces CPU cache misses → faster execution

**3. Whole-Stage Code Generation (WSCG):**
- Fuses multiple operators into a single compiled Java function
- Eliminates virtual function calls between operators
- Generated code is JIT-compiled by the JVM
- Example: `filter → project → aggregate` becomes one tight loop instead of 3 function calls

**Photon** extends these ideas further in Databricks:
- Written in C++ (not JVM) → fully avoids JVM overhead
- True SIMD vectorized execution
- Built specifically for analytical SQL and Delta Lake

---

### Q53. What is lazy evaluation in Spark? Why does it exist and what triggers execution?
**Answer:**

**Lazy evaluation**: Spark does NOT execute transformations when they're called. It builds a logical plan (DAG) and only executes when an **action** is called.

**Transformations (lazy):**
- `select`, `filter`, `map`, `groupBy`, `join`, `withColumn`, `repartition`, `union`, etc.
- These add nodes to the DAG — no computation happens

**Actions (trigger execution):**
- `count()`, `collect()`, `show()`, `first()`, `take(n)`
- `write.save()`, `write.parquet()`, `write.format().save()`
- `foreach()`, `foreachPartition()`

**Why lazy evaluation:**
1. **Optimization**: Catalyst optimizer can see the full transformation chain and apply logical → physical plan optimizations (predicate pushdown, constant folding, pruning)
2. **Fault tolerance**: Spark can recompute lost partitions by replaying the DAG from stable sources
3. **Efficiency**: Spark combines multiple narrow transformations into a single pass (pipeline)

**Common gotcha:**
```python
# This is WRONG for counting intermediate results:
df_filtered = df.filter(col("age") > 18)
print(f"Filtered count: {df_filtered.count()}")  # triggers action #1
df_final = df_filtered.groupBy("country").count()
df_final.show()  # triggers action #2 -- re-reads and re-filters the source!

# CORRECT: cache if reusing
df_filtered = df.filter(col("age") > 18).cache()
print(f"Filtered count: {df_filtered.count()}")
df_filtered.groupBy("country").count().show()
df_filtered.unpersist()
```

---

### Q54. What is the Catalyst Optimizer? Walk through how a query gets optimized.
**Answer:**

**Catalyst** is Spark's query optimizer — a rule-based + cost-based optimizer that transforms a logical query plan into an efficient physical plan.

**Optimization pipeline:**

```
SQL / DataFrame API
        ↓
1. Unresolved Logical Plan (AST)
   - Column names not yet resolved
        ↓
2. Analysis (Catalog lookup)
   - Resolve column names, types, aliases
        ↓
3. Resolved Logical Plan
        ↓
4. Logical Optimization (Rule-based)
   - Predicate pushdown: push filters as close to source as possible
   - Constant folding: evaluate constants at plan time (WHERE 1+1 → WHERE 2)
   - Column pruning: drop unused columns early
   - Boolean simplification
        ↓
5. Optimized Logical Plan
        ↓
6. Physical Planning (multiple strategies)
   - Choose join strategy (BHJ vs SMJ vs SHJ)
   - Choose scan strategy (full scan vs partition pruning)
        ↓
7. Physical Plans (multiple candidates)
        ↓
8. Cost-Based Optimization (CBO)
   - If statistics available: pick lowest-cost join order
   - Statistics from: ANALYZE TABLE, file-level stats in Delta log
        ↓
9. Selected Physical Plan → Code Generation (Tungsten WSCG)
        ↓
10. Execution
```

**Enabling CBO:**
```python
spark.conf.set("spark.sql.cbo.enabled", "true")
spark.conf.set("spark.sql.statistics.histogram.enabled", "true")
# Collect stats:
spark.sql("ANALYZE TABLE my_table COMPUTE STATISTICS FOR ALL COLUMNS")
```

---

## 9. Autoloader & Streaming

### Q55. What is Databricks Autoloader and how does it differ from `spark.readStream`?
**Answer:**

**Autoloader** (`cloudFiles` format) is Databricks' incremental file ingestion service that monitors cloud storage for new files and processes them exactly-once in a streaming fashion.

```python
df = spark.readStream.format("cloudFiles") \
    .option("cloudFiles.format", "json") \
    .option("cloudFiles.schemaLocation", "/checkpoints/schema/") \
    .load("s3://my-bucket/incoming/")
```

**vs. plain `spark.readStream` on file source:**

| Feature | Plain readStream (file source) | Autoloader (cloudFiles) |
|---|---|---|
| File discovery | Lists all files on every trigger | Incremental: only new files (via cloud events or listing) |
| Scalability | Slows down with millions of files | Handles billions of files efficiently |
| Schema inference | Manual or infer once | Auto-infers + evolves schema over time |
| Exactly-once | Yes (via checkpoint) | Yes (via checkpoint) |
| Rescue data | No | Yes — bad records go to `_rescued_data` column |

**Two file discovery modes:**
1. **Directory listing mode** (default): lists the directory, compares with checkpoint, finds new files. Good for most cases.
2. **File notification mode** (AWS SQS / Azure Event Grid): cloud sends events when files land → near-zero latency, scales to extremely high file rates.

**Schema evolution with Autoloader:**
```python
# Enable automatic schema evolution
.option("cloudFiles.inferColumnTypes", "true") \
.option("cloudFiles.schemaEvolutionMode", "addNewColumns")  
# Options: addNewColumns, rescue, failOnNewColumns, none
```

---

### Q56. Explain Spark Structured Streaming. What is a micro-batch and what is continuous processing?
**Answer:**

**Structured Streaming** treats streaming as an unbounded table — new data is appended as rows; queries on this table produce results continuously.

**Micro-batch mode (default):**
- Process data in small batches at fixed intervals (`trigger`)
- Low latency: 100ms to minutes depending on trigger
- Exactly-once semantics via checkpointing
- Most operations supported

```python
query = df.writeStream \
    .outputMode("append") \
    .format("delta") \
    .option("checkpointLocation", "/checkpoints/my_stream/") \
    .trigger(processingTime="30 seconds") \  # micro-batch every 30s
    .toTable("catalog.schema.events")
```

**Trigger types:**
| Trigger | Behavior |
|---|---|
| `processingTime="1 minute"` | Run every 1 minute |
| `once=True` | Run once, process all available data, stop |
| `availableNow=True` | Like `once` but with micro-batch (respects maxFilesPerTrigger) |
| `continuous="1 second"` | Continuous processing mode |

**Continuous processing mode** (experimental):
- Processes each record as it arrives — no batching
- Latency: ~1ms vs 100ms+ for micro-batch
- Limitation: only supports stateless map/filter operations
- Not widely used in production

**Output modes:**
- `append`: only new rows written (for stateless queries or windowed aggregations)
- `update`: only changed rows (works with aggregations)
- `complete`: entire result table rewritten (for aggregations, small result sets)

---

### Q57. How do you handle late-arriving data and watermarks in Spark Structured Streaming?
**Answer:**

**Problem**: In streaming, events often arrive out-of-order (network delays, mobile apps offline). If you compute a window aggregate, late data arrives after the window should be closed.

**Watermark**: A moving threshold that tells Spark "I will accept data up to X time late — after that, I drop it."

```python
from pyspark.sql.functions import window, col

windowed_counts = df \
    .withWatermark("event_time", "10 minutes") \  # accept up to 10 min late data
    .groupBy(
        window(col("event_time"), "5 minutes"),    # 5-min tumbling window
        col("user_id")
    ) \
    .count()
```

**How it works:**
- Watermark = max(event_time seen) - late_threshold
- Windows with end_time < watermark are finalized and dropped from state
- Events with event_time < watermark are dropped (too late)

**Window types:**
```python
# Tumbling window: non-overlapping fixed-size windows
window(col("event_time"), "5 minutes")

# Sliding window: overlapping
window(col("event_time"), "10 minutes", "5 minutes")  # 10-min window, slides every 5 min

# Session window (Spark 3.2+): gaps between events define window boundaries
session_window(col("event_time"), "30 minutes")  # new window after 30 min of inactivity
```

**State store**: Spark maintains an in-memory + RocksDB state store for streaming aggregations. Watermarks control how long state is retained.

---

### Q58. What is a streaming checkpoint and what happens if you lose it?
**Answer:**

**Checkpoint**: A fault-tolerance mechanism that saves the stream's progress (offsets processed) and aggregation state to durable storage.

**Contents of a checkpoint directory:**
```
/checkpoints/my_stream/
├── commits/           ← completed micro-batches (for dedup)
├── offsets/           ← what data has been read (source offsets)
├── metadata           ← stream metadata
└── state/             ← stateful aggregation state (watermark, aggregations)
```

**What happens if you lose the checkpoint:**

| Scenario | Impact |
|---|---|
| Source supports replay (Kafka, Delta CDF) | Restart stream from beginning or latest offset — risk of reprocessing or data gaps |
| Source is Autoloader | Restart from last saved file list — may miss or reprocess files |
| Stateful aggregation (windowed counts) | **State is lost** — counters reset to 0, incorrect aggregation results |
| Stateless stream (pure filter/map) | Just replay from source — no data loss if source is replayable |

**Best practices:**
1. Store checkpoints on **durable, reliable storage** (S3, ADLS) — not ephemeral local disk
2. **Back up checkpoint directories** for critical streams
3. If checkpoint must be deleted (schema change, logic change): accept the reset, document the gap
4. Use **`idempotent sinks`** (Delta Lake with merge, Kafka with dedup keys) so replaying is safe

**Checkpoint on schema change**: If you change the stream schema incompatibly (rename column, change type), you must delete checkpoint and restart — old state is incompatible with new schema.

---

### Q59. How does Autoloader handle schema evolution and what is the `_rescued_data` column?
**Answer:**

**Schema evolution modes in Autoloader:**

| Mode | Behavior |
|---|---|
| `addNewColumns` | New columns in incoming files are added to the schema; old queries not broken |
| `rescue` | New/unexpected columns go into `_rescued_data` (JSON column) instead of failing |
| `failOnNewColumns` | Fail the stream if a new column is detected — forces human review |
| `none` | No evolution; new columns dropped silently |

**`_rescued_data` column:**
- Captures rows (or columns) that don't match the current schema
- Contains a JSON string with all columns that were not mapped
- Useful for: audit, debugging, handling heterogeneous data from different producers

```python
# Check rescued data
rescued = spark.read.format("delta").load("/tables/bronze_events") \
    .filter(col("_rescued_data").isNotNull()) \
    .select("_rescued_data", "event_time", "source_file")

# Parse rescued data for specific field
from pyspark.sql.functions import get_json_object
rescued.withColumn("new_field", get_json_object(col("_rescued_data"), "$.new_field_name"))
```

**Schema inference location:**
- Always set `cloudFiles.schemaLocation` to a persistent path
- Autoloader stores inferred schema there across restarts
- Without it, Autoloader re-infers schema on each restart (slower, potential inconsistency)

---

## 10. Scenario-Based Questions

*(Mirroring Ansh Lamba's video style: 17 real-world scenario questions)*

---

### Q60. Scenario: Your ETL pipeline is reading 10TB of raw data daily and the job is taking 6 hours. The business needs it under 1 hour. How do you approach this?
**Answer:**

**Step 1: Profile the current pipeline**
- Check Spark UI: which stages are slowest? What are the shuffle sizes? Is there spill?
- Is the bottleneck: reading (I/O), transforming (CPU/memory), or writing (I/O)?

**Step 2: Quick wins (likely < 1 hour effort)**
```
a) Switch to incremental processing:
   - Don't process 10TB daily if most of it hasn't changed
   - Use Delta CDF or Autoloader to process only NEW/CHANGED data
   - This alone can reduce data volume from 10TB to 100GB if only 1% changes daily

b) Enable Photon:
   - If not already on Photon cluster, switching can give 2-5x speedup for SQL transforms

c) Fix partition + Z-order on source tables:
   - If source is Delta, ensure it's partitioned by date and Z-ordered by commonly filtered columns
   - Reduces data read from 10TB to the actual relevant subset
```

**Step 3: Architecture improvements**
```
d) Implement medallion architecture properly:
   - Bronze: raw ingest with Autoloader (fast, schema-flexible)
   - Silver: apply business rules incrementally (only new bronze data)
   - Gold: aggregate incrementally (only changed silver data)

e) Right-size cluster with autoscaling:
   - If bottleneck is CPU: more workers (m5.4xlarge × 20)
   - If bottleneck is memory: memory-optimized instances (r5.4xlarge)

f) Optimize join strategy:
   - Broadcast small dimension tables
   - Partition-prune source tables using partition columns in join condition
```

**Step 4: Measure incrementally**
- Implement change #1, measure, then #2, etc.
- Target: incremental processing alone should get you under 2 hours; + Photon + Z-order should hit <1 hour.

---

### Q61. Scenario: A streaming job consuming from an event hub/Kafka is processing 1M events/second but has a backlog growing to 30 minutes. What do you investigate?
**Answer:**

**Symptoms**: Input rate > processing rate → backlog grows.

**Step 1: Check current throughput**
```python
# In Spark UI → Streaming tab:
# inputRowsPerSecond vs processedRowsPerSecond
# If processed < input → you're falling behind
```

**Step 2: Diagnose root cause**

| Check | Tool | Resolution |
|---|---|---|
| Cluster too small | Spark UI Executors tab: all CPUs busy | Scale up workers |
| I/O bottleneck on sink | Stage metrics: write time >> compute time | Switch to async writes, use Delta merge batch instead of per-event |
| State store explosion | Streaming tab: state rows growing | Add watermark to expire old state |
| Micro-batch trigger too frequent | Streaming metrics: trigger interval < processing time | Increase trigger interval |
| Shuffle in stateful operation | Stage tab: large shuffle | Reduce shuffle by repartitioning on state key |
| Skewed Kafka partition | Some tasks 10x slower | Rebalance Kafka partitions; use repartition() after read |

**Step 3: Tune**
```python
# Limit max records per micro-batch to control burst:
.option("maxOffsetsPerTrigger", 500000)

# For Kafka: set fetch size appropriately
.option("kafka.max.partition.fetch.bytes", "1048576")

# Increase trigger interval to allow larger batches (more efficient):
.trigger(processingTime="60 seconds")  # was 10 seconds → larger, fewer batches

# Enable enhanced autoscaling for streaming:
spark.conf.set("spark.databricks.streaming.enhancedAutoscaling.enabled", "true")
```

---

### Q62. Scenario: A MERGE operation on a 500GB Delta table is taking 2 hours. How do you optimize it?
**Answer:**

**Root cause of slow MERGE:**
1. MERGE does a full table scan to find matching rows (if no partition filter)
2. Rewrites affected files even for single-row changes
3. Large shuffle to perform the join between source and target

**Optimization strategy:**

**1. Add partition filter to MERGE condition:**
```sql
-- BEFORE: full table scan
MERGE INTO target t USING source s ON t.id = s.id
WHEN MATCHED THEN UPDATE ...

-- AFTER: limit scan to recent partition
MERGE INTO target t USING source s
ON t.date = s.date AND t.id = s.id  -- date partition filter limits files scanned
WHEN MATCHED THEN UPDATE ...
```

**2. Pre-filter the source:**
```sql
MERGE INTO target t
USING (
  SELECT * FROM source
  WHERE date >= DATEADD(day, -7, CURRENT_DATE())  -- only recent changes
) s ON t.id = s.id AND t.date = s.date
```

**3. Enable low-shuffle MERGE (Databricks optimization):**
```python
spark.conf.set("spark.databricks.delta.merge.enableLowShuffle", "true")
# Uses the Delta file statistics to skip files that can't match the merge condition
```

**4. Z-Order target table on join key:**
```sql
OPTIMIZE target ZORDER BY (id);  -- co-locates rows with same id → fewer files per merge
```

**5. Use insertOnly MERGE when no updates:**
```sql
MERGE INTO target t USING source s ON t.id = s.id
WHEN NOT MATCHED THEN INSERT *;
-- This is faster than full MERGE — set: spark.databricks.delta.merge.insertOnly = true
```

**6. Consider alternative patterns for high-frequency updates:**
- Append micro-updates to a staging table
- Periodically merge in batches (e.g., hourly) rather than per-record

---

### Q63. Scenario: You discover that a data pipeline silently ingested bad data (nulls in a NOT NULL column) for the past 3 days. How do you recover?
**Answer:**

**Step 1: Assess the damage**
```sql
-- How many bad records?
SELECT COUNT(*) as bad_rows, MAX(_commit_timestamp) as last_bad_commit
FROM target_table
WHERE critical_column IS NULL;

-- What versions were affected?
SELECT version, timestamp, operation
FROM (DESCRIBE HISTORY target_table)
WHERE timestamp >= DATEADD(day, -3, CURRENT_DATE())
ORDER BY version ASC;
```

**Step 2: Find the clean version using Time Travel**
```sql
-- Check version before corruption started
SELECT COUNT(*) FROM target_table VERSION AS OF 45
WHERE critical_column IS NULL;  -- should be 0 for clean version
```

**Step 3: Restore options**

*Option A: Full restore (if corruption is pervasive)*
```sql
-- Restore entire table to clean version (Databricks RESTORE command)
RESTORE TABLE target_table TO VERSION AS OF 45;
-- This creates a new commit pointing to version 45's file set
-- Does NOT delete any files — just moves the pointer
```

*Option B: Surgical fix (if only specific records are bad)*
```sql
-- Delete bad records
DELETE FROM target_table WHERE critical_column IS NULL;

-- Re-ingest from source for the affected time window
INSERT INTO target_table
SELECT * FROM source_table
WHERE event_date BETWEEN '2025-01-12' AND '2025-01-15'
  AND critical_column IS NOT NULL;
```

**Step 4: Add data quality guardrails**
```python
# In DLT pipeline:
@dlt.table(
    name="silver_events",
    expect_all_or_fail={"critical_column_not_null": "critical_column IS NOT NULL"}
)

# Or in notebook pipeline:
assert df.filter(col("critical_column").isNull()).count() == 0, \
    "FAIL: critical_column contains nulls — aborting write"
```

---

### Q64. Scenario: You need to design a Unity Catalog access control model for a company with 3 business units (Finance, Marketing, HR) sharing a single Databricks workspace. Each unit has analysts, engineers, and managers. How do you design it?
**Answer:**

**Catalog structure:**
```
Metastore
├── catalog: raw_data        (landing zone — restricted to platform team)
├── catalog: finance         (Finance BU)
│   ├── schema: bronze
│   ├── schema: silver
│   └── schema: gold
├── catalog: marketing       (Marketing BU)
│   ├── schema: bronze
│   ├── schema: silver
│   └── schema: gold
├── catalog: hr              (HR BU — highly restricted, PII)
│   ├── schema: bronze
│   ├── schema: silver
│   └── schema: gold
└── catalog: shared          (cross-BU shared tables — read-only for all)
    └── schema: reference_data
```

**Group structure (synced from SCIM/IdP):**
```
databricks_account_admins
  finance_engineers        → USE CATALOG finance; USE SCHEMA finance.*; SELECT/MODIFY bronze,silver,gold
  finance_analysts         → USE CATALOG finance; USE SCHEMA finance.gold; SELECT gold
  finance_managers         → USE CATALOG finance; SELECT ALL in gold; no bronze/silver
  marketing_engineers      → same pattern for marketing catalog
  hr_engineers             → USE CATALOG hr; special PII handling
  hr_analysts              → masked view access only (no raw PII)
  platform_engineers       → METASTORE ADMIN or catalog-level admin
```

**Granting permissions:**
```sql
-- Finance engineers: full ETL access
GRANT USE CATALOG ON CATALOG finance TO `finance_engineers`;
GRANT USE SCHEMA ON ALL SCHEMAS IN CATALOG finance TO `finance_engineers`;
GRANT ALL PRIVILEGES ON ALL TABLES IN CATALOG finance TO `finance_engineers`;

-- Finance analysts: gold layer only
GRANT USE CATALOG ON CATALOG finance TO `finance_analysts`;
GRANT USE SCHEMA ON SCHEMA finance.gold TO `finance_analysts`;
GRANT SELECT ON ALL TABLES IN SCHEMA finance.gold TO `finance_analysts`;

-- HR PII protection: column masking
ALTER TABLE hr.silver.employees
  ALTER COLUMN ssn SET MASK mask_ssn_for_non_hr;
ALTER TABLE hr.silver.employees
  ALTER COLUMN salary SET MASK mask_salary_for_analysts;

-- Shared catalog: all BUs read-only
GRANT USE CATALOG ON CATALOG shared TO `finance_engineers`, `marketing_engineers`, `hr_engineers`;
GRANT SELECT ON ALL TABLES IN CATALOG shared TO `finance_engineers`, `marketing_engineers`;
```

**Audit monitoring:**
```sql
-- Who accessed HR tables?
SELECT user_identity.email, action_name, request_params.table_full_name, event_time
FROM system.access.audit
WHERE request_params.table_full_name LIKE 'hr.%'
ORDER BY event_time DESC;
```

---

### Q65. Scenario: Your Databricks workspace is being migrated from AWS to Azure. What is your migration strategy?
**Answer:**

**Phase 1: Assessment (2-4 weeks)**
```
Inventory:
- Workspaces, users, groups (SCIM configuration)
- Clusters, cluster policies, instance pools
- Jobs/Workflows definitions
- Notebooks (workspace files)
- Delta tables (location, size, schema)
- Mount points → External Locations mapping
- Secrets scopes
- Unity Catalog configuration
- Network topology (VPC/VNET, PrivateLink)
- Service principals and PATs
- External connections (JDBC, REST APIs, AD integrations)
```

**Phase 2: Infrastructure Setup (2-3 weeks)**
```
Azure:
- Create Azure Databricks workspace in VNET
- Set up ADLS Gen2 storage accounts (mirror S3 bucket structure)
- Configure Managed Identity for storage access
- Set up Unity Catalog metastore (Azure region)
- Configure Private Endpoints
- Set up Azure AD groups / SCIM provisioning
```

**Phase 3: Data Migration**
```
Delta tables:
  Option A: distcp / Azure Data Factory copy (S3 → ADLS) + re-register in UC
  Option B: DEEP CLONE via dual-cloud Spark cluster (temporary)
  Option C: Export as Parquet + re-import (loss of history)

Recommended: Use Azure Data Factory to copy Delta files byte-for-byte
→ preserves _delta_log → preserves Time Travel history
→ Then CREATE TABLE USING DELTA LOCATION 'abfss://...' in new UC
```

**Phase 4: Application Migration**
```
- Export Jobs as JSON (Databricks CLI: databricks jobs list/get)
- Update all storage paths: s3:// → abfss://
- Update JDBC connection strings
- Recreate secrets in Azure Key Vault backed scopes
- Run notebooks in parallel on both platforms for validation
```

**Phase 5: Cutover**
```
- Dual-write period: write to both AWS and Azure tables
- Validate row counts, schema, query results
- Freeze writes to AWS
- Final sync of Delta log
- Switch DNS / API endpoints to Azure
- Monitor for 48 hours
- Decommission AWS resources
```

---

### Q66. Scenario: Design an incremental data loading pipeline from an RDBMS source to Delta Lake using Databricks.
**Answer:**

**Full architecture:**

```
RDBMS (PostgreSQL/Oracle/SQL Server)
    → [JDBC incremental extract]
    → Bronze Delta Table (raw, append-only)
    → [Silver transformation: dedup + schema enforcement]
    → Silver Delta Table
    → [Gold aggregation]
    → Gold Delta Table
```

**Bronze layer — JDBC incremental load:**
```python
from pyspark.sql.functions import col, current_timestamp

# Read high-watermark from checkpoint table
watermark = spark.sql("""
  SELECT COALESCE(MAX(last_loaded_timestamp), '1900-01-01') 
  FROM pipeline_control.watermarks
  WHERE table_name = 'orders'
""").collect()[0][0]

# JDBC incremental extract
df_new = spark.read.format("jdbc") \
    .option("url", jdbc_url) \
    .option("dbtable", f"(SELECT * FROM orders WHERE updated_at > '{watermark}') t") \
    .option("user", dbutils.secrets.get("scope", "db_user")) \
    .option("password", dbutils.secrets.get("scope", "db_password")) \
    .option("numPartitions", 20) \
    .option("partitionColumn", "order_id") \
    .option("lowerBound", 1) \
    .option("upperBound", 10000000) \
    .load() \
    .withColumn("_ingested_at", current_timestamp()) \
    .withColumn("_source_file", lit("jdbc:orders"))

# Append to bronze
df_new.write.format("delta").mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable("bronze.orders_raw")

# Update watermark
spark.sql(f"""
  MERGE INTO pipeline_control.watermarks w
  USING (SELECT 'orders' as table_name, MAX(updated_at) as ts FROM bronze.orders_raw) s
  ON w.table_name = s.table_name
  WHEN MATCHED THEN UPDATE SET last_loaded_timestamp = s.ts
  WHEN NOT MATCHED THEN INSERT *
""")
```

**Silver layer — dedup and enforce schema:**
```python
# Use CDF or version-based incremental processing
last_version = get_last_processed_version("orders_silver")

changes = spark.read.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingVersion", last_version) \
    .table("bronze.orders_raw") \
    .filter(col("_change_type") == "insert")

# Deduplicate (last write wins)
from pyspark.sql.window import Window
w = Window.partitionBy("order_id").orderBy(desc("updated_at"))
deduped = changes.withColumn("rn", row_number().over(w)).filter("rn = 1").drop("rn")

# Merge into silver
DeltaTable.forName(spark, "silver.orders") \
    .alias("t") \
    .merge(deduped.alias("s"), "t.order_id = s.order_id") \
    .whenMatchedUpdateAll() \
    .whenNotMatchedInsertAll() \
    .execute()
```

---

### Q67. Scenario: Explain how you would debug a PySpark job that is failing with `java.lang.OutOfMemoryError: GC overhead limit exceeded`.
**Answer:**

**What this error means**: The JVM is spending more than 98% of time in garbage collection and recovering less than 2% of heap — effectively stuck in a GC loop.

**Common causes and solutions:**

**1. Too much data collected to driver**
```python
# BAD: collects entire DataFrame to driver
all_data = df.collect()  # 10M rows → OOM

# GOOD: write to storage instead
df.write.format("delta").save("/output/path")
# Or limit what you collect:
sample = df.limit(1000).collect()
```

**2. High executor memory pressure (not driver)**
```
Check Spark UI → Executors tab → GC Time column
If GC time > 10% of task time → executors need more memory

Fix:
- Increase spark.executor.memory (cluster config)
- Reduce spark.sql.shuffle.partitions (less data per task)
- Add more workers (less data per executor)
- Enable off-heap: spark.memory.offHeap.enabled=true
```

**3. Exploding broadcast variable**
```python
# BAD: broadcasting a large table
df.join(broadcast(large_df), "key")  # large_df is 5GB → OOM

# Fix: remove broadcast hint, let Spark choose strategy
df.join(large_df, "key")
# Or lower broadcast threshold:
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "10m")  # prevent auto-broadcast
```

**4. Unbounded streaming state**
```python
# Add watermark to expire old state
df.withWatermark("event_time", "2 hours") \
  .groupBy(window("event_time", "1 hour"), "user_id") \
  .count()
# Without watermark, state grows forever → OOM on streaming job
```

**Debugging workflow:**
```
1. Enable GC logging: spark.executor.extraJavaOptions=-verbose:gc -XX:+PrintGCDetails
2. Check Spark UI: Executors → GC Time, Memory Used
3. Check Stage metrics: Spill (Memory) high → memory pressure
4. Check Task metrics: which tasks are failing? What's the data volume?
5. Reduce data size: add more filters, partition pruning, sample to reproduce locally
```

---

### Q68. Scenario: You need to process 1 billion rows with a complex multi-join query. The query plan shows 5 shuffles. How do you reduce shuffle count?
**Answer:**

**5-shuffle query analysis:**
```python
df.explain()  # Show physical plan
# Look for Exchange nodes = shuffle points
```

**Strategies to eliminate shuffles:**

**1. Broadcast small tables (eliminates shuffle for small-large joins):**
```python
# If any join table < 10MB (or up to 8GB with manual broadcast):
result = large_df \
    .join(broadcast(dim_date), "date_id") \       # no shuffle
    .join(broadcast(dim_product), "product_id") \ # no shuffle
    .join(broadcast(dim_store), "store_id")        # no shuffle
# 3 shuffles → 0 shuffles for these joins
```

**2. Bucketing both sides of a large join:**
```python
# Pre-bucket tables once at creation (expensive upfront, saves every future join):
large_orders.write \
    .bucketBy(200, "customer_id") \
    .sortBy("customer_id") \
    .saveAsTable("bucketed_orders")

large_customers.write \
    .bucketBy(200, "customer_id") \
    .sortBy("customer_id") \
    .saveAsTable("bucketed_customers")

# Future joins on customer_id: no shuffle!
spark.read.table("bucketed_orders") \
    .join(spark.read.table("bucketed_customers"), "customer_id")
```

**3. Combine multiple aggregations in one pass:**
```python
# BAD: 3 separate groupBy → 3 shuffles
count_by_region = df.groupBy("region").count()
sum_by_region = df.groupBy("region").sum("amount")
avg_by_region = df.groupBy("region").avg("amount")

# GOOD: one groupBy → 1 shuffle
from pyspark.sql.functions import count, sum, avg
result = df.groupBy("region").agg(
    count("*").alias("cnt"),
    sum("amount").alias("total"),
    avg("amount").alias("average")
)
```

**4. Repartition before chained joins on same key:**
```python
# Repartition once on the join key → subsequent joins on same key use same partitioning
df_orders = df_orders.repartition(200, "customer_id")
df_transactions = df_transactions.repartition(200, "customer_id")
# Join: no additional shuffle since both already partitioned the same way
```

**5. Use AQE to automatically coalesce and avoid redundant shuffles:**
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
```

---

### Q69. Scenario: Implement a Type 2 Slowly Changing Dimension (SCD2) in Delta Lake.
**Answer:**

**SCD Type 2**: When a record changes, we don't update in-place. Instead we close the old record (set `end_date`, mark `is_current = false`) and insert a new record.

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col, current_date, lit, when

# Source: incoming changes (customer updates)
source_df = spark.read.table("staging.customer_updates")

# Target: SCD2 dimension table
target = DeltaTable.forName(spark, "gold.dim_customers")

# Step 1: Identify changed records (rows where attributes differ)
# Step 2: Expire old records and insert new ones using MERGE

# The key: merge with TWO conditions
# 1. Match on business key AND is_current = true AND attributes changed → close old record
# 2. Match on business key AND is_current = true AND no change → do nothing
# 3. Not matched → insert new record

target.alias("t").merge(
    source_df.alias("s"),
    "t.customer_id = s.customer_id AND t.is_current = true"
).whenMatchedUpdate(
    condition="""
        t.customer_name != s.customer_name OR
        t.customer_email != s.customer_email OR
        t.customer_region != s.customer_region
    """,
    set={
        "is_current": "false",
        "end_date": "current_date()",
        "updated_at": "current_timestamp()"
    }
).whenNotMatchedInsert(
    values={
        "customer_id": "s.customer_id",
        "customer_name": "s.customer_name",
        "customer_email": "s.customer_email",
        "customer_region": "s.customer_region",
        "start_date": "current_date()",
        "end_date": "lit('9999-12-31')",
        "is_current": "true",
        "updated_at": "current_timestamp()"
    }
).execute()

# Step 3: Insert new versions of changed records (those we just closed)
changed = source_df.alias("s") \
    .join(
        target.toDF().filter("is_current = false AND end_date = current_date()").alias("t"),
        "customer_id"
    ) \
    .select(
        col("s.customer_id"), col("s.customer_name"),
        col("s.customer_email"), col("s.customer_region"),
        current_date().alias("start_date"),
        lit("9999-12-31").alias("end_date"),
        lit(True).alias("is_current"),
        current_timestamp().alias("updated_at")
    )

changed.write.format("delta").mode("append").saveAsTable("gold.dim_customers")
```

**Alternative: Use DLT APPLY CHANGES for simpler SCD2:**
```sql
APPLY CHANGES INTO LIVE.dim_customers
FROM stream(LIVE.customer_cdc)
KEYS (customer_id)
SEQUENCE BY operation_timestamp
STORED AS SCD TYPE 2
TRACK HISTORY ON * EXCEPT (operation_timestamp);
```

---

### Q70. Scenario: How would you implement monitoring for a production Databricks platform serving 200 data engineers and 50 data scientists?
**Answer:**

**Monitoring architecture layers:**

**1. Job & Pipeline Health (operational)**
```sql
-- Real-time failure dashboard (query system tables)
SELECT 
  j.name AS job_name,
  r.state,
  r.start_time,
  DATEDIFF(minute, r.start_time, COALESCE(r.end_time, CURRENT_TIMESTAMP())) AS duration_min,
  r.run_page_url
FROM system.lakeflow.job_runs r
JOIN system.lakeflow.jobs j ON r.job_id = j.job_id
WHERE r.start_time > DATEADD(hour, -24, CURRENT_TIMESTAMP())
  AND r.state IN ('FAILED', 'TIMED_OUT')
ORDER BY r.start_time DESC;
```

**2. Cost & DBU monitoring (financial)**
```sql
-- Daily DBU by team/project (via cluster tags)
SELECT 
  usage_date,
  custom_tags['team'] as team,
  SUM(usage_quantity) as dbus,
  SUM(usage_quantity * list_price) as cost_usd
FROM system.billing.usage
JOIN system.billing.list_prices USING (sku_name, cloud)
WHERE usage_start_time > DATEADD(day, -30, CURRENT_DATE())
GROUP BY usage_date, team
ORDER BY cost_usd DESC;
```

**3. Query performance (SQL Warehouse)**
```sql
-- Slow query detection
SELECT 
  user_name,
  query_text,
  duration / 1000 AS duration_seconds,
  rows_produced,
  read_bytes / 1e9 AS gb_read
FROM system.query.history
WHERE duration > 300000  -- > 5 minutes
  AND start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())
ORDER BY duration DESC;
```

**4. Security & audit monitoring**
```sql
-- Detect unusual data access patterns
SELECT user_identity.email, COUNT(*) as query_count,
  SUM(CASE WHEN action_name = 'getTable' THEN 1 ELSE 0 END) as tables_accessed
FROM system.access.audit
WHERE event_time > DATEADD(hour, -1, CURRENT_TIMESTAMP())
GROUP BY user_identity.email
HAVING query_count > 1000  -- flag potential data exfiltration
ORDER BY query_count DESC;
```

**5. Platform health alerting**
```
Databricks SQL Dashboard → Databricks Alerts:
  - Alert: any job failure in last 30 minutes → Slack #data-ops
  - Alert: DBU spend > $500 in last hour → email finance + platform lead
  - Alert: any query > 30 min runtime → Slack #data-platform
  - Alert: cluster start failures > 3 in last hour → PagerDuty

External integration:
  - Export system tables to external monitoring (Datadog, Grafana)
  - Webhook notifications from Jobs → Slack/PagerDuty
  - Azure Monitor / CloudWatch integration for VM-level metrics
```

**6. Capacity planning**
```sql
-- Identify peak usage patterns for capacity planning
SELECT 
  HOUR(usage_start_time) as hour_of_day,
  DAYOFWEEK(usage_start_time) as day_of_week,
  AVG(usage_quantity) as avg_dbus,
  MAX(usage_quantity) as max_dbus
FROM system.billing.usage
WHERE usage_start_time > DATEADD(month, -1, CURRENT_DATE())
  AND sku_name LIKE '%ALL_PURPOSE%'
GROUP BY 1, 2
ORDER BY avg_dbus DESC;
```

---

*End of Catalog — 70 Q&A covering all major Databricks Admin & Engineering interview domains.*

---

## Quick Reference: Key Numbers to Know

| Item | Value |
|---|---|
| Delta checkpoint frequency | Every 10 commits |
| VACUUM default retention | 7 days (168 hours) |
| Delta log retention default | 30 days |
| Auto broadcast join threshold | 10 MB (configurable) |
| Default shuffle partitions | 200 |
| Photon DBU multiplier | ~2x (but faster → better price/perf) |
| Liquid Clustering target file size | 128 MB |
| AQE skew factor default | 5x |
| Spark memory fraction | 0.6 (60% of JVM heap) |
| Instance Pool idle termination | 60 min (configurable) |
| Autoloader schema inference checkpoint | Required for production |
