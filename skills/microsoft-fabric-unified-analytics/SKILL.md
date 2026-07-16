---
name: microsoft-fabric-unified-analytics
description: End-to-end analytics platform using Microsoft Fabric with Lakehouse, Dataflow Gen2, PySpark notebooks, and Power BI following Medallion Architecture
triggers:
  - "build a Microsoft Fabric analytics solution"
  - "implement medallion architecture in Fabric"
  - "create a lakehouse with bronze silver gold layers"
  - "use Dataflow Gen2 for data transformation"
  - "write PySpark notebooks in Microsoft Fabric"
  - "set up semantic model in Fabric"
  - "design unified analytics platform"
  - "process retail data with Fabric"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates how to build a production-grade unified analytics platform using **Microsoft Fabric**, implementing the **Medallion Architecture** (Bronze → Silver → Gold) with **OneLake**, **Dataflow Gen2**, **Fabric Notebooks (PySpark)**, **Semantic Models**, and **Power BI**.

## What This Project Does

Microsoft Fabric Unified Analytics Platform showcases:
- **Lakehouse Architecture**: Organize data in Bronze (raw), Silver (cleansed), and Gold (business-ready) layers
- **Unified SaaS Platform**: Single environment for ingestion, processing, modeling, and visualization
- **OneLake Storage**: Centralized data lake foundation for all workloads
- **Low-Code + Code**: Combine Dataflow Gen2 (visual) with PySpark notebooks (code)
- **Semantic Modeling**: Create reusable business metrics and KPIs
- **Native BI Integration**: Power BI directly connected to the platform

## Architecture Overview

```
Raw Data → Bronze Layer (OneLake)
    ↓
Dataflow Gen2 → Silver Layer (Cleansed)
    ↓
Fabric Notebooks (PySpark) → Gold Layer (Business KPIs)
    ↓
Semantic Model → Power BI Reports
```

## Prerequisites

- **Microsoft Fabric Workspace**: With appropriate capacity (F64 or higher recommended)
- **Microsoft Fabric License**: Premium or trial capacity
- **Power BI Pro/Premium**: For report publishing
- **Python 3.8+**: For local notebook development (optional)
- **Azure Storage Account**: If ingesting from external sources

## Setting Up Microsoft Fabric Environment

### 1. Create Fabric Workspace

```python
# Fabric workspaces are created via the web UI
# Navigate to: https://app.fabric.microsoft.com
# Click "Workspaces" → "New Workspace"
# Name: "RetailAnalyticsPlatform"
# Assign Fabric capacity
```

### 2. Create Lakehouse

In your Fabric workspace:
1. Click **New** → **Lakehouse**
2. Name: `retail_analytics_lakehouse`
3. This creates OneLake storage with Delta tables support

### 3. Organize Medallion Layers

Create folder structure in your lakehouse:

```
Files/
├── bronze/
│   ├── online_retail/
│   │   └── online_retail.csv
├── silver/
│   ├── online_retail_cleaned/
└── gold/
    ├── revenue_trends/
    ├── product_performance/
    ├── customer_analytics/
    └── rfm_segmentation/
```

## Data Ingestion to Bronze Layer

### Upload Raw Data via Lakehouse UI

```python
# Files can be uploaded directly via the Lakehouse explorer
# Or programmatically using Fabric APIs
# Place CSV files in: Files/bronze/online_retail/
```

### Using Fabric Notebook for Ingestion

```python
# Fabric Notebook - Data Ingestion
from pyspark.sql import SparkSession

# Fabric provides pre-configured Spark session
spark = SparkSession.builder.getOrCreate()

# Read from external source (e.g., Azure Blob)
storage_account = "your_storage_account"
container = "raw-data"
sas_token = mssparkutils.credentials.getSecret("KeyVault", "SASToken")

df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load(f"wasbs://{container}@{storage_account}.blob.core.windows.net/online_retail.csv")

# Write to Bronze layer as Delta table
df.write.format("delta") \
    .mode("overwrite") \
    .save("Files/bronze/online_retail")

print(f"Ingested {df.count()} records to Bronze layer")
```

