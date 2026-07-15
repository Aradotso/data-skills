---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering with Microsoft Fabric, Lakehouse, Dataflow Gen2, PySpark, and Power BI using Medallion Architecture
triggers:
  - "set up Microsoft Fabric lakehouse"
  - "implement medallion architecture in Fabric"
  - "create dataflow gen2 transformation"
  - "write PySpark notebook for Fabric"
  - "build semantic model in Fabric"
  - "design unified analytics platform"
  - "configure OneLake storage layers"
  - "process data with Fabric notebooks"
---

# Microsoft Fabric Unified Analytics Platform Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end unified analytics platforms using Microsoft Fabric, implementing the Medallion Architecture (Bronze → Silver → Gold) with OneLake, Dataflow Gen2, PySpark notebooks, Semantic Models, and Power BI.

## What This Project Does

Microsoft Fabric Unified Analytics Platform demonstrates how to:

- Build a complete lakehouse architecture using Microsoft Fabric's unified SaaS platform
- Implement Medallion Architecture (Bronze → Silver → Gold layers) for data quality progression
- Ingest and transform data using Dataflow Gen2
- Process business logic with Fabric Notebooks (PySpark)
- Create reusable Semantic Models for consistent analytics
- Deliver interactive dashboards through Power BI
- Manage all analytics workloads in a single unified environment with OneLake

This is a production-inspired retail analytics solution showcasing modern Data Engineering practices on Microsoft Fabric.

## Prerequisites

- Microsoft Fabric workspace (trial or paid)
- Power BI Pro or Premium Per User license
- Access to Microsoft Fabric Lakehouse
- Basic understanding of PySpark and data transformation concepts

## Lakehouse Architecture Setup

### Creating the Lakehouse

1. **Navigate to Microsoft Fabric workspace**
2. **Create a new Lakehouse** named `retail_analytics_lakehouse`
3. **Create the Medallion folder structure**:

```
lakehouse/
├── Files/
│   ├── bronze/          # Raw source data
│   ├── silver/          # Cleansed and enriched
│   └── gold/            # Business-ready datasets
└── Tables/              # Delta tables for analytics
```

### Bronze Layer Setup

Upload raw CSV data to the Bronze layer:

```python
# Upload data via Fabric Lakehouse UI or programmatically
# Expected files in bronze/:
# - online_retail.csv (raw transaction data)
```

Bronze layer stores raw, unmodified data as ingested from source systems. This preserves data lineage and enables reprocessing.

## Dataflow Gen2 Configuration

### Creating a Dataflow for Silver Layer

Dataflow Gen2 provides low-code data transformation capabilities.

**Key Transformation Steps**:

1. **Connect to Bronze layer data source**
2. **Remove duplicates**:
   - Remove rows with duplicate InvoiceNo + StockCode combinations
3. **Handle missing values**:
   - Filter out rows where CustomerID is null
   - Remove records with invalid Quantity or UnitPrice
4. **Type conversions**:
   - Ensure InvoiceDate is datetime
   - Convert Quantity to integer
   - Convert UnitPrice to decimal
5. **Business enrichment**:
   - Add `line_total` column: `Quantity * UnitPrice`
   - Extract `year` from InvoiceDate
   - Extract `month` from InvoiceDate
   - Create `is_return` flag: `if Quantity < 0 then True else False`

**Destination Configuration**:
- Output: Silver layer in Lakehouse
- Format: Delta table
- Table name: `retail_transactions_silver`

## PySpark Notebooks for Gold Layer

### Notebook: Gold Layer Business Transformations

Create a Fabric Notebook to generate curated business datasets:

```python
# Import required libraries
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.window import Window
from datetime import datetime, timedelta

# Initialize Spark session (automatically configured in Fabric)
spark = spark

# Read Silver layer data
df_silver = spark.read.format("delta").load("Tables/retail_transactions_silver")

# Display schema
df_silver.printSchema()
df_silver.show(5)
```

### Business KPIs Table

```python
# Calculate key business metrics
kpis = df_silver.agg(
    sum("line_total").alias("total_revenue"),
    countDistinct("InvoiceNo").alias("total_orders"),
    countDistinct("CustomerID").alias("total_customers"),
    countDistinct("StockCode").alias("total_products"),
    avg("line_total").alias("avg_order_value")
).withColumn("metric_date", current_date())

# Write to Gold layer
kpis.write.format("delta").mode("overwrite").saveAsTable("gold_business_kpis")

print("Business KPIs table created successfully")
```

### Revenue Trends Table

