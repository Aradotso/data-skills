---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering with Microsoft Fabric using Lakehouse, Medallion Architecture, Dataflow Gen2, PySpark notebooks, and Power BI
triggers:
  - "set up microsoft fabric lakehouse"
  - "implement medallion architecture in fabric"
  - "create fabric dataflow gen2 pipeline"
  - "write pyspark notebooks for fabric"
  - "build semantic model in microsoft fabric"
  - "configure onelake storage layers"
  - "transform data bronze to silver to gold"
  - "design unified analytics platform"
---

# Microsoft Fabric Unified Analytics Platform Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end unified analytics platforms using **Microsoft Fabric**. The project demonstrates production-grade data engineering following the **Medallion Architecture** (Bronze → Silver → Gold) with OneLake storage, Dataflow Gen2 transformations, PySpark notebooks, semantic modeling, and Power BI integration.

## What This Project Does

Microsoft Fabric Unified Analytics Platform provides:

- **Lakehouse Architecture**: Organized data storage using OneLake with Bronze, Silver, and Gold layers
- **Medallion Pattern**: Progressive data refinement from raw to business-ready datasets
- **Dataflow Gen2**: Low-code ETL transformations for data quality improvement
- **PySpark Processing**: Business logic implementation in Fabric Notebooks
- **Semantic Models**: Centralized business layer with reusable KPIs and metrics
- **Power BI Integration**: Native visualization on trusted data assets
- **Unified SaaS Platform**: All components within a single Microsoft Fabric workspace

## Prerequisites

- Microsoft Fabric capacity (Trial, F64, or higher)
- Microsoft Fabric workspace with Contributor or Admin role
- Power BI Premium or Fabric capacity license
- Azure Storage account (optional for external data sources)

## Lakehouse Setup

### Create a Fabric Lakehouse

1. Navigate to Microsoft Fabric workspace
2. Create new Lakehouse:

```python
# Lakehouse is created via Fabric UI, then accessed in notebooks
# Reference the lakehouse in your notebook

from pyspark.sql import SparkSession

# Fabric automatically configures Spark session with lakehouse context
spark = SparkSession.builder.getOrCreate()

# Verify lakehouse connection
display(spark.catalog.listDatabases())
```

### Organize Medallion Layers

Create folder structure in Files section:

```
Files/
├── bronze/           # Raw source data
│   ├── transactions/
│   ├── customers/
│   └── products/
├── silver/          # Cleansed data
│   ├── transactions/
│   ├── customers/
│   └── products/
└── gold/            # Business-ready datasets
    ├── kpis/
    ├── revenue_trends/
    ├── customer_analytics/
    └── product_performance/
```

## Data Ingestion to Bronze Layer

### Using Dataflow Gen2 (Low-Code)

1. Create Dataflow Gen2 in Fabric workspace
2. Add data source (CSV, SQL, API, etc.)
3. Apply minimal transformations (preserve raw data)
4. Configure destination:
   - **Destination**: Lakehouse
   - **Folder**: `Files/bronze/transactions`
   - **File format**: Parquet

### Using Notebook (Code-Based)

```python
# Load raw data into bronze layer
from pyspark.sql import functions as F
from datetime import datetime

# Read source data (example: CSV from external storage)
bronze_df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("abfss://container@storage.dfs.core.windows.net/source/transactions.csv")

# Add metadata columns
bronze_df = bronze_df \
    .withColumn("ingestion_timestamp", F.current_timestamp()) \
    .withColumn("source_file", F.input_file_name())

# Write to bronze layer
bronze_path = "Files/bronze/transactions"
bronze_df.write \
    .mode("overwrite") \
    .format("delta") \
    .save(bronze_path)

# Create delta table for SQL access
spark.sql(f"""
    CREATE TABLE IF NOT EXISTS bronze_transactions
    USING DELTA
    LOCATION '{bronze_path}'
""")

print(f"✓ Bronze layer ingestion complete: {bronze_df.count()} records")
```

## Silver Layer Transformation

### Dataflow Gen2 Transformations

Common transformations for Silver layer:

```python
# Example transformations to apply in Dataflow Gen2:
# 1. Remove duplicates (Group By > All Rows)
# 2. Handle null values (Replace Values or Remove Empty)
# 3. Standardize data types (Change Type)
# 4. Add calculated columns (Add Column > Custom Column)
# 
# Custom Column examples:
# line_total = [Quantity] * [UnitPrice]
# year = Date.Year([InvoiceDate])
# month = Date.Month([InvoiceDate])
# is_return = if [Quantity] < 0 then "Yes" else "No"
```

