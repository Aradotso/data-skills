---
name: microsoft-fabric-unified-analytics
description: End-to-end unified analytics platform using Microsoft Fabric, Lakehouse, Medallion Architecture, Dataflow Gen2, PySpark, and Power BI for retail data engineering.
triggers:
  - "build a Microsoft Fabric lakehouse"
  - "implement medallion architecture in Fabric"
  - "create dataflow gen2 transformations"
  - "process data with Fabric notebooks"
  - "design unified analytics platform"
  - "build retail analytics with OneLake"
  - "create Power BI semantic model"
  - "set up fabric data engineering pipeline"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in building end-to-end unified analytics platforms using **Microsoft Fabric**. The project demonstrates modern data engineering practices using Lakehouse architecture, Medallion layers (Bronze → Silver → Gold), Dataflow Gen2 for data integration, PySpark notebooks for transformations, and Power BI for business intelligence.

## What This Project Does

Microsoft Fabric Unified Analytics Platform is a production-inspired reference implementation that shows how to:

- **Unify data engineering workflows** in a single SaaS platform
- **Implement Medallion Architecture** (Bronze, Silver, Gold layers)
- **Ingest and transform data** using Dataflow Gen2
- **Process data at scale** with PySpark in Fabric Notebooks
- **Create semantic models** for consistent business metrics
- **Build interactive dashboards** with native Power BI integration
- **Store everything in OneLake** for unified data governance

**Use Case**: Retail analytics pipeline transforming raw transaction data into trusted business insights.

## Architecture Overview

```
OneLake (Unified Storage)
    ↓
Lakehouse (Medallion Architecture)
    ├── Bronze Layer (Raw data)
    ├── Silver Layer (Cleansed data via Dataflow Gen2)
    └── Gold Layer (Business-ready data via PySpark)
        ↓
    Semantic Model (Business metrics)
        ↓
    Power BI (Dashboards & Reports)
```

## Prerequisites

- **Microsoft Fabric** workspace with appropriate license
- **OneLake** storage (automatically provisioned)
- **Power BI** access (included in Fabric)
- **Python 3.8+** for local development (optional)
- **PySpark** knowledge for notebook development

## Setting Up a Fabric Lakehouse

### 1. Create a Lakehouse

```python
# In Microsoft Fabric UI:
# 1. Navigate to your workspace
# 2. Click "+ New" → "Lakehouse"
# 3. Name it: "RetailAnalyticsLakehouse"

# Lakehouse will automatically create:
# - Tables folder (managed Delta tables)
# - Files folder (raw files, organized by medallion layers)
```

### 2. Create Medallion Layer Folders

In your Lakehouse **Files** section, create:

```
Files/
├── bronze/          # Raw source data
│   ├── online_retail/
│   └── customers/
├── silver/          # Cleansed data
│   ├── online_retail_clean/
│   └── customers_clean/
└── gold/           # Business-ready aggregates
    ├── revenue_trends/
    ├── product_performance/
    └── customer_segments/
```

## Data Ingestion with Dataflow Gen2

### Creating a Dataflow Gen2

```python
# In Fabric UI:
# 1. Navigate to workspace → "+ New" → "Dataflow Gen2"
# 2. Name: "BronzeToSilver_DataCleaning"

# Sample Power Query M transformations (in Dataflow Gen2 editor):
```

```m
// Load Bronze data
let
    Source = Lakehouse.Contents(null),
    BronzeData = Source{[workspaceId="YOUR_WORKSPACE_ID"]}[Data],
    OnlineRetail = BronzeData{[lakehouseId="YOUR_LAKEHOUSE_ID"]}[Data],
    BronzeTable = OnlineRetail{[Name="bronze",ItemKind="Folder"]}[Data],
    RetailData = BronzeTable{[Name="online_retail"]}[Data],
    
    // Remove duplicates
    DuplicatesRemoved = Table.Distinct(RetailData, {"InvoiceNo", "StockCode", "InvoiceDate"}),
    
    // Handle missing values
    RemovedNulls = Table.SelectRows(DuplicatesRemoved, each [CustomerID] <> null and [Description] <> null),
    
    // Add business columns
    AddLineTotal = Table.AddColumn(RemovedNulls, "line_total", each [Quantity] * [UnitPrice], type number),
    AddYear = Table.AddColumn(AddLineTotal, "year", each Date.Year([InvoiceDate]), Int64.Type),
    AddMonth = Table.AddColumn(AddYear, "month", each Date.Month([InvoiceDate]), Int64.Type),
    AddIsReturn = Table.AddColumn(AddMonth, "is_return", each if [Quantity] < 0 then true else false, type logical),
    
    // Standardize data types
    ChangedTypes = Table.TransformColumnTypes(AddIsReturn, {
        {"InvoiceNo", type text},
        {"Quantity", Int64.Type},
        {"UnitPrice", type number},
        {"CustomerID", type text}
    })
in
    ChangedTypes
```

