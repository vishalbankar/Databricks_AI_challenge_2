# Databricks-AI-challenge 2 (Advanced)
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


# Day 03 📒 Notebooks vs Job |Parameterized Notebooks

Here's what I covered:
📒 Notebooks vs Jobs - 
The Production Gap Notebooks = development playground. we click Run, watch it execute, fix errors live. 
Jobs = production reality. Runs at 2 AM when you're asleep, retries on failure, emails alerts. 
Same code. Completely different purpose.

🎛️ Parameterized Notebooks - Making Code Reusable Added dbutils.widgets to accept parameters like catalog, run_date, write_mode. 
The catch? Widget values are ALWAYS strings. Spent an hour debugging why my dates weren't comparing properly before I realized I needed to cast them 😅

On Day 02 I had one big notebook doing Bronze → Silver → Gold. 
Today I learned to break it into pieces that actually run themselves in production.

🔗 The Orchestrator Pattern Split my monolith into 3 notebooks:
01_bronze_ingest (raw data in)
02_silver_clean (quality checks)
03_gold_features (analytics-ready features)
orchestrator (the master brain using UI by creating job and adding each layer notebook as tasks and and setting dependency on bronze and silver)

Each notebook does ONE thing. Returns SUCCESS or FAIL. The orchestrator calls them in sequence using dbutils.notebook.run() and validates each step before moving forward.

⏰ Scheduling - Set It and Forget It Created a Databricks Job with cron schedule: 0 0 9 * * ? (daily at 2 AM) Added 2 automatic retries with 5-minute delays Configured email alerts on failure

Take a look at day 01 parcticle learing in notebook 
Day 03 [Link to Notebook](Day_03)

# Day 04 Structured Streaming
# Micro-batch | Checkpointing | Streaming -> Delta

Structured Streaming — and honestly, it was the most mind-shifting concept so far.

I always thought streaming meant completely different code. It doesn't.

Here's what actually changed my thinking:

🔷 Batch vs Streaming — it's just two lines
Batch:  spark.read.format('delta').table(...)
Stream: spark.readStream.format('delta').table(...)

That's it. The transformations — filter(), groupBy(), withColumn() — are identical. Only the read and write methods change. The same ecommerce pipeline I built in Day 2 works as a stream with minimal changes.

🔷 Micro-batch — not what I expected
Structured Streaming doesn't process one event at a time. It collects all new data since the last run, processes it as a batch, writes results, then waits for the next trigger. This is called micro-batch and it's why the API looks exactly like batch code.

🔷 Checkpointing — the memory of a stream
Every micro-batch writes a checkpoint — a small folder that records exactly what was already processed. When the stream restarts after a failure or notebook re-run, it reads the checkpoint and continues from where it left off. No reprocessing. No duplicates.

I tested this today:
→ Run 1: streamed all existing ecommerce events (Oct + Nov 2019)
→ Appended 3 new December events to the source table
→ Run 2: stream picked up ONLY the 3 new rows

The checkpoint is what made that work.

🔷 trigger(availableNow=True) — the key for notebooks
Instead of running the stream continuously, availableNow processes all pending data and stops automatically. No manual interruption needed. Perfect for scheduled jobs and learning.

Take a look at day 01 parcticle learing in notebook Day 04
Day 04 [Link to Notebook](Day_04/Day_04_Structured%20Streaming.ipynb)
