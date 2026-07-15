---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering on Microsoft Fabric using Lakehouse, Dataflow Gen2, PySpark, and Power BI with Medallion Architecture
triggers:
  - how do I build a lakehouse in Microsoft Fabric
  - implement medallion architecture with fabric
  - use dataflow gen2 for data transformation
  - create fabric notebooks with pyspark
  - build semantic models in microsoft fabric
  - set up bronze silver gold layers in fabric
  - create power bi reports from fabric lakehouse
  - use onelake for unified analytics
---

# Microsoft Fabric Unified Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in building end-to-end unified analytics platforms using Microsoft Fabric, implementing the Medallion Architecture (Bronze → Silver → Gold) with OneLake, Dataflow Gen2, Fabric Notebooks (PySpark), Semantic Models, and Power BI.

## What This Project Does

This project demonstrates a production-ready unified analytics platform that:

- Implements Lakehouse architecture on Microsoft Fabric
- Follows Medallion Architecture pattern (Bronze → Silver → Gold layers)
- Uses Dataflow Gen2 for low-code data transformations
- Leverages Fabric Notebooks with PySpark for business logic
- Creates centralized Semantic Models for consistent KPIs
- Integrates natively with Power BI for visualization
- Stores all data in OneLake for unified storage

The solution processes retail transaction data through progressive refinement stages, from raw ingestion to business-ready analytics.

## Prerequisites

- Microsoft Fabric workspace access
- Power BI Pro or Premium license
- Basic Python/PySpark knowledge
- Understanding of data lakehouse concepts

## Core Architecture

### Medallion Layer Structure

```
Lakehouse/
├── Files/
│   ├── bronze/          # Raw source data
│   ├── silver/          # Cleansed and enriched
│   └── gold/            # Business-ready analytics
└── Tables/
    ├── bronze_*         # Bronze Delta tables
    ├── silver_*         # Silver Delta tables
    └── gold_*           # Gold Delta tables
```

## Setting Up a Fabric Lakehouse

### 1. Create Lakehouse

In Microsoft Fabric workspace:
1. Click **+ New**
2. Select **Lakehouse**
3. Name it (e.g., `retail_analytics_lakehouse`)
4. Create the Bronze, Silver, Gold folder structure

### 2. Configure OneLake Integration

OneLake is automatically configured when you create a Lakehouse. All data is stored in Delta Lake format with ACID transactions.

## Working with Dataflow Gen2

### Creating a Dataflow

Dataflow Gen2 provides low-code ETL for Bronze → Silver transformations.

**Key Transformations:**

```powerquery
// Remove duplicates
= Table.Distinct(Source, {"InvoiceNo", "StockCode", "CustomerID"})

// Handle missing values
= Table.ReplaceValue(PreviousStep, null, "", Replacer.ReplaceValue, {"Description"})

// Add business columns
= Table.AddColumn(PreviousStep, "line_total", 
    each [Quantity] * [UnitPrice], type number)

= Table.AddColumn(PreviousStep, "year", 
    each Date.Year([InvoiceDate]), type number)

= Table.AddColumn(PreviousStep, "month", 
    each Date.Month([InvoiceDate]), type number)

= Table.AddColumn(PreviousStep, "is_return", 
    each if [Quantity] < 0 then true else false, type logical)

// Standardize data types
= Table.TransformColumnTypes(PreviousStep, {
    {"InvoiceNo", type text},
    {"StockCode", type text},
    {"Quantity", Int64.Type},
    {"InvoiceDate", type datetime},
    {"UnitPrice", type number},
    {"CustomerID", type text}
})
```

**Publishing to Silver Layer:**

1. Configure destination as Lakehouse
2. Set table name with `silver_` prefix
3. Choose **Replace** or **Append** load method
4. Enable **Publish to OneLake**

## Fabric Notebooks with PySpark

### Connecting to Lakehouse

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.window import Window

