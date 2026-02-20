# Databricks-AI-challenge---2
Learning and Practice purpose

# challenge data setup 
follow the instructions form below file 
Challenge setup - [Link to Notebook](code/setup.ipynb)

# Day 01 - Delta conversion & optimization
📦 Delta Lake vs Parquet
Parquet is just a file. Delta is a table system built on Parquet - adding a _delta_log/ transaction log that enables ACID transactions, time travel, schema enforcement, and OPTIMIZE.

⚠️ The Small File Problem
Every small append creates a new Parquet file. 1000 small files = 1000 Spark tasks. Queries that should take seconds will eventually start taking minutes. This could lead to real production pain point.

🔧 OPTIMIZE & ZORDER 
OPTIMIZE is Delta Lake's solution to the small file problem
OPTIMIZE compacts many small files into fewer large ones (~1GB target). ZORDER physically reorders data within files so similar values are co-located enabling Spark to skip irrelevant files entirely.

When to run OPTIMIZE:
1. After bulk loads
2. After many small appends accumulate
3. On a schedule (nightly job is common in production this i searched on google)

💡 The real lesson of the day?
When I ran dbutils.fs.ls() after overwriting a Delta table, I saw 222 files instead of the expected 112. Turns out Delta NEVER immediately deletes old files on overwrite it just marks them deleted in the transaction log and keeps them for time travel. VACUUM is what actually cleans up disk.

Also learned that CREATE TABLE ... LOCATION '/tmp/...' fails in Databricks Free Edition (Unity Catalog restriction) -> the fix💡 saveAsTable() for managed tables works fine .

Take a look at day 01 parcticle learing in notebook 

Day 01 [Link to Notebook](code/Day_01/delta_conversion_and_Optimization.ipynb)
