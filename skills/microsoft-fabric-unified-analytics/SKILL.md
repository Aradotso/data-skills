---
name: microsoft-fabric-unified-analytics
description: End-to-end unified analytics platform using Microsoft Fabric with Lakehouse, Medallion Architecture, Dataflow Gen2, PySpark notebooks, and Power BI for retail analytics
triggers:
  - "how do I build a unified analytics platform with Microsoft Fabric"
  - "set up medallion architecture in microsoft fabric lakehouse"
  - "create dataflow gen2 transformations in fabric"
  - "process data with pyspark in fabric notebooks"
  - "build semantic models in microsoft fabric"
  - "design bronze silver gold layers in onelake"
  - "implement retail analytics with fabric and power bi"
  - "transform data from bronze to gold in fabric"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates building an end-to-end unified analytics platform using Microsoft Fabric with Lakehouse architecture following the Medallion pattern (Bronze → Silver → Gold). It covers data ingestion, transformation with Dataflow Gen2 and PySpark, semantic modeling, and Power BI visualization for retail analytics scenarios.

## What This Project Does

- **Unified SaaS Platform**: Single platform for ingestion, storage, transformation, and visualization
- **Lakehouse Architecture**: OneLake-based storage with Delta Lake format
- **Medallion Layers**: Progressive data refinement from raw (Bronze) to cleansed (Silver) to business-ready (Gold)
- **Low-Code & Code Integration**: Dataflow Gen2 for transformations + PySpark notebooks for business logic
- **Semantic Layer**: Centralized business metrics and KPIs
- **Native Power BI**: Direct integration for analytics and dashboards

## Prerequisites

- Microsoft Fabric capacity (F64 or higher recommended)
- Power BI Pro or Premium license
- Azure account (for Fabric workspace)
- Basic knowledge of PySpark and Power Query M
- Understanding of dimensional modeling

## Lakehouse Setup

### Creating a Lakehouse

```python
# Lakehouse is created through Microsoft Fabric UI
# Structure your Lakehouse with Medallion layers:
# /Files/bronze/     - Raw source data
# /Files/silver/     - Cleansed data
# /Files/gold/       - Business-ready datasets
# /Tables/           - Delta tables for analytics
```

### Bronze Layer - Raw Data Ingestion

Store raw CSV/Parquet files in Bronze layer preserving source format:

```
bronze/
├── orders/
│   └── online_retail.csv
├── customers/
│   └── customer_data.csv
└── products/
    └── product_catalog.csv
```

## Dataflow Gen2 Transformations

### Bronze to Silver Pipeline

Dataflow Gen2 applies low-code transformations to cleanse and enrich data:

**Key Transformations:**
- Remove duplicates
- Handle missing values
- Standardize data types
- Add business attributes

**Power Query M Example (in Dataflow Gen2):**

```m
let
    Source = Lakehouse.Contents(null),
    BronzeData = Source{[workspaceId="YOUR_WORKSPACE_ID"]}[Data],
    Orders = BronzeData{[lakehouseTable="bronze.orders"]}[Data],
    
    // Remove duplicates
    RemoveDuplicates = Table.Distinct(Orders, {"InvoiceNo", "StockCode"}),
    
    // Filter invalid records
    FilteredRows = Table.SelectRows(RemoveDuplicates, 
        each [Quantity] > 0 and [UnitPrice] > 0),
    
    // Add calculated columns
    AddLineTotal = Table.AddColumn(FilteredRows, "line_total", 
        each [Quantity] * [UnitPrice], type number),
    
    // Add date dimensions
    AddYear = Table.AddColumn(AddLineTotal, "year", 
        each Date.Year([InvoiceDate]), Int64.Type),
    AddMonth = Table.AddColumn(AddYear, "month", 
        each Date.Month([InvoiceDate]), Int64.Type),
    
    // Identify returns
    AddIsReturn = Table.AddColumn(AddMonth, "is_return", 
        each if Text.StartsWith([InvoiceNo], "C") then true else false, 
        type logical),
    
    // Change data types
    ChangeTypes = Table.TransformColumnTypes(AddIsReturn,{
        {"InvoiceNo", type text},
        {"StockCode", type text},
        {"Quantity", Int64.Type},
        {"InvoiceDate", type datetime},
        {"UnitPrice", type number},
        {"CustomerID", type text},
        {"Country", type text}
    })
in
    ChangeTypes
```

### Publishing to Silver Layer

Configure Dataflow Gen2 destination:
- **Destination Type**: Lakehouse
- **Table Name**: `silver.orders_cleansed`
- **Update Method**: Replace or Append
- **Format**: Delta

