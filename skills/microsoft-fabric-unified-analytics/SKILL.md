---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering with Microsoft Fabric, implementing Medallion Architecture using Lakehouse, Dataflow Gen2, PySpark notebooks, and Power BI.
triggers:
  - how do I build a lakehouse in microsoft fabric
  - implement medallion architecture with fabric
  - create dataflow gen2 transformations
  - process data with pyspark in fabric notebooks
  - build semantic models in microsoft fabric
  - set up bronze silver gold layers in fabric
  - use onelake for data engineering
  - create power bi reports from fabric lakehouse
---

# Microsoft Fabric Unified Analytics Platform Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in building end-to-end analytics platforms using **Microsoft Fabric**, following the **Medallion Architecture** pattern (Bronze → Silver → Gold). The project demonstrates unified data engineering across ingestion, transformation, semantic modeling, and visualization within a single SaaS platform using OneLake, Dataflow Gen2, Fabric Notebooks (PySpark), Semantic Models, and Power BI.

## What This Project Does

This project implements a production-ready retail analytics platform that:

- **Organizes data** using Medallion Architecture (Bronze/Silver/Gold layers)
- **Ingests raw data** into the Bronze layer via OneLake
- **Transforms data** using Dataflow Gen2 for the Silver layer
- **Processes business logic** with PySpark in Fabric Notebooks for the Gold layer
- **Creates semantic models** for consistent business metrics
- **Delivers insights** through native Power BI integration

The architecture unifies all analytics components in Microsoft Fabric instead of managing separate Azure services (ADF, Databricks, Synapse, etc.).

## Prerequisites

To work with this project, you need:

- **Microsoft Fabric capacity** (Trial, F2, or higher)
- **Fabric workspace** with appropriate permissions
- **OneLake access** for data storage
- **Power BI** (included with Fabric)
- Basic knowledge of **PySpark** and **Python**

## Project Structure

```
microsoft-fabric-unified-analytics-platformre/
├── architecture/          # Architecture diagrams and screenshots
├── notebooks/            # PySpark notebooks for Gold layer
├── case-study/           # Technical documentation
└── README.md
```

## Setting Up the Lakehouse

### 1. Create a Lakehouse in Fabric

In your Microsoft Fabric workspace:

```python
# No code needed - use Fabric UI:
# 1. Open your Fabric workspace
# 2. Click "+ New" → "Lakehouse"
# 3. Name it (e.g., "retail_analytics_lakehouse")
# 4. This creates your OneLake storage automatically
```

### 2. Organize Medallion Layers

Create folder structure in your Lakehouse:

```
Files/
├── bronze/           # Raw data
│   ├── orders/
│   └── customers/
├── silver/           # Cleansed data
│   ├── orders_clean/
│   └── customers_clean/
└── gold/            # Business-ready data
    ├── kpis/
    ├── revenue_trends/
    └── customer_segments/
```

## Working with Dataflow Gen2

### Creating a Dataflow Gen2

Dataflow Gen2 handles Bronze → Silver transformations using Power Query:

```m
// Example Power Query M code for Dataflow Gen2
let
    // Load from Bronze layer
    Source = Lakehouse.Contents(null),
    bronze_orders = Source{[workspaceId="YOUR_WORKSPACE_ID"]}[Data],
    orders_table = bronze_orders{[lakehouseTable="bronze_orders"]}[Data],
    
    // Remove duplicates
    RemoveDuplicates = Table.Distinct(orders_table, {"InvoiceNo", "StockCode"}),
    
    // Handle missing values
    ReplaceNull = Table.ReplaceValue(RemoveDuplicates, null, 0, Replacer.ReplaceValue, {"Quantity"}),
    
    // Add calculated columns
    AddLineTotal = Table.AddColumn(ReplaceNull, "line_total", each [Quantity] * [UnitPrice], type number),
    AddYear = Table.AddColumn(AddLineTotal, "year", each Date.Year([InvoiceDate]), Int64.Type),
    AddMonth = Table.AddColumn(AddYear, "month", each Date.Month([InvoiceDate]), Int64.Type),
    
    // Add business logic
    AddIsReturn = Table.AddColumn(AddMonth, "is_return", each if [Quantity] < 0 then true else false, type logical),
    
    // Change data types
    ChangeTypes = Table.TransformColumnTypes(AddIsReturn,{
        {"InvoiceNo", type text},
        {"StockCode", type text},
        {"Quantity", Int64.Type},
        {"InvoiceDate", type datetime},
        {"UnitPrice", type number},
        {"CustomerID", type text}
    })
in
    ChangeTypes
```