```python
# Aggregate revenue by year and month
revenue_trends = df_silver.groupBy("year", "month").agg(
    sum("line_total").alias("monthly_revenue"),
    countDistinct("InvoiceNo").alias("monthly_orders"),
    countDistinct("CustomerID").alias("monthly_customers")
).orderBy("year", "month")

# Calculate month-over-month growth
window_spec = Window.orderBy("year", "month")
revenue_trends = revenue_trends.withColumn(
    "prev_month_revenue",
    lag("monthly_revenue").over(window_spec)
).withColumn(
    "revenue_growth_pct",
    when(col("prev_month_revenue").isNotNull(),
         ((col("monthly_revenue") - col("prev_month_revenue")) / col("prev_month_revenue") * 100))
    .otherwise(None)
)

# Write to Gold layer
revenue_trends.write.format("delta").mode("overwrite").saveAsTable("gold_revenue_trends")

print("Revenue trends table created successfully")
```

### Product Performance Table

```python
# Calculate product-level metrics
product_performance = df_silver.groupBy("StockCode", "Description").agg(
    sum("line_total").alias("product_revenue"),
    sum("Quantity").alias("units_sold"),
    countDistinct("InvoiceNo").alias("orders_count"),
    countDistinct("CustomerID").alias("unique_customers"),
    avg("UnitPrice").alias("avg_unit_price")
).withColumn(
    "revenue_per_order",
    col("product_revenue") / col("orders_count")
)

# Rank products by revenue
window_spec = Window.orderBy(col("product_revenue").desc())
product_performance = product_performance.withColumn(
    "revenue_rank",
    row_number().over(window_spec)
)

# Write to Gold layer
product_performance.write.format("delta").mode("overwrite").saveAsTable("gold_product_performance")

print("Product performance table created successfully")
```

### Customer Analytics Table

```python
# Calculate customer-level metrics
customer_analytics = df_silver.groupBy("CustomerID", "Country").agg(
    sum("line_total").alias("total_spend"),
    countDistinct("InvoiceNo").alias("total_orders"),
    count("*").alias("total_items_purchased"),
    min("InvoiceDate").alias("first_purchase_date"),
    max("InvoiceDate").alias("last_purchase_date")
).withColumn(
    "avg_order_value",
    col("total_spend") / col("total_orders")
).withColumn(
    "customer_lifetime_days",
    datediff(col("last_purchase_date"), col("first_purchase_date"))
)

# Write to Gold layer
customer_analytics.write.format("delta").mode("overwrite").saveAsTable("gold_customer_analytics")

print("Customer analytics table created successfully")
```

### RFM Segmentation

```python
# Calculate RFM metrics (Recency, Frequency, Monetary)
current_date = df_silver.agg(max("InvoiceDate")).collect()[0][0]

rfm = df_silver.groupBy("CustomerID").agg(
    datediff(lit(current_date), max("InvoiceDate")).alias("recency"),
    countDistinct("InvoiceNo").alias("frequency"),
    sum("line_total").alias("monetary")
)

# Calculate RFM scores using quartiles
rfm_with_scores = rfm.withColumn(
    "r_score",
    when(col("recency") <= rfm.approxQuantile("recency", [0.25], 0.01)[0], 4)
    .when(col("recency") <= rfm.approxQuantile("recency", [0.50], 0.01)[0], 3)
    .when(col("recency") <= rfm.approxQuantile("recency", [0.75], 0.01)[0], 2)
    .otherwise(1)
).withColumn(
    "f_score",
    when(col("frequency") >= rfm.approxQuantile("frequency", [0.75], 0.01)[0], 4)
    .when(col("frequency") >= rfm.approxQuantile("frequency", [0.50], 0.01)[0], 3)
    .when(col("frequency") >= rfm.approxQuantile("frequency", [0.25], 0.01)[0], 2)
    .otherwise(1)
).withColumn(
    "m_score",
    when(col("monetary") >= rfm.approxQuantile("monetary", [0.75], 0.01)[0], 4)
    .when(col("monetary") >= rfm.approxQuantile("monetary", [0.50], 0.01)[0], 3)
    .when(col("monetary") >= rfm.approxQuantile("monetary", [0.25], 0.01)[0], 2)
    .otherwise(1)
)

# Create customer segments
rfm_segmented = rfm_with_scores.withColumn(
    "customer_segment",
    when((col("r_score") >= 3) & (col("f_score") >= 3) & (col("m_score") >= 3), "Champions")
    .when((col("r_score") >= 3) & (col("f_score") >= 2), "Loyal Customers")
    .when((col("r_score") >= 3), "Recent Customers")
    .when((col("f_score") >= 3) & (col("m_score") >= 3), "Big Spenders")
    .when((col("r_score") <= 2) & (col("f_score") <= 2), "At Risk")
    .otherwise("Needs Attention")
)

# Write to Gold layer
rfm_segmented.write.format("delta").mode("overwrite").saveAsTable("gold_customer_rfm_segmentation")

print("RFM segmentation table created successfully")
```