### PySpark Silver Transformation

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Read from bronze
bronze_df = spark.read.format("delta").load("Files/bronze/transactions")

# Data quality transformations
silver_df = bronze_df \
    .dropDuplicates(["InvoiceNo", "StockCode", "InvoiceDate"]) \
    .filter(F.col("Quantity").isNotNull()) \
    .filter(F.col("UnitPrice").isNotNull()) \
    .filter(F.col("CustomerID").isNotNull())

# Business enrichment
silver_df = silver_df \
    .withColumn("line_total", F.col("Quantity") * F.col("UnitPrice")) \
    .withColumn("year", F.year(F.col("InvoiceDate"))) \
    .withColumn("month", F.month(F.col("InvoiceDate"))) \
    .withColumn("is_return", F.when(F.col("Quantity") < 0, "Yes").otherwise("No")) \
    .withColumn("processed_timestamp", F.current_timestamp())

# Data type standardization
silver_df = silver_df \
    .withColumn("Quantity", F.col("Quantity").cast("integer")) \
    .withColumn("UnitPrice", F.col("UnitPrice").cast("decimal(10,2)")) \
    .withColumn("CustomerID", F.col("CustomerID").cast("string"))

# Write to silver layer
silver_path = "Files/silver/transactions"
silver_df.write \
    .mode("overwrite") \
    .format("delta") \
    .partitionBy("year", "month") \
    .save(silver_path)

spark.sql(f"""
    CREATE TABLE IF NOT EXISTS silver_transactions
    USING DELTA
    LOCATION '{silver_path}'
""")

print(f"✓ Silver layer transformation complete: {silver_df.count()} records")
```

## Gold Layer Business Logic

### Revenue Analytics

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Read silver data
silver_df = spark.read.format("delta").load("Files/silver/transactions")

# Calculate revenue metrics by period
revenue_trends = silver_df \
    .filter(F.col("is_return") == "No") \
    .groupBy("year", "month") \
    .agg(
        F.sum("line_total").alias("total_revenue"),
        F.countDistinct("InvoiceNo").alias("total_orders"),
        F.countDistinct("CustomerID").alias("unique_customers"),
        F.avg("line_total").alias("avg_order_value"),
        F.count("*").alias("total_items_sold")
    ) \
    .withColumn("revenue_per_customer", 
                F.col("total_revenue") / F.col("unique_customers")) \
    .orderBy("year", "month")

# Write to gold layer
gold_revenue_path = "Files/gold/revenue_trends"
revenue_trends.write \
    .mode("overwrite") \
    .format("delta") \
    .save(gold_revenue_path)

spark.sql(f"""
    CREATE TABLE IF NOT EXISTS gold_revenue_trends
    USING DELTA
    LOCATION '{gold_revenue_path}'
""")
```

### Product Performance

```python
# Product performance analysis
product_performance = silver_df \
    .filter(F.col("is_return") == "No") \
    .groupBy("StockCode", "Description") \
    .agg(
        F.sum("Quantity").alias("total_quantity_sold"),
        F.sum("line_total").alias("total_revenue"),
        F.countDistinct("InvoiceNo").alias("order_frequency"),
        F.countDistinct("CustomerID").alias("unique_buyers"),
        F.avg("UnitPrice").alias("avg_unit_price")
    ) \
    .withColumn("revenue_rank", 
                F.row_number().over(Window.orderBy(F.desc("total_revenue"))))

# Categorize products
product_performance = product_performance \
    .withColumn("performance_tier",
                F.when(F.col("revenue_rank") <= 10, "Top")
                 .when(F.col("revenue_rank") <= 50, "High")
                 .when(F.col("revenue_rank") <= 200, "Medium")
                 .otherwise("Low"))

gold_product_path = "Files/gold/product_performance"
product_performance.write \
    .mode("overwrite") \
    .format("delta") \
    .save(gold_product_path)

spark.sql(f"""
    CREATE TABLE IF NOT EXISTS gold_product_performance
    USING DELTA
    LOCATION '{gold_product_path}'
""")
```

### Customer RFM Segmentation

