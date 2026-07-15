---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering with Microsoft Fabric, Lakehouse, Dataflow Gen2, PySpark, and Power BI using Medallion Architecture
triggers:
  - "set up microsoft fabric lakehouse"
  - "implement medallion architecture in fabric"
  - "create dataflow gen2 transformations"
  - "build pyspark notebooks for fabric"
  - "configure semantic model in fabric"
  - "design unified analytics platform"
  - "process data with fabric notebooks"
  - "create gold layer datasets"
---

# Microsoft Fabric Unified Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end unified analytics platforms using **Microsoft Fabric**, implementing the **Medallion Architecture** (Bronze → Silver → Gold) with OneLake, Dataflow Gen2, PySpark notebooks, Semantic Models, and Power BI.

## What This Project Does

The Microsoft Fabric Unified Analytics Platform demonstrates:

- **Unified SaaS Analytics**: Single platform for ingestion, transformation, modeling, and visualization
- **Lakehouse Architecture**: OneLake storage with Bronze/Silver/Gold layers
- **Dataflow Gen2**: Low-code data transformation and quality improvement
- **PySpark Processing**: Business logic implementation in Fabric Notebooks
- **Semantic Modeling**: Centralized business metrics and KPIs
- **Power BI Integration**: Native visualization on trusted data

**Key Use Case**: Retail analytics transforming raw transaction data into business insights through progressive data refinement.

## Architecture Overview

```
Raw Data → Bronze Layer → Dataflow Gen2 → Silver Layer → 
Fabric Notebook (PySpark) → Gold Layer → Semantic Model → Power BI
```

**Layers**:
- 🟤 **Bronze**: Raw source data preservation
- ⚪ **Silver**: Cleansed and enriched data
- 🟡 **Gold**: Business-ready analytical datasets

## Setting Up Microsoft Fabric

### Prerequisites

1. Microsoft Fabric capacity (F2 or higher recommended)
2. Microsoft Fabric workspace
3. Power BI Pro or Premium license
4. Source data (CSV, JSON, or database connections)

### Initial Lakehouse Setup

```python
# Create Lakehouse structure in Fabric UI:
# 1. Navigate to your Fabric workspace
# 2. Click "+ New" → Lakehouse
# 3. Name: "RetailAnalyticsLH"
# 4. Create folders: bronze/, silver/, gold/

# Folder structure:
# lakehouse/
#   Files/
#     bronze/
#       sales/
#       products/
#       customers/
#     silver/
#       sales_cleaned/
#     gold/
#       revenue_trends/
#       customer_segments/
#       product_performance/
```

## Bronze Layer: Data Ingestion

### Load Raw Data to Bronze

```python
# Fabric Notebook - Bronze data ingestion
from pyspark.sql import SparkSession
from pyspark.sql.functions import current_timestamp

# Initialize Spark session (automatic in Fabric)
spark = spark

# Read raw CSV data
raw_sales_df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("Files/raw/sales.csv")

# Add ingestion metadata
bronze_df = raw_sales_df.withColumn("ingestion_timestamp", current_timestamp())

# Write to Bronze layer (Delta format)
bronze_df.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Files/bronze/sales/")

print(f"Bronze layer loaded: {bronze_df.count()} records")
```

### Verify Bronze Layer

```python
# Read Bronze data
bronze_sales = spark.read.format("delta").load("Files/bronze/sales/")

# Display schema and sample
bronze_sales.printSchema()
display(bronze_sales.limit(10))

# Data quality checks
print(f"Total records: {bronze_sales.count()}")
print(f"Null values per column:")
bronze_sales.select([count(when(col(c).isNull(), c)).alias(c) for c in bronze_sales.columns]).show()
```

## Silver Layer: Data Transformation with Dataflow Gen2

### Dataflow Gen2 Transformations

**Via Fabric UI**:

1. Create Dataflow Gen2:
   - Navigate to workspace → "+ New" → Dataflow Gen2
   - Name: "SalesDataCleansing"

2. Connect to Bronze Layer:
   - Add data source → OneLake → Select Bronze table
   - Import data

3. Apply Transformations:
   - **Remove Duplicates**: Based on InvoiceNo + StockCode
   - **Handle Missing Values**: Remove rows with null CustomerID
   - **Change Data Types**: Ensure correct types (Date, Decimal, Integer)
   - **Add Calculated Columns**:
     ```m
     // Power Query M - Add line_total
     = Table.AddColumn(#"Previous Step", "line_total", 
         each [Quantity] * [UnitPrice])
     
     // Add year and month
     = Table.AddColumn(#"Previous Step", "year", 
         each Date.Year([InvoiceDate]))
     = Table.AddColumn(#"Previous Step", "month", 
         each Date.Month([InvoiceDate]))
     
     // Add is_return flag
     = Table.AddColumn(#"Previous Step", "is_return", 
         each if Text.StartsWith([InvoiceNo], "C") then true else false)
     ```

