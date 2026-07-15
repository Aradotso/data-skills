---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering on Microsoft Fabric using Lakehouse, Medallion Architecture, Dataflow Gen2, PySpark notebooks, and Power BI for unified analytics.
triggers:
  - "how do I build a lakehouse in Microsoft Fabric"
  - "set up medallion architecture with bronze silver gold layers"
  - "create dataflow gen2 for data transformation"
  - "use fabric notebooks with pyspark for data processing"
  - "implement unified analytics platform with microsoft fabric"
  - "build semantic model from lakehouse gold layer"
  - "connect power bi to fabric lakehouse"
  - "process retail data using fabric and onelake"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This skill teaches AI agents how to build end-to-end unified analytics platforms using **Microsoft Fabric**. The project demonstrates a production-grade implementation of the **Medallion Architecture (Bronze → Silver → Gold)** for retail analytics, utilizing Microsoft Fabric's unified SaaS capabilities including OneLake storage, Dataflow Gen2 for low-code transformations, Fabric Notebooks with PySpark for business logic, Semantic Models, and native Power BI integration.

Unlike traditional multi-service architectures, Microsoft Fabric consolidates data ingestion, storage, transformation, semantic modeling, and business intelligence into a single platform with shared storage (OneLake).

## Project Structure

```
microsoft-fabric-unified-analytics-platform/
├── architecture/          # Architecture diagrams and screenshots
├── notebooks/            # PySpark transformation notebooks
├── case-study/          # Technical documentation
├── images/              # Project assets
└── README.md
```

## Prerequisites

- **Microsoft Fabric Capacity**: F2 or higher (F64+ recommended for production)
- **Microsoft Fabric Workspace**: With Lakehouse enabled
- **Power BI Pro or Premium**: For report development
- **Azure Account**: If sourcing data from Azure services
- **Python 3.10+**: For local notebook development (optional)

## Installation & Setup

### 1. Create Microsoft Fabric Workspace

```text
1. Navigate to app.fabric.microsoft.com
2. Click "Workspaces" → "New workspace"
3. Name: "Unified-Analytics-Platform"
4. Assign Fabric capacity (F2 minimum)
5. Enable "Lakehouse" workload
```

### 2. Create Lakehouse

```text
1. In workspace, click "+ New" → "Lakehouse"
2. Name: "retail_analytics_lakehouse"
3. This creates OneLake storage with lakehouse capabilities
```

### 3. Create Medallion Architecture Folders

In the Lakehouse Files section, create folder structure:

```text
Files/
├── bronze/           # Raw source data
├── silver/           # Cleansed data
└── gold/            # Business-ready curated data
```

## Architecture Pattern: Medallion Layers

### Bronze Layer (Raw Data)

**Purpose**: Preserve raw source data exactly as received, immutable historical record.

**Typical Sources**:
- CSV files from blob storage
- API extracts
- Database exports
- Event streams

**Example Bronze Structure**:
```text
bronze/
├── customers/
│   └── customers_2024.csv
├── products/
│   └── products_2024.csv
└── transactions/
    └── transactions_2024.csv
```

### Silver Layer (Cleansed Data)

**Purpose**: Cleaned, standardized, enriched data with business context.

**Transformations Applied**:
- Remove duplicates
- Handle missing values
- Standardize data types
- Add derived columns
- Business enrichment

### Gold Layer (Business-Ready Data)

**Purpose**: Curated datasets optimized for analytics, reporting, and ML.

**Typical Outputs**:
- Aggregated KPIs
- Customer segments
- Revenue trends
- Product performance metrics
- Executive dashboards

## Dataflow Gen2: Silver Layer Transformations

### Creating a Dataflow Gen2

```text
1. In workspace, click "+ New" → "Dataflow Gen2"
2. Name: "Bronze_to_Silver_Transformation"
3. Get Data → "OneLake data hub" → Select Bronze tables
4. Apply transformations using Power Query
5. Set destination to Silver layer
```

### Common Transformation Patterns

**Remove Duplicates**:
```powerquery
// Power Query M
= Table.Distinct(Source, {"CustomerID", "InvoiceNo"})
```