**Configure destination:**
- Destination: Lakehouse
- Table: `silver/orders_clean`
- Update method: Replace

## PySpark Processing in Fabric Notebooks

### Loading Data from Silver Layer

```python
# Load cleansed data from Silver layer
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

# Read from Silver layer (Delta tables)
df_orders = spark.read.format("delta").load("Tables/silver_orders_clean")
df_customers = spark.read.format("delta").load("Tables/silver_customers_clean")

# Display schema
df_orders.printSchema()
```

### Creating Gold Layer: Business KPIs

```python
from pyspark.sql.functions import sum, count, avg, max, min, countDistinct

# Calculate key business metrics
kpis = df_orders.agg(
    sum("line_total").alias("total_revenue"),
    count("InvoiceNo").alias("total_orders"),
    countDistinct("CustomerID").alias("unique_customers"),
    avg("line_total").alias("avg_order_value"),
    max("InvoiceDate").alias("last_order_date"),
    min("InvoiceDate").alias("first_order_date")
)

# Save to Gold layer
kpis.write.format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable("gold_business_kpis")
```

### Creating Gold Layer: Revenue Trends

```python
# Aggregate revenue by year and month
revenue_trends = df_orders.groupBy("year", "month") \
    .agg(
        sum("line_total").alias("monthly_revenue"),
        count("InvoiceNo").alias("order_count"),
        countDistinct("CustomerID").alias("customer_count"),
        avg("line_total").alias("avg_transaction")
    ) \
    .orderBy("year", "month")

# Add month name for reporting
revenue_trends = revenue_trends.withColumn(
    "month_name",
    when(col("month") == 1, "January")
    .when(col("month") == 2, "February")
    .when(col("month") == 3, "March")
    .when(col("month") == 4, "April")
    .when(col("month") == 5, "May")
    .when(col("month") == 6, "June")
    .when(col("month") == 7, "July")
    .when(col("month") == 8, "August")
    .when(col("month") == 9, "September")
    .when(col("month") == 10, "October")
    .when(col("month") == 11, "November")
    .otherwise("December")
)

# Write to Gold layer
revenue_trends.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold_revenue_trends")
```

### Creating Gold Layer: Product Performance

```python
# Analyze product performance
product_performance = df_orders.groupBy("StockCode", "Description") \
    .agg(
        sum("Quantity").alias("total_quantity_sold"),
        sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("transaction_count"),
        avg("UnitPrice").alias("avg_price"),
        countDistinct("CustomerID").alias("unique_buyers")
    ) \
    .orderBy(col("total_revenue").desc())

# Calculate revenue contribution percentage
total_revenue = df_orders.select(sum("line_total")).collect()[0][0]
product_performance = product_performance.withColumn(
    "revenue_percentage",
    round((col("total_revenue") / total_revenue) * 100, 2)
)

# Save to Gold layer
product_performance.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold_product_performance")
```

### Creating Gold Layer: RFM Customer Segmentation

```python
from pyspark.sql.window import Window
from datetime import datetime

# Calculate RFM metrics
reference_date = df_orders.agg(max("InvoiceDate")).collect()[0][0]

rfm = df_orders.groupBy("CustomerID") \
    .agg(
        datediff(lit(reference_date), max("InvoiceDate")).alias("recency"),
        countDistinct("InvoiceNo").alias("frequency"),
        sum("line_total").alias("monetary")
    )

# Create RFM scores using quartiles
for col_name in ["recency", "frequency", "monetary"]:
    quartiles = rfm.approxQuantile(col_name, [0.25, 0.5, 0.75], 0.01)
    
    if col_name == "recency":
        # Lower recency is better
        rfm = rfm.withColumn(
            f"{col_name}_score",
            when(col(col_name) <= quartiles[0], 4)
            .when(col(col_name) <= quartiles[1], 3)
            .when(col(col_name) <= quartiles[2], 2)
            .otherwise(1)
        )
    else:
        # Higher frequency/monetary is better
        rfm = rfm.withColumn(
            f"{col_name}_score",
            when(col(col_name) <= quartiles[0], 1)
            .when(col(col_name) <= quartiles[1], 2)
            .when(col(col_name) <= quartiles[2], 3)
            .otherwise(4)
        )

# Create overall RFM score
rfm = rfm.withColumn(
    "rfm_score",
    concat(col("recency_score"), col("frequency_score"), col("monetary_score"))
)

# Segment customers
rfm = rfm.withColumn(
    "customer_segment",
    when(col("rfm_score").isin("444", "443", "434"), "Champions")
    .when(col("rfm_score").isin("344", "343", "334", "244"), "Loyal Customers")
    .when(col("rfm_score").isin("442", "441", "432", "431"), "Potential Loyalists")
    .when(col("rfm_score").isin("414", "413", "424"), "Recent Customers")
    .when(col("rfm_score").isin("333", "332", "323"), "Promising")
    .when(col("rfm_score").isin("241", "242", "231"), "At Risk")
    .when(col("rfm_score").isin("144", "143", "134"), "Can't Lose")
    .otherwise("Others")
)

# Save to Gold layer
rfm.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold_customer_rfm")
```