### Dataflow Destination Configuration

```python
# In Dataflow Gen2 destination settings:
# - Destination: Lakehouse
# - Workspace: [Your Workspace]
# - Lakehouse: RetailAnalyticsLakehouse
# - Root Folder: Files/silver
# - Table name: online_retail_clean
# - Update method: Replace
```

## PySpark Processing in Fabric Notebooks

### Example: Bronze to Silver Notebook

```python
# Load Bronze data
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, year, month, when, sum as _sum, count, countDistinct
from pyspark.sql.types import DoubleType

# Access Fabric Lakehouse
bronze_path = "Files/bronze/online_retail/*.csv"
df_bronze = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load(bronze_path)

# Data quality transformations
df_clean = df_bronze \
    .dropna(subset=["CustomerID", "Description", "InvoiceDate"]) \
    .dropDuplicates(["InvoiceNo", "StockCode", "InvoiceDate"]) \
    .filter(col("Quantity") > 0) \
    .filter(col("UnitPrice") > 0)

# Add business logic columns
df_silver = df_clean \
    .withColumn("line_total", col("Quantity") * col("UnitPrice")) \
    .withColumn("year", year(col("InvoiceDate"))) \
    .withColumn("month", month(col("InvoiceDate"))) \
    .withColumn("is_return", when(col("Quantity") < 0, True).otherwise(False))

# Write to Silver layer as Delta table
silver_table_path = "Tables/silver_online_retail"
df_silver.write.format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save(silver_table_path)

print(f"Silver layer created: {df_silver.count()} records")
```

### Example: Silver to Gold - Revenue Trends

```python
# Create Gold layer aggregates
from pyspark.sql.window import Window
from pyspark.sql.functions import lag, round as sql_round

# Load Silver data
df_silver = spark.read.format("delta").load("Tables/silver_online_retail")

# Calculate monthly revenue trends
df_revenue_trends = df_silver \
    .groupBy("year", "month") \
    .agg(
        sql_round(_sum("line_total"), 2).alias("total_revenue"),
        countDistinct("InvoiceNo").alias("total_orders"),
        countDistinct("CustomerID").alias("unique_customers"),
        sql_round(_sum("line_total") / countDistinct("InvoiceNo"), 2).alias("avg_order_value")
    ) \
    .orderBy("year", "month")

# Add month-over-month growth
window_spec = Window.orderBy("year", "month")
df_revenue_trends = df_revenue_trends \
    .withColumn("prev_month_revenue", lag("total_revenue").over(window_spec)) \
    .withColumn("mom_growth_pct", 
                when(col("prev_month_revenue").isNotNull(),
                     sql_round((col("total_revenue") - col("prev_month_revenue")) / col("prev_month_revenue") * 100, 2))
                .otherwise(None))

# Write to Gold layer
gold_table_path = "Tables/gold_revenue_trends"
df_revenue_trends.write.format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save(gold_table_path)

# Display results
display(df_revenue_trends)
```

### Example: Customer RFM Segmentation

```python
from pyspark.sql.functions import datediff, max as _max, lit, current_date
from pyspark.sql import Window

# Load Silver data
df_silver = spark.read.format("delta").load("Tables/silver_online_retail")

# Calculate RFM metrics
max_date = df_silver.agg(_max("InvoiceDate")).collect()[0][0]

df_rfm = df_silver \
    .groupBy("CustomerID") \
    .agg(
        datediff(lit(max_date), _max("InvoiceDate")).alias("recency"),
        countDistinct("InvoiceNo").alias("frequency"),
        sql_round(_sum("line_total"), 2).alias("monetary")
    )

# Create RFM quartiles
def assign_rfm_score(df, column_name):
    """Assign RFM score (1-4) based on quartiles"""
    quantiles = df.approxQuantile(column_name, [0.25, 0.5, 0.75], 0.01)
    
    return when(col(column_name) <= quantiles[0], 1) \
        .when(col(column_name) <= quantiles[1], 2) \
        .when(col(column_name) <= quantiles[2], 3) \
        .otherwise(4)

# Note: Recency is inverted (lower is better)
df_rfm = df_rfm \
    .withColumn("r_score", 5 - assign_rfm_score(df_rfm, "recency")) \
    .withColumn("f_score", assign_rfm_score(df_rfm, "frequency")) \
    .withColumn("m_score", assign_rfm_score(df_rfm, "monetary"))

# Create customer segments
df_rfm = df_rfm \
    .withColumn("rfm_segment", 
                when((col("r_score") >= 3) & (col("f_score") >= 3) & (col("m_score") >= 3), "Champions")
                .when((col("r_score") >= 3) & (col("f_score") >= 2), "Loyal Customers")
                .when((col("r_score") >= 3), "Potential Loyalists")
                .when((col("f_score") >= 3) & (col("m_score") >= 3), "At Risk")
                .when(col("r_score") <= 2, "Lost")
                .otherwise("Others"))

# Write to Gold layer
gold_rfm_path = "Tables/gold_customer_segments"
df_rfm.write.format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save(gold_rfm_path)

# Show segment distribution
df_rfm.groupBy("rfm_segment").count().orderBy(col("count").desc()).show()
```