**Handle Missing Values**:
```powerquery
// Replace null with 0 for quantity
= Table.ReplaceValue(Source, null, 0, Replacer.ReplaceValue, {"Quantity"})
```

**Add Calculated Columns**:
```powerquery
// Add line total
= Table.AddColumn(Source, "line_total", each [Quantity] * [UnitPrice], type number)
```

**Extract Date Components**:
```powerquery
// Add year and month
= Table.AddColumn(Source, "year", each Date.Year([InvoiceDate]), Int64.Type)
= Table.AddColumn(#"Added year", "month", each Date.Month([InvoiceDate]), Int64.Type)
```

**Flag Returns**:
```powerquery
// Identify return transactions
= Table.AddColumn(Source, "is_return", each Text.StartsWith([InvoiceNo], "C"), type logical)
```

### Dataflow Gen2 Destination Configuration

```text
Output Settings:
├── Destination: Lakehouse
├── Lakehouse: retail_analytics_lakehouse
├── Target folder: Files/silver/
├── File format: Parquet (recommended) or Delta
├── Partition by: year, month (for large datasets)
└── Update method: Replace or Append
```

## Fabric Notebooks: Gold Layer with PySpark

### Creating a Fabric Notebook

```text
1. In workspace, click "+ New" → "Notebook"
2. Name: "Silver_to_Gold_Business_Logic"
3. Default language: PySpark (Python)
4. Runtime: Standard (Spark 3.4+)
```

### Notebook: Read Silver Data

```python
# Read Silver layer data from OneLake
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, count, avg, datediff, max, current_date
from pyspark.sql.window import Window

# Initialize Spark session (automatically configured in Fabric)
spark = SparkSession.builder.appName("GoldLayerProcessing").getOrCreate()

# Read Silver data (adjust path to your lakehouse)
transactions_df = spark.read.format("parquet").load(
    "Files/silver/transactions/"
)

customers_df = spark.read.format("parquet").load(
    "Files/silver/customers/"
)

products_df = spark.read.format("parquet").load(
    "Files/silver/products/"
)

# Display schema and sample
transactions_df.printSchema()
transactions_df.show(5)
```

### Notebook: Revenue KPIs

```python
# Calculate revenue KPIs by year and month
revenue_kpis = transactions_df.filter(
    col("is_return") == False
).groupBy("year", "month").agg(
    sum("line_total").alias("total_revenue"),
    count("InvoiceNo").distinct().alias("total_orders"),
    count("CustomerID").distinct().alias("unique_customers"),
    avg("line_total").alias("avg_order_value")
).orderBy("year", "month")

# Write to Gold layer
revenue_kpis.write.format("delta").mode("overwrite").save(
    "Tables/gold_revenue_kpis"
)

revenue_kpis.show()
```

### Notebook: Product Performance

```python
# Product performance analysis
product_performance = transactions_df.alias("t").join(
    products_df.alias("p"),
    col("t.StockCode") == col("p.StockCode"),
    "inner"
).filter(
    col("t.is_return") == False
).groupBy(
    col("p.StockCode"),
    col("p.Description")
).agg(
    sum("t.Quantity").alias("total_quantity_sold"),
    sum("t.line_total").alias("total_revenue"),
    count("t.InvoiceNo").distinct().alias("total_orders"),
    avg("t.UnitPrice").alias("avg_unit_price")
).orderBy(col("total_revenue").desc())

# Write to Gold layer
product_performance.write.format("delta").mode("overwrite").save(
    "Tables/gold_product_performance"
)

product_performance.show(10)
```

### Notebook: Customer Segmentation (RFM)