# Fabric notebooks automatically connect to attached lakehouse
# Access OneLake paths directly
lakehouse_path = "Files/silver/"
```

### Reading Silver Data

```python
# Read Delta table from Silver layer
df_transactions = spark.read.format("delta").table("silver_transactions")

# Or read from Files
df_transactions = spark.read.format("delta") \
    .load("Files/silver/transactions")

# Show schema
df_transactions.printSchema()
```

### Creating Gold Business Metrics

**Revenue Trends:**

```python
from pyspark.sql.functions import col, sum, count, avg, month, year

# Calculate monthly revenue metrics
gold_revenue_trends = df_transactions \
    .filter(col("is_return") == False) \
    .groupBy(
        year("InvoiceDate").alias("year"),
        month("InvoiceDate").alias("month")
    ) \
    .agg(
        sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("total_transactions"),
        countDistinct("CustomerID").alias("unique_customers"),
        avg("line_total").alias("avg_transaction_value")
    ) \
    .orderBy("year", "month")

# Write to Gold layer as Delta table
gold_revenue_trends.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable("gold_revenue_trends")
```

**Product Performance:**

```python
# Product-level analytics
gold_product_performance = df_transactions \
    .filter(col("is_return") == False) \
    .groupBy("StockCode", "Description") \
    .agg(
        sum("Quantity").alias("total_quantity_sold"),
        sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("total_orders"),
        avg("UnitPrice").alias("avg_unit_price"),
        countDistinct("CustomerID").alias("unique_customers")
    ) \
    .withColumn("revenue_rank", 
        rank().over(Window.orderBy(col("total_revenue").desc()))
    )

gold_product_performance.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold_product_performance")
```

**Customer RFM Segmentation:**

```python
from pyspark.sql.functions import max, datediff, current_date

# Calculate RFM metrics
reference_date = df_transactions.agg(max("InvoiceDate")).collect()[0][0]

gold_customer_rfm = df_transactions \
    .filter(col("is_return") == False) \
    .groupBy("CustomerID") \
    .agg(
        datediff(lit(reference_date), max("InvoiceDate")).alias("recency"),
        count("InvoiceNo").alias("frequency"),
        sum("line_total").alias("monetary")
    ) \
    .withColumn("rfm_score",
        expr("CASE " +
             "WHEN recency <= 30 AND frequency >= 10 AND monetary >= 1000 THEN 'Champion' " +
             "WHEN recency <= 60 AND frequency >= 5 AND monetary >= 500 THEN 'Loyal' " +
             "WHEN recency <= 90 AND frequency >= 3 THEN 'Potential' " +
             "WHEN recency > 180 THEN 'At Risk' " +
             "ELSE 'Regular' END")
    )

gold_customer_rfm.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold_customer_rfm")
```

### Executive KPI Summary

```python
# Single-row KPI table for dashboard
gold_executive_kpi = df_transactions.filter(col("is_return") == False) \
    .agg(
        sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("total_transactions"),
        countDistinct("CustomerID").alias("total_customers"),
        countDistinct("StockCode").alias("total_products"),
        avg("line_total").alias("avg_order_value")
    )

gold_executive_kpi.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold_executive_kpi")
```

## Working with Delta Tables

### Optimizing Delta Tables

```python
# Optimize Gold tables for query performance
spark.sql("OPTIMIZE gold_revenue_trends")
spark.sql("OPTIMIZE gold_product_performance ZORDER BY (total_revenue)")
spark.sql("OPTIMIZE gold_customer_rfm ZORDER BY (CustomerID)")

# Vacuum old versions (keep 7 days history)
spark.sql("VACUUM gold_revenue_trends RETAIN 168 HOURS")
```

### Time Travel Queries

```python
# Query historical versions
df_version = spark.read.format("delta") \
    .option("versionAsOf", 5) \
    .table("gold_revenue_trends")

# Query as of timestamp
df_timestamp = spark.read.format("delta") \
    .option("timestampAsOf", "2026-07-01") \
    .table("gold_product_performance")
