---
name: microsoft-fabric-unified-analytics
description: End-to-end analytics platform using Microsoft Fabric with Lakehouse, Medallion Architecture, Dataflow Gen2, PySpark notebooks, and Power BI integration
triggers:
  - "build a Microsoft Fabric analytics pipeline"
  - "implement Medallion architecture in Fabric"
  - "create Lakehouse with bronze silver gold layers"
  - "use Dataflow Gen2 for data transformation"
  - "write PySpark notebooks in Fabric"
  - "set up OneLake data storage"
  - "design semantic models for Power BI"
  - "integrate Power BI with Fabric Lakehouse"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end analytics solutions using Microsoft Fabric, implementing the Medallion Architecture (Bronze → Silver → Gold) with Lakehouse storage, Dataflow Gen2 transformations, PySpark processing, and Power BI integration.

## What This Project Does

Microsoft Fabric Unified Analytics Platform demonstrates:

- **Lakehouse Architecture**: OneLake-backed storage with Bronze/Silver/Gold layers
- **Medallion Pattern**: Progressive data refinement from raw to business-ready
- **Dataflow Gen2**: Low-code ETL transformations
- **Fabric Notebooks**: PySpark-based data processing and business logic
- **Semantic Models**: Centralized business metrics layer
- **Power BI Integration**: Native visualization and reporting

The solution transforms raw retail transaction data into trusted analytical assets through a unified SaaS platform.

## Prerequisites

- Microsoft Fabric workspace (Trial or paid capacity)
- Power BI account
- Basic understanding of PySpark and Python
- Familiarity with data warehousing concepts

## Architecture Overview

```
Data Source → Dataflow Gen2 → Lakehouse (Bronze) 
  → Dataflow Gen2 (cleansing) → Lakehouse (Silver)
  → Fabric Notebook (PySpark) → Lakehouse (Gold)
  → Semantic Model → Power BI
```

## Setting Up the Lakehouse

### 1. Create a Fabric Lakehouse

In your Microsoft Fabric workspace:

1. Click **+ New** → **Lakehouse**
2. Name it (e.g., `retail_analytics_lakehouse`)
3. Create folder structure in **Files**:
   - `bronze/`
   - `silver/`
   - `gold/`

### 2. Lakehouse Folder Structure

```
Files/
├── bronze/           # Raw ingested data
│   └── online_retail/
├── silver/           # Cleansed and standardized
│   └── online_retail_clean/
└── gold/             # Business-ready aggregates
    ├── revenue_trends/
    ├── product_performance/
    ├── customer_analytics/
    └── rfm_segmentation/
```

## Data Ingestion with Dataflow Gen2

### Creating a Dataflow Gen2

1. **New** → **Dataflow Gen2**
2. **Get Data** → Choose your source (CSV, SQL, API, etc.)
3. Apply transformations using Power Query
4. Set destination to **Lakehouse** → `bronze` folder

### Common Dataflow Transformations

```powerquery
// Remove duplicates
= Table.Distinct(Source, {"InvoiceNo", "StockCode"})

// Handle nulls
= Table.ReplaceValue(PreviousStep, null, "", Replacer.ReplaceValue, {"Description"})

// Add custom column for line total
= Table.AddColumn(PreviousStep, "line_total", each [Quantity] * [UnitPrice])

// Parse dates
= Table.TransformColumnTypes(PreviousStep, {{"InvoiceDate", type datetime}})

// Add year and month
= Table.AddColumn(PreviousStep, "year", each Date.Year([InvoiceDate]))
= Table.AddColumn(PreviousStep, "month", each Date.Month([InvoiceDate]))
```

### Dataflow Publishing

Configure destination:
- **Lakehouse**: Select your lakehouse
- **Table**: `bronze_online_retail` (for Bronze)
- **Update method**: Replace or Append

## PySpark Processing in Fabric Notebooks

### Reading from Lakehouse (Bronze → Silver)

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, when, trim, upper, year, month

# Read from Bronze layer
bronze_df = spark.read.format("delta").load("Files/bronze/online_retail")