### Running All Transformations

```python
# Complete Gold layer processing pipeline
def process_gold_layer():
    """
    Execute all Gold layer transformations
    """
    try:
        # Read Silver data
        df_silver = spark.read.format("delta").load("Tables/retail_transactions_silver")
        
        # Generate all Gold tables
        print("Creating Business KPIs...")
        # [KPIs code here]
        
        print("Creating Revenue Trends...")
        # [Revenue trends code here]
        
        print("Creating Product Performance...")
        # [Product performance code here]
        
        print("Creating Customer Analytics...")
        # [Customer analytics code here]
        
        print("Creating RFM Segmentation...")
        # [RFM code here]
        
        print("✅ Gold layer processing completed successfully!")
        
    except Exception as e:
        print(f"❌ Error processing Gold layer: {str(e)}")
        raise

# Execute pipeline
process_gold_layer()
```

## Semantic Model Configuration

### Creating the Semantic Model

1. **Navigate to Lakehouse** in Fabric workspace
2. **Select "New semantic model"** from top menu
3. **Select Gold layer tables**:
   - gold_business_kpis
   - gold_revenue_trends
   - gold_product_performance
   - gold_customer_analytics
   - gold_customer_rfm_segmentation

### Defining Relationships

In the Semantic Model view:

```
gold_product_performance[StockCode] ← Many-to-One → retail_transactions_silver[StockCode]
gold_customer_analytics[CustomerID] ← One-to-One → gold_customer_rfm_segmentation[CustomerID]
```

### Creating DAX Measures

```dax
-- Total Revenue
Total Revenue = SUM('gold_business_kpis'[total_revenue])

-- Average Order Value
Avg Order Value = AVERAGE('retail_transactions_silver'[line_total])

-- Customer Count
Total Customers = DISTINCTCOUNT('retail_transactions_silver'[CustomerID])

-- Revenue Growth %
Revenue Growth % = 
DIVIDE(
    [Total Revenue] - CALCULATE([Total Revenue], PREVIOUSMONTH('retail_transactions_silver'[InvoiceDate])),
    CALCULATE([Total Revenue], PREVIOUSMONTH('retail_transactions_silver'[InvoiceDate])),
    0
) * 100

-- Top Products by Revenue
Top 10 Products = 
CALCULATE(
    [Total Revenue],
    TOPN(10, 'gold_product_performance', 'gold_product_performance'[product_revenue], DESC)
)
```

## Power BI Dashboard Integration

### Connecting to Semantic Model

1. **Open Power BI Desktop** or **use Fabric Power BI**
2. **Get Data** → **Power BI semantic models**
3. **Select your Fabric semantic model**
4. **Import or DirectQuery** (DirectQuery recommended for live data)

### Key Visualizations

```dax
-- For card visuals
Total Revenue Card = FORMAT([Total Revenue], "$#,##0")
Total Customers Card = FORMAT([Total Customers], "#,##0")
Avg Order Value Card = FORMAT([Avg Order Value], "$#,##0.00")

-- For trend charts
Monthly Revenue = 
SUMMARIZE(
    'gold_revenue_trends',
    'gold_revenue_trends'[year],
    'gold_revenue_trends'[month],
    "Revenue", [monthly_revenue]
)

-- For customer segmentation
Customers by Segment = 
SUMMARIZE(
    'gold_customer_rfm_segmentation',
    'gold_customer_rfm_segmentation'[customer_segment],
    "Count", COUNTROWS('gold_customer_rfm_segmentation')
)
```

## Common Patterns

### Incremental Data Processing

```python
# Process only new data since last run
from delta.tables import DeltaTable

# Read existing Gold table
gold_table = DeltaTable.forName(spark, "gold_revenue_trends")

# Get max date from Gold
max_processed_date = spark.sql("""
    SELECT MAX(InvoiceDate) as max_date 
    FROM gold_revenue_trends
""").collect()[0]["max_date"]

# Filter Silver for new records only
df_new = df_silver.filter(col("InvoiceDate") > max_processed_date)

# Process and merge
if df_new.count() > 0:
    new_revenue_trends = df_new.groupBy("year", "month").agg(
        sum("line_total").alias("monthly_revenue")
    )
    
    # Merge into existing table
    gold_table.alias("target").merge(
        new_revenue_trends.alias("source"),
        "target.year = source.year AND target.month = source.month"
    ).whenMatchedUpdate(set={
        "monthly_revenue": "source.monthly_revenue"
    }).whenNotMatchedInsertAll().execute()
```