## Working with Semantic Models

### Creating a Semantic Model

```python
# In Fabric UI:
# 1. Navigate to your Lakehouse
# 2. Select Gold layer tables
# 3. Click "New semantic model"
# 4. Name: "RetailAnalytics_SemanticModel"
# 5. Select tables:
#    - gold_revenue_trends
#    - gold_product_performance
#    - gold_customer_segments
```

### Define DAX Measures

```dax
-- Total Revenue
Total Revenue = SUM(gold_revenue_trends[total_revenue])

-- Average Order Value
Avg Order Value = AVERAGE(gold_revenue_trends[avg_order_value])

-- Customer Lifetime Value
Customer LTV = 
SUMX(
    gold_customer_segments,
    gold_customer_segments[monetary]
)

-- Revenue Growth Rate
Revenue Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = CALCULATE([Total Revenue], PREVIOUSMONTH(gold_revenue_trends[Date]))
RETURN
    DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0) * 100

-- Active Customers
Active Customers = DISTINCTCOUNT(gold_revenue_trends[unique_customers])
```

## Power BI Integration

### Connecting to Semantic Model

```python
# Power BI automatically connects to Fabric Semantic Models
# In Power BI Service or Desktop:
# 1. Create new report
# 2. Get Data → Power BI semantic models
# 3. Select "RetailAnalytics_SemanticModel"
# 4. Build visuals using pre-defined measures
```

### Example Report Layout

```
Dashboard Components:
├── KPI Cards
│   ├── Total Revenue
│   ├── Total Orders
│   ├── Avg Order Value
│   └── Active Customers
├── Time Series
│   └── Revenue Trend (Line chart)
├── Product Analysis
│   ├── Top 10 Products (Bar chart)
│   └── Category Performance (Treemap)
└── Customer Segments
    ├── RFM Distribution (Donut chart)
    └── Segment Metrics (Table)
```

## Common Patterns

### Pattern 1: Incremental Load Pattern

```python
# Load only new data since last run
from pyspark.sql.functions import to_date

# Read checkpoint
checkpoint_path = "Files/checkpoints/last_load_date.txt"
try:
    last_load_date = spark.read.text(checkpoint_path).first()[0]
except:
    last_load_date = "1900-01-01"

# Incremental load
df_incremental = df_bronze \
    .filter(col("InvoiceDate") > last_load_date)

# Process and merge with existing Silver data
df_silver_existing = spark.read.format("delta").load("Tables/silver_online_retail")
df_silver_combined = df_silver_existing.union(df_incremental).dropDuplicates()

# Write back
df_silver_combined.write.format("delta") \
    .mode("overwrite") \
    .save("Tables/silver_online_retail")

# Update checkpoint
from datetime import datetime
new_checkpoint = datetime.now().strftime("%Y-%m-%d")
spark.createDataFrame([(new_checkpoint,)], ["date"]) \
    .write.mode("overwrite").text(checkpoint_path)
```

### Pattern 2: Data Quality Checks

```python
def run_data_quality_checks(df, layer_name):
    """Run automated DQ checks"""
    
    checks = {
        "total_records": df.count(),
        "null_customer_ids": df.filter(col("CustomerID").isNull()).count(),
        "negative_quantities": df.filter(col("Quantity") < 0).count(),
        "zero_prices": df.filter(col("UnitPrice") <= 0).count(),
        "duplicate_invoices": df.count() - df.dropDuplicates(["InvoiceNo", "StockCode"]).count()
    }
    
    # Log results
    print(f"=== Data Quality Report: {layer_name} ===")
    for check, value in checks.items():
        print(f"{check}: {value}")
    
    # Alert if thresholds exceeded
    if checks["null_customer_ids"] > 100:
        print(f"⚠️ WARNING: High null CustomerIDs: {checks['null_customer_ids']}")
    
    return checks

# Usage
dq_results = run_data_quality_checks(df_silver, "Silver Layer")
```