```python
from datetime import datetime

# Calculate RFM metrics
max_date = silver_df.agg(F.max("InvoiceDate")).collect()[0][0]

customer_rfm = silver_df \
    .filter(F.col("is_return") == "No") \
    .groupBy("CustomerID") \
    .agg(
        F.datediff(F.lit(max_date), F.max("InvoiceDate")).alias("recency"),
        F.countDistinct("InvoiceNo").alias("frequency"),
        F.sum("line_total").alias("monetary")
    )

# Calculate RFM scores (1-5 scale)
for metric in ["recency", "frequency", "monetary"]:
    ascending = True if metric == "recency" else False
    customer_rfm = customer_rfm.withColumn(
        f"{metric}_score",
        F.ntile(5).over(Window.orderBy(F.col(metric).asc() if ascending else F.col(metric).desc()))
    )

# Create RFM segment
customer_rfm = customer_rfm \
    .withColumn("rfm_score", 
                F.concat(F.col("recency_score"), 
                        F.col("frequency_score"), 
                        F.col("monetary_score"))) \
    .withColumn("customer_segment",
                F.when((F.col("recency_score") >= 4) & (F.col("frequency_score") >= 4), "Champions")
                 .when((F.col("recency_score") >= 3) & (F.col("frequency_score") >= 3), "Loyal")
                 .when((F.col("recency_score") >= 4) & (F.col("frequency_score") <= 2), "Promising")
                 .when((F.col("recency_score") <= 2) & (F.col("frequency_score") >= 3), "At Risk")
                 .otherwise("Needs Attention"))

gold_customer_path = "Files/gold/customer_analytics"
customer_rfm.write \
    .mode("overwrite") \
    .format("delta") \
    .save(gold_customer_path)

spark.sql(f"""
    CREATE TABLE IF NOT EXISTS gold_customer_analytics
    USING DELTA
    LOCATION '{gold_customer_path}'
""")
```

## Semantic Model Configuration

### Create Semantic Model

1. In Fabric workspace, navigate to your Lakehouse
2. Select **New Semantic Model**
3. Choose Gold layer tables:
   - `gold_revenue_trends`
   - `gold_product_performance`
   - `gold_customer_analytics`

### Define Relationships (DAX)

```dax
// Example Date dimension (create in Power BI)
DateTable = 
ADDCOLUMNS(
    CALENDAR(DATE(2020,1,1), DATE(2026,12,31)),
    "Year", YEAR([Date]),
    "Month", MONTH([Date]),
    "MonthName", FORMAT([Date], "MMM"),
    "Quarter", "Q" & FORMAT([Date], "Q"),
    "YearMonth", FORMAT([Date], "YYYY-MM")
)

// Mark as date table
MARK AS DATE TABLE DateTable[Date]
```

### Key Measures

```dax
// Total Revenue
Total Revenue = 
SUM(gold_revenue_trends[total_revenue])

// Revenue Growth %
Revenue Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD(DateTable[Date], -1, MONTH)
    )
RETURN
DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0)

// Average Order Value
Avg Order Value = 
AVERAGE(gold_revenue_trends[avg_order_value])

// Customer Lifetime Value
Customer LTV = 
AVERAGEX(
    VALUES(gold_customer_analytics[CustomerID]),
    CALCULATE(SUM(gold_customer_analytics[monetary]))
)

// Top Products by Revenue
Top 10 Products Revenue = 
CALCULATE(
    [Total Revenue],
    TOPN(10, gold_product_performance, gold_product_performance[total_revenue], DESC)
)
```

## Power BI Report Development

### Connect to Semantic Model

```python
# Power BI automatically connects to Fabric semantic models
# No separate connection configuration needed

# In Power BI Desktop:
# 1. Get Data > Power Platform > Fabric Semantic Models
# 2. Select your semantic model
# 3. Choose tables/measures to use
```

### Key Visualizations

**Revenue Dashboard:**
- Line chart: Revenue trends over time
- Card visuals: Total revenue, growth %, unique customers
- Bar chart: Revenue by country
- Table: Top 10 products

**Customer Analytics:**
- Donut chart: Customer segmentation distribution
- Scatter chart: RFM analysis (Frequency vs Monetary)
- Table: Customer details with RFM scores

**Product Performance:**
- Tree map: Revenue by product category
- Bar chart: Top products by quantity sold
- Table: Product performance metrics

## Common Patterns

### Incremental Load Pattern

```python
from pyspark.sql import functions as F
from delta.tables import DeltaTable

# Check if table exists
bronze_path = "Files/bronze/transactions"
if DeltaTable.isDeltaTable(spark, bronze_path):
    # Get last ingestion timestamp
    last_ingestion = spark.read.format("delta").load(bronze_path) \
        .agg(F.max("ingestion_timestamp")).collect()[0][0]
    
    # Read only new records
    new_df = spark.read.format("csv") \
        .option("header", "true") \
        .load("source/transactions.csv") \
        .filter(F.col("LoadDate") > last_ingestion)
    
    # Append new records
    new_df.write.format("delta").mode("append").save(bronze_path)
else:
    # First load - full refresh
    df = spark.read.format("csv") \
        .option("header", "true") \
        .load("source/transactions.csv")
    df.write.format("delta").mode("overwrite").save(bronze_path)
```