```python
from pyspark.sql.functions import datediff, max as spark_max, current_date

# Calculate RFM metrics
reference_date = transactions_df.agg(spark_max("InvoiceDate")).collect()[0][0]

rfm_df = transactions_df.filter(
    col("is_return") == False
).groupBy("CustomerID").agg(
    datediff(current_date(), spark_max("InvoiceDate")).alias("recency"),
    count("InvoiceNo").distinct().alias("frequency"),
    sum("line_total").alias("monetary_value")
)

# RFM scoring (simplified - use ntile for production)
from pyspark.sql.functions import when

rfm_scored = rfm_df.withColumn(
    "r_score",
    when(col("recency") <= 30, 5)
    .when(col("recency") <= 60, 4)
    .when(col("recency") <= 90, 3)
    .when(col("recency") <= 180, 2)
    .otherwise(1)
).withColumn(
    "f_score",
    when(col("frequency") >= 20, 5)
    .when(col("frequency") >= 10, 4)
    .when(col("frequency") >= 5, 3)
    .when(col("frequency") >= 2, 2)
    .otherwise(1)
).withColumn(
    "m_score",
    when(col("monetary_value") >= 5000, 5)
    .when(col("monetary_value") >= 2000, 4)
    .when(col("monetary_value") >= 1000, 3)
    .when(col("monetary_value") >= 500, 2)
    .otherwise(1)
)

# Assign customer segments
rfm_segments = rfm_scored.withColumn(
    "customer_segment",
    when((col("r_score") >= 4) & (col("f_score") >= 4) & (col("m_score") >= 4), "Champions")
    .when((col("r_score") >= 3) & (col("f_score") >= 3), "Loyal Customers")
    .when((col("r_score") >= 4) & (col("f_score") <= 2), "Potential Loyalists")
    .when((col("r_score") <= 2) & (col("f_score") >= 3), "At Risk")
    .otherwise("Needs Attention")
)

# Write to Gold layer
rfm_segments.write.format("delta").mode("overwrite").save(
    "Tables/gold_customer_segments"
)

rfm_segments.show(10)
```

### Notebook: Customer Analytics

```python
# Customer behavior analytics
customer_analytics = transactions_df.alias("t").join(
    customers_df.alias("c"),
    col("t.CustomerID") == col("c.CustomerID"),
    "inner"
).filter(
    col("t.is_return") == False
).groupBy(
    col("c.CustomerID"),
    col("c.Country")
).agg(
    count("t.InvoiceNo").distinct().alias("total_orders"),
    sum("t.Quantity").alias("total_items_purchased"),
    sum("t.line_total").alias("total_spent"),
    avg("t.line_total").alias("avg_order_value"),
    spark_max("t.InvoiceDate").alias("last_purchase_date")
).orderBy(col("total_spent").desc())

# Write to Gold layer
customer_analytics.write.format("delta").mode("overwrite").save(
    "Tables/gold_customer_analytics"
)

customer_analytics.show(10)
```

### Notebook Best Practices

```python
# 1. Use Delta Lake for ACID transactions and time travel
# Write with Delta
df.write.format("delta").mode("overwrite").option("overwriteSchema", "true").save(
    "Tables/gold_table_name"
)

# 2. Partition large datasets for performance
df.write.format("delta").partitionBy("year", "month").mode("overwrite").save(
    "Tables/gold_partitioned_table"
)

# 3. Use caching for iterative operations
cached_df = df.cache()
cached_df.count()  # Triggers caching

# 4. Monitor Spark execution
# In Fabric Notebook, use %%info magic
# %%info
# Shows Spark session details, executors, memory

# 5. Handle schema evolution
df.write.format("delta").mode("append").option("mergeSchema", "true").save(
    "Tables/gold_evolving_table"
)
```

## Semantic Model Creation

### Create Semantic Model from Gold Tables

```text
1. In Lakehouse, navigate to "Gold" tables section
2. Select tables: gold_revenue_kpis, gold_product_performance, gold_customer_analytics
3. Click "New semantic model"
4. Name: "Retail_Analytics_Semantic_Model"
5. Select all gold tables to include
6. Click "Create"
```

### Define Relationships

```text
In Semantic Model designer:

1. Open relationship view
2. Create relationships:
   - gold_customer_analytics[CustomerID] → gold_customer_segments[CustomerID]
   - gold_product_performance[StockCode] → transactions[StockCode] (if needed)
3. Set cardinality and cross-filter direction
4. Validate relationships
```

### Create Measures (DAX)