## PySpark Notebooks - Silver to Gold

### Notebook Configuration

```python
# Fabric Notebook automatically connects to attached Lakehouse
# Import required libraries
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.window import Window
from datetime import datetime

# Spark session is pre-configured in Fabric
spark
```

### Reading Silver Data

```python
# Read cleansed data from Silver layer
df_orders = spark.read.format("delta").load("Tables/silver.orders_cleansed")

# Display schema
df_orders.printSchema()

# Preview data
display(df_orders.limit(10))
```

### Gold Layer - Revenue Analytics

```python
# Create revenue summary by month
df_revenue_monthly = df_orders \
    .filter(col("is_return") == False) \
    .groupBy("year", "month") \
    .agg(
        sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("total_orders"),
        countDistinct("CustomerID").alias("unique_customers")
    ) \
    .orderBy("year", "month")

# Write to Gold layer as Delta table
df_revenue_monthly.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable("gold.revenue_monthly")

print("✅ Gold table created: gold.revenue_monthly")
```

### Product Performance Analysis

```python
# Product-level metrics
df_product_performance = df_orders \
    .filter(col("is_return") == False) \
    .groupBy("StockCode", "Description") \
    .agg(
        sum("line_total").alias("total_revenue"),
        sum("Quantity").alias("total_quantity_sold"),
        count("InvoiceNo").alias("order_count"),
        countDistinct("CustomerID").alias("unique_customers"),
        avg("UnitPrice").alias("avg_unit_price")
    ) \
    .withColumn("revenue_rank", 
                rank().over(Window.orderBy(desc("total_revenue"))))

# Save to Gold
df_product_performance.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold.product_performance")
```

### Customer Segmentation (RFM Analysis)

```python
# Calculate Recency, Frequency, Monetary values
from pyspark.sql.types import IntegerType

# Reference date for recency calculation
reference_date = df_orders.agg(max("InvoiceDate")).collect()[0][0]

df_rfm = df_orders \
    .filter(col("is_return") == False) \
    .groupBy("CustomerID") \
    .agg(
        datediff(lit(reference_date), max("InvoiceDate")).alias("recency"),
        countDistinct("InvoiceNo").alias("frequency"),
        sum("line_total").alias("monetary")
    )

# Create RFM segments using quartiles
df_rfm_segments = df_rfm \
    .withColumn("R_score", ntile(4).over(Window.orderBy(col("recency")))) \
    .withColumn("F_score", ntile(4).over(Window.orderBy(desc("frequency")))) \
    .withColumn("M_score", ntile(4).over(Window.orderBy(desc("monetary")))) \
    .withColumn("RFM_score", 
                concat(col("R_score"), col("F_score"), col("M_score")))

# Add customer segment labels
df_rfm_final = df_rfm_segments \
    .withColumn("customer_segment",
        when((col("R_score") >= 3) & (col("F_score") >= 3) & (col("M_score") >= 3), 
             "Champions")
        .when((col("R_score") >= 3) & (col("F_score") >= 2), "Loyal Customers")
        .when((col("R_score") >= 3) & (col("M_score") >= 3), "Big Spenders")
        .when(col("R_score") >= 3, "Promising")
        .when((col("F_score") >= 3) & (col("M_score") >= 3), "At Risk")
        .when(col("R_score") <= 2, "Lost")
        .otherwise("Others")
    )

# Save RFM analysis to Gold
df_rfm_final.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold.customer_rfm_segments")
```

### Customer Metrics Summary

```python
# Customer-level aggregated metrics
df_customer_metrics = df_orders \
    .filter(col("is_return") == False) \
    .groupBy("CustomerID", "Country") \
    .agg(
        sum("line_total").alias("lifetime_value"),
        count("InvoiceNo").alias("total_orders"),
        avg("line_total").alias("avg_order_value"),
        min("InvoiceDate").alias("first_purchase_date"),
        max("InvoiceDate").alias("last_purchase_date")
    ) \
    .withColumn("customer_tenure_days",
                datediff(col("last_purchase_date"), col("first_purchase_date")))

# Save to Gold
df_customer_metrics.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold.customer_metrics")
```

### KPI Dashboard Data

```python
# Executive KPIs
df_kpis = df_orders \
    .filter(col("is_return") == False) \
    .agg(
        sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("total_orders"),
        countDistinct("CustomerID").alias("total_customers"),
        countDistinct("StockCode").alias("total_products"),
        avg("line_total").alias("avg_order_value")
    )

# Add calculated metrics
df_kpis = df_kpis \
    .withColumn("revenue_per_customer", 
                col("total_revenue") / col("total_customers")) \
    .withColumn("calculation_date", current_timestamp())

# Save to Gold
df_kpis.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold.executive_kpis")

display(df_kpis)
```