```

### Describing Delta History

```python
# View table history
history_df = spark.sql("DESCRIBE HISTORY gold_revenue_trends")
history_df.select("version", "timestamp", "operation", "operationMetrics").show()
```

## Creating Semantic Models

### Connecting Gold Tables

In Microsoft Fabric:
1. Navigate to your Lakehouse
2. Click **New semantic model**
3. Select Gold layer tables:
   - `gold_revenue_trends`
   - `gold_product_performance`
   - `gold_customer_rfm`
   - `gold_executive_kpi`

### Defining Relationships

Create relationships in the semantic model:

```dax
// Example: Link products to transactions via StockCode
Relationship: gold_product_performance[StockCode] → silver_transactions[StockCode]
Cardinality: One-to-Many
Cross-filter direction: Single
```

### Creating DAX Measures

```dax
// Total Revenue
Total Revenue = SUM(gold_revenue_trends[total_revenue])

// Revenue Growth MoM
Revenue Growth MoM = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD(gold_revenue_trends[InvoiceDate], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue)

// Customer Lifetime Value
Customer LTV = 
DIVIDE(
    SUM(gold_customer_rfm[monetary]),
    DISTINCTCOUNT(gold_customer_rfm[CustomerID])
)

// Top Products by Revenue
Top 10 Products Revenue = 
CALCULATE(
    SUM(gold_product_performance[total_revenue]),
    TOPN(10, ALL(gold_product_performance), gold_product_performance[total_revenue], DESC)
)
```

## Power BI Integration

### Connecting to Semantic Model

Power BI automatically connects to Fabric semantic models:

1. Open Power BI Desktop
2. **Get Data** → **Power Platform** → **Power BI semantic models**
3. Select your Fabric semantic model
4. Build reports using connected data

### Sample Report Visuals

**Revenue Dashboard Card:**
- Visual: Card
- Field: `Total Revenue`
- Format: Currency

**Monthly Trend Line Chart:**
- X-axis: `gold_revenue_trends[month]`
- Y-axis: `[Total Revenue]`
- Legend: `gold_revenue_trends[year]`

**Product Performance Table:**
- Columns: `Description`, `total_quantity_sold`, `total_revenue`, `revenue_rank`
- Sorting: `revenue_rank` ascending

**Customer Segmentation Donut:**
- Legend: `gold_customer_rfm[rfm_score]`
- Values: `COUNT(CustomerID)`

## Common Patterns

### Pattern: Incremental Load to Silver

```python
from delta.tables import DeltaTable

# Read last processed timestamp
checkpoint_path = "Files/checkpoints/silver_transactions"
try:
    last_processed = spark.read.parquet(checkpoint_path) \
        .select(max("processed_timestamp")).collect()[0][0]
except:
    last_processed = "1900-01-01"

# Read only new records
df_incremental = spark.read.format("delta") \
    .table("bronze_transactions") \
    .filter(col("_ingestion_timestamp") > last_processed)

# Apply transformations
df_transformed = df_incremental \
    .withColumn("line_total", col("Quantity") * col("UnitPrice")) \
    .filter(col("CustomerID").isNotNull())

# Merge into Silver table
silver_table = DeltaTable.forName(spark, "silver_transactions")

silver_table.alias("target") \
    .merge(
        df_transformed.alias("source"),
        "target.InvoiceNo = source.InvoiceNo AND target.StockCode = source.StockCode"
    ) \
    .whenMatchedUpdateAll() \
    .whenNotMatchedInsertAll() \
    .execute()

# Update checkpoint
spark.createDataFrame([(current_timestamp(),)], ["processed_timestamp"]) \
    .write.mode("overwrite").parquet(checkpoint_path)
```

### Pattern: Data Quality Checks

```python
from pyspark.sql.functions import when, col, count

# Define quality rules
df_quality = df_transactions \
    .select(
        count("*").alias("total_records"),
        count(when(col("CustomerID").isNull(), 1)).alias("missing_customer_id"),
        count(when(col("Quantity") <= 0, 1)).alias("negative_quantity"),
        count(when(col("UnitPrice") <= 0, 1)).alias("invalid_price"),
        count(when(col("InvoiceDate").isNull(), 1)).alias("missing_date")
    )

