---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering on Microsoft Fabric using Lakehouse, Dataflow Gen2, PySpark, and Power BI with Medallion Architecture
triggers:
  - "how do I build a Lakehouse on Microsoft Fabric"
  - "implement Medallion Architecture with Bronze Silver Gold layers"
  - "use Dataflow Gen2 for data transformation"
  - "process data with Fabric Notebooks and PySpark"
  - "create a Semantic Model in Microsoft Fabric"
  - "build unified analytics platform with OneLake"
  - "design end-to-end data pipeline in Fabric"
  - "integrate Power BI with Fabric Lakehouse"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end unified analytics platforms using **Microsoft Fabric**. The project demonstrates a production-ready implementation of the **Medallion Architecture** (Bronze → Silver → Gold) using OneLake, Dataflow Gen2, Fabric Notebooks with PySpark, Semantic Models, and Power BI.

## What This Project Does

Microsoft Fabric Unified Analytics Platform showcases how to:

- Build a **Lakehouse architecture** on Microsoft Fabric's unified SaaS platform
- Implement **Medallion Architecture** with Bronze (raw), Silver (cleansed), and Gold (curated) layers
- Use **Dataflow Gen2** for low-code data transformations
- Process data with **Fabric Notebooks** and PySpark for business logic
- Create **Semantic Models** for consistent KPIs and metrics
- Integrate **Power BI** for business intelligence dashboards
- Leverage **OneLake** as unified storage across all workloads

The project uses retail analytics as the use case, transforming transactional data into trusted business insights.

## Prerequisites

Before using this project, you need:

1. **Microsoft Fabric Workspace** (capacity-based or trial)
2. **Microsoft Fabric Lakehouse** created in your workspace
3. **Power BI Premium** or Fabric capacity
4. Source data (CSV files or other formats)
5. Basic knowledge of PySpark and Power BI

## Project Structure

```
project/
├── architecture/          # Architecture diagrams and screenshots
├── notebooks/            # PySpark transformation notebooks
├── case-study/           # Technical documentation
└── README.md
```

## Medallion Architecture Implementation

### Bronze Layer (Raw Data)

The Bronze layer stores raw, unprocessed data from source systems. In Microsoft Fabric Lakehouse:

1. **Create Lakehouse folders:**
   - Navigate to your Fabric Lakehouse
   - Create folders: `bronze/`, `silver/`, `gold/`

2. **Upload raw data to Bronze:**
   - Use Fabric UI to upload CSV/Parquet files
   - Or use Fabric Pipelines to automate ingestion

**PySpark to read Bronze data:**

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

# Initialize Spark session (automatically configured in Fabric)
spark = SparkSession.builder.appName("BronzeReader").getOrCreate()

# Read raw data from Bronze layer
bronze_df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("Files/bronze/online_retail.csv")

# Display schema
bronze_df.printSchema()

# Show sample data
display(bronze_df.limit(10))
```

### Silver Layer (Cleansed Data)

The Silver layer contains cleansed, standardized, and deduplicated data.

#### Using Dataflow Gen2

1. **Create Dataflow Gen2:**
   - In Fabric workspace, click **New** → **Dataflow Gen2**
   - Connect to Bronze layer data source
   - Apply transformations in Power Query

2. **Key transformations:**
   - Remove null values: `Table.SelectRows(Source, each [InvoiceNo] <> null)`
   - Remove duplicates: `Table.Distinct(PreviousStep)`
   - Change data types: `Table.TransformColumnTypes(...)`
   - Add calculated columns: `Table.AddColumn(...)`

3. **Data destination:**
   - Set destination to **Lakehouse**
   - Target folder: `silver/online_retail_clean`
   - Update method: **Replace**

#### Using PySpark in Fabric Notebook

```python
from pyspark.sql.functions import col, when, year, month, regexp_replace

# Read Bronze data
bronze_df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("Files/bronze/online_retail.csv")

# Data quality transformations
silver_df = bronze_df \
    .filter(col("InvoiceNo").isNotNull()) \
    .filter(col("CustomerID").isNotNull()) \
    .filter(col("Quantity") > 0) \
    .filter(col("UnitPrice") > 0) \
    .dropDuplicates()

# Add business enrichment columns
silver_df = silver_df \
    .withColumn("line_total", col("Quantity") * col("UnitPrice")) \
    .withColumn("year", year(col("InvoiceDate"))) \
    .withColumn("month", month(col("InvoiceDate"))) \
    .withColumn("is_return", when(col("InvoiceNo").startswith("C"), True).otherwise(False))

# Standardize data types
silver_df = silver_df \
    .withColumn("InvoiceNo", regexp_replace(col("InvoiceNo"), "C", "")) \
    .withColumn("CustomerID", col("CustomerID").cast("integer"))

# Write to Silver layer (Delta format for optimization)
silver_df.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Tables/silver_online_retail")

print(f"Silver layer created: {silver_df.count()} records")
```

### Gold Layer (Business-Ready Data)

The Gold layer contains aggregated, business-ready datasets optimized for analytics.

#### Revenue Analysis (Gold Layer)

```python
from pyspark.sql.functions import sum, count, avg, round