## Dataflow Gen2: Bronze to Silver Transformation

### Creating a Dataflow Gen2

1. In Fabric workspace: **New** → **Dataflow Gen2**
2. **Get data** → **Lakehouse** → Select Bronze layer table
3. Apply transformations using Power Query M language

### Key Transformations (Power Query M)

```m
let
    // Load from Bronze
    Source = Lakehouse.Contents(null),
    BronzeData = Source{[workspaceId="YOUR_WORKSPACE_ID"]}[Data],
    
    // Remove duplicates
    RemovedDuplicates = Table.Distinct(BronzeData, {"InvoiceNo", "StockCode"}),
    
    // Handle missing values
    RemovedNulls = Table.SelectRows(RemovedDuplicates, each [CustomerID] <> null),
    
    // Add business columns
    AddedLineTotal = Table.AddColumn(RemovedNulls, "line_total", 
        each [Quantity] * [UnitPrice], type number),
    
    // Extract date components
    AddedYear = Table.AddColumn(AddedLineTotal, "year", 
        each Date.Year([InvoiceDate]), Int64.Type),
    
    AddedMonth = Table.AddColumn(AddedYear, "month", 
        each Date.Month([InvoiceDate]), Int64.Type),
    
    // Flag returns
    AddedIsReturn = Table.AddColumn(AddedMonth, "is_return", 
        each if Text.StartsWith([InvoiceNo], "C") then true else false, 
        type logical),
    
    // Change data types
    ChangedTypes = Table.TransformColumnTypes(AddedIsReturn, {
        {"Quantity", Int64.Type},
        {"UnitPrice", Currency.Type},
        {"CustomerID", type text}
    })
in
    ChangedTypes
```

4. **Data destination**: Lakehouse → `silver/online_retail_cleaned`
5. **Publish** the dataflow

### Refresh Dataflow Programmatically

```python
# Using Fabric REST API
import requests
import os

workspace_id = os.getenv("FABRIC_WORKSPACE_ID")
dataflow_id = os.getenv("DATAFLOW_ID")
access_token = os.getenv("FABRIC_ACCESS_TOKEN")

url = f"https://api.fabric.microsoft.com/v1/workspaces/{workspace_id}/dataflows/{dataflow_id}/refresh"

headers = {
    "Authorization": f"Bearer {access_token}",
    "Content-Type": "application/json"
}

response = requests.post(url, headers=headers)
print(f"Dataflow refresh status: {response.status_code}")
```

## Fabric Notebooks: Silver to Gold with PySpark

### Revenue Trends Analysis

```python
# Fabric Notebook - Gold Layer: Revenue Trends
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, round, year, month, count

spark = SparkSession.builder.getOrCreate()

# Read Silver layer
silver_df = spark.read.format("delta").load("Files/silver/online_retail_cleaned")

# Filter out returns
valid_sales = silver_df.filter(col("is_return") == False)

# Aggregate revenue by year and month
revenue_trends = valid_sales.groupBy("year", "month") \
    .agg(
        sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("total_transactions"),
        round(sum("line_total") / count("InvoiceNo"), 2).alias("avg_order_value")
    ) \
    .orderBy("year", "month")

# Write to Gold layer
revenue_trends.write.format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Files/gold/revenue_trends")

# Display results
display(revenue_trends)
```

### Product Performance Analysis

```python
# Fabric Notebook - Gold Layer: Product Performance
from pyspark.sql.functions import col, sum, count, round, desc

# Top products by revenue
product_performance = valid_sales.groupBy("StockCode", "Description") \
    .agg(
        sum("line_total").alias("total_revenue"),
        sum("Quantity").alias("total_quantity_sold"),
        count("InvoiceNo").alias("transaction_count"),
        round(sum("line_total") / sum("Quantity"), 2).alias("avg_unit_revenue")
    ) \
    .filter(col("total_revenue") > 0) \
    .orderBy(desc("total_revenue")) \
    .limit(100)

# Write to Gold layer
product_performance.write.format("delta") \
    .mode("overwrite") \
    .save("Files/gold/product_performance")

display(product_performance)
```