### Data Quality Checks

```python
from pyspark.sql import functions as F

def data_quality_check(df, table_name):
    """
    Perform data quality checks on DataFrame
    """
    checks = {
        "total_records": df.count(),
        "null_counts": {},
        "duplicate_count": df.count() - df.dropDuplicates().count()
    }
    
    # Check nulls for each column
    for col in df.columns:
        null_count = df.filter(F.col(col).isNull()).count()
        if null_count > 0:
            checks["null_counts"][col] = null_count
    
    # Log results
    print(f"\n=== Data Quality Report: {table_name} ===")
    print(f"Total Records: {checks['total_records']}")
    print(f"Duplicate Records: {checks['duplicate_count']}")
    if checks["null_counts"]:
        print("Null Counts:")
        for col, count in checks["null_counts"].items():
            print(f"  - {col}: {count}")
    else:
        print("✓ No null values detected")
    
    return checks

# Usage
silver_df = spark.read.format("delta").load("Files/silver/transactions")
quality_report = data_quality_check(silver_df, "silver_transactions")
```

### Parameterized Processing

```python
# Create notebook parameters cell (first cell with tag "parameters")
# Default values:
layer = "silver"  # bronze, silver, gold
entity = "transactions"  # transactions, customers, products
mode = "overwrite"  # overwrite, append

# Processing logic using parameters
source_path = f"Files/{layer}/{entity}"
target_path = f"Files/{layer}/{entity}_processed"

df = spark.read.format("delta").load(source_path)

# Apply transformations based on parameters
if layer == "silver":
    df = apply_silver_transformations(df, entity)
elif layer == "gold":
    df = apply_gold_transformations(df, entity)

df.write.format("delta").mode(mode).save(target_path)
```

## Troubleshooting

### Lakehouse Not Accessible

```python
# Verify lakehouse is attached to notebook
# In notebook: Settings > Lakehouse > Add
# Ensure correct permissions (Contributor or Admin)

# Test connection
try:
    databases = spark.catalog.listDatabases()
    print("✓ Lakehouse connected")
    for db in databases:
        print(f"  - {db.name}")
except Exception as e:
    print(f"✗ Lakehouse connection failed: {str(e)}")
```

### Delta Table Not Found

```python
# Refresh metadata
spark.sql("REFRESH TABLE bronze_transactions")

# Or recreate table reference
spark.sql("""
    CREATE TABLE IF NOT EXISTS bronze_transactions
    USING DELTA
    LOCATION 'Files/bronze/transactions'
""")
```

### Dataflow Gen2 Refresh Errors

Check in Dataflow Gen2 settings:
- Destination lakehouse has write permissions
- Folder path exists in Files section
- No schema conflicts with existing data

### Power BI Semantic Model Not Updating

```python
# Trigger semantic model refresh via REST API
import requests
import os

# Get access token (requires Fabric API permissions)
workspace_id = os.getenv("FABRIC_WORKSPACE_ID")
dataset_id = os.getenv("FABRIC_DATASET_ID")
access_token = os.getenv("FABRIC_ACCESS_TOKEN")

url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/refreshes"
headers = {"Authorization": f"Bearer {access_token}"}

response = requests.post(url, headers=headers)
if response.status_code == 202:
    print("✓ Semantic model refresh triggered")
else:
    print(f"✗ Refresh failed: {response.text}")
```

### Memory Errors in Notebooks

```python
# Optimize Spark configuration for large datasets
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")

# Process in batches
batch_size = 100000
total_records = bronze_df.count()

for offset in range(0, total_records, batch_size):
    batch_df = bronze_df.limit(batch_size).offset(offset)
    # Process batch
    process_batch(batch_df)
```

## Best Practices

1. **Medallion Architecture**: Always separate Bronze (raw), Silver (cleansed), Gold (business-ready)
2. **Delta Lake Format**: Use Delta format for ACID transactions and time travel
3. **Partitioning**: Partition large tables by date columns for query performance
4. **Data Quality**: Implement quality checks at each layer transition
5. **Metadata**: Add ingestion timestamps and source tracking columns
6. **Semantic Layer**: Define business logic once in semantic models, reuse across reports
7. **Incremental Processing**: Implement incremental loads for efficiency
8. **Documentation**: Document transformations and business rules in notebook markdown cells

This skill enables comprehensive development of unified analytics platforms using Microsoft Fabric's integrated capabilities.