# Data cleansing transformations
silver_df = (
    bronze_df
    .filter(col("Quantity") > 0)  # Remove negative quantities
    .filter(col("UnitPrice") > 0)  # Remove invalid prices
    .filter(col("CustomerID").isNotNull())  # Remove null customers
    .withColumn("Description", trim(upper(col("Description"))))  # Standardize
    .withColumn("Country", trim(upper(col("Country"))))
    .dropDuplicates(["InvoiceNo", "StockCode"])  # Remove duplicates
)

# Write to Silver layer
(
    silver_df
    .write
    .mode("overwrite")
    .format("delta")
    .option("overwriteSchema", "true")
    .save("Files/silver/online_retail_clean")
)
```

### Creating Gold Layer Aggregates

#### Revenue Trends

```python
from pyspark.sql.functions import sum, count, avg, round

# Read from Silver
silver_df = spark.read.format("delta").load("Files/silver/online_retail_clean")

# Calculate revenue trends by year/month
revenue_trends = (
    silver_df
    .groupBy("year", "month")
    .agg(
        round(sum(col("Quantity") * col("UnitPrice")), 2).alias("total_revenue"),
        count("InvoiceNo").alias("order_count"),
        round(avg(col("Quantity") * col("UnitPrice")), 2).alias("avg_order_value")
    )
    .orderBy("year", "month")
)

# Write to Gold layer
(
    revenue_trends
    .write
    .mode("overwrite")
    .format("delta")
    .save("Files/gold/revenue_trends")
)
```

#### Product Performance

```python
from pyspark.sql.window import Window

product_performance = (
    silver_df
    .groupBy("StockCode", "Description")
    .agg(
        round(sum(col("Quantity") * col("UnitPrice")), 2).alias("total_revenue"),
        sum("Quantity").alias("units_sold"),
        count("InvoiceNo").alias("order_count"),
        round(avg("UnitPrice"), 2).alias("avg_price")
    )
    .orderBy(col("total_revenue").desc())
)

# Write to Gold
(
    product_performance
    .write
    .mode("overwrite")
    .format("delta")
    .save("Files/gold/product_performance")
)
```

#### Customer Analytics

```python
from pyspark.sql.functions import countDistinct, first

customer_analytics = (
    silver_df
    .groupBy("CustomerID")
    .agg(
        round(sum(col("Quantity") * col("UnitPrice")), 2).alias("total_spend"),
        count("InvoiceNo").alias("order_count"),
        countDistinct("StockCode").alias("unique_products"),
        round(avg(col("Quantity") * col("UnitPrice")), 2).alias("avg_order_value"),
        first("Country").alias("country")
    )
)

# Write to Gold
(
    customer_analytics
    .write
    .mode("overwrite")
    .format("delta")
    .save("Files/gold/customer_analytics")
)
```

#### RFM Segmentation

```python
from pyspark.sql.functions import max, datediff, current_date, ntile

# Calculate RFM metrics
rfm_df = (
    silver_df
    .groupBy("CustomerID")
    .agg(
        datediff(current_date(), max("InvoiceDate")).alias("recency"),
        count("InvoiceNo").alias("frequency"),
        round(sum(col("Quantity") * col("UnitPrice")), 2).alias("monetary")
    )
)

# Create RFM scores using quartiles
window_spec = Window.orderBy(col("recency").desc())
rfm_with_scores = (
    rfm_df
    .withColumn("r_score", ntile(4).over(Window.orderBy(col("recency"))))
    .withColumn("f_score", ntile(4).over(Window.orderBy(col("frequency").desc())))
    .withColumn("m_score", ntile(4).over(Window.orderBy(col("monetary").desc())))
)

# Create customer segments
rfm_segmented = (
    rfm_with_scores
    .withColumn(
        "segment",
        when((col("r_score") >= 3) & (col("f_score") >= 3) & (col("m_score") >= 3), "Champions")
        .when((col("r_score") >= 2) & (col("f_score") >= 3), "Loyal Customers")
        .when((col("r_score") >= 3) & (col("f_score") <= 2), "Potential Loyalists")
        .when((col("r_score") >= 3) & (col("f_score") == 1), "New Customers")
        .when((col("r_score") <= 2) & (col("f_score") >= 3), "At Risk")
        .when((col("r_score") == 1) & (col("f_score") <= 2), "Hibernating")
        .otherwise("Lost")
    )
)

# Write to Gold
(
    rfm_segmented
    .write
    .mode("overwrite")
    .format("delta")
    .save("Files/gold/rfm_segmentation")
)
```

### Delta Table Optimization

```python
# Optimize Delta tables for better performance
from delta.tables import DeltaTable