4. Set Destination:
   - Configure data destination → Lakehouse
   - Target: "Files/silver/sales_cleaned/"
   - Save and Publish

### Read Silver Data in Notebook

```python
# Read transformed Silver data
silver_sales = spark.read.format("delta").load("Files/silver/sales_cleaned/")

# Verify transformations
print(f"Silver records: {silver_sales.count()}")
print(f"Columns: {silver_sales.columns}")

# Check data quality
display(silver_sales.describe())
```

## Gold Layer: Business Logic with PySpark

### Revenue Trends Analysis

```python
from pyspark.sql.functions import sum, col, year, month, dayofmonth, to_date

# Read Silver data
sales_df = spark.read.format("delta").load("Files/silver/sales_cleaned/")

# Calculate daily revenue trends
revenue_trends = sales_df \
    .filter(col("is_return") == False) \
    .groupBy(
        to_date("InvoiceDate").alias("date"),
        year("InvoiceDate").alias("year"),
        month("InvoiceDate").alias("month")
    ) \
    .agg(
        sum("line_total").alias("total_revenue"),
        sum("Quantity").alias("total_quantity"),
        count("InvoiceNo").alias("transaction_count"),
        countDistinct("CustomerID").alias("unique_customers")
    ) \
    .orderBy("date")

# Write to Gold layer
revenue_trends.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Files/gold/revenue_trends/")

print(f"Revenue trends created: {revenue_trends.count()} records")
display(revenue_trends.limit(20))
```

### Product Performance Analysis

```python
from pyspark.sql.functions import round, avg

# Product performance metrics
product_performance = sales_df \
    .filter(col("is_return") == False) \
    .groupBy("StockCode", "Description") \
    .agg(
        sum("line_total").alias("total_revenue"),
        sum("Quantity").alias("total_quantity_sold"),
        countDistinct("InvoiceNo").alias("transaction_count"),
        countDistinct("CustomerID").alias("unique_customers"),
        avg("UnitPrice").alias("avg_unit_price")
    ) \
    .withColumn("avg_transaction_value", 
                round(col("total_revenue") / col("transaction_count"), 2)) \
    .orderBy(col("total_revenue").desc())

# Save to Gold layer
product_performance.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Files/gold/product_performance/")

display(product_performance.limit(20))
```

### Customer Segmentation (RFM Analysis)

```python
from pyspark.sql.functions import datediff, lit, max as spark_max, min as spark_min
from pyspark.sql.window import Window

# Calculate RFM metrics
reference_date = sales_df.agg(spark_max("InvoiceDate")).collect()[0][0]

customer_rfm = sales_df \
    .filter(col("is_return") == False) \
    .groupBy("CustomerID") \
    .agg(
        datediff(lit(reference_date), spark_max("InvoiceDate")).alias("recency"),
        countDistinct("InvoiceNo").alias("frequency"),
        sum("line_total").alias("monetary")
    )

# Create RFM scores (1-5 scale)
from pyspark.sql.functions import percent_rank

windowSpec = Window.orderBy("recency")
rfm_scored = customer_rfm \
    .withColumn("r_rank", percent_rank().over(windowSpec)) \
    .withColumn("r_score", 
                when(col("r_rank") <= 0.2, 5)
                .when(col("r_rank") <= 0.4, 4)
                .when(col("r_rank") <= 0.6, 3)
                .when(col("r_rank") <= 0.8, 2)
                .otherwise(1))

windowSpec = Window.orderBy(col("frequency").desc())
rfm_scored = rfm_scored \
    .withColumn("f_rank", percent_rank().over(windowSpec)) \
    .withColumn("f_score", 
                when(col("f_rank") <= 0.2, 5)
                .when(col("f_rank") <= 0.4, 4)
                .when(col("f_rank") <= 0.6, 3)
                .when(col("f_rank") <= 0.8, 2)
                .otherwise(1))

windowSpec = Window.orderBy(col("monetary").desc())
rfm_scored = rfm_scored \
    .withColumn("m_rank", percent_rank().over(windowSpec)) \
    .withColumn("m_score", 
                when(col("m_rank") <= 0.2, 5)
                .when(col("m_rank") <= 0.4, 4)
                .when(col("m_rank") <= 0.6, 3)
                .when(col("m_rank") <= 0.8, 2)
                .otherwise(1))

# Create customer segments
rfm_segments = rfm_scored \
    .withColumn("rfm_score", col("r_score") + col("f_score") + col("m_score")) \
    .withColumn("customer_segment",
                when(col("rfm_score") >= 13, "Champions")
                .when(col("rfm_score") >= 10, "Loyal Customers")
                .when(col("rfm_score") >= 7, "Potential Loyalists")
                .when(col("rfm_score") >= 5, "At Risk")
                .otherwise("Lost")) \
    .select("CustomerID", "recency", "frequency", "monetary", 
            "r_score", "f_score", "m_score", "rfm_score", "customer_segment")

# Save to Gold layer
rfm_segments.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Files/gold/customer_segments/")

print(f"Customer segments created: {rfm_segments.count()} customers")
display(rfm_segments.groupBy("customer_segment").count().orderBy(col("count").desc()))
```

