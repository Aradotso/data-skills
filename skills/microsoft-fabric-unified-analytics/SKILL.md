---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering with Microsoft Fabric using Lakehouse, Dataflow Gen2, PySpark notebooks, and Medallion Architecture for unified analytics
triggers:
  - "setup microsoft fabric lakehouse project"
  - "implement medallion architecture bronze silver gold"
  - "create fabric dataflow gen2 transformation"
  - "write pyspark notebook for fabric lakehouse"
  - "configure onelake storage layers"
  - "build semantic model in microsoft fabric"
  - "process data using fabric notebooks"
  - "design unified analytics platform"
---

# Microsoft Fabric Unified Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates how to build a unified analytics platform using **Microsoft Fabric**, implementing the **Medallion Architecture** (Bronze → Silver → Gold) with OneLake storage, Dataflow Gen2 for data integration, Fabric Notebooks with PySpark for transformations, and Power BI for visualization.

Microsoft Fabric consolidates data ingestion, transformation, semantic modeling, and business intelligence into a single SaaS platform, eliminating the complexity of managing multiple disconnected services.

## Architecture

The solution follows a **Lakehouse architecture** with three layers:

- **🟤 Bronze Layer**: Raw source data preservation
- **⚪ Silver Layer**: Cleansed and enriched data
- **🟡 Gold Layer**: Business-ready datasets and KPIs

## Prerequisites

- Microsoft Fabric workspace with Lakehouse enabled
- Microsoft Fabric capacity or trial
- Power BI Premium or Fabric capacity
- Python 3.x (for local development/testing)
- PySpark knowledge for notebook development

## Lakehouse Setup

### Create Lakehouse Structure

In Microsoft Fabric, create a Lakehouse with the following folder structure:

```
lakehouse/
├── Files/
│   ├── bronze/
│   │   └── online_retail/
│   ├── silver/
│   │   └── online_retail/
│   └── gold/
│       ├── revenue_trends/
│       ├── product_performance/
│       ├── customer_analytics/
│       └── rfm_segmentation/
└── Tables/
```

### Load Bronze Data

Upload raw CSV/Parquet files to the `bronze/online_retail/` directory in your Lakehouse Files section. The Bronze layer preserves raw data exactly as received.

## Dataflow Gen2 Configuration

### Create Dataflow for Silver Layer

Dataflow Gen2 provides low-code transformations to clean and enrich Bronze data:

1. **Create New Dataflow Gen2** in your Fabric workspace
2. **Connect to Lakehouse** Bronze layer
3. **Apply Transformations**:
   - Remove duplicates
   - Handle missing values
   - Standardize data types
   - Add business columns

### Example Transformations (Power Query M)

```m
let
    Source = Lakehouse.Contents(null){[workspaceId="YOUR_WORKSPACE_ID"]}[Data],
    BronzeData = Source{[lakehouseId="YOUR_LAKEHOUSE_ID"]}[Data],
    OnlineRetail = BronzeData{[name="bronze/online_retail"]}[Data],
    
    // Remove duplicates
    RemoveDuplicates = Table.Distinct(OnlineRetail),
    
    // Handle missing values
    RemoveNulls = Table.SelectRows(RemoveDuplicates, each [CustomerID] <> null and [Description] <> null),
    
    // Change data types
    TypedData = Table.TransformColumnTypes(RemoveNulls,{
        {"InvoiceNo", type text},
        {"StockCode", type text},
        {"Quantity", Int64.Type},
        {"InvoiceDate", type datetime},
        {"UnitPrice", type number},
        {"CustomerID", type text},
        {"Country", type text}
    }),
    
    // Add business columns
    AddLineTotal = Table.AddColumn(TypedData, "line_total", each [Quantity] * [UnitPrice], type number),
    AddYear = Table.AddColumn(AddLineTotal, "year", each Date.Year([InvoiceDate]), Int64.Type),
    AddMonth = Table.AddColumn(AddYear, "month", each Date.Month([InvoiceDate]), Int64.Type),
    AddIsReturn = Table.AddColumn(AddMonth, "is_return", each if [Quantity] < 0 then true else false, type logical)
in
    AddIsReturn
```