## Creating Semantic Models

### Define Semantic Model in Fabric

After creating Gold tables, build a semantic model:

```python
# Use Fabric UI to create semantic model:
# 1. Navigate to your Lakehouse
# 2. Click "New semantic model"
# 3. Select Gold layer tables
# 4. Define relationships and measures
```

### Example DAX Measures

```dax
// Total Revenue
Total Revenue = SUM(gold_revenue_trends[monthly_revenue])

// Revenue Growth %
Revenue Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD('gold_revenue_trends'[date], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0)

// Customer Lifetime Value
Customer LTV = 
AVERAGEX(
    SUMMARIZE(
        gold_customer_rfm,
        gold_customer_rfm[CustomerID],
        "TotalSpent", [Total Revenue]
    ),
    [TotalSpent]
)

// Active Customers
Active Customers = 
CALCULATE(
    DISTINCTCOUNT(gold_revenue_trends[CustomerID]),
    gold_customer_rfm[recency] <= 90
)
```

## Data Quality and Validation

### Validate Silver Layer Quality

```python
# Data quality checks after Dataflow Gen2
def validate_silver_quality(df, table_name):
    """Validate data quality in Silver layer"""
    
    # Check for nulls in critical columns
    critical_cols = ["InvoiceNo", "StockCode", "Quantity", "InvoiceDate"]
    null_counts = df.select([
        count(when(col(c).isNull(), c)).alias(c) 
        for c in critical_cols
    ])
    
    print(f"=== Quality Check: {table_name} ===")
    null_counts.show()
    
    # Check for duplicates
    total_rows = df.count()
    distinct_rows = df.dropDuplicates(["InvoiceNo", "StockCode"]).count()
    duplicate_count = total_rows - distinct_rows
    
    print(f"Total rows: {total_rows}")
    print(f"Distinct rows: {distinct_rows}")
    print(f"Duplicates: {duplicate_count}")
    
    # Validate data ranges
    df.select(
        min("InvoiceDate").alias("min_date"),
        max("InvoiceDate").alias("max_date"),
        min("Quantity").alias("min_qty"),
        max("Quantity").alias("max_qty")
    ).show()
    
    return duplicate_count == 0 and null_counts.first()[0] == 0

# Run validation
df_silver = spark.read.format("delta").load("Tables/silver_orders_clean")
is_valid = validate_silver_quality(df_silver, "silver_orders_clean")
```

## Common Patterns

### Pattern 1: Incremental Load to Bronze

```python
# Load only new data into Bronze layer
from datetime import datetime, timedelta

# Get last load timestamp
last_load = spark.sql("""
    SELECT MAX(load_timestamp) as last_load 
    FROM bronze_orders_metadata
""").collect()[0]["last_load"]

# Load incremental data
new_data = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load(f"abfss://source-data/*.csv") \
    .filter(col("InvoiceDate") > last_load)

# Append to Bronze
new_data.withColumn("load_timestamp", current_timestamp()) \
    .write.format("delta") \
    .mode("append") \
    .saveAsTable("bronze_orders")
```

### Pattern 2: Gold Layer Refresh Strategy

```python
# Full refresh with audit metadata
def refresh_gold_table(table_name, df):
    """Refresh Gold table with audit trail"""
    
    # Add metadata
    df_with_meta = df.withColumn("refresh_timestamp", current_timestamp()) \
                      .withColumn("refresh_id", monotonically_increasing_id())
    
    # Write with overwrite
    df_with_meta.write.format("delta") \
        .mode("overwrite") \
        .option("overwriteSchema", "true") \
        .saveAsTable(table_name)
    
    # Log refresh
    refresh_log = spark.createDataFrame([
        (table_name, datetime.now(), df.count())
    ], ["table_name", "refresh_time", "row_count"])
    
    refresh_log.write.format("delta") \
        .mode("append") \
        .saveAsTable("gold_refresh_log")

# Use pattern
refresh_gold_table("gold_revenue_trends", revenue_trends)
```

