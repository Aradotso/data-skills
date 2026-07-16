---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering with Microsoft Fabric using Lakehouse, Medallion Architecture, Dataflow Gen2, PySpark notebooks, and Power BI for unified analytics
triggers:
  - "build a unified analytics platform with Microsoft Fabric"
  - "implement medallion architecture in Fabric lakehouse"
  - "create dataflow gen2 transformations"
  - "process data with Fabric notebooks and PySpark"
  - "build semantic model in Microsoft Fabric"
  - "design end-to-end data pipeline in Fabric"
  - "set up bronze silver gold layers in OneLake"
  - "integrate Power BI with Fabric lakehouse"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build production-grade unified analytics platforms using Microsoft Fabric. It covers implementing Medallion Architecture (Bronze → Silver → Gold), data ingestion with Dataflow Gen2, PySpark transformations in Fabric Notebooks, semantic modeling, and Power BI integration—all within a single SaaS platform powered by OneLake.

## Overview

Microsoft Fabric is a unified analytics platform that consolidates data engineering, data science, data warehousing, and business intelligence into a single SaaS offering. This project demonstrates how to:

- **Organize data** using Lakehouse and Medallion Architecture
- **Ingest and transform** data with Dataflow Gen2
- **Process business logic** using Fabric Notebooks (PySpark)
- **Model analytics** with Semantic Models
- **Visualize insights** with Power BI

All workloads share a common storage foundation through **OneLake**, eliminating data silos and integration complexity.

## Prerequisites

- **Microsoft Fabric workspace** (capacity or trial)
- **Power BI Premium** or Fabric capacity
- Access to **Fabric Lakehouse**
- Basic understanding of **PySpark** and **Power Query M**

## Core Architecture

### Medallion Architecture Layers

```
Bronze Layer (Raw)
    ↓
Silver Layer (Cleansed)
    ↓
Gold Layer (Business-Ready)
    ↓
Semantic Model
    ↓
Power BI Reports
```

## Getting Started

### 1. Create a Fabric Lakehouse

In Microsoft Fabric workspace:

1. Click **+ New** → **Lakehouse**
2. Name it (e.g., `retail_analytics_lakehouse`)
3. The lakehouse automatically connects to OneLake

### 2. Organize Folder Structure

Create folders in the lakehouse **Files** section:

```
Files/
├── bronze/
│   ├── online_retail/
├── silver/
│   ├── online_retail_clean/
└── gold/
    ├── revenue_summary/
    ├── product_performance/
    └── customer_analytics/
```

You can create folders via the Fabric UI or programmatically using notebooks.

## Data Ingestion with Dataflow Gen2

### Creating a Dataflow Gen2