### Data Quality Checks

```python
# Validate data quality before writing to Gold
def validate_data_quality(df, table_name):
    """
    Perform data quality checks
    """
    # Check for nulls in critical columns
    null_counts = df.select([
        count(when(col(c).isNull(), c)).alias(c)
        for c in df.columns
    ])
    
    print(f"Null counts for {table_name}:")
    null_counts.show()
    
    # Check for negative values in monetary columns
    if "line_total" in df.columns:
        negative_count = df.filter(col("line_total") < 0).count()
        print(f"Records with negative line_total: {negative_count}")
    
    # Check row count
    row_count = df.count()
    print(f"Total rows: {row_count}")
    
    return row_count > 0

# Use before writing
if validate_data_quality(revenue_trends, "gold_revenue_trends"):
    revenue_trends.write.format("delta").mode("overwrite").saveAsTable("gold_revenue_trends")
else:
    raise Exception("Data quality validation failed")
```

### Partitioning Strategy

```python
# Partition Gold tables by year and month for performance
revenue_trends.write.format("delta") \
    .partitionBy("year", "month") \
    .mode("overwrite") \
    .saveAsTable("gold_revenue_trends")

# Query partitioned data efficiently
df_2024 = spark.read.format("delta").load("Tables/gold_revenue_trends") \
    .filter(col("year") == 2024)
```

## Environment Configuration

### Fabric Workspace Settings

Store configuration in notebook parameters or environment:

```python
# Lakehouse paths
BRONZE_PATH = "Files/bronze/"
SILVER_PATH = "Files/silver/"
GOLD_PATH = "Tables/"

# Table names
SILVER_TABLE = "retail_transactions_silver"
GOLD_KPI_TABLE = "gold_business_kpis"
GOLD_REVENUE_TABLE = "gold_revenue_trends"
GOLD_PRODUCT_TABLE = "gold_product_performance"
GOLD_CUSTOMER_TABLE = "gold_customer_analytics"
GOLD_RFM_TABLE = "gold_customer_rfm_segmentation"

# Processing configuration
INCREMENTAL_MODE = True
PARTITION_ENABLED = True
DATA_QUALITY_CHECKS = True
```

### Accessing Lakehouse Resources

```python
# Read from Lakehouse files
df_bronze = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load(f"Files/bronze/online_retail.csv")

# Read from Lakehouse tables
df_silver = spark.read.format("delta").table(SILVER_TABLE)

# Write to Lakehouse tables
df_gold.write.format("delta") \
    .mode("overwrite") \
    .option("mergeSchema", "true") \
    .saveAsTable(GOLD_KPI_TABLE)
```

## Troubleshooting

### Issue: Dataflow Gen2 fails to write to Lakehouse

**Solution**: Ensure workspace has proper permissions and Lakehouse is in the same workspace as Dataflow.

```
- Check workspace role (Admin, Member, or Contributor required)
- Verify Lakehouse destination path exists
- Ensure Dataflow has "Write" permission to Lakehouse
```

### Issue: PySpark notebook cannot find Delta table

**Solution**: Verify table registration and use correct table reference:

```python
# List available tables
spark.sql("SHOW TABLES").show()

# Refresh table metadata
spark.catalog.refreshTable("retail_transactions_silver")

# Use fully qualified table name if needed
df = spark.read.format("delta").table("lakehouse_name.retail_transactions_silver")
```

### Issue: Semantic model shows incorrect relationships

**Solution**: Verify data types and cardinality:

```python
# Check data types in Gold tables
df = spark.read.format("delta").table("gold_product_performance")
df.printSchema()

# Ensure join keys have matching types
df.select("StockCode").distinct().count()  # Should match Silver StockCode count
```

### Issue: Power BI dashboard shows outdated data

**Solution**: Ensure DirectQuery mode or schedule refresh:

```
- Use DirectQuery connection for real-time data
- For Import mode, configure scheduled refresh in Fabric
- Verify Semantic Model refresh hasn't failed
```

### Issue: Performance degradation in large datasets

**Solution**: Implement optimization strategies:

```python
# Enable Delta Lake optimizations
spark.sql("OPTIMIZE gold_revenue_trends")
spark.sql("VACUUM gold_revenue_trends RETAIN 168 HOURS")

# Use partitioning
df.write.format("delta").partitionBy("year", "month").saveAsTable("gold_revenue_trends")

# Cache frequently accessed data
df_silver.cache()

# Use broadcast joins for small dimension tables
from pyspark.sql.functions import broadcast
df_result = df_large.join(broadcast(df_small), "key")
```

This skill provides comprehensive guidance for building unified analytics platforms with Microsoft Fabric following production-grade Data Engineering practices.