### Customer Analytics

```python
# Fabric Notebook - Gold Layer: Customer Analytics
from pyspark.sql.functions import col, sum, count, countDistinct, round

customer_analytics = valid_sales.groupBy("CustomerID") \
    .agg(
        sum("line_total").alias("total_spent"),
        count("InvoiceNo").alias("total_orders"),
        countDistinct("StockCode").alias("unique_products_purchased"),
        round(sum("line_total") / count("InvoiceNo"), 2).alias("avg_order_value")
    ) \
    .filter(col("total_spent") > 0)

# Write to Gold layer
customer_analytics.write.format("delta") \
    .mode("overwrite") \
    .save("Files/gold/customer_analytics")

display(customer_analytics)
```

### RFM Segmentation

```python
# Fabric Notebook - Gold Layer: RFM Segmentation
from pyspark.sql.functions import col, datediff, max, count, sum, lit, current_date
from pyspark.sql.window import Window

# Calculate RFM metrics
max_date = valid_sales.select(max("InvoiceDate")).collect()[0][0]

rfm = valid_sales.groupBy("CustomerID") \
    .agg(
        datediff(lit(max_date), max("InvoiceDate")).alias("recency"),
        count("InvoiceNo").alias("frequency"),
        sum("line_total").alias("monetary")
    )

# Create RFM scores using ntile
window_spec = Window.orderBy(col("recency").desc())
rfm_scored = rfm.withColumn("r_score", ntile(5).over(window_spec))

window_spec = Window.orderBy(col("frequency"))
rfm_scored = rfm_scored.withColumn("f_score", ntile(5).over(window_spec))

window_spec = Window.orderBy(col("monetary"))
rfm_scored = rfm_scored.withColumn("m_score", ntile(5).over(window_spec))

# Create RFM segment
rfm_final = rfm_scored.withColumn("rfm_score", 
    col("r_score") * 100 + col("f_score") * 10 + col("m_score"))

# Segment classification
from pyspark.sql.functions import when

rfm_final = rfm_final.withColumn("customer_segment",
    when(col("rfm_score") >= 444, "Champions")
    .when(col("rfm_score") >= 334, "Loyal Customers")
    .when(col("rfm_score") >= 224, "Potential Loyalists")
    .when(col("rfm_score") >= 144, "At Risk")
    .otherwise("Lost")
)

# Write to Gold layer
rfm_final.write.format("delta") \
    .mode("overwrite") \
    .save("Files/gold/rfm_segmentation")

display(rfm_final)
```

## Creating Semantic Model

### 1. Create Semantic Model from Lakehouse

1. In Lakehouse explorer, go to **Tables** tab
2. Select Gold layer tables
3. Click **New semantic model**
4. Name: `RetailAnalyticsModel`

### 2. Define Relationships (DAX)

Open the semantic model in Power BI Desktop or Fabric Model View:

```dax
// Create Calendar table
Calendar = 
ADDCOLUMNS(
    CALENDAR(DATE(2020, 1, 1), DATE(2023, 12, 31)),
    "Year", YEAR([Date]),
    "Month", MONTH([Date]),
    "MonthName", FORMAT([Date], "MMMM"),
    "Quarter", "Q" & ROUNDUP(MONTH([Date])/3, 0)
)

// Create relationship: revenue_trends[year, month] → Calendar[Year, Month]
```

### 3. Create Measures (DAX)