```dax
// Total Revenue
Total Revenue = SUM('gold_revenue_kpis'[total_revenue])

// YoY Revenue Growth
YoY Revenue Growth = 
VAR CurrentYearRevenue = CALCULATE([Total Revenue], YEAR('gold_revenue_kpis'[year]) = YEAR(TODAY()))
VAR PreviousYearRevenue = CALCULATE([Total Revenue], YEAR('gold_revenue_kpis'[year]) = YEAR(TODAY()) - 1)
RETURN
DIVIDE(CurrentYearRevenue - PreviousYearRevenue, PreviousYearRevenue, 0)

// Average Order Value
Avg Order Value = AVERAGE('gold_customer_analytics'[avg_order_value])

// Customer Count
Total Customers = DISTINCTCOUNT('gold_customer_analytics'[CustomerID])

// Champions Count
Champions = CALCULATE(
    COUNTROWS('gold_customer_segments'),
    'gold_customer_segments'[customer_segment] = "Champions"
)
```

## Power BI Report Development

### Connect to Semantic Model

```text
1. Open Power BI Desktop
2. Get Data → "Power BI semantic models"
3. Select workspace: "Unified-Analytics-Platform"
4. Select: "Retail_Analytics_Semantic_Model"
5. Click "Connect" (live connection)
```

### Report Layout Example

```text
Page 1: Executive Dashboard
├── KPI Cards
│   ├── Total Revenue ([Total Revenue])
│   ├── YoY Growth ([YoY Revenue Growth])
│   ├── Total Customers ([Total Customers])
│   └── Avg Order Value ([Avg Order Value])
├── Line Chart: Revenue Trend (year, month → Total Revenue)
├── Bar Chart: Top 10 Products (Description → total_revenue)
└── Donut Chart: Customer Segments (customer_segment → count)

Page 2: Product Performance
├── Table: Product Details (all product metrics)
├── Scatter Plot: Price vs Volume (avg_unit_price, total_quantity_sold)
└── Slicer: Year, Month

Page 3: Customer Analytics
├── Map: Revenue by Country (Country → total_spent)
├── Stacked Bar: Segment Distribution (customer_segment → count)
├── Table: Top Customers (CustomerID, total_spent, total_orders)
└── Funnel: Customer Journey
```

### Publish to Fabric

```text
1. In Power BI Desktop, click "Publish"
2. Select workspace: "Unified-Analytics-Platform"
3. Report publishes to Fabric workspace
4. Can be viewed in app.fabric.microsoft.com
```

## Configuration Patterns

### Environment Variables for Connections

```python
# If connecting to external sources
import os
from pyspark.sql import SparkSession

# Azure Blob Storage configuration
storage_account = os.getenv("AZURE_STORAGE_ACCOUNT")
storage_key = os.getenv("AZURE_STORAGE_KEY")
container = os.getenv("AZURE_CONTAINER_NAME")

spark.conf.set(
    f"fs.azure.account.key.{storage_account}.dfs.core.windows.net",
    storage_key
)

# Read from Azure Blob
df = spark.read.format("csv").option("header", "true").load(
    f"abfss://{container}@{storage_account}.dfs.core.windows.net/raw_data/*.csv"
)
```

### Lakehouse Connection in Notebook

```python
# Access default lakehouse (no configuration needed in Fabric)
# Tables are accessible via spark.read.table()
df = spark.read.table("retail_analytics_lakehouse.gold_revenue_kpis")

# Files are accessible via relative paths
df = spark.read.format("parquet").load("Files/silver/transactions/")

# Use notebookutils for lakehouse operations
from notebookutils import mssparkutils

# List files in lakehouse
files = mssparkutils.fs.ls("Files/silver/")
for file in files:
    print(file.name, file.size)
```

### Delta Lake Configuration

```python
# Enable Delta Lake optimizations
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")

# Z-order optimization for common queries
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "Tables/gold_customer_analytics")
delta_table.optimize().executeZOrderBy("CustomerID", "Country")

# Vacuum old versions (7 days retention)
delta_table.vacuum(168)  # hours
```

## Common Workflows

### End-to-End Pipeline Execution