### Executive KPI Summary

```python
from datetime import datetime

# Calculate key business metrics
total_revenue = sales_df.filter(col("is_return") == False) \
    .agg(sum("line_total")).collect()[0][0]

total_orders = sales_df.filter(col("is_return") == False) \
    .select("InvoiceNo").distinct().count()

total_customers = sales_df.select("CustomerID").distinct().count()

avg_order_value = total_revenue / total_orders if total_orders > 0 else 0

# Create KPI summary
kpi_data = [(
    datetime.now(),
    round(total_revenue, 2),
    total_orders,
    total_customers,
    round(avg_order_value, 2)
)]

kpi_schema = ["report_date", "total_revenue", "total_orders", 
              "total_customers", "avg_order_value"]

kpi_df = spark.createDataFrame(kpi_data, kpi_schema)

# Save to Gold layer
kpi_df.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Files/gold/executive_kpis/")

display(kpi_df)
```

## Semantic Model Configuration

### Create Semantic Model

1. In Fabric workspace → Select Lakehouse → New Semantic Model
2. Select Gold layer tables:
   - `revenue_trends`
   - `product_performance`
   - `customer_segments`
   - `executive_kpis`

### Define Relationships (DAX)

```dax
// Create Calendar table
Calendar = 
ADDCOLUMNS(
    CALENDARAUTO(),
    "Year", YEAR([Date]),
    "Month", MONTH([Date]),
    "MonthName", FORMAT([Date], "MMMM"),
    "Quarter", "Q" & QUARTER([Date]),
    "YearMonth", FORMAT([Date], "YYYY-MM")
)

// Relationship: Calendar[Date] → revenue_trends[date]
```

### Create Measures

```dax
// Total Revenue
Total Revenue = 
SUM(revenue_trends[total_revenue])

// Revenue Growth MoM
Revenue Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD(Calendar[Date], -1, MONTH)
    )
RETURN
DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0)

// Average Order Value
Average Order Value = 
DIVIDE(
    SUM(revenue_trends[total_revenue]),
    SUM(revenue_trends[transaction_count]),
    0
)

// Customer Count
Total Customers = 
DISTINCTCOUNT(customer_segments[CustomerID])

// Top Products by Revenue
Top 10 Products Revenue = 
CALCULATE(
    SUM(product_performance[total_revenue]),
    TOPN(10, ALL(product_performance), product_performance[total_revenue], DESC)
)
```

## Power BI Report Development

### Connect to Semantic Model

```text
1. Open Power BI Desktop
2. Get Data → Power BI Semantic Models
3. Select your Fabric workspace → Select Semantic Model
4. Load data
```

### Sample Visualizations

**Revenue Dashboard Page**:
- Card: Total Revenue
- Card: Revenue Growth %
- Line Chart: Revenue Trends over Time (Calendar[Date], [Total Revenue])
- Bar Chart: Top 10 Products (product_performance[Description], [Total Revenue])

**Customer Analytics Page**:
- Donut Chart: Customer Segments (customer_segments[customer_segment], Count)
- Table: RFM Analysis (CustomerID, recency, frequency, monetary, customer_segment)
- Card: Total Customers

## Common Patterns

### Delta Lake Optimization

```python
# Optimize Delta tables for better query performance
from delta.tables import DeltaTable

# Optimize Gold layer table
deltaTable = DeltaTable.forPath(spark, "Files/gold/revenue_trends/")
deltaTable.optimize().executeCompaction()

# Vacuum old versions (7 days retention)
deltaTable.vacuum(168)  # hours

print("Delta optimization complete")
```

### Incremental Data Processing

```python
from pyspark.sql.functions import lit

# Track last processed timestamp
last_processed = spark.read.format("delta") \
    .load("Files/silver/sales_cleaned/") \
    .agg(spark_max("ingestion_timestamp")) \
    .collect()[0][0]

# Process only new records
new_records = spark.read.format("delta") \
    .load("Files/bronze/sales/") \
    .filter(col("ingestion_timestamp") > lit(last_processed))

if new_records.count() > 0:
    # Apply transformations and append
    processed = new_records  # Apply your transformations here
    
    processed.write \
        .format("delta") \
        .mode("append") \
        .save("Files/silver/sales_cleaned/")
    
    print(f"Processed {new_records.count()} new records")
else:
    print("No new records to process")
```