# Read Silver data
silver_df = spark.read.format("delta").load("Tables/silver_online_retail")

# Calculate revenue metrics by year and month
revenue_by_period = silver_df \
    .groupBy("year", "month") \
    .agg(
        round(sum("line_total"), 2).alias("total_revenue"),
        count("InvoiceNo").alias("transaction_count"),
        round(avg("line_total"), 2).alias("avg_transaction_value"),
        count(when(col("is_return") == True, 1)).alias("return_count")
    ) \
    .orderBy("year", "month")

# Write to Gold layer
revenue_by_period.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Tables/gold_revenue_analysis")

display(revenue_by_period)
```

#### Product Performance (Gold Layer)

```python
# Product-level analysis
product_performance = silver_df \
    .groupBy("StockCode", "Description") \
    .agg(
        round(sum("line_total"), 2).alias("total_sales"),
        sum("Quantity").alias("total_quantity_sold"),
        count("InvoiceNo").alias("transaction_count"),
        round(avg("UnitPrice"), 2).alias("avg_unit_price")
    ) \
    .orderBy(col("total_sales").desc())

# Write to Gold layer
product_performance.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Tables/gold_product_performance")

# Show top 20 products
display(product_performance.limit(20))
```

#### Customer Segmentation (RFM Analysis)

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import max, datediff, current_date, ntile

# Calculate RFM metrics
customer_rfm = silver_df \
    .groupBy("CustomerID") \
    .agg(
        max("InvoiceDate").alias("last_purchase_date"),
        count("InvoiceNo").alias("frequency"),
        round(sum("line_total"), 2).alias("monetary")
    )

# Calculate Recency (days since last purchase)
customer_rfm = customer_rfm \
    .withColumn("recency", datediff(current_date(), col("last_purchase_date")))

# Create RFM quintiles
window_spec = Window.orderBy(col("recency").asc())
customer_rfm = customer_rfm.withColumn("r_score", ntile(5).over(window_spec))

window_spec = Window.orderBy(col("frequency").desc())
customer_rfm = customer_rfm.withColumn("f_score", ntile(5).over(window_spec))

window_spec = Window.orderBy(col("monetary").desc())
customer_rfm = customer_rfm.withColumn("m_score", ntile(5).over(window_spec))

# Calculate RFM score
customer_rfm = customer_rfm \
    .withColumn("rfm_score", col("r_score") + col("f_score") + col("m_score"))

# Segment customers
customer_rfm = customer_rfm.withColumn(
    "customer_segment",
    when((col("r_score") >= 4) & (col("f_score") >= 4), "Champions")
    .when((col("r_score") >= 3) & (col("f_score") >= 3), "Loyal")
    .when(col("r_score") >= 4, "Potential Loyalists")
    .when(col("f_score") >= 4, "Promising")
    .when(col("r_score") <= 2, "At Risk")
    .otherwise("Others")
)

# Write to Gold layer
customer_rfm.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Tables/gold_customer_segmentation")

display(customer_rfm)
```

## Working with Semantic Models

After creating Gold layer datasets, build a Semantic Model to define business logic and relationships.

### Create Semantic Model

1. **Navigate to Lakehouse:**
   - Open your Fabric Lakehouse
   - Click **New Semantic Model**

2. **Select Gold tables:**
   - `gold_revenue_analysis`
   - `gold_product_performance`
   - `gold_customer_segmentation`

3. **Define relationships:**
   - Create relationships between dimension and fact tables
   - Set cardinality and cross-filter direction

4. **Create measures (DAX):**

```dax
// Total Revenue
Total Revenue = SUM(gold_revenue_analysis[total_revenue])

// Average Transaction Value
Avg Transaction Value = AVERAGE(gold_revenue_analysis[avg_transaction_value])

// Customer Count
Customer Count = DISTINCTCOUNT(gold_customer_segmentation[CustomerID])

// Revenue Growth %
Revenue Growth % = 
DIVIDE(
    [Total Revenue] - CALCULATE([Total Revenue], DATEADD(Date[Date], -1, MONTH)),
    CALCULATE([Total Revenue], DATEADD(Date[Date], -1, MONTH)),
    0
)

// Top Product by Revenue
Top Product = 
TOPN(
    1,
    VALUES(gold_product_performance[Description]),
    [Total Revenue],
    DESC
)
```

## Power BI Integration

### Connect Power BI to Semantic Model

1. **Open Power BI Desktop**
2. **Get Data** → **Power Platform** → **Semantic Models**
3. Select your Fabric Semantic Model
4. Build reports using pre-defined measures

### Create Executive Dashboard

```dax
// Key Performance Indicators
Total Customers = DISTINCTCOUNT(gold_customer_segmentation[CustomerID])

Total Transactions = SUM(gold_revenue_analysis[transaction_count])

Return Rate = 
DIVIDE(
    SUM(gold_revenue_analysis[return_count]),
    SUM(gold_revenue_analysis[transaction_count]),
    0
) * 100

// Time Intelligence
YTD Revenue = TOTALYTD([Total Revenue], Date[Date])

Previous Month Revenue = CALCULATE([Total Revenue], DATEADD(Date[Date], -1, MONTH))
```