4. **Set Destination** to Lakehouse `silver/online_retail/` as Delta table
5. **Publish and Refresh** the dataflow

## Fabric Notebooks (PySpark)

### Bronze to Silver Processing

Create a Fabric Notebook to validate Bronze data quality:

```python
# Load Bronze data
bronze_df = spark.read.format("delta").load("Files/bronze/online_retail/")

# Data quality checks
print(f"Total records: {bronze_df.count()}")
print(f"Null CustomerID: {bronze_df.filter(col('CustomerID').isNull()).count()}")
print(f"Duplicate records: {bronze_df.count() - bronze_df.dropDuplicates().count()}")

# Display sample
display(bronze_df.limit(10))
```

### Silver to Gold Transformations

Create business-ready datasets in the Gold layer:

#### Revenue Trends Analysis

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, count, avg, year, month, to_date, round

# Initialize Spark session (already available in Fabric Notebooks)
# spark is pre-configured

# Read Silver data
silver_df = spark.read.format("delta").load("Files/silver/online_retail/")

# Filter out returns
sales_df = silver_df.filter(col("is_return") == False)

# Calculate revenue trends
revenue_trends = (
    sales_df
    .groupBy("year", "month")
    .agg(
        sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("transaction_count"),
        avg("line_total").alias("avg_order_value"),
        count(col("CustomerID").distinct()).alias("unique_customers")
    )
    .orderBy("year", "month")
    .withColumn("total_revenue", round(col("total_revenue"), 2))
    .withColumn("avg_order_value", round(col("avg_order_value"), 2))
)

# Write to Gold layer
(
    revenue_trends
    .write
    .format("delta")
    .mode("overwrite")
    .option("overwriteSchema", "true")
    .save("Files/gold/revenue_trends/")
)

print("Revenue trends written to Gold layer")
display(revenue_trends)
```

#### Product Performance Analysis

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import dense_rank

# Product performance metrics
product_performance = (
    sales_df
    .groupBy("StockCode", "Description")
    .agg(
        sum("Quantity").alias("total_quantity_sold"),
        sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("transaction_count"),
        avg("UnitPrice").alias("avg_unit_price"),
        count(col("CustomerID").distinct()).alias("unique_buyers")
    )
    .withColumn("total_revenue", round(col("total_revenue"), 2))
    .withColumn("avg_unit_price", round(col("avg_unit_price"), 2))
)

# Add ranking
window_spec = Window.orderBy(col("total_revenue").desc())
product_ranked = product_performance.withColumn(
    "revenue_rank",
    dense_rank().over(window_spec)
)

# Write to Gold layer
(
    product_ranked
    .write
    .format("delta")
    .mode("overwrite")
    .option("overwriteSchema", "true")
    .save("Files/gold/product_performance/")
)

print("Product performance written to Gold layer")
display(product_ranked.limit(20))
```

#### Customer Analytics

```python
from pyspark.sql.functions import min, max, countDistinct

# Customer behavior analysis
customer_analytics = (
    sales_df
    .groupBy("CustomerID", "Country")
    .agg(
        sum("line_total").alias("total_spent"),
        count("InvoiceNo").alias("total_orders"),
        avg("line_total").alias("avg_order_value"),
        countDistinct("StockCode").alias("unique_products_purchased"),
        min("InvoiceDate").alias("first_purchase_date"),
        max("InvoiceDate").alias("last_purchase_date")
    )
    .withColumn("total_spent", round(col("total_spent"), 2))
    .withColumn("avg_order_value", round(col("avg_order_value"), 2))
)

# Write to Gold layer
(
    customer_analytics
    .write
    .format("delta")
    .mode("overwrite")
    .option("overwriteSchema", "true")
    .save("Files/gold/customer_analytics/")
)

print("Customer analytics written to Gold layer")
display(customer_analytics.limit(20))
```

#### RFM Segmentation