```dax
// Total Revenue
Total Revenue = 
SUM(revenue_trends[total_revenue])

// Total Transactions
Total Transactions = 
SUM(revenue_trends[total_transactions])

// Average Order Value
Avg Order Value = 
AVERAGE(revenue_trends[avg_order_value])

// Revenue Growth %
Revenue Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD(Calendar[Date], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0)

// Customer Lifetime Value
Customer LTV = 
AVERAGE(customer_analytics[total_spent])

// Top Product Revenue
Top Product Revenue = 
CALCULATE(
    SUM(product_performance[total_revenue]),
    TOPN(1, ALL(product_performance), product_performance[total_revenue])
)
```

## Creating Power BI Reports

### Connect to Semantic Model

```python
# Power BI connects directly to the Fabric semantic model
# No additional code needed - use the Power BI service or Desktop
```

### Key Visualizations

1. **Revenue Trends**: Line chart with `Calendar[Date]` and `[Total Revenue]`
2. **Product Performance**: Table with top products by revenue
3. **Customer Segments**: Pie chart of RFM segments
4. **KPI Cards**: Total Revenue, Total Transactions, Avg Order Value

## Scheduling and Orchestration

### Create Fabric Pipeline

```python
# Fabric Pipelines are created via UI
# Navigate to workspace → New → Data Pipeline
# Name: "RetailAnalytics_ETL_Pipeline"

# Add activities:
# 1. Dataflow Gen2 activity → Select your dataflow
# 2. Notebook activity → Select PySpark notebooks
# 3. Refresh Semantic Model activity

# Set triggers:
# - Schedule: Daily at 2:00 AM
# - Or event-based when new files arrive in Bronze
```

### Using Fabric REST API for Pipeline Execution

```python
import requests
import os

workspace_id = os.getenv("FABRIC_WORKSPACE_ID")
pipeline_id = os.getenv("PIPELINE_ID")
access_token = os.getenv("FABRIC_ACCESS_TOKEN")

url = f"https://api.fabric.microsoft.com/v1/workspaces/{workspace_id}/pipelines/{pipeline_id}/jobs"

headers = {
    "Authorization": f"Bearer {access_token}",
    "Content-Type": "application/json"
}

response = requests.post(url, headers=headers)
job_id = response.json()["jobId"]
print(f"Pipeline started with job ID: {job_id}")
```

## Common Patterns

### Delta Table Optimization

```python
# Optimize Delta tables for better query performance
from delta.tables import DeltaTable

# Optimize Gold layer tables
delta_table = DeltaTable.forPath(spark, "Files/gold/revenue_trends")
delta_table.optimize().executeCompaction()
delta_table.vacuum(168)  # Clean up old files (7 days retention)

print("Delta table optimized")
```

### Incremental Data Loading

```python
# Load only new/changed data from Silver to Gold
from delta.tables import DeltaTable
from pyspark.sql.functions import col

# Read Silver with watermark
silver_df = spark.read.format("delta").load("Files/silver/online_retail_cleaned")

# Get last processed timestamp from Gold layer
try:
    gold_df = spark.read.format("delta").load("Files/gold/revenue_trends")
    last_processed = gold_df.select(max("InvoiceDate")).collect()[0][0]
    
    # Filter only new records
    new_records = silver_df.filter(col("InvoiceDate") > last_processed)
except:
    # First run - process all data
    new_records = silver_df

# Process and append
revenue_trends = new_records.groupBy("year", "month") \
    .agg(sum("line_total").alias("total_revenue"))

revenue_trends.write.format("delta") \
    .mode("append") \
    .save("Files/gold/revenue_trends")
```

### Error Handling and Logging

```python
# Robust error handling in Fabric Notebooks
from datetime import datetime
import json

def log_execution(status, message, details=None):
    """Log execution details to a monitoring table"""
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "status": status,
        "message": message,
        "details": details
    }
    
    # Write to monitoring table
    log_df = spark.createDataFrame([log_entry])
    log_df.write.format("delta").mode("append").save("Files/monitoring/execution_logs")

try:
    # Your transformation logic
    df = spark.read.format("delta").load("Files/silver/online_retail_cleaned")
    result = df.groupBy("year").agg(sum("line_total").alias("revenue"))
    result.write.format("delta").mode("overwrite").save("Files/gold/revenue_trends")
    
    log_execution("SUCCESS", "Revenue trends updated", {"row_count": result.count()})
    
except Exception as e:
    log_execution("ERROR", "Failed to update revenue trends", {"error": str(e)})
    raise
```