```text
Step 1: Ingest Bronze Data
├── Upload CSV to Files/bronze/ OR
├── Use Dataflow Gen2 to fetch from source
└── Validate raw data arrival

Step 2: Run Dataflow Gen2
├── Execute "Bronze_to_Silver_Transformation"
├── Monitors data quality transformations
└── Writes to Files/silver/

Step 3: Execute Fabric Notebook
├── Run "Silver_to_Gold_Business_Logic"
├── PySpark processes business logic
└── Writes to Tables/gold_*

Step 4: Refresh Semantic Model
├── Manual: Click "Refresh now" in semantic model
├── Scheduled: Configure refresh schedule
└── Validates relationships and measures

Step 5: View Power BI Report
├── Navigate to report in workspace
├── Interacts with live data
└── Share with business users
```

### Incremental Load Pattern

```python
# Read last processed timestamp
from pyspark.sql.functions import col, lit, max as spark_max

# Track processing watermark
watermark_table = "Tables/processing_watermark"

try:
    last_processed = spark.read.format("delta").load(watermark_table)
    max_timestamp = last_processed.select("last_timestamp").collect()[0][0]
except:
    max_timestamp = "1970-01-01 00:00:00"  # Initial load

# Read only new data
new_data = spark.read.format("parquet").load("Files/silver/transactions/").filter(
    col("InvoiceDate") > lit(max_timestamp)
)

if new_data.count() > 0:
    # Process new data
    processed_data = new_data.groupBy("year", "month").agg(
        sum("line_total").alias("revenue")
    )
    
    # Append to gold
    processed_data.write.format("delta").mode("append").save(
        "Tables/gold_revenue_kpis"
    )
    
    # Update watermark
    current_max = new_data.agg(spark_max("InvoiceDate")).collect()[0][0]
    watermark_df = spark.createDataFrame([(current_max,)], ["last_timestamp"])
    watermark_df.write.format("delta").mode("overwrite").save(watermark_table)
```

### Data Quality Checks

```python
# Data quality validation before Silver layer
from pyspark.sql.functions import col, isnan, isnull, when, count

def data_quality_report(df, table_name):
    """Generate data quality metrics"""
    
    # Row count
    total_rows = df.count()
    
    # Null counts per column
    null_counts = df.select([
        count(when(isnull(c) | isnan(c), c)).alias(c) for c in df.columns
    ])
    
    # Duplicate count (assuming CustomerID + InvoiceNo uniqueness)
    if "CustomerID" in df.columns and "InvoiceNo" in df.columns:
        duplicate_count = total_rows - df.select("CustomerID", "InvoiceNo").distinct().count()
    else:
        duplicate_count = 0
    
    print(f"=== Data Quality Report: {table_name} ===")
    print(f"Total Rows: {total_rows}")
    print(f"Duplicate Rows: {duplicate_count}")
    print("Null Counts:")
    null_counts.show()
    
    return {
        "table": table_name,
        "total_rows": total_rows,
        "duplicates": duplicate_count,
        "null_counts": null_counts
    }

# Run quality checks
bronze_df = spark.read.format("parquet").load("Files/bronze/transactions/")
quality_report = data_quality_report(bronze_df, "bronze_transactions")
```

## Troubleshooting

### Issue: Dataflow Gen2 Fails with "Access Denied"

**Solution**: Verify workspace role assignments and lakehouse permissions.

```text
1. Navigate to Workspace Settings
2. Check "Manage access"
3. Ensure service principal or user has "Contributor" role
4. In Lakehouse, verify "Manage permissions" includes dataflow identity
```

### Issue: Notebook Cannot Read Lakehouse Tables

**Solution**: Explicitly attach lakehouse to notebook.

```text
1. Open notebook
2. Click "Add lakehouse" in left panel
3. Select "Existing lakehouse"
4. Choose "retail_analytics_lakehouse"
5. Click "Add"
```

### Issue: Semantic Model Refresh Fails

**Solution**: Check for schema mismatches and credentials.

```python
# In notebook, validate Delta table schema
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "Tables/gold_revenue_kpis")
print(delta_table.toDF().schema)

# Ensure columns match semantic model expectations
```