### Error Handling and Logging

```python
import logging
from datetime import datetime

# Configure logging
logger = logging.getLogger(__name__)

def process_with_error_handling():
    try:
        # Read source data
        source_df = spark.read.format("delta").load("Files/bronze/sales/")
        
        # Validation checks
        if source_df.count() == 0:
            raise ValueError("Source data is empty")
        
        # Process data
        result_df = source_df  # Your transformations here
        
        # Write results
        result_df.write.format("delta").mode("overwrite").save("Files/gold/output/")
        
        # Log success
        log_entry = [(datetime.now(), "SUCCESS", "Processing completed", result_df.count())]
        log_df = spark.createDataFrame(log_entry, ["timestamp", "status", "message", "records"])
        log_df.write.format("delta").mode("append").save("Files/logs/processing_log/")
        
        print("✓ Processing successful")
        
    except Exception as e:
        # Log error
        log_entry = [(datetime.now(), "ERROR", str(e), 0)]
        log_df = spark.createDataFrame(log_entry, ["timestamp", "status", "message", "records"])
        log_df.write.format("delta").mode("append").save("Files/logs/processing_log/")
        
        print(f"✗ Processing failed: {str(e)}")
        raise

process_with_error_handling()
```

## Troubleshooting

### Dataflow Gen2 Issues

**Problem**: Dataflow fails to refresh

```text
Solution:
1. Check source connectivity (Bronze layer accessible)
2. Verify data types in transformations
3. Check memory limits (split large datasets)
4. Review Power Query diagnostics
5. Ensure destination Lakehouse has write permissions
```

### PySpark Performance Issues

```python
# Enable adaptive query execution
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")

# Cache frequently accessed data
frequently_used_df = spark.read.format("delta").load("Files/silver/sales_cleaned/")
frequently_used_df.cache()

# Repartition for better parallelism
optimized_df = frequently_used_df.repartition(200, "CustomerID")

# Use broadcast for small dimension tables
from pyspark.sql.functions import broadcast
small_table = spark.read.format("delta").load("Files/silver/products/")
result = large_table.join(broadcast(small_table), "StockCode")
```

### Delta Lake Schema Evolution

```python
# Enable schema evolution for Delta writes
df_with_new_column = existing_df.withColumn("new_field", lit(None))

df_with_new_column.write \
    .format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .save("Files/gold/revenue_trends/")

# Or overwrite with schema evolution
df_with_new_column.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Files/gold/revenue_trends/")
```

### Semantic Model Refresh Issues

```text
Problem: Semantic model not updating with latest data

Solution:
1. Verify Gold layer Delta tables have new data
2. Refresh Semantic Model in Fabric (Settings → Refresh Now)
3. Check gateway connection if using on-premises data
4. Configure scheduled refresh (Settings → Scheduled Refresh)
5. Review refresh history for errors
```

### Workspace Capacity Management

```python
# Monitor Lakehouse storage
def check_storage_usage():
    from notebookutils import mssparkutils
    
    # List all Delta tables
    tables = mssparkutils.fs.ls("Files/")
    
    for table in tables:
        if table.isDir:
            files = mssparkutils.fs.ls(table.path)
            total_size = sum([f.size for f in files])
            print(f"{table.name}: {total_size / (1024**3):.2f} GB")

check_storage_usage()
```

## Best Practices

1. **Medallion Architecture**: Always maintain clear separation between Bronze/Silver/Gold
2. **Delta Format**: Use Delta Lake for all Lakehouse tables (ACID transactions, time travel)
3. **Schema Validation**: Enforce schemas in Bronze layer to catch data quality issues early
4. **Incremental Processing**: Implement watermarking for large datasets
5. **Semantic Layer**: Define all business metrics in Semantic Model, not in Power BI
6. **Documentation**: Comment complex transformations in notebooks
7. **Version Control**: Export notebooks and Dataflows for version control
8. **Monitoring**: Implement logging and alerting for pipeline failures

## Environment Configuration

Store sensitive configuration in environment variables or Azure Key Vault:

```python
# Reference environment variables
from notebookutils import mssparkutils

# Get secrets from Key Vault (if configured)
api_key = mssparkutils.credentials.getSecret("your-keyvault", "api-key")
connection_string = mssparkutils.credentials.getSecret("your-keyvault", "connection-string")

# Use in connections
df = spark.read \
    .format("jdbc") \
    .option("url", connection_string) \
    .option("dbtable", "source_table") \
    .load()
```

This skill provides AI agents with comprehensive knowledge to build unified analytics platforms using Microsoft Fabric's integrated components following industry best practices.