### Using Fabric Shortcuts

```python
# Create shortcuts to external data without copying
# This is done via Lakehouse UI:
# 1. Right-click on Files/bronze → New Shortcut
# 2. Select source: Azure Data Lake Storage Gen2, OneLake, S3, etc.
# 3. Provide connection details using environment variables
# 4. Data is accessible without duplication

# Access shortcut data in notebook
external_data = spark.read.format("delta").load("Files/bronze/external_shortcut/data")
```

## Troubleshooting

### Dataflow Gen2 Fails to Refresh

**Issue**: Dataflow refresh fails with timeout error

**Solution**:
```m
// In Dataflow, add query folding optimization
// Ensure source queries are delegated to source system

// Check query diagnostics
// Tools → Query Diagnostics → Start Diagnostics

// Optimize by reducing data volume early
let
    Source = Lakehouse.Contents(null),
    FilteredEarly = Table.SelectRows(Source, each [InvoiceDate] >= #date(2023, 1, 1))
in
    FilteredEarly
```

### Notebook Spark Session Memory Issues

**Issue**: PySpark job fails with OutOfMemoryError

**Solution**:
```python
# Configure Spark session with more memory
spark.conf.set("spark.executor.memory", "8g")
spark.conf.set("spark.driver.memory", "8g")
spark.conf.set("spark.sql.shuffle.partitions", "200")

# Use partitioning for large datasets
df.repartition(100).write.format("delta").save("Files/gold/large_dataset")

# Process data in batches
batch_size = 100000
for i in range(0, total_rows, batch_size):
    batch_df = df.limit(batch_size).offset(i)
    # Process batch
```

### Delta Table Schema Evolution

**Issue**: New columns in source data break Delta writes

**Solution**:
```python
# Enable schema merging
df.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("append") \
    .save("Files/silver/online_retail_cleaned")

# Or explicitly overwrite schema
df.write.format("delta") \
    .option("overwriteSchema", "true") \
    .mode("overwrite") \
    .save("Files/silver/online_retail_cleaned")
```

### Semantic Model Refresh Fails

**Issue**: Semantic model shows refresh errors

**Solution**:
```python
# Verify Gold tables are accessible
# Check table permissions in Lakehouse
# Ensure semantic model has workspace access

# Refresh via API with detailed logging
import requests

url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/refreshes"
headers = {"Authorization": f"Bearer {access_token}"}

response = requests.post(url, headers=headers)
if response.status_code != 202:
    print(f"Refresh failed: {response.text}")
```

## Best Practices

1. **Medallion Layers**: Keep Bronze immutable, Silver cleansed, Gold business-ready
2. **Delta Format**: Use Delta Lake for ACID transactions and time travel
3. **Partitioning**: Partition large tables by date for query performance
4. **Incremental Loads**: Process only changed data to reduce compute costs
5. **Monitoring**: Implement logging and alerting for pipeline failures
6. **Security**: Use Azure Key Vault for secrets, managed identities for authentication
7. **Testing**: Validate transformations in notebooks before productionizing
8. **Documentation**: Comment complex PySpark logic and DAX measures

## Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric/)
- [Dataflow Gen2 Guide](https://learn.microsoft.com/en-us/fabric/data-factory/dataflow-gen2-overview)
- [Fabric Notebooks](https://learn.microsoft.com/en-us/fabric/data-engineering/how-to-use-notebook)
- [Delta Lake on Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-optimization-and-v-order)
- [Original Project Repository](https://github.com/Abdoulkarimouattara/microsoft-fabric-unified-analytics-platformre)