## Semantic Model Configuration

### Creating Semantic Model

1. Navigate to your Lakehouse in Fabric
2. Select **New Semantic Model**
3. Choose Gold layer tables:
   - `gold.revenue_monthly`
   - `gold.product_performance`
   - `gold.customer_rfm_segments`
   - `gold.customer_metrics`
   - `gold.executive_kpis`

### Defining Relationships (DAX)

```dax
// Create relationships in Model View
// customer_metrics[CustomerID] → customer_rfm_segments[CustomerID]
// One-to-One relationship
```

### Creating Measures

```dax
// Total Revenue
Total Revenue = SUM(revenue_monthly[total_revenue])

// Revenue Growth MoM
Revenue Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD(revenue_monthly[InvoiceDate], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0)

// Customer Lifetime Value
Avg Customer LTV = AVERAGE(customer_metrics[lifetime_value])

// Repeat Customer Rate
Repeat Customer Rate = 
DIVIDE(
    COUNTROWS(FILTER(customer_metrics, customer_metrics[total_orders] > 1)),
    COUNTROWS(customer_metrics),
    0
)

// Top 10 Products Revenue
Top 10 Products Revenue = 
CALCULATE(
    SUM(product_performance[total_revenue]),
    TOPN(10, product_performance, product_performance[total_revenue], DESC)
)
```

### Calculated Columns

```dax
// In customer_metrics table
Customer Tier = 
SWITCH(
    TRUE(),
    customer_metrics[lifetime_value] >= 10000, "Platinum",
    customer_metrics[lifetime_value] >= 5000, "Gold",
    customer_metrics[lifetime_value] >= 1000, "Silver",
    "Bronze"
)

// In revenue_monthly table
Month Name = FORMAT(DATE(revenue_monthly[year], revenue_monthly[month], 1), "MMM")
```

## Power BI Integration

### Connecting to Semantic Model

```python
# Power BI automatically connects to Fabric Semantic Models
# No separate connection configuration needed
```

**In Power BI Desktop:**
1. Get Data → Power BI Semantic Models
2. Select your Fabric workspace
3. Choose the semantic model
4. Select required tables/measures

### Example Report Structure

**Page 1: Executive Dashboard**
- Card visuals for KPIs (Total Revenue, Customers, Orders)
- Line chart: Revenue trend by month
- Donut chart: Revenue by customer segment
- Table: Top 10 products by revenue

**Page 2: Customer Analytics**
- Matrix: RFM segments with counts and metrics
- Scatter plot: Frequency vs Monetary with Recency color
- Bar chart: Customer distribution by country
- Card: Repeat customer rate

**Page 3: Product Performance**
- Table: Product performance ranked by revenue
- Treemap: Products by category and revenue
- Column chart: Top products by quantity sold

## Orchestration with Fabric Pipelines

### Creating a Pipeline

```python
# Fabric Pipelines are created through the UI
# Example pipeline structure:

# 1. Trigger: Scheduled (daily at 2 AM)
# 2. Activity 1: Run Dataflow Gen2 (Bronze → Silver)
# 3. Activity 2: Run Notebook (Silver → Gold transformations)
# 4. Activity 3: Refresh Semantic Model
# 5. Notification: Send email on completion/failure
```

### Pipeline Configuration

**Schedule Trigger:**
```json
{
  "type": "ScheduleTrigger",
  "recurrence": {
    "frequency": "Day",
    "interval": 1,
    "startTime": "2024-01-01T02:00:00Z",
    "timeZone": "UTC"
  }
}
```

## Delta Lake Optimization

### Optimize Tables

```python
# Optimize Delta tables for better performance
spark.sql("OPTIMIZE gold.revenue_monthly")
spark.sql("OPTIMIZE gold.product_performance")
spark.sql("OPTIMIZE gold.customer_rfm_segments")

# Z-order optimization for frequently filtered columns
spark.sql("OPTIMIZE gold.product_performance ZORDER BY (StockCode)")
spark.sql("OPTIMIZE gold.customer_metrics ZORDER BY (CustomerID, Country)")
```

### Vacuum Old Versions

```python
# Remove old file versions (retention: 7 days)
spark.sql("VACUUM gold.revenue_monthly RETAIN 168 HOURS")
```

## Common Patterns

### Incremental Loading