### Pattern 3: Cross-Layer Data Lineage

```python
# Track data lineage across layers
lineage_metadata = {
    "source_layer": "silver",
    "source_table": "silver_orders_clean",
    "target_layer": "gold",
    "target_table": "gold_revenue_trends",
    "transformation_logic": "Monthly aggregation with KPIs",
    "created_by": spark.sparkContext.sparkUser(),
    "created_at": datetime.now()
}

# Save lineage
lineage_df = spark.createDataFrame([lineage_metadata])
lineage_df.write.format("delta") \
    .mode("append") \
    .saveAsTable("metadata_lineage")
```

## Troubleshooting

### Issue: Dataflow Gen2 Fails to Refresh

**Symptoms:** Dataflow shows "Failed" status

**Solutions:**
```python
# Check table locks
# Run in notebook:
spark.sql("SHOW LOCKS silver_orders_clean").show()

# Optimize Delta table
spark.sql("OPTIMIZE silver_orders_clean")
spark.sql("VACUUM silver_orders_clean RETAIN 168 HOURS")

# Check for schema changes
df_bronze = spark.read.format("delta").load("Tables/bronze_orders")
df_bronze.printSchema()
```

### Issue: PySpark Notebook Performance Degradation

**Symptoms:** Long execution times on Gold transformations

**Solutions:**
```python
# Enable adaptive query execution
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")

# Repartition before aggregations
df_optimized = df_orders.repartition(8, "CustomerID")

# Cache intermediate results
df_orders.cache()
revenue_trends = df_orders.groupBy("year", "month").agg(...)
df_orders.unpersist()

# Use broadcast for small dimension tables
from pyspark.sql.functions import broadcast
result = df_large.join(broadcast(df_small), "key")
```

### Issue: Semantic Model Shows Incorrect Totals

**Symptoms:** Power BI visuals don't match source data

**Solutions:**
```python
# Validate Gold table totals match Silver
silver_total = spark.sql("""
    SELECT SUM(line_total) as silver_revenue 
    FROM silver_orders_clean
""").collect()[0]["silver_revenue"]

gold_total = spark.sql("""
    SELECT SUM(monthly_revenue) as gold_revenue 
    FROM gold_revenue_trends
""").collect()[0]["gold_revenue"]

print(f"Silver: {silver_total}")
print(f"Gold: {gold_total}")
print(f"Match: {abs(silver_total - gold_total) < 0.01}")

# Check for duplicate relationships in semantic model
# Review in Fabric UI: Model view → Check relationship cardinality
```

### Issue: OneLake Storage Costs Growing

**Symptoms:** Unexpected storage consumption

**Solutions:**
```python
# Analyze table sizes
spark.sql("""
    SELECT 
        table_name,
        location,
        sizeInBytes / 1024 / 1024 / 1024 as size_gb
    FROM 
        SYSTEM.tables
    WHERE 
        table_name LIKE 'bronze_%'
        OR table_name LIKE 'silver_%'
        OR table_name LIKE 'gold_%'
    ORDER BY 
        size_gb DESC
""").show()

# Run VACUUM to remove old versions
for table in ["bronze_orders", "silver_orders_clean", "gold_revenue_trends"]:
    spark.sql(f"VACUUM {table} RETAIN 168 HOURS")
    print(f"Vacuumed {table}")

# Enable table optimization
spark.sql("ALTER TABLE gold_revenue_trends SET TBLPROPERTIES ('delta.autoOptimize.optimizeWrite' = 'true')")
```

## Best Practices

1. **Layer Separation**: Keep Bronze immutable, transform in Silver, aggregate in Gold
2. **Delta Format**: Use Delta Lake for all layers for ACID compliance and time travel
3. **Partition Strategy**: Partition large tables by date in Silver/Gold for performance
4. **Incremental Processing**: Load only new/changed data where possible
5. **Metadata Tracking**: Log all transformations and refreshes for auditing
6. **Schema Evolution**: Use `mergeSchema` option when schema changes are expected
7. **Resource Management**: Cache strategically and unpersist when done
8. **Validation**: Always validate data quality between layers

## Additional Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric/)
- [Medallion Architecture Guide](https://learn.microsoft.com/en-us/azure/databricks/lakehouse/medallion)
- [PySpark API Reference](https://spark.apache.org/docs/latest/api/python/)
- [Power Query M Reference](https://learn.microsoft.com/en-us/powerquery-m/)

---

This skill enables AI agents to guide developers through building complete analytics platforms using Microsoft Fabric's unified approach, from raw data ingestion through business intelligence delivery.