```python
from pyspark.sql.functions import datediff, lit, current_date, ntile
from pyspark.sql.window import Window

# Calculate RFM metrics
reference_date = sales_df.agg(max("InvoiceDate")).collect()[0][0]

rfm_df = (
    sales_df
    .groupBy("CustomerID")
    .agg(
        datediff(lit(reference_date), max("InvoiceDate")).alias("recency"),
        countDistinct("InvoiceNo").alias("frequency"),
        sum("line_total").alias("monetary")
    )
    .withColumn("monetary", round(col("monetary"), 2))
)

# Create RFM scores using quartiles
r_window = Window.orderBy(col("recency"))
f_window = Window.orderBy(col("frequency").desc())
m_window = Window.orderBy(col("monetary").desc())

rfm_scored = (
    rfm_df
    .withColumn("r_score", ntile(4).over(r_window))
    .withColumn("f_score", ntile(4).over(f_window))
    .withColumn("m_score", ntile(4).over(m_window))
)

# Create RFM segment
rfm_segmented = rfm_scored.withColumn(
    "rfm_segment",
    when((col("r_score") >= 3) & (col("f_score") >= 3) & (col("m_score") >= 3), "Champions")
    .when((col("r_score") >= 3) & (col("f_score") >= 2), "Loyal Customers")
    .when((col("r_score") >= 3) & (col("m_score") >= 3), "Big Spenders")
    .when((col("r_score") >= 2) & (col("f_score") >= 2), "Potential Loyalists")
    .when(col("r_score") >= 3, "Recent Customers")
    .when((col("f_score") <= 2) & (col("r_score") <= 2), "At Risk")
    .when(col("r_score") <= 2, "Lost Customers")
    .otherwise("Others")
)

# Write to Gold layer
(
    rfm_segmented
    .write
    .format("delta")
    .mode("overwrite")
    .option("overwriteSchema", "true")
    .save("Files/gold/rfm_segmentation/")
)

print("RFM segmentation written to Gold layer")
display(rfm_segmented.groupBy("rfm_segment").count().orderBy(col("count").desc()))
```

### Register Gold Tables

After writing Gold datasets, register them as Lakehouse tables:

```python
# Register as managed tables for Semantic Model access
spark.sql("CREATE TABLE IF NOT EXISTS gold_revenue_trends USING DELTA LOCATION 'Files/gold/revenue_trends/'")
spark.sql("CREATE TABLE IF NOT EXISTS gold_product_performance USING DELTA LOCATION 'Files/gold/product_performance/'")
spark.sql("CREATE TABLE IF NOT EXISTS gold_customer_analytics USING DELTA LOCATION 'Files/gold/customer_analytics/'")
spark.sql("CREATE TABLE IF NOT EXISTS gold_rfm_segmentation USING DELTA LOCATION 'Files/gold/rfm_segmentation/'")

print("Gold tables registered successfully")
```

## Semantic Model Configuration

### Create Semantic Model

1. In Fabric Lakehouse, navigate to **Reporting** tab
2. Select **New Semantic Model**
3. Choose Gold tables to include
4. Define relationships between tables (if applicable)

### Define Measures (DAX)

Create calculated measures in the Semantic Model:

```dax
Total Revenue = SUM(gold_revenue_trends[total_revenue])

Average Order Value = AVERAGE(gold_revenue_trends[avg_order_value])

Revenue Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = CALCULATE([Total Revenue], DATEADD(gold_revenue_trends[month], -1, MONTH))
RETURN DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0)

Top Product Revenue = CALCULATE(SUM(gold_product_performance[total_revenue]), TOPN(10, gold_product_performance, gold_product_performance[total_revenue], DESC))

Customer Lifetime Value = SUMX(gold_customer_analytics, gold_customer_analytics[total_spent])
```

## Power BI Integration

Power BI automatically connects to the Semantic Model:

1. Create **New Report** from Semantic Model
2. Build visualizations using Gold tables and measures
3. Publish report to Fabric workspace

### Key Visuals

- **Revenue Trend**: Line chart showing revenue over time
- **Product Performance**: Bar chart of top products by revenue
- **Customer Segments**: Pie chart showing RFM distribution
- **KPI Cards**: Total revenue, customer count, avg order value

## Common Patterns

### Incremental Data Loading