# Optimize revenue_trends table
delta_table = DeltaTable.forPath(spark, "Files/gold/revenue_trends")
delta_table.optimize().executeCompaction()

# Vacuum old files (7-day retention)
delta_table.vacuum(168)  # hours

# Z-order optimization for common filters
delta_table.optimize().executeZOrderBy("year", "month")
```

## Creating Semantic Models

### 1. Load Gold Tables as Semantic Model

In Lakehouse:
1. Navigate to **Gold** tables
2. Select tables → **New semantic model**
3. Choose tables: `revenue_trends`, `product_performance`, `customer_analytics`, `rfm_segmentation`

### 2. Define Relationships

```dax
// In Power BI Desktop or Fabric Semantic Model
// Create relationships between tables if needed
// Example: Link customer_analytics to rfm_segmentation on CustomerID
```

### 3. Create Measures

```dax
// Total Revenue
Total Revenue = 
SUM('revenue_trends'[total_revenue])

// Year-over-Year Growth
YoY Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD('revenue_trends'[year], -1, YEAR)
    )
RETURN
DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0)

// Average Order Value
Avg Order Value = 
AVERAGE('revenue_trends'[avg_order_value])

// Customer Lifetime Value
Customer LTV = 
AVERAGE('customer_analytics'[total_spend])

// Top Products
Top 10 Products = 
TOPN(
    10,
    'product_performance',
    'product_performance'[total_revenue],
    DESC
)
```

## Power BI Integration

### Connecting to Semantic Model

```python
# Power BI automatically connects to Fabric semantic models
# No connection string needed - use OneLake integration

# In Power BI Desktop:
# File → Get Data → Power BI semantic models
# Select your Fabric workspace and semantic model
```

### Common DAX Patterns

```dax
// Date Table (if needed)
Date Table = 
ADDCOLUMNS(
    CALENDAR(DATE(2020, 1, 1), DATE(2026, 12, 31)),
    "Year", YEAR([Date]),
    "Month", MONTH([Date]),
    "Quarter", QUARTER([Date]),
    "MonthName", FORMAT([Date], "MMMM")
)

// Running Total
Running Total Revenue = 
CALCULATE(
    [Total Revenue],
    FILTER(
        ALLSELECTED('revenue_trends'[year], 'revenue_trends'[month]),
        'revenue_trends'[year] & 'revenue_trends'[month] <= 
        MAX('revenue_trends'[year]) & MAX('revenue_trends'[month])
    )
)

// Rank Products
Product Rank = 
RANKX(
    ALL('product_performance'[Description]),
    [Total Revenue],,
    DESC,
    Dense
)
```

## Common Patterns

### Pattern 1: Incremental Load

```python
from pyspark.sql.functions import col, max as spark_max
from datetime import datetime, timedelta

# Get last processed date
last_run_df = spark.read.format("delta").load("Files/silver/online_retail_clean")
last_date = last_run_df.select(spark_max("InvoiceDate")).collect()[0][0]

# Read only new data from Bronze
bronze_df = spark.read.format("delta").load("Files/bronze/online_retail")
new_data = bronze_df.filter(col("InvoiceDate") > last_date)

# Process and append to Silver
processed = new_data.transform(cleansing_function)
(
    processed
    .write
    .mode("append")
    .format("delta")
    .save("Files/silver/online_retail_clean")
)
```

### Pattern 2: Data Quality Checks

```python
from pyspark.sql.functions import col, count, when

def data_quality_report(df):
    """Generate data quality metrics"""
    total_rows = df.count()
    
    quality_metrics = df.select(
        count("*").alias("total_records"),
        count(when(col("CustomerID").isNull(), 1)).alias("null_customers"),
        count(when(col("Quantity") <= 0, 1)).alias("invalid_quantity"),
        count(when(col("UnitPrice") <= 0, 1)).alias("invalid_price"),
        count(when(col("Description").isNull(), 1)).alias("null_description")
    )
    
    return quality_metrics

# Run quality checks
quality_report = data_quality_report(bronze_df)
quality_report.show()

