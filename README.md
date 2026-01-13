
# BigMart Sales Analytics Platform (Databricks + PySpark)
## Overview

This project implements a production-grade analytics pipeline for retail sales data using Databricks, PySpark, and Delta Lake.  
The goal is to transform raw transactional data into reliable, query-ready datasets that support business insights such as outlet performance, sales distribution, and trend analysis.

The project is intentionally designed to reflect real-world data engineering practices rather than a learning lab. It emphasizes correctness, maintainability, and clarity over shortcuts.

---

## Problem Statement

Retail organizations often receive sales data as flat files stored in object storage. These files are:

- Not analytics-ready
- Prone to schema drift and data quality issues
- Difficult to use directly for reporting and decision-making

The core problem is to **ingest raw sales data, clean and standardize it, and produce trusted analytical datasets** that can be queried consistently and scheduled to run automatically as new data arrives.

This project solves that problem by building a structured ETL pipeline that:
- Separates raw, cleaned, and aggregated data
- Enforces schema and transformation boundaries
- Produces stable Delta tables suitable for downstream analytics

---
## High-Level Architecture

```
Amazon S3 (Raw CSV)
|
v
Databricks ETL Pipeline (PySpark)
|
v
Bronze Layer (Raw Delta)
|
v
Silver Layer (Cleaned Delta)
|
v
Gold Layer (Aggregated Delta)
|
v
SQL / BI / Analytics Consumers

````

---

## Architecture Design and Rationale

### Object Storage (Amazon S3)
S3 is used as the landing zone for raw data because it is:
- Cost-effective for large volumes
- Decoupled from compute
- Suitable for batch-based ingestion patterns

Raw files are treated as immutable inputs. No transformations occur at this stage.

---

### Databricks + PySpark

Databricks is used as the processing layer due to:
- Native support for Apache Spark
- Built-in orchestration for ETL pipelines
- Tight integration with Delta Lake and Unity Catalog

PySpark is chosen over SQL-only pipelines because:
- Complex transformations are easier to express and review
- Business logic can be versioned and tested as code
- The same code scales from small datasets to large volumes

---

### Delta Lake Storage Layers

The pipeline follows a layered data model to enforce separation of concerns.

| Layer   | Purpose | Design Reasoning |
|--------|--------|------------------|
| Bronze | Raw ingestion | Preserves source data exactly as received for traceability and reprocessing |
| Silver | Cleaned data | Applies data quality rules and standardization |
| Gold | Aggregations | Optimized for analytics and reporting use cases |

This structure makes failures easier to isolate and reduces the cost of change when business logic evolves.

---

### Unity Catalog & External Locations

Unity Catalog is used to:
- Securely access S3 without embedding credentials
- Manage table ownership and governance
- Provide consistent object naming across environments

External locations ensure that storage and compute remain decoupled while maintaining strong access control.

---

## End-to-End Data Flow

1. Raw sales CSV files are stored in S3
2. The ETL pipeline reads the data into a Bronze Delta table
3. Data is cleaned and standardized into a Silver table
4. Business-level aggregations are produced in a Gold table
5. The pipeline is scheduled to run automatically on a defined cadence

Each stage is materialized as a Delta table to ensure durability and observability.

---

## Data Transformations and Insights

### Key Transformations

- Handling missing values in numerical and categorical fields
- Standardizing inconsistent text attributes
- Enforcing consistent schema across runs
- Aggregating sales metrics by outlet

### Example Business Insight

**Total and Average Sales by Outlet**

This aggregation allows stakeholders to:
- Identify high-performing outlets
- Compare outlet efficiency
- Support inventory and marketing decisions

```python
df.groupBy("Outlet_Identifier").agg(
    sum("Item_Outlet_Sales").alias("total_sales"),
    avg("Item_Outlet_Sales").alias("avg_sales")
)
````

The logic is intentionally simple and transparent, making it easy to validate and extend.

---

## Scheduling and Automation

The pipeline is scheduled using Databricks’ native pipeline scheduler rather than an external job wrapper.

Design choice:

* Pipelines already manage dependencies and execution state
* Reduces operational complexity
* Aligns with Databricks’ recommended operational model

---

## Monitoring and Reliability

* Each pipeline run is tracked with execution status and duration
* Table-level metrics are available for performance review
* Failures are isolated to specific layers (Bronze, Silver, or Gold)

This makes the system suitable for production monitoring and on-call support.

---

## Repository Structure

```
.
├── transformations/
│   └── bigmart_etl.py
├── README.md
```

The project intentionally avoids unnecessary abstraction. All transformation logic is centralized and version-controlled.

---

## Screenshots

*(Placeholders for documentation and reviews)*

* **Pipeline Graph:** Visual lineage showing Bronze → Silver → Gold execution
* **SQL Query Results:** Output of aggregated Gold table
* **Pipeline Run History:** Successful scheduled executions with timestamps

---

## Design Tradeoffs

| Decision                       | Tradeoff                                                                |
| ------------------------------ | ----------------------------------------------------------------------- |
| Batch ETL instead of streaming | Simpler operations, acceptable latency for business use case            |
| PySpark over SQL-only          | Slightly higher learning curve, better long-term maintainability        |
| Materialized tables            | Higher storage cost, significantly better reliability and debuggability |

---

## Future Enhancements

* Incremental ingestion using file-based tracking
* Data quality expectations with enforced constraints
* Partitioning and optimization for large-scale datasets
* Integration with BI tools for reporting

---

## Summary

This project demonstrates how to build a clean, production-ready analytics pipeline using Databricks and PySpark.
The focus is on correctness, clarity, and real-world design decisions rather than shortcuts or demos.

It is suitable for:

* Technical interviews
* Code reviews
* Extension into larger data platforms

```
```