## Common Patterns and Best Practices

### 1. Incremental Loading Pattern

```python
from datetime import datetime, timedelta

# Get last processed date
last_processed = spark.read.format("delta").load("Tables/silver_online_retail") \
    .agg(max("InvoiceDate")).collect()[0][0]

# Read only new data from Bronze
incremental_df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("Files/bronze/online_retail.csv") \
    .filter(col("InvoiceDate") > last_processed)

# Apply transformations and append to Silver
# ... transformation logic ...

incremental_df.write \
    .format("delta") \
    .mode("append") \
    .save("Tables/silver_online_retail")
```

### 2. Data Quality Validation

```python
from pyspark.sql.functions import col, count, when, isnan

def validate_data_quality(df, table_name):
    """Validate data quality and log issues"""
    
    # Count nulls per column
    null_counts = df.select([
        count(when(col(c).isNull() | isnan(c), c)).alias(c)
        for c in df.columns
    ])
    
    print(f"=== Data Quality Report: {table_name} ===")
    display(null_counts)
    
    # Count duplicates
    duplicate_count = df.count() - df.dropDuplicates().count()
    print(f"Duplicate records: {duplicate_count}")
    
    # Record count
    print(f"Total records: {df.count()}")
    
    return null_counts

# Usage
validate_data_quality(silver_df, "silver_online_retail")
```

### 3. Parameterized Notebook Execution

```python
# Define notebook parameters
# These can be passed from Fabric Pipelines
bronze_path = "Files/bronze/online_retail.csv"
silver_table = "Tables/silver_online_retail"
gold_table = "Tables/gold_revenue_analysis"

# Use parameters in your code
df = spark.read.format("csv") \
    .option("header", "true") \
    .load(bronze_path)

# ... processing logic ...

df.write.format("delta") \
    .mode("overwrite") \
    .save(silver_table)
```

### 4. Delta Lake Optimization

```python
# Optimize Delta tables for better query performance
spark.sql(f"OPTIMIZE delta.`Tables/silver_online_retail`")

# Z-ORDER by frequently filtered columns
spark.sql(f"""
    OPTIMIZE delta.`Tables/silver_online_retail`
    ZORDER BY (CustomerID, InvoiceDate)
""")

# Vacuum old files (7 days retention)
spark.sql(f"""
    VACUUM delta.`Tables/silver_online_retail`
    RETAIN 168 HOURS
""")
```

## Troubleshooting

### Issue: "Path does not exist" Error

```python
# Check if path exists before reading
from pyspark.sql.utils import AnalysisException

try:
    df = spark.read.format("delta").load("Tables/silver_online_retail")
except AnalysisException as e:
    print(f"Table not found: {e}")
    print("Creating new table...")
    # Initialize empty table
```

### Issue: Schema Evolution Conflicts

```python
# Enable schema evolution for Delta tables
df.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .option("mergeSchema", "true") \
    .save("Tables/silver_online_retail")
```

### Issue: Memory Errors with Large Datasets

```python
# Process data in partitions
df = spark.read.format("csv") \
    .option("header", "true") \
    .load("Files/bronze/large_dataset.csv") \
    .repartition(10)  # Increase parallelism

# Cache intermediate results
df.cache()

# Use broadcast joins for small dimension tables
from pyspark.sql.functions import broadcast

result = large_df.join(
    broadcast(small_df),
    "key_column",
    "left"
)
```

### Issue: Dataflow Gen2 Timeout

- **Split large transformations** into multiple Dataflows
- **Enable staging** in Dataflow settings
- **Increase refresh timeout** in workspace settings
- **Filter data early** to reduce processing volume

## Configuration Best Practices

1. **Workspace Organization:**
   - Use separate workspaces for Dev, Test, Prod
   - Enable workspace identity for OneLake access
   - Configure capacity settings appropriately

2. **Lakehouse Structure:**
   - Follow Medallion Architecture consistently
   - Use Delta Lake format for Silver and Gold layers
   - Implement folder structure: `bronze/`, `silver/`, `gold/`

3. **Notebook Configuration:**
   - Set Spark configuration at session level
   - Use environment variables for sensitive configs
   - Document notebook parameters clearly

4. **Semantic Model:**
   - Define relationships explicitly
   - Use star schema design
   - Create reusable measures
   - Enable incremental refresh for large datasets

5. **Security:**
   - Use workspace roles for access control
   - Enable row-level security (RLS) in Semantic Model
   - Store credentials in Azure Key Vault
   - Reference secrets via environment variables: `os.getenv("FABRIC_TOKEN")`

## Next Steps

After mastering the basics:

1. Implement **Fabric Pipelines** for orchestration
2. Add **Real-Time Intelligence** with Eventstream
3. Enable **Delta Lake optimization** (OPTIMIZE, VACUUM)
4. Set up **CI/CD** with deployment pipelines
5. Implement **data governance** with Purview integration
6. Add **monitoring and alerting** for data quality

This skill provides the foundation for building production-grade unified analytics platforms on Microsoft Fabric.