# Write quality metrics to monitoring table
(
    quality_report
    .withColumn("check_timestamp", current_timestamp())
    .write
    .mode("append")
    .format("delta")
    .save("Files/monitoring/data_quality_logs")
)
```

### Pattern 3: Parameterized Notebooks

```python
# Define notebook parameters
dbutils.widgets.text("layer", "gold", "Target layer")
dbutils.widgets.text("table_name", "revenue_trends", "Table name")
dbutils.widgets.dropdown("mode", "overwrite", ["overwrite", "append"], "Write mode")

# Get parameter values
layer = dbutils.widgets.get("layer")
table_name = dbutils.widgets.get("table_name")
mode = dbutils.widgets.get("mode")

# Use in processing
output_path = f"Files/{layer}/{table_name}"
df.write.mode(mode).format("delta").save(output_path)
```

### Pattern 4: Error Handling

```python
from pyspark.sql.utils import AnalysisException

def safe_write_delta(df, path, mode="overwrite"):
    """Write DataFrame with error handling"""
    try:
        (
            df
            .write
            .mode(mode)
            .format("delta")
            .option("mergeSchema", "true")
            .save(path)
        )
        print(f"✓ Successfully wrote to {path}")
        return True
    except AnalysisException as e:
        print(f"✗ Analysis error: {str(e)}")
        return False
    except Exception as e:
        print(f"✗ Unexpected error: {str(e)}")
        return False

# Usage
success = safe_write_delta(revenue_trends, "Files/gold/revenue_trends")
```

## Configuration

### Lakehouse Settings

```python
# Configure Spark session for optimization
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")
```

### Delta Table Properties

```python
# Set table properties
spark.sql("""
    ALTER TABLE revenue_trends 
    SET TBLPROPERTIES (
        'delta.autoOptimize.optimizeWrite' = 'true',
        'delta.autoOptimize.autoCompact' = 'true',
        'delta.deletedFileRetentionDuration' = 'interval 7 days'
    )
""")
```

## Troubleshooting

### Issue: "Delta table not found"

```python
# Verify path and permissions
dbutils.fs.ls("Files/gold/")

# Check if table exists
try:
    df = spark.read.format("delta").load("Files/gold/revenue_trends")
    print("Table exists")
except:
    print("Table not found - check path")
```

### Issue: Schema Evolution

```python
# Enable schema merging
(
    df
    .write
    .mode("append")
    .format("delta")
    .option("mergeSchema", "true")  # Allow schema changes
    .save("Files/silver/online_retail_clean")
)
```

### Issue: Performance Problems

```python
# Check table statistics
spark.sql("DESCRIBE EXTENDED revenue_trends").show(truncate=False)

# Optimize and vacuum
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "Files/gold/revenue_trends")
delta_table.optimize().executeCompaction()
delta_table.vacuum(168)  # Remove files older than 7 days
```

### Issue: Memory Errors

```python
# Use partitioning for large datasets
(
    df
    .repartition(10)  # Adjust based on data size
    .write
    .mode("overwrite")
    .format("delta")
    .save("Files/gold/product_performance")
)

# Or use coalesce for writing
df.coalesce(1).write.format("delta").save(path)
```

### Issue: Dataflow Gen2 Refresh Failures

Check:
1. Source connectivity and credentials
2. Lakehouse permissions
3. Data type mismatches
4. Memory limits (reduce batch size)

### Issue: Power BI Refresh Errors

```python
# Ensure semantic model tables are properly registered
spark.sql("REFRESH TABLE revenue_trends")
spark.sql("MSCK REPAIR TABLE product_performance")
```

## Best Practices

1. **Layer Separation**: Keep Bronze (raw), Silver (cleansed), Gold (business) strictly separated
2. **Idempotency**: Ensure notebooks can be re-run safely
3. **Incremental Processing**: Use Delta Lake's MERGE for efficient updates
4. **Monitoring**: Log data quality metrics and processing statistics
5. **Documentation**: Add markdown cells in notebooks explaining business logic
6. **Version Control**: Export notebooks and sync with Git
7. **Security**: Use workspace roles and OneLake security features
8. **Cost Management**: Use auto-pause for compute and optimize storage with vacuum

## Additional Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/fabric/)
- [Delta Lake Documentation](https://docs.delta.io/)
- [PySpark API Reference](https://spark.apache.org/docs/latest/api/python/)
- [Power BI DAX Reference](https://dax.guide/)