1. In your Fabric workspace: **+ New** → **Dataflow Gen2**
2. Connect to your data source (CSV, database, API, etc.)
3. Apply Power Query transformations
4. Set destination to Lakehouse **bronze/** folder

### Example: Bronze Layer Ingestion

In Dataflow Gen2, use Power Query M to load raw data:

```m
let
    Source = Csv.Document(
        Web.Contents("https://your-data-source.com/online_retail.csv"),
        [Delimiter=",", Columns=8, Encoding=65001, QuoteStyle=QuoteStyle.None]
    ),
    PromotedHeaders = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    ChangedType = Table.TransformColumnTypes(PromotedHeaders,{
        {"InvoiceNo", type text},
        {"StockCode", type text},
        {"Description", type text},
        {"Quantity", Int64.Type},
        {"InvoiceDate", type datetime},
        {"UnitPrice", type number},
        {"CustomerID", type text},
        {"Country", type text}
    })
in
    ChangedType
```

**Destination**: Set to `Lakehouse` → `Files/bronze/online_retail/`

### Example: Silver Layer Transformation

Continue in Dataflow Gen2 to create cleansed data:

```m
let
    Source = Lakehouse.Contents(null){[workspaceId="YOUR_WORKSPACE_ID"]}[Data],
    BronzeData = Source{[lakehouseId="YOUR_LAKEHOUSE_ID"]}[Data],
    OnlineRetail = BronzeData{[itemName="bronze"]}[Data]{[itemName="online_retail"]}[Data],
    
    // Remove nulls in key columns
    FilteredRows = Table.SelectRows(OnlineRetail, each [CustomerID] <> null and [InvoiceNo] <> null),
    
    // Remove duplicates
    RemovedDuplicates = Table.Distinct(FilteredRows),
    
    // Add business columns
    AddedLineTotal = Table.AddColumn(RemovedDuplicates, "line_total", each [Quantity] * [UnitPrice], type number),
    AddedYear = Table.AddColumn(AddedLineTotal, "year", each Date.Year([InvoiceDate]), Int64.Type),
    AddedMonth = Table.AddColumn(AddedYear, "month", each Date.Month([InvoiceDate]), Int64.Type),
    AddedIsReturn = Table.AddColumn(AddedMonth, "is_return", each [Quantity] < 0, type logical),
    
    // Filter out returns for analysis
    FilteredPositive = Table.SelectRows(AddedIsReturn, each [is_return] = false)
in
    FilteredPositive
```

**Destination**: Set to `Lakehouse` → `Files/silver/online_retail_clean/`

## PySpark Processing in Fabric Notebooks

### Creating a Fabric Notebook

1. **+ New** → **Notebook**
2. Default language: **PySpark (Python)**
3. Auto-attached to Spark compute

### Reading Data from OneLake

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

# Read Silver layer data
df_silver = spark.read.format("delta").load("Files/silver/online_retail_clean/")

# Show schema
df_silver.printSchema()

# Preview data
display(df_silver.limit(10))
```

### Gold Layer: Revenue Summary

```python
# Aggregate revenue by year and month
revenue_summary = df_silver.groupBy("year", "month").agg(
    F.sum("line_total").alias("total_revenue"),
    F.countDistinct("InvoiceNo").alias("total_orders"),
    F.countDistinct("CustomerID").alias("unique_customers"),
    F.sum("Quantity").alias("total_quantity")
).orderBy("year", "month")

# Write to Gold layer as Delta table
revenue_summary.write.format("delta").mode("overwrite").save("Files/gold/revenue_summary/")

# Register as table for SQL access
revenue_summary.write.format("delta").mode("overwrite").saveAsTable("gold_revenue_summary")
```

### Gold Layer: Product Performance

```python
# Product-level analytics
product_performance = df_silver.groupBy("StockCode", "Description").agg(
    F.sum("line_total").alias("total_revenue"),
    F.sum("Quantity").alias("total_quantity_sold"),
    F.countDistinct("InvoiceNo").alias("order_count"),
    F.countDistinct("CustomerID").alias("customer_count")
).orderBy(F.desc("total_revenue"))

# Add revenue rank
from pyspark.sql.window import Window

window_spec = Window.orderBy(F.desc("total_revenue"))
product_performance = product_performance.withColumn(
    "revenue_rank",
    F.row_number().over(window_spec)
)

# Save to Gold
product_performance.write.format("delta").mode("overwrite").save("Files/gold/product_performance/")
product_performance.write.format("delta").mode("overwrite").saveAsTable("gold_product_performance")
```

### Gold Layer: Customer Analytics (RFM Segmentation)

```python
from pyspark.sql.window import Window
from datetime import datetime

# Calculate reference date (max date in dataset)
max_date = df_silver.agg(F.max("InvoiceDate")).collect()[0][0]

# Calculate RFM metrics
customer_rfm = df_silver.groupBy("CustomerID").agg(
    F.datediff(F.lit(max_date), F.max("InvoiceDate")).alias("recency"),
    F.countDistinct("InvoiceNo").alias("frequency"),
    F.sum("line_total").alias("monetary")
)

# Quartile-based scoring
def calculate_rfm_score(df, column, ascending=True):
    quartiles = df.approxQuantile(column, [0.25, 0.5, 0.75], 0.01)
    if ascending:
        return F.when(F.col(column) <= quartiles[0], 4)\
                .when(F.col(column) <= quartiles[1], 3)\
                .when(F.col(column) <= quartiles[2], 2)\
                .otherwise(1)
    else:
        return F.when(F.col(column) <= quartiles[0], 1)\
                .when(F.col(column) <= quartiles[1], 2)\
                .when(F.col(column) <= quartiles[2], 3)\
                .otherwise(4)

customer_rfm = customer_rfm.withColumn("r_score", calculate_rfm_score(customer_rfm, "recency", ascending=True))
customer_rfm = customer_rfm.withColumn("f_score", calculate_rfm_score(customer_rfm, "frequency", ascending=False))
customer_rfm = customer_rfm.withColumn("m_score", calculate_rfm_score(customer_rfm, "monetary", ascending=False))

# Combined RFM score
customer_rfm = customer_rfm.withColumn(
    "rfm_score",
    (F.col("r_score") + F.col("f_score") + F.col("m_score")) / 3
)

# Customer segment
customer_rfm = customer_rfm.withColumn(
    "customer_segment",
    F.when(F.col("rfm_score") >= 3.5, "Champions")
    .when(F.col("rfm_score") >= 2.5, "Loyal")
    .when(F.col("rfm_score") >= 1.5, "At Risk")
    .otherwise("Inactive")
)

# Save to Gold
customer_rfm.write.format("delta").mode("overwrite").save("Files/gold/customer_analytics/")
customer_rfm.write.format("delta").mode("overwrite").saveAsTable("gold_customer_analytics")
```

### Optimizing Delta Tables

```python
# Optimize Delta table for better query performance
spark.sql("OPTIMIZE gold_revenue_summary")
spark.sql("OPTIMIZE gold_product_performance")
spark.sql("OPTIMIZE gold_customer_analytics")

# Z-ORDER by frequently filtered columns
spark.sql("OPTIMIZE gold_product_performance ZORDER BY (StockCode)")
spark.sql("OPTIMIZE gold_customer_analytics ZORDER BY (customer_segment)")
```

## Creating a Semantic Model

### Option 1: Auto-generate from Lakehouse

1. Open your Lakehouse
2. Navigate to **SQL analytics endpoint**
3. Select Gold tables → **New semantic model**
4. Choose tables and relationships
5. Define measures

### Option 2: Define Measures in DAX

In the Semantic Model editor:

```dax
// Total Revenue
Total Revenue = SUM('gold_revenue_summary'[total_revenue])

// Year-over-Year Growth
YoY Growth % = 
VAR CurrentYear = SUM('gold_revenue_summary'[total_revenue])
VAR PreviousYear = CALCULATE(
    SUM('gold_revenue_summary'[total_revenue]),
    DATEADD('gold_revenue_summary'[year], -1, YEAR)
)
RETURN
DIVIDE(CurrentYear - PreviousYear, PreviousYear, 0)

// Average Order Value
Average Order Value = 
DIVIDE(
    [Total Revenue],
    SUM('gold_revenue_summary'[total_orders]),
    0
)

// Customer Lifetime Value
Customer LTV = 
AVERAGEX(
    'gold_customer_analytics',
    'gold_customer_analytics'[monetary]
)
```

### Defining Relationships

```dax
// If using separate dimension tables
RELATIONSHIP(
    'gold_product_performance'[StockCode],
    'dim_products'[StockCode],
    MANY_TO_ONE
)
```

## Power BI Integration

### Connecting to Semantic Model

1. Open **Power BI Desktop**
2. **Get Data** → **Power BI semantic models**
3. Select your Fabric semantic model
4. Build reports using pre-defined measures

### DirectLake Mode

Fabric semantic models support **DirectLake** mode, which queries Delta tables directly in OneLake without import or DirectQuery overhead:

- Faster performance than DirectQuery
- No data duplication like Import mode
- Real-time data access

## Common Patterns

### Pattern 1: Incremental Load to Bronze

```python
from datetime import datetime, timedelta

# Get last processed date
last_date = spark.sql("""
    SELECT MAX(InvoiceDate) as max_date 
    FROM bronze_online_retail
""").collect()[0]['max_date']

# Load only new data
new_data = spark.read.format("csv")\
    .option("header", "true")\
    .load("source_path")\
    .filter(F.col("InvoiceDate") > last_date)

# Append to Bronze
new_data.write.format("delta").mode("append").save("Files/bronze/online_retail/")
```

### Pattern 2: Type 2 SCD in Silver

```python
from delta.tables import DeltaTable

# Read existing Silver table
silver_table = DeltaTable.forPath(spark, "Files/silver/online_retail_clean/")

# New data with effective dates
new_records = df_bronze\
    .withColumn("effective_from", F.current_timestamp())\
    .withColumn("effective_to", F.lit(None).cast("timestamp"))\
    .withColumn("is_current", F.lit(True))

# Merge with SCD Type 2 logic
silver_table.alias("target").merge(
    new_records.alias("source"),
    "target.InvoiceNo = source.InvoiceNo AND target.is_current = true"
).whenMatchedUpdate(
    condition="target.Quantity != source.Quantity OR target.UnitPrice != source.UnitPrice",
    set={
        "effective_to": "source.effective_from",
        "is_current": "false"
    }
).whenNotMatchedInsert(
    values={
        "InvoiceNo": "source.InvoiceNo",
        "StockCode": "source.StockCode",
        "Quantity": "source.Quantity",
        "UnitPrice": "source.UnitPrice",
        "effective_from": "source.effective_from",
        "effective_to": "source.effective_to",
        "is_current": "source.is_current"
    }
).execute()
```

### Pattern 3: Data Quality Checks

```python
# Define data quality rules
def run_quality_checks(df, layer_name):
    checks = {
        "total_rows": df.count(),
        "null_customer_ids": df.filter(F.col("CustomerID").isNull()).count(),
        "negative_quantities": df.filter(F.col("Quantity") < 0).count(),
        "invalid_prices": df.filter(F.col("UnitPrice") <= 0).count()
    }
    
    # Log results
    quality_df = spark.createDataFrame([{
        "layer": layer_name,
        "check_timestamp": datetime.now(),
        **checks
    }])
    
    quality_df.write.format("delta").mode("append").save("Files/monitoring/data_quality/")
    
    return checks

# Run checks
quality_results = run_quality_checks(df_silver, "silver")
print(quality_results)
```

## Configuration

### Environment Variables

```python
import os

# Access Fabric workspace and lakehouse IDs
WORKSPACE_ID = os.environ.get("FABRIC_WORKSPACE_ID")
LAKEHOUSE_ID = os.environ.get("FABRIC_LAKEHOUSE_ID")

# Data source connection strings
SOURCE_CONNECTION_STRING = os.environ.get("DATA_SOURCE_CONNECTION_STRING")
```

### Notebook Parameters

Use Fabric notebook parameters for dynamic execution:

```python
# Define parameters in notebook cell tagged as "parameters"
start_date = "2024-01-01"  # default value
end_date = "2024-12-31"

# Use in processing
df_filtered = df_silver.filter(
    (F.col("InvoiceDate") >= start_date) & 
    (F.col("InvoiceDate") <= end_date)
)
```

### Spark Configuration

```python
# Configure Spark session in notebook
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")
```

## Troubleshooting

### Issue: Dataflow Gen2 Timeout

**Symptom**: Large data ingestion fails or times out

**Solution**:
- Split data into smaller batches
- Use incremental refresh in Dataflow Gen2 settings
- Increase dataflow timeout in workspace settings

### Issue: Notebook Out of Memory

**Symptom**: PySpark job fails with OOM error

**Solution**:
```python
# Repartition data for better memory distribution
df = df.repartition(200)

# Use broadcast for small dimension tables
from pyspark.sql.functions import broadcast
df_joined = df_large.join(broadcast(df_small), "key")

# Write in smaller batches
df.write.format("delta")\
    .option("maxRecordsPerFile", 100000)\
    .mode("overwrite")\
    .save("Files/gold/output/")
```

### Issue: Delta Table Version Conflict

**Symptom**: `ConcurrentAppendException` or version conflict

**Solution**:
```python
# Enable optimistic concurrency
spark.conf.set("spark.databricks.delta.optimisticTransaction.enabled", "true")

# Or use explicit locking
from delta.tables import DeltaTable
delta_table = DeltaTable.forPath(spark, "Files/gold/table/")
delta_table.vacuum(168)  # Clean up old versions older than 7 days
```

### Issue: Semantic Model Not Refreshing

**Symptom**: Power BI shows stale data

**Solution**:
1. Check semantic model refresh history in Fabric
2. Verify Delta table is committed:
   ```python
   spark.sql("DESCRIBE HISTORY gold_revenue_summary").show()
   ```
3. Manually refresh semantic model or configure scheduled refresh

### Issue: Slow Query Performance

**Symptom**: Power BI reports load slowly

**Solution**:
```python
# Optimize Delta tables
spark.sql("OPTIMIZE gold_revenue_summary ZORDER BY (year, month)")

# Create aggregations in Gold layer
# Use file statistics
spark.sql("ANALYZE TABLE gold_revenue_summary COMPUTE STATISTICS")
```

## Best Practices

1. **Layered Processing**: Always maintain Bronze (raw) → Silver (clean) → Gold (business) separation
2. **Idempotent Pipelines**: Design transformations to be rerunnable without side effects
3. **Delta Lake**: Use Delta format for ACID transactions and time travel
4. **Semantic Layer**: Centralize business logic in the semantic model, not in reports
5. **Incremental Processing**: Avoid full reprocessing; use watermarks and incremental patterns
6. **Monitoring**: Implement data quality checks and logging at each layer
7. **Optimization**: Regularly run `OPTIMIZE` and `VACUUM` on Delta tables
8. **Security**: Use workspace roles and row-level security in semantic models

## Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric/)
- [Dataflow Gen2 Guide](https://learn.microsoft.com/en-us/fabric/data-factory/dataflows-gen2-overview)
- [Delta Lake on Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-overview)
- [Semantic Models](https://learn.microsoft.com/en-us/fabric/data-engineering/semantic-models)
