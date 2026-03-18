# Databricks Internals: Complete Deep Dive

> A comprehensive guide to understanding how Databricks works under the hood — from file system architecture to compute engine internals.

---

## Table of Contents

1. [Core Architecture Overview](#1-core-architecture-overview)
2. [The File System: How Data is Actually Stored](#2-the-file-system-how-data-is-actually-stored)
3. [Delta Lake Internals](#3-delta-lake-internals)
4. [Unity Catalog Architecture](#4-unity-catalog-architecture)
5. [Compute Engine: Spark on Databricks](#5-compute-engine-spark-on-databricks)
6. [Table Types and Storage Management](#6-table-types-and-storage-management)
7. [Query Execution Flow](#7-query-execution-flow)
8. [Medallion Architecture in Practice](#8-medallion-architecture-in-practice)
9. [Debugging and Diagnostics](#9-debugging-and-diagnostics)
10. [Common Problems and Solutions](#10-common-problems-and-solutions)

---

## 1. Core Architecture Overview

### The Fundamental Truth

```
┌─────────────────────────────────────────────────────────────┐
│                    DATABRICKS PLATFORM                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐ │
│  │   Compute    │     │   Compute    │     │   Compute    │ │
│  │   (Spark)    │     │   (Spark)    │     │   (Spark)    │ │
│  │   Cluster 1  │     │   Cluster 2  │     │   Cluster N  │ │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘ │
│         │                    │                    │          │
│         └────────────────────┼────────────────────┘          │
│                              │                               │
│                     ┌────────▼────────┐                      │
│                     │  Unity Catalog  │                      │
│                     │  (Metadata Layer)│                     │
│                     └────────┬────────┘                      │
│                              │                               │
└──────────────────────────────┼───────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   YOUR CLOUD STORAGE │
                    │   (ADLS/S3/GCS)      │
                    │                      │
                    │   /bronze/           │
                    │   /silver/           │
                    │   /gold/             │
                    └─────────────────────┘
```

### Key Concept: Separation of Compute and Storage

**Databricks does NOT store your data.** This is the most important concept to understand.

| Component | What It Does | Where It Lives |
|-----------|--------------|----------------|
| **Compute** | Processes data (Spark clusters) | Databricks Platform |
| **Storage** | Stores actual data files | Your Cloud (ADLS, S3, GCS) |
| **Unity Catalog** | Stores metadata about data | Databricks Platform |
| **Delta Log** | Tracks table transactions | Your Cloud Storage |

---

## 2. The File System: How Data is Actually Stored

### 2.1 Cloud Storage Structure

When you create a table in Databricks, here's what actually happens in your cloud storage:

```
adls://your-storage-account.dfs.core.windows.net/container-name/
│
├── tables/
│   └── company_data/
│       └── sales/
│           └── orders/
│               ├── part-00001-abc123.parquet
│               ├── part-00002-def456.parquet
│               ├── part-00003-ghi789.parquet
│               ├── _delta_log/
│               │   ├── 00000000000000000000.json
│               │   ├── 00000000000000000001.json
│               │   ├── 00000000000000000002.json
│               │   └── _last_checkpoint
│               └── _checkpoint/
│
├── volumes/
│   └── (for unstructured data)
│
└── _unity_catalog/
    └── (metadata managed by Unity Catalog)
```

### 2.2 Parquet Files Explained

**Parquet** is a columnar storage format. Here's why it matters:

```
Row-Oriented (CSV, JSON):
┌─────┬────────┬───────┬────────┐
│ id  │ name   │ age   │ city   │
├─────┼────────┼───────┼────────┤
│ 1   │ Alice  │ 30    │ NYC    │
│ 2   │ Bob    │ 25    │ LA     │
│ 3   │ Carol  │ 35    │ Chicago│
└─────┴────────┴───────┴────────┘
→ Reading "age" column requires scanning ALL rows

Columnar (Parquet):
┌─────┬─────┬─────┐   ┌─────┬─────┬─────┬─────┐   ┌─────┬─────┬────────┐
│  1  │  2  │  3  │   │Alice│ Bob │Carol│     │   │ 30  │ 25  │ 35     │
└─────┴─────┴─────┘   └─────┴─────┴─────┴─────┘   └─────┴─────┴────────┘
    id                    name                        age
→ Reading "age" only scans the age column (MUCH FASTER)
```

### 2.3 File Paths in Databricks

You'll encounter three types of paths:

```sql
-- 1. Unity Catalog Path (Recommended)
SELECT * FROM company_data.sales.orders;

-- 2. Hive-style Path
SELECT * FROM delta.`/mnt/company_data/sales/orders`;

-- 3. Cloud Storage Path
SELECT * FROM delta.`abfss://container@storageaccount.dfs.core.windows.net/tables/orders`;
```

---

## 3. Delta Lake Internals

### 3.1 What is Delta Lake?

Delta Lake is an **open-source storage layer** that brings ACID transactions to data lakes.

```
┌────────────────────────────────────────────────────────┐
│                  DELTA TABLE                           │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Data Files (Parquet)        Transaction Log (_delta_log)
│  ┌────────────────────┐      ┌─────────────────────────┐
│  │ part-00001.parquet │      │ 000000.json             │
│  │ part-00002.parquet │      │ - ADD file1.parquet     │
│  │ part-00003.parquet │      │ - ADD file2.parquet     │
│  └────────────────────┘      │ - operation: WRITE      │
│                              │ - timestamp: ...        │
│                              ├─────────────────────────┤
│                              │ 000001.json             │
│                              │ - REMOVE file1.parquet  │
│                              │ - ADD file4.parquet     │
│                              │ - operation: UPDATE     │
│                              └─────────────────────────┘
└────────────────────────────────────────────────────────┘
```

### 3.2 The Transaction Log (_delta_log)

Every operation on a Delta table is recorded in the transaction log:

**Initial Table Creation (00000000000000000000.json):**
```json
{
  "protocol": {
    "minReaderVersion": 1,
    "minWriterVersion": 2
  }
}
{
  "metaData": {
    "id": "abc123-def456",
    "format": {
      "provider": "parquet",
      "options": {}
    },
    "schemaString": "{\"type\":\"struct\",\"fields\":[{\"name\":\"order_id\",\"type\":\"integer\"},{\"name\":\"product\",\"type\":\"string\"},{\"name\":\"price\",\"type\":\"integer\"}]}",
    "partitionColumns": [],
    "configuration": {},
    "createdTime": 1679875200000
  }
}
{
  "txn": {
    "appId": "Spark",
    "version": 1
  }
}
```

**After INSERT (00000000000000000001.json):**
```json
{
  "add": {
    "path": "part-00001-abc123-c0000.snappy.parquet",
    "partitionValues": {},
    "size": 1234,
    "modificationTime": 1679875300000,
    "dataChange": true,
    "stats": "{\"numRecords\":100,\"minValues\":{\"order_id\":1},\"maxValues\":{\"order_id\":100}}"
  }
}
```

**After UPDATE (00000000000000000002.json):**
```json
{
  "remove": {
    "path": "part-00001-abc123-c0000.snappy.parquet",
    "dataChange": true,
    "dropStats": true
  }
}
{
  "add": {
    "path": "part-00002-def456-c0000.snappy.parquet",
    "partitionValues": {},
    "size": 1456,
    "modificationTime": 1679875400000,
    "dataChange": true,
    "stats": "{\"numRecords\":100}"
  }
}
```

### 3.3 Checkpoint Files

Over time, the transaction log grows. Delta Lake creates **checkpoint files** to speed up table initialization:

```
_delta_log/
├── 00000000000000000000.json
├── 00000000000000000001.json
├── ...
├── 00000000000000000999.json
├── 00000000000000001000.checkpoint.parquet  ← Checkpoint (binary)
├── 00000000000000001000.json
├── ...
└── _last_checkpoint  ← Points to latest checkpoint
```

**_last_checkpoint content:**
```json
{
  "version": 1000,
  "size": 12345,
  "parts": null
}
```

### 3.4 How Time Travel Works

Delta Lake's time travel feature queries historical versions:

```sql
-- View all versions
DESCRIBE HISTORY orders;

-- Query version 5
SELECT * FROM orders VERSION AS OF 5;

-- Query by timestamp
SELECT * FROM orders TIMESTAMP AS OF '2024-01-15 10:00:00';

-- Restore to previous version
RESTORE TABLE orders TO VERSION AS OF 5;
```

**Under the hood:**
1. Delta Lake reads the transaction log up to version 5
2. Identifies all active files at that version
3. Queries only those files

---

## 4. Unity Catalog Architecture

### 4.1 Three-Level Namespace

```
┌─────────────────────────────────────────────────────────────┐
│                    UNITY CATALOG HIERARCHY                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CATALOG (company_data)                                      │
│    │                                                         │
│    ├── SCHEMA / DATABASE (sales)                             │
│    │     │                                                   │
│    │     ├── TABLE (orders)                                  │
│    │     ├── TABLE (customers)                               │
│    │     └── VIEW (order_summary)                            │
│    │                                                         │
│    ├── SCHEMA / DATABASE (inventory)                         │
│    │     │                                                   │
│    │     ├── TABLE (products)                                │
│    │     └── TABLE (warehouses)                              │
│    │                                                         │
│    └── SCHEMA / DATABASE (analytics)                         │
│          │                                                   │
│          └── TABLE (daily_sales)                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Where Unity Catalog Metadata Lives

Unity Catalog stores metadata in a **separate metastore**:

```sql
-- Full table reference
catalog_name.schema_name.table_name
-- Example:
company_data.sales.orders
```

**Metadata stored in Unity Catalog:**
- Table schema (column names, types)
- Table location in cloud storage
- Table properties
- Permissions and access control
- Column-level lineage

**Metadata NOT stored in Unity Catalog:**
- Actual data (stored in your cloud)
- Transaction log (stored with data in cloud)

### 4.3 Creating the Hierarchy

```sql
-- Step 1: Create Catalog (top-level container)
CREATE CATALOG company_data;
USE CATALOG company_data;

-- Step 2: Create Schema/Database (logical grouping)
CREATE SCHEMA sales;
USE SCHEMA sales;

-- Step 3: Create Table
CREATE TABLE orders (
    order_id INT,
    product STRING,
    price INT,
    order_date TIMESTAMP
)
USING DELTA;
```

---

## 5. Compute Engine: Spark on Databricks

### 5.1 Cluster Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DATABRICKS CLUSTER                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              DRIVER NODE                              │   │
│  │  - SparkContext                                       │   │
│  │  - Query Planning                                     │   │
│  │  - Task Scheduling                                    │   │
│  │  - Result Aggregation                                 │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   WORKER    │  │   WORKER    │  │   WORKER    │          │
│  │   NODE 1    │  │   NODE 2    │  │   NODE N    │          │
│  │  ┌───────┐  │  │  ┌───────┐  │  │  ┌───────┐  │          │
│  │  │Executor│  │  │  │Executor│  │  │  │Executor│  │          │
│  │  │ Task 1│  │  │  │ Task 2│  │  │  │ Task N│  │          │
│  │  └───────┘  │  │  └───────┘  │  │  └───────┘  │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Query Execution Flow

```
User submits SQL/Python code
        │
        ▼
┌───────────────────┐
│   Driver Node     │
│ - Parses query    │
│ - Creates Plan    │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Logical Plan     │
│  (What to do)     │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Optimized Plan   │
│  (Catalyst Opt.)  │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Physical Plan    │
│  (How to do)      │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Task Distribution│
│  to Executors     │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Worker Nodes     │
│  Process data     │
│  from cloud       │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Results sent     │
│  back to Driver   │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Results to User  │
└───────────────────┘
```

### 5.3 Example: What Happens When You Run a Query

```sql
SELECT product, SUM(price) as total
FROM company_data.sales.orders
WHERE order_id > 100
GROUP BY product;
```

**Step-by-step execution:**

1. **Parsing:** SQL → Abstract Syntax Tree (AST)
2. **Logical Plan:**
   ```
   Project [product, SUM(price) AS total]
   +- Filter (order_id > 100)
      +- TableScan orders
   ```
3. **Optimization (Catalyst):**
   ```
   Project [product, SUM(price) AS total]
   +- HashAggregate [product], [SUM(price)]
      +- Filter (order_id > 100)
         +- TableScan orders
   ```
4. **Physical Plan:**
   ```
   HashAggregate [product], [SUM(price)]
   +- Exchange HashPartitioning [product]
      +- Filter (order_id > 100)
         +- FileScan Parquet [order_id, product, price]
   ```
5. **Task Execution:**
   - Driver splits work into tasks
   - Executors read Parquet files from cloud storage
   - Filter applied (order_id > 100)
   - Aggregation performed
   - Results sent to driver

---

## 6. Table Types and Storage Management

### 6.1 Managed Tables vs External Tables

| Aspect | Managed Table | External Table |
|--------|--------------|----------------|
| **Storage Location** | Databricks manages (default location) | You specify (LOCATION clause) |
| **DROP TABLE** | Deletes data AND metadata | Deletes only metadata |
| **Use Case** | Temporary/experimental tables | Production tables, shared data |
| **Control** | Databricks controls lifecycle | You control lifecycle |

### 6.2 Creating Each Type

**Managed Table:**
```sql
CREATE TABLE managed_orders (
    order_id INT,
    product STRING,
    price INT
)
USING DELTA;

-- Data stored at:
-- abfss://container@storageaccount.dfs.core.windows.net/
--   hive/warehouse/managed_orders/
```

**External Table:**
```sql
CREATE TABLE external_orders (
    order_id INT,
    product STRING,
    price INT
)
USING DELTA
LOCATION 'abfss://container@storageaccount.dfs.core.windows.net/tables/orders/';

-- Data stored at your specified location
```

### 6.3 Finding Where Data Lives

```sql
-- See full table details
DESCRIBE DETAIL orders;
```

**Output:**
```
+--------+--------------------+
| Location | abfss://container@storageaccount.dfs.core.windows.net/tables/orders/ |
| Format   | delta           |
| Provider | delta           |
| ...      | ...             |
+--------+--------------------+
```

---

## 7. Query Execution Flow

### 7.1 Complete Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        QUERY EXECUTION FLOW                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. USER SUBMITS QUERY                                               │
│     %sql                                                             │
│     SELECT * FROM company_data.sales.orders                          │
│                                                                      │
│                          │                                           │
│                          ▼                                           │
│  2. DATABRICKS PLATFORM                                            │
│     - Authenticates user                                           │
│     - Validates permissions (Unity Catalog)                         │
│     - Routes to appropriate cluster                                 │
│                                                                      │
│                          │                                           │
│                          ▼                                           │
│  3. DRIVER NODE                                                     │
│     - Parses SQL                                                   │
│     - Creates logical plan                                         │
│     - Optimizes with Catalyst                                       │
│     - Generates physical plan                                       │
│     - Schedules tasks                                               │
│                                                                      │
│                          │                                           │
│                          ▼                                           │
│  4. EXECUTOR NODES (Workers)                                        │
│     - Read Parquet files from cloud storage                         │
│     - Apply filters/projections                                     │
│     - Perform computations                                          │
│     - Return results to driver                                      │
│                                                                      │
│                          │                                           │
│                          ▼                                           │
│  5. CLOUD STORAGE (ADLS/S3/GCS)                                     │
│     - Stores Parquet data files                                     │
│     - Stores _delta_log (transaction log)                           │
│     - Stores checkpoints                                            │
│                                                                      │
│                          │                                           │
│                          ▼                                           │
│  6. RESULTS RETURNED TO USER                                        │
│     - Displayed in notebook                                         │
│     - Or returned to application                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Understanding Spark UI

When a query runs, access the Spark UI to see:

- **SQL Tab:** Query execution details
- **Stages:** How work was divided
- **Tasks:** Individual unit of work
- **Storage:** Cached data
- **Executors:** Worker node status

---

## 8. Medallion Architecture in Practice

### 8.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                      MEDALLION ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐                                                   │
│  │   SOURCES    │                                                   │
│  │  (APIs, DBs) │                                                   │
│  └──────┬───────┘                                                   │
│         │                                                           │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │   BRONZE     │  ← Raw data (immutable)                          │
│  │   LAYER      │  - Original format preserved                      │
│  │              │  - Append-only                                   │
│  │              │  - Full history                                  │
│  └──────┬───────┘                                                   │
│         │                                                           │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │   SILVER     │  ← Cleaned/conformed data                        │
│  │   LAYER      │  - Deduplicated                                  │
│  │              │  - Validated                                     │
│  │              │  - Enriched                                      │
│  └──────┬───────┘                                                   │
│         │                                                           │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │   GOLD       │  ← Business-level aggregates                      │
│  │   LAYER      │  - Summarized                                    │
│  │              │  - Business metrics                              │
│  │              │  - Ready for reporting                           │
│  └──────────────┘                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 Implementation Example

```sql
-- Create Bronze layer (raw ingestion)
CREATE CATALOG company_data;
CREATE SCHEMA company_data.bronze;

CREATE TABLE company_data.bronze.raw_orders (
    order_json STRING,
    ingestion_time TIMESTAMP
)
USING DELTA
LOCATION 'abfss://container@storageaccount.dfs.core.windows.net/bronze/raw_orders/';

-- Create Silver layer (cleaned)
CREATE SCHEMA company_data.silver;

CREATE TABLE company_data.silver.orders (
    order_id INT,
    product STRING,
    price INT,
    quantity INT,
    order_date TIMESTAMP,
    customer_id INT
)
USING DELTA
LOCATION 'abfss://container@storageaccount.dfs.core.windows.net/silver/orders/';

-- Create Gold layer (aggregated)
CREATE SCHEMA company_data.gold;

CREATE TABLE company_data.gold.daily_sales (
    sale_date DATE,
    product STRING,
    total_revenue DECIMAL(18,2),
    total_quantity INT
)
USING DELTA
LOCATION 'abfss://container@storageaccount.dfs.core.windows.net/gold/daily_sales/';
```

### 8.3 Data Flow Pipeline

```python
# Bronze: Ingest raw data
df_raw = spark.read.json("abfss://container@storageaccount.dfs.core.windows.net/raw/orders/")
df_raw.write.mode("append").saveAsTable("company_data.bronze.raw_orders")

# Silver: Clean and transform
df_silver = spark.sql("""
    SELECT 
        get_json_object(order_json, '$.order_id') as order_id,
        get_json_object(order_json, '$.product') as product,
        get_json_object(order_json, '$.price') as price,
        get_json_object(order_json, '$.quantity') as quantity,
        to_timestamp(get_json_object(order_json, '$.order_date')) as order_date
    FROM company_data.bronze.raw_orders
    WHERE order_json IS NOT NULL
""")
df_silver.write.mode("overwrite").saveAsTable("company_data.silver.orders")

# Gold: Aggregate
df_gold = spark.sql("""
    SELECT 
        DATE(order_date) as sale_date,
        product,
        SUM(price * quantity) as total_revenue,
        SUM(quantity) as total_quantity
    FROM company_data.silver.orders
    GROUP BY DATE(order_date), product
""")
df_gold.write.mode("overwrite").saveAsTable("company_data.gold.daily_sales")
```

---

## 9. Debugging and Diagnostics

### 9.1 Diagnostic Commands Reference

```sql
-- 1. Check all catalogs
SHOW CATALOGS;

-- 2. Check schemas in current catalog
SHOW SCHEMAS;

-- 3. Check tables in current schema
SHOW TABLES;

-- 4. Get table schema
DESCRIBE orders;

-- 5. Get detailed table information
DESCRIBE DETAIL orders;

-- 6. Get table creation info and location
DESCRIBE EXTENDED orders;

-- 7. View table history (time travel)
DESCRIBE HISTORY orders;

-- 8. Check table properties
SHOW TBLPROPERTIES orders;

-- 9. Check partition information
SHOW PARTITIONS orders;

-- 10. View table statistics
ANALYZE TABLE orders COMPUTE STATISTICS;
```

### 9.2 Common Diagnostic Scenarios

**Scenario 1: Table Not Found**
```sql
-- Problem
SELECT * FROM orders;  -- Error: Table not found

-- Debug steps
SHOW CATALOGS;              -- Verify current catalog
SHOW SCHEMAS;               -- Verify current schema
SHOW TABLES;                -- Check if table exists
DESCRIBE EXTENDED orders;   -- Get full table info
```

**Scenario 2: Data Not Visible**
```sql
-- Problem
SELECT * FROM orders;  -- Returns empty results

-- Debug steps
DESCRIBE HISTORY orders;     -- Check if data was written
DESCRIBE DETAIL orders;      -- Verify table location
-- Check cloud storage for Parquet files
```

**Scenario 3: Query Running Slow**
```sql
-- Debug steps
-- 1. Check Spark UI for execution details
-- 2. Check table size
DESCRIBE DETAIL orders;
-- 3. Check if statistics are available
ANALYZE TABLE orders COMPUTE STATISTICS;
-- 4. Consider OPTIMIZE for small files
OPTIMIZE orders;
```

### 9.3 Understanding Error Messages

```
Error: TABLE_OR_VIEW_NOT_FOUND

Causes:
1. Wrong catalog selected
2. Wrong schema selected
3. Table doesn't exist
4. Typo in table name

Solution:
USE CATALOG correct_catalog;
USE SCHEMA correct_schema;
SHOW TABLES LIKE 'order%';
```

```
Error: PATH_NOT_FOUND

Causes:
1. External table location doesn't exist
2. Cloud storage path incorrect
3. Permissions issue

Solution:
DESCRIBE DETAIL table_name;  -- Check location
-- Verify path exists in cloud storage
```

---

## 10. Common Problems and Solutions

### Problem 1: "I created a table but can't find it"

**Root Cause:** Usually a catalog/schema issue

**Solution:**
```sql
-- Check where you are
SELECT CURRENT_CATALOG(), CURRENT_SCHEMA();

-- List all tables across all schemas
SHOW TABLES IN catalog_name.schema_name;

-- Use fully qualified name
SELECT * FROM catalog_name.schema_name.table_name;
```

### Problem 2: "DROP TABLE didn't free up storage"

**Root Cause:** Table was external (LOCATION specified)

**Explanation:**
- Managed tables: DROP deletes data + metadata
- External tables: DROP deletes only metadata

**Solution:**
```sql
-- Check if table is external
DESCRIBE EXTENDED table_name;

-- Manually delete data from cloud storage if needed
```

### Problem 3: "Query is reading too much data"

**Root Cause:** Missing filters, small files, or no partitioning

**Solution:**
```sql
-- Add filters
SELECT * FROM orders WHERE order_date > '2024-01-01';

-- Compact small files
OPTIMIZE orders;

-- Consider partitioning
CREATE TABLE orders_partitioned (
    order_id INT,
    product STRING,
    order_date DATE
)
USING DELTA
PARTITIONED BY (order_date);
```

### Problem 4: "Time travel query returns old data"

**Root Cause:** Querying specific version/timestamp

**Solution:**
```sql
-- Check current version
DESCRIBE HISTORY orders;

-- Query latest (default)
SELECT * FROM orders;

-- Query specific version (intentional)
SELECT * FROM orders VERSION AS OF 5;
```

### Problem 5: "Concurrent writes failing"

**Root Cause:** Optimistic concurrency control

**Solution:**
```sql
-- Retry the operation (usually succeeds on retry)
-- Or use MERGE instead of UPDATE/DELETE
MERGE INTO orders target
USING updates source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

---

## Appendix: Quick Reference

### Essential Commands

```sql
-- Setup
CREATE CATALOG catalog_name;
CREATE SCHEMA schema_name;
USE CATALOG catalog_name;
USE SCHEMA schema_name;

-- Table operations
CREATE TABLE table_name (col1 TYPE, col2 TYPE) USING DELTA;
INSERT INTO table_name VALUES (...);
SELECT * FROM table_name;
UPDATE table_name SET col = value WHERE condition;
DELETE FROM table_name WHERE condition;

-- Diagnostics
SHOW CATALOGS;
SHOW SCHEMAS;
SHOW TABLES;
DESCRIBE DETAIL table_name;
DESCRIBE HISTORY table_name;

-- Maintenance
OPTIMIZE table_name;
VACUUM table_name RETAIN 168 HOURS;
```

### File Structure Reference

```
Cloud Storage/
├── tables/
│   └── catalog/
│       └── schema/
│           └── table_name/
│               ├── part-*.parquet
│               └── _delta_log/
│                   ├── *.json
│                   ├── *.checkpoint.parquet
│                   └── _last_checkpoint
└── _unity_catalog/
    └── (metadata)
```

### Architecture Summary

```
User → Databricks Platform → Unity Catalog (metadata)
                              ↓
                         Compute Cluster (Spark)
                              ↓
                         Cloud Storage (your data)
```

---

## Further Learning Resources

1. **Delta Lake Documentation:** https://docs.delta.io/
2. **Databricks Documentation:** https://docs.databricks.com/
3. **Spark Documentation:** https://spark.apache.org/docs/latest/
4. **Unity Catalog Documentation:** https://docs.databricks.com/data-governance/unity-catalog/