```python
from delta.tables import DeltaTable

# Read existing Gold table
existing_table = DeltaTable.forPath(spark, "Files/gold/revenue_trends/")

# Merge new data (upsert pattern)
(
    existing_table.alias("existing")
    .merge(
        new_revenue_trends.alias("new"),
        "existing.year = new.year AND existing.month = new.month"
    )
    .whenMatchedUpdateAll()
    .whenNotMatchedInsertAll()
    .execute()
)
```

### Data Quality Validation

```python
def validate_data_quality(df, table_name):
    """Validate data quality before writing to Gold"""
    null_counts = df.select([count(when(col(c).isNull(), c)).alias(c) for c in df.columns])
    
    print(f"Data Quality Report for {table_name}")
    print(f"Total rows: {df.count()}")
    print("Null counts:")
    null_counts.show()
    
    # Check for critical nulls
    critical_columns = ["CustomerID", "InvoiceDate", "line_total"]
    for col_name in critical_columns:
        null_count = df.filter(col(col_name).isNull()).count()
        if null_count > 0:
            print(f"WARNING: {null_count} null values in critical column {col_name}")

# Use before writing to Gold
validate_data_quality(revenue_trends, "revenue_trends")
```

### Environment-Specific Configuration

```python
# Use Fabric workspace context
import os

# Get workspace and lakehouse from environment
WORKSPACE_ID = os.getenv("FABRIC_WORKSPACE_ID")
LAKEHOUSE_ID = os.getenv("FABRIC_LAKEHOUSE_ID")

# Dynamic path construction
def get_layer_path(layer, dataset):
    """Generate lakehouse path based on layer"""
    return f"Files/{layer}/{dataset}/"

# Usage
silver_path = get_layer_path("silver", "online_retail")
gold_path = get_layer_path("gold", "revenue_trends")
```

## Troubleshooting

### Dataflow Gen2 Not Refreshing

**Issue**: Dataflow fails to load data from Bronze layer

**Solution**: 
- Verify Lakehouse connection credentials
- Check file path and format in Bronze layer
- Ensure Fabric capacity has sufficient resources
- Review refresh history for specific error messages

### PySpark Notebook Timeout

**Issue**: Notebook times out during large data processing

**Solution**:
```python
# Optimize with partition pruning
df = spark.read.format("delta").load("Files/silver/online_retail/") \
    .filter(col("year") == 2023)  # Partition filter

# Cache frequently accessed data
df.cache()

# Repartition for better parallelism
df = df.repartition(8, "CustomerID")
```

### Delta Table Schema Conflicts

**Issue**: Schema mismatch when writing to existing Delta table

**Solution**:
```python
# Option 1: Overwrite with schema merge
df.write.format("delta") \
    .mode("overwrite") \
    .option("mergeSchema", "true") \
    .save("Files/gold/revenue_trends/")

# Option 2: Explicit schema evolution
df.write.format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Files/gold/revenue_trends/")
```

### Semantic Model Not Showing Tables

**Issue**: Gold tables not appearing in Semantic Model

**Solution**:
- Ensure tables are registered in Lakehouse SQL endpoint
- Refresh Semantic Model metadata
- Check table permissions in workspace
- Verify tables are properly formatted as Delta tables

### Power BI Report Performance

**Issue**: Slow report rendering with large datasets

**Solution**:
- Use DirectLake mode for optimal performance
- Create aggregated tables in Gold layer for summary views
- Define proper relationships in Semantic Model
- Use Import mode for smaller, frequently accessed data

## Best Practices

1. **Medallion Layers**: Always preserve raw data in Bronze, clean in Silver, and aggregate in Gold
2. **Delta Format**: Use Delta Lake for ACID transactions and time travel
3. **Incremental Processing**: Implement merge patterns for efficient updates
4. **Data Quality**: Validate data at each layer transition
5. **Partitioning**: Partition large tables by date for query performance
6. **Documentation**: Document transformations and business logic in notebooks
7. **Version Control**: Export notebooks and track changes externally
8. **Workspace Organization**: Use separate workspaces for dev/test/prod environments

## Additional Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric/)
- [Dataflow Gen2 Guide](https://learn.microsoft.com/en-us/fabric/data-factory/dataflows-gen2-overview)
- [Delta Lake on Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-optimization-and-v-order)
- [PySpark API Reference](https://spark.apache.org/docs/latest/api/python/)