```python
# Read only new data since last load
from delta.tables import DeltaTable

# Get last processed timestamp
last_timestamp = spark.sql("""
    SELECT MAX(last_modified) as max_ts 
    FROM gold.metadata_log 
    WHERE table_name = 'orders'
""").collect()[0]['max_ts']

# Read incremental data
df_incremental = df_orders.filter(col("InvoiceDate") > last_timestamp)

# Merge into Gold table
delta_table = DeltaTable.forName(spark, "gold.revenue_monthly")

delta_table.alias("target").merge(
    df_incremental.alias("source"),
    "target.year = source.year AND target.month = source.month"
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .execute()
```

### Error Handling in Notebooks

```python
try:
    # Data processing logic
    df_result = df_orders.filter(col("Quantity") > 0)
    
    # Write to Gold
    df_result.write.format("delta").mode("overwrite").saveAsTable("gold.orders")
    
    # Log success
    mssparkutils.notebook.exit("SUCCESS")
    
except Exception as e:
    # Log error details
    error_msg = f"Pipeline failed: {str(e)}"
    print(error_msg)
    
    # Exit with failure
    mssparkutils.notebook.exit(error_msg)
```

### Parameterized Notebooks

```python
# Get parameters from pipeline
date_param = mssparkutils.notebook.getArgument("process_date", "2024-01-01")
layer = mssparkutils.notebook.getArgument("layer", "silver")

print(f"Processing date: {date_param}, Layer: {layer}")

# Use parameters in logic
df_filtered = df_orders.filter(col("InvoiceDate") == date_param)
```

## Troubleshooting

### Dataflow Gen2 Issues

**Problem:** Dataflow fails with memory errors
```
Solution: Reduce data volume per refresh or enable Fast Copy
- In Dataflow settings, enable "Enhanced compute engine"
- Consider partitioning source data
```

**Problem:** Connection timeout to Lakehouse
```
Solution: Verify Lakehouse is in same workspace
- Check workspace permissions
- Refresh Lakehouse connection in Dataflow
```

### Notebook Execution Errors

**Problem:** Delta table not found
```python
# Verify table exists
spark.sql("SHOW TABLES IN silver").show()

# Create table if missing
if not spark.catalog.tableExists("silver.orders_cleansed"):
    df.write.format("delta").saveAsTable("silver.orders_cleansed")
```

**Problem:** Out of memory in PySpark
```python
# Repartition large dataframes
df_large = df_large.repartition(100)

# Persist intermediate results
df_intermediate.cache()

# Unpersist when done
df_intermediate.unpersist()
```

### Semantic Model Refresh Failures

**Problem:** Semantic model refresh timeout
```
Solution: Optimize data model
- Remove unused columns from Gold tables
- Create aggregated summary tables
- Use incremental refresh policies
```

**Problem:** Relationship errors in Power BI
```
Solution: Validate cardinality
- Ensure CustomerID is unique in dimension tables
- Check for null values in relationship keys
- Use DirectQuery for very large tables
```

### Performance Optimization

**Slow queries in Gold layer:**
```python
# Partition large tables by date
df_orders.write \
    .format("delta") \
    .partitionBy("year", "month") \
    .mode("overwrite") \
    .saveAsTable("silver.orders_partitioned")

# Add Z-order indexes
spark.sql("OPTIMIZE gold.product_performance ZORDER BY (revenue_rank)")
```

**Power BI report slow:**
```dax
// Use SUMMARIZE for aggregations instead of calculated columns
Revenue Summary = 
SUMMARIZE(
    revenue_monthly,
    revenue_monthly[year],
    revenue_monthly[month],
    "TotalRevenue", SUM(revenue_monthly[total_revenue])
)
```

## Best Practices

1. **Medallion Architecture**: Always maintain separation between Bronze (raw), Silver (cleansed), and Gold (business)
2. **Delta Format**: Use Delta Lake for ACID transactions and time travel
3. **Incremental Processing**: Process only new/changed data to optimize performance
4. **Idempotent Pipelines**: Ensure rerunning pipelines produces same results
5. **Schema Evolution**: Use `mergeSchema` option when schemas change
6. **Monitoring**: Track pipeline execution times and data volumes
7. **Documentation**: Comment complex transformations in notebooks and DAX
8. **Security**: Use workspace roles and row-level security in semantic models

## Additional Resources

- Microsoft Fabric Documentation: https://learn.microsoft.com/fabric/
- Delta Lake Best Practices: https://docs.delta.io/latest/best-practices.html
- Power BI DAX Reference: https://dax.guide/
