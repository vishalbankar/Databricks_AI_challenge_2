# Databricks-AI-challenge---2
Learning and Practice purpose

# challenge data setup 
follow the instructions form below file 
Challenge setup - [Link to Notebook](code/setup.ipynb)

# Day 01 - Delta conversion & optimization
📦 Delta Lake vs Parquet

Parquet is just a file. Delta is a table system built on Parquet - adding a _delta_log/ transaction log that enables ACID transactions, time travel, schema enforcement, and OPTIMIZE.

⚠️ The Small File Problem

- Every small append creates a new Parquet file. 1000 small files = 1000 Spark tasks. Queries that should take seconds will eventually start taking minutes. This could lead to real production pain point.

🔧 OPTIMIZE & ZORDER 
- OPTIMIZE is Delta Lake's solution to the small file problem
OPTIMIZE compacts many small files into fewer large ones (~1GB target). ZORDER physically reorders data within files so similar values are co-located enabling Spark to skip irrelevant files entirely.

When to run OPTIMIZE:
1. After bulk loads
2. After many small appends accumulate
3. On a schedule (nightly job is common in production this i searched on google)

💡 The real lesson of the day?
- When I ran dbutils.fs.ls() after overwriting a Delta table, I saw 222 files instead of the expected 112. Turns out Delta NEVER immediately deletes old files on overwrite it just marks them deleted in the transaction log and keeps them for time travel. VACUUM is what actually cleans up disk.

Also learned that CREATE TABLE ... LOCATION '/tmp/...' fails in Databricks Free Edition (Unity Catalog restriction) -> the fix💡 saveAsTable() for managed tables works fine .

Take a look at day 01 parcticle learing in notebook 

Day 01 [Link to Notebook](code/Day_01/delta_conversion_and_Optimization.ipynb)


-------------------------------------------------------------------------------------

# Day 02 : Medallion Architecture & Feature Engineering in PySpark

Bronze -> Silver -> Gold | Data Quality | User-Level Aggregations

**Medallion Architecture**

> **What Is It?**
- The Medallion Architecture (also called multi-hop architecture) is a data design pattern recommended by Databricks for organizing data in a Lakehouse. Data moves through three layers of progressively improving quality. Each layer has a strict purpose and no layer should be skipped.

| **Layer**  | **Also Called**          | **Data State**                                                            | **Who Uses It**                    | **Format**                |
|------------|--------------------------|---------------------------------------------------------------------------|------------------------------------|---------------------------|
| **Bronze** | Raw / Landing Zone       | Exactly as received. No cleanup. Store everything including corrupt rows. | Data Engineers only                | Delta (strings preferred) |
| **Silver** | Validated / Cleaned      | Deduped, null-handled, quality-checked, typed schema, enriched.           | Engineers + Data Scientists        | Delta (typed)             |
| **Gold**   | Curated / Business-Ready | Aggregated, feature-engineered, one row per entity (user/product).        | Analysts + ML Engineers + Business | Delta (optimized)         |


--- 
**How Data Flows — The Multi-Hop Pipeline**

> `CSV Files` -> [Ingest + Add Metadata] -> `Bronze Delta Table` -> [Clean + Validate + Dedup] -> `Silver Delta Table` -> [Aggregate + Feature Engineer] -> `Gold Delta Table`

- **`Bronze (Hop 1)`**: Read raw CSV files. Write to Delta with minimal transformation. Add _ingested_at timestamp and _source_file columns. Store everything — even corrupt records.

- **`Silver (Hop 2)`**: Read from Bronze. Apply schema enforcement (cast to proper types). Handle nulls. Remove exact duplicates. Quarantine bad records to a separate table. Write clean data.

- **`Gold (Hop 3)`**: Read from Silver. Create user-level aggregations (total spend, session count, favourite brand, purchase rate). One row per user. This is the feature table.

Take a look at day 01 parcticle learing in notebook 

Day 02 [Link to Notebook](code/Day_02/Day_02_medallian_Architecture_Feature_Engg.ipynb)