### Pattern 3: Parameterized Notebooks

```python
# Fabric Notebook with parameters
# Define parameters (can be overridden in pipelines)
dbutils.widgets.text("layer", "silver", "Data Layer")
dbutils.widgets.text("table_name", "online_retail", "Table Name")
dbutils.widgets.text("mode", "overwrite", "Write Mode")

# Get parameter values
layer = dbutils.widgets.get("layer")
table_name = dbutils.widgets.get("table_name")
mode = dbutils.widgets.get("mode")

# Dynamic processing
input_path = f"Tables/{layer}_{table_name}"
output_path = f"Tables/{layer}_processed_{table_name}"

df = spark.read.format("delta").load(input_path)
# ... apply transformations ...
df_processed.write.format("delta").mode(mode).save(output_path)
```

## Configuration

### Notebook Environment Variables

```python
# Reference workspace and lakehouse
WORKSPACE_ID = "00000000-0000-0000-0000-000000000000"  # Get from Fabric UI
LAKEHOUSE_ID = "00000000-0000-0000-0000-000000000000"  # Get from Fabric UI

# Storage paths
BRONZE_PATH = "Files/bronze"
SILVER_PATH = "Files/silver"
GOLD_PATH = "Files/gold"
DELTA_TABLES_PATH = "Tables"

# Processing configurations
SPARK_CONFIGS = {
    "spark.sql.adaptive.enabled": "true",
    "spark.sql.adaptive.coalescePartitions.enabled": "true",
    "spark.databricks.delta.optimizeWrite.enabled": "true"
}

# Apply configs
for key, value in SPARK_CONFIGS.items():
    spark.conf.set(key, value)
```

## Troubleshooting

### Issue: Dataflow Gen2 fails to write to Lakehouse

**Solution**: Check workspace permissions and lakehouse destination settings.

```python
# Verify lakehouse connection in Dataflow:
# 1. Ensure workspace contributor role
# 2. Confirm lakehouse exists in same workspace
# 3. Check destination folder path (use Files/silver, not /Files/silver)
```

### Issue: PySpark notebook can't find Delta tables

**Solution**: Use correct path format and verify table exists.

```python
# Correct path formats:
df = spark.read.format("delta").load("Tables/silver_online_retail")  # ✓ Correct
df = spark.read.format("delta").load("Files/silver/online_retail")   # ✗ Wrong

# List available tables
display(spark.sql("SHOW TABLES"))

# Describe table location
spark.sql("DESCRIBE EXTENDED silver_online_retail").show(truncate=False)
```

### Issue: Out of memory errors in PySpark

**Solution**: Optimize partitioning and use incremental processing.

```python
# Repartition large datasets
df_large = df.repartition(200, "CustomerID")

# Use broadcast for small dimension tables
from pyspark.sql.functions import broadcast
df_result = df_large.join(broadcast(df_small), "key")

# Process in batches
for year in [2020, 2021, 2022]:
    df_batch = df.filter(col("year") == year)
    # Process batch...
```

### Issue: Semantic model refresh fails

**Solution**: Check data source credentials and table schemas.

```python
# Ensure:
# 1. Semantic model has proper lakehouse connection
# 2. Gold tables use consistent schema
# 3. No breaking schema changes between refreshes
# 4. Refresh schedule aligns with data pipeline completion
```

## Best Practices

1. **Always use Delta format** for tables (ACID transactions, time travel)
2. **Partition large tables** by date or high-cardinality columns
3. **Implement data quality checks** at each medallion layer
4. **Use semantic models** for consistent business logic
5. **Version control notebooks** using Fabric Git integration
6. **Monitor OneLake storage** and clean up unused data
7. **Use incremental loads** for large datasets
8. **Document data lineage** through comments and metadata

## Additional Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric/)
- [OneLake Overview](https://learn.microsoft.com/en-us/fabric/onelake/)
- [PySpark API Reference](https://spark.apache.org/docs/latest/api/python/)
- [Power BI Service](https://powerbi.microsoft.com/)

---

This skill enables AI coding agents to assist with building production-ready unified analytics platforms using Microsoft Fabric's complete suite of data engineering and business intelligence capabilities.