### Issue: Power BI Report Shows Blank Visuals

**Solution**: Verify data exists in gold tables and relationships are correct.

```python
# Quick check in notebook
spark.read.format("delta").load("Tables/gold_revenue_kpis").show()
spark.read.format("delta").load("Tables/gold_customer_analytics").count()
```

```text
In Semantic Model:
1. Open "Model view"
2. Validate relationships show solid lines (not dotted)
3. Check cardinality matches data (1:many, many:1, etc.)
4. Test measures in "Data view" before publishing
```

### Issue: Slow PySpark Query Performance

**Solution**: Optimize with partitioning, caching, and broadcast joins.

```python
# Partition by frequently filtered columns
df.write.format("delta").partitionBy("year", "month").mode("overwrite").save(
    "Tables/gold_partitioned_table"
)

# Cache intermediate results
cached_df = df.cache()
cached_df.count()

# Broadcast small dimension tables
from pyspark.sql.functions import broadcast

result = large_fact.join(
    broadcast(small_dimension),
    "key"
)

# Check query plan
df.explain()
```

### Issue: Delta Table Version Conflicts

**Solution**: Use time travel or vacuum appropriately.

```python
from delta.tables import DeltaTable

# Read specific version
df_v5 = spark.read.format("delta").option("versionAsOf", 5).load(
    "Tables/gold_revenue_kpis"
)

# Restore to previous version
delta_table = DeltaTable.forPath(spark, "Tables/gold_revenue_kpis")
delta_table.restoreToVersion(5)

# Clean up old versions (if needed)
delta_table.vacuum(168)  # Retain 7 days
```

## Advanced Patterns

### Pipeline Orchestration with Fabric Pipelines

```text
Create Fabric Pipeline:
1. New → "Data pipeline"
2. Name: "End_to_End_Analytics_Pipeline"
3. Add activities:
   ├── Dataflow Gen2 Activity → "Bronze_to_Silver_Transformation"
   ├── Notebook Activity → "Silver_to_Gold_Business_Logic"
   └── Refresh Semantic Model Activity → "Retail_Analytics_Semantic_Model"
4. Set dependencies: Dataflow → Notebook → Semantic Model
5. Schedule trigger: Daily at 2 AM UTC
```

### Parameterized Notebooks

```python
# Define notebook parameters
# In cell 1 with tag "parameters"
year_filter = 2024
month_filter = 12

# Use parameters in processing
filtered_df = spark.read.format("delta").load("Tables/gold_revenue_kpis").filter(
    (col("year") == year_filter) & (col("month") == month_filter)
)

# When calling notebook from pipeline, pass parameters
# Pipeline Notebook Activity → Parameters → year_filter: 2024, month_filter: 12
```

### Monitoring and Logging

```python
# Custom logging in notebooks
import logging
from datetime import datetime

# Setup logger
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# Log processing steps
logger.info(f"Starting Silver to Gold processing at {datetime.now()}")

try:
    # Your transformation logic
    df = spark.read.format("parquet").load("Files/silver/transactions/")
    result = df.groupBy("year").agg(sum("line_total").alias("revenue"))
    result.write.format("delta").mode("overwrite").save("Tables/gold_revenue")
    
    logger.info(f"Successfully wrote {result.count()} rows to gold layer")
except Exception as e:
    logger.error(f"Processing failed: {str(e)}")
    raise

# View logs in Fabric Notebook monitoring section
```

## Key Takeaways

- **Unified Platform**: Microsoft Fabric eliminates the need for multiple disconnected services
- **OneLake Storage**: Single source of truth for all analytics workloads
- **Medallion Architecture**: Systematic data refinement from raw to business-ready
- **Low-Code + Code**: Dataflow Gen2 for transformations, PySpark for complex logic
- **Semantic Layer**: Centralized business metrics for consistent reporting
- **Native Integration**: Power BI connects directly without data movement

This skill enables AI agents to guide developers through building production-grade unified analytics platforms on Microsoft Fabric, following modern data engineering best practices.