# Write quality metrics
df_quality.write \
    .format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable("quality_metrics")
```

### Pattern: Parameterized Notebook Execution

```python
# Get parameters from notebook execution
dbutils.widgets.text("processing_date", "2026-07-15")
dbutils.widgets.dropdown("layer", "gold", ["bronze", "silver", "gold"])

processing_date = dbutils.widgets.get("processing_date")
layer = dbutils.widgets.get("layer")

# Use in transformations
df_filtered = df_transactions \
    .filter(col("InvoiceDate") == processing_date)
```

## Troubleshooting

### Issue: Cannot Read Delta Table

**Problem:** `AnalysisException: Table or view not found`

**Solution:**
```python
# Check if table exists
spark.sql("SHOW TABLES IN default").show()

# Verify lakehouse attachment
print(spark.conf.get("spark.sql.warehouse.dir"))

# Explicitly specify schema
df = spark.read.format("delta").table("default.gold_revenue_trends")
```

### Issue: Schema Evolution Errors

**Problem:** `AnalysisException: A schema mismatch detected`

**Solution:**
```python
# Enable schema evolution
df.write \
    .format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable("gold_revenue_trends")

# Or overwrite with new schema
df.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable("gold_revenue_trends")
```

### Issue: Dataflow Gen2 Performance

**Problem:** Slow transformations in Dataflow Gen2

**Solution:**
- Enable **Query Folding** by keeping transformations simple
- Use **Data Gateway** for on-premises sources
- Partition large datasets before loading
- Consider Fabric Notebooks for complex logic

### Issue: Power BI Refresh Failures

**Problem:** Semantic model refresh timeout

**Solution:**
```python
# Optimize Gold tables
spark.sql("OPTIMIZE gold_revenue_trends")

# Reduce data volume with filters
gold_filtered = df.filter(col("InvoiceDate") >= "2025-01-01")

# Create aggregated summary tables
gold_summary = df.groupBy("year", "month").agg(...)
```

### Issue: Memory Errors in Notebooks

**Problem:** `OutOfMemoryError` during large transformations

**Solution:**
```python
# Use partitioning
df.repartition(100).write.format("delta").saveAsTable("target")

# Process in batches
for batch in range(0, total_rows, batch_size):
    df_batch = df.limit(batch_size).offset(batch)
    # process batch

# Enable adaptive query execution
spark.conf.set("spark.sql.adaptive.enabled", "true")
```

## Best Practices

1. **Lakehouse Organization:**
   - Keep Bronze as immutable archive
   - Apply quality rules in Silver
   - Create business-ready aggregates in Gold

2. **Delta Lake Optimization:**
   - Run `OPTIMIZE` on frequently queried tables
   - Use `ZORDER` for common filter columns
   - Set appropriate retention with `VACUUM`

3. **Semantic Model Design:**
   - Create reusable DAX measures
   - Define clear relationships between tables
   - Use calculated columns sparingly

4. **Performance:**
   - Partition large datasets appropriately
   - Cache frequently accessed DataFrames
   - Use broadcast joins for small dimension tables

5. **Governance:**
   - Document data lineage in layer transitions
   - Implement quality checks at each stage
   - Use workspace roles for access control

## Environment Variables

When working with external data sources:

```python
# Use Fabric secrets or Key Vault
storage_account = os.environ.get("STORAGE_ACCOUNT_NAME")
api_key = os.environ.get("API_KEY")
```

## Additional Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/fabric/)
- [OneLake Overview](https://learn.microsoft.com/fabric/onelake/)
- [Dataflow Gen2 Guide](https://learn.microsoft.com/fabric/data-factory/dataflows-gen2-overview)
- [Delta Lake on Fabric](https://learn.microsoft.com/fabric/data-engineering/lakehouse-overview)
