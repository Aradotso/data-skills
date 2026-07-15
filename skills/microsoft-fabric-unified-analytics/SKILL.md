---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering with Microsoft Fabric, Lakehouse, Dataflow Gen2, PySpark, and Power BI using Medallion Architecture
triggers:
  - "build a Microsoft Fabric lakehouse pipeline"
  - "set up medallion architecture in Fabric"
  - "create dataflow gen2 transformations"
  - "write PySpark notebooks for Fabric"
  - "design unified analytics with OneLake"
  - "implement bronze silver gold layers"
  - "configure semantic models in Fabric"
  - "integrate Power BI with Fabric lakehouse"
---

# Microsoft Fabric Unified Analytics Platform Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end unified analytics platforms using Microsoft Fabric, implementing the Medallion Architecture (Bronze → Silver → Gold) with OneLake, Dataflow Gen2, PySpark notebooks, Semantic Models, and Power BI.

## What This Project Does

The Microsoft Fabric Unified Analytics Platform demonstrates a production-grade data engineering solution that:

- **Unifies the analytics lifecycle** in a single SaaS platform (Microsoft Fabric)
- **Implements Medallion Architecture** with Bronze (raw), Silver (cleansed), and Gold (business-ready) layers
- **Uses OneLake** as the unified storage foundation across all workloads
- **Transforms data** using Dataflow Gen2 (low-code) and Fabric Notebooks (PySpark)
- **Creates semantic models** for consistent business metrics
- **Delivers insights** through native Power BI integration

Unlike traditional multi-service architectures, everything runs within Microsoft Fabric's unified platform.

## Prerequisites

- Active Microsoft Fabric workspace
- Microsoft Fabric capacity (F2 or higher)
- Power BI Pro or Premium Per User license
- Azure subscription (for source data if using external sources)
- Python 3.8+ (for local notebook development)

## Core Architecture Patterns

### Medallion Architecture Layers

```
Lakehouse/
├── Bronze/          # Raw source data (immutable)
│   ├── customers/
│   ├── orders/
│   └── products/
├── Silver/          # Cleansed and enriched
│   ├── customers_clean/
│   ├── orders_enriched/
│   └── products_standardized/
└── Gold/            # Business-ready aggregations
    ├── revenue_trends/
    ├── customer_analytics/
    ├── product_performance/
    └── rfm_segmentation/
```

### Fabric Lakehouse Setup

1. **Create a Lakehouse** in your Fabric workspace
2. **Organize folders** following Medallion Architecture
3. **Set up permissions** for workspace collaboration

```python
# Access Lakehouse in Fabric Notebook
from pyspark.sql import SparkSession

# Fabric automatically configures Spark session
spark = SparkSession.builder.getOrCreate()

# Reference OneLake paths
bronze_path = "abfss://workspace@onelake.dfs.fabric.microsoft.com/lakehouse/Files/bronze/"
silver_path = "abfss://workspace@onelake.dfs.fabric.microsoft.com/lakehouse/Files/silver/"
gold_path = "abfss://workspace@onelake.dfs.fabric.microsoft.com/lakehouse/Files/gold/"
```

## Bronze Layer: Raw Data Ingestion

### Upload Raw Data to Bronze

Bronze layer preserves raw source data without transformations.

```python
from pyspark.sql import DataFrame
from pyspark.sql.types import *

# Define schema for raw orders
orders_schema = StructType([
    StructField("InvoiceNo", StringType(), True),
    StructField("StockCode", StringType(), True),
    StructField("Description", StringType(), True),
    StructField("Quantity", IntegerType(), True),
    StructField("InvoiceDate", StringType(), True),
    StructField("UnitPrice", DoubleType(), True),
    StructField("CustomerID", StringType(), True),
    StructField("Country", StringType(), True)
])

# Read from source (CSV example)
raw_orders_df = spark.read \
    .option("header", "true") \
    .schema(orders_schema) \
    .csv("path/to/source/orders.csv")

# Write to Bronze layer (preserve raw format)
bronze_orders_path = f"{bronze_path}orders/"
raw_orders_df.write \
    .mode("overwrite") \
    .format("delta") \
    .save(bronze_orders_path)

print(f"Bronze layer: {raw_orders_df.count()} records ingested")
```

### Bronze Layer Best Practices

- **Immutability**: Never modify Bronze data
- **Partitioning**: Partition by ingestion date for tracking
- **Metadata**: Include ingestion timestamp

```python
from pyspark.sql.functions import current_timestamp

# Add ingestion metadata
raw_orders_df_with_metadata = raw_orders_df \
    .withColumn("ingestion_timestamp", current_timestamp())

# Partition by date
raw_orders_df_with_metadata.write \
    .mode("append") \
    .partitionBy("ingestion_date") \
    .format("delta") \
    .save(bronze_orders_path)
```

## Silver Layer: Data Cleansing with Dataflow Gen2

### Dataflow Gen2 Configuration

Dataflow Gen2 provides low-code transformations using Power Query.

**Key Transformations for Silver Layer:**

1. **Remove duplicates** based on business key
2. **Handle missing values** (drop, fill, or flag)
3. **Standardize data types** (dates, decimals, strings)
4. **Add business enrichment** (calculated columns)

### Dataflow Gen2 Example Transformations

```m
// Power Query M code for Dataflow Gen2
let
    // Load from Bronze
    Source = Lakehouse.Contents("bronze/orders"),
    
    // Remove duplicates
    RemoveDuplicates = Table.Distinct(Source, {"InvoiceNo", "StockCode"}),
    
    // Handle missing CustomerID
    FillMissingCustomer = Table.ReplaceValue(
        RemoveDuplicates,
        null,
        "GUEST",
        Replacer.ReplaceValue,
        {"CustomerID"}
    ),
    
    // Standardize date format
    ParseDate = Table.TransformColumns(
        FillMissingCustomer,
        {{"InvoiceDate", DateTime.FromText, type datetime}}
    ),
    
    // Add line total
    AddLineTotal = Table.AddColumn(
        ParseDate,
        "LineTotal",
        each [Quantity] * [UnitPrice],
        type number
    ),
    
    // Add time dimensions
    AddYear = Table.AddColumn(AddLineTotal, "Year", each Date.Year([InvoiceDate]), Int64.Type),
    AddMonth = Table.AddColumn(AddYear, "Month", each Date.Month([InvoiceDate]), Int64.Type),
    
    // Flag returns (negative quantity)
    AddReturnFlag = Table.AddColumn(
        AddMonth,
        "IsReturn",
        each [Quantity] < 0,
        type logical
    )
in
    AddReturnFlag
```

### PySpark Alternative for Silver Layer

```python
from pyspark.sql.functions import col, when, year, month, abs as spark_abs

# Read from Bronze
bronze_orders = spark.read.format("delta").load(bronze_orders_path)

# Data quality transformations
silver_orders = bronze_orders \
    .dropDuplicates(["InvoiceNo", "StockCode"]) \
    .fillna({"CustomerID": "GUEST"}) \
    .withColumn("InvoiceDate", col("InvoiceDate").cast("timestamp")) \
    .withColumn("LineTotal", col("Quantity") * col("UnitPrice")) \
    .withColumn("Year", year(col("InvoiceDate"))) \
    .withColumn("Month", month(col("InvoiceDate"))) \
    .withColumn("IsReturn", when(col("Quantity") < 0, True).otherwise(False))

# Write to Silver layer
silver_orders_path = f"{silver_path}orders_clean/"
silver_orders.write \
    .mode("overwrite") \
    .format("delta") \
    .save(silver_orders_path)

print(f"Silver layer: {silver_orders.count()} clean records")
```

## Gold Layer: Business Aggregations with PySpark

### Revenue Trends Aggregation

```python
from pyspark.sql.functions import sum, count, avg, round as spark_round

# Read from Silver
silver_orders = spark.read.format("delta").load(silver_orders_path)

# Aggregate revenue by year and month
revenue_trends = silver_orders \
    .filter(col("IsReturn") == False) \
    .groupBy("Year", "Month") \
    .agg(
        spark_round(sum("LineTotal"), 2).alias("TotalRevenue"),
        count("InvoiceNo").alias("OrderCount"),
        spark_round(avg("LineTotal"), 2).alias("AvgOrderValue")
    ) \
    .orderBy("Year", "Month")

# Write to Gold layer
gold_revenue_path = f"{gold_path}revenue_trends/"
revenue_trends.write \
    .mode("overwrite") \
    .format("delta") \
    .save(gold_revenue_path)

# Display sample
revenue_trends.show(10)
```

### Product Performance Analysis

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import rank, desc

# Product-level aggregation
product_performance = silver_orders \
    .filter(col("IsReturn") == False) \
    .groupBy("StockCode", "Description") \
    .agg(
        spark_round(sum("LineTotal"), 2).alias("TotalRevenue"),
        sum("Quantity").alias("TotalQuantitySold"),
        count("InvoiceNo").alias("OrderFrequency")
    )

# Rank products by revenue
window_spec = Window.orderBy(desc("TotalRevenue"))
product_ranked = product_performance \
    .withColumn("RevenueRank", rank().over(window_spec))

# Save top performers
gold_products_path = f"{gold_path}product_performance/"
product_ranked.write \
    .mode("overwrite") \
    .format("delta") \
    .save(gold_products_path)

# Show top 10 products
product_ranked.filter(col("RevenueRank") <= 10).show(truncate=False)
```

### Customer RFM Segmentation

```python
from pyspark.sql.functions import max as spark_max, min as spark_min, datediff, current_date

# Calculate RFM metrics
customer_rfm = silver_orders \
    .filter(col("IsReturn") == False) \
    .filter(col("CustomerID") != "GUEST") \
    .groupBy("CustomerID") \
    .agg(
        datediff(current_date(), spark_max("InvoiceDate")).alias("Recency"),
        count("InvoiceNo").alias("Frequency"),
        spark_round(sum("LineTotal"), 2).alias("Monetary")
    )

# Define RFM segments using quartiles
from pyspark.sql.functions import when, percentile_approx

# Calculate quartile thresholds
r_quartiles = customer_rfm.approxQuantile("Recency", [0.25, 0.5, 0.75], 0.01)
f_quartiles = customer_rfm.approxQuantile("Frequency", [0.25, 0.5, 0.75], 0.01)
m_quartiles = customer_rfm.approxQuantile("Monetary", [0.25, 0.5, 0.75], 0.01)

# Assign RFM scores (1-4)
customer_segmented = customer_rfm \
    .withColumn("R_Score", 
        when(col("Recency") <= r_quartiles[0], 4)
        .when(col("Recency") <= r_quartiles[1], 3)
        .when(col("Recency") <= r_quartiles[2], 2)
        .otherwise(1)
    ) \
    .withColumn("F_Score",
        when(col("Frequency") >= f_quartiles[2], 4)
        .when(col("Frequency") >= f_quartiles[1], 3)
        .when(col("Frequency") >= f_quartiles[0], 2)
        .otherwise(1)
    ) \
    .withColumn("M_Score",
        when(col("Monetary") >= m_quartiles[2], 4)
        .when(col("Monetary") >= m_quartiles[1], 3)
        .when(col("Monetary") >= m_quartiles[0], 2)
        .otherwise(1)
    )

# Define customer segments
customer_segmented = customer_segmented \
    .withColumn("Segment",
        when((col("R_Score") >= 3) & (col("F_Score") >= 3) & (col("M_Score") >= 3), "Champions")
        .when((col("R_Score") >= 3) & (col("F_Score") <= 2), "Potential Loyalists")
        .when((col("R_Score") <= 2) & (col("F_Score") >= 3), "At Risk")
        .when((col("R_Score") <= 2) & (col("F_Score") <= 2), "Lost")
        .otherwise("Regular")
    )

# Save RFM segmentation
gold_rfm_path = f"{gold_path}customer_rfm/"
customer_segmented.write \
    .mode("overwrite") \
    .format("delta") \
    .save(gold_rfm_path)
```

## Semantic Model Configuration

### Create Semantic Model in Fabric

1. **Navigate to Lakehouse** in Fabric workspace
2. **Select "New Semantic Model"** from ribbon
3. **Choose Gold layer tables** to include
4. **Define relationships** between fact and dimension tables

### Define DAX Measures

```dax
// Total Revenue
Total Revenue = 
SUMX(
    'revenue_trends',
    'revenue_trends'[TotalRevenue]
)

// Year-over-Year Growth
YoY Revenue Growth % = 
VAR CurrentYearRevenue = 
    CALCULATE([Total Revenue], YEAR('revenue_trends'[Year]) = MAX('revenue_trends'[Year]))
VAR PreviousYearRevenue = 
    CALCULATE([Total Revenue], YEAR('revenue_trends'[Year]) = MAX('revenue_trends'[Year]) - 1)
RETURN
    DIVIDE(CurrentYearRevenue - PreviousYearRevenue, PreviousYearRevenue, 0) * 100

// Average Order Value
Avg Order Value = 
AVERAGEX(
    'revenue_trends',
    'revenue_trends'[AvgOrderValue]
)

// Customer Count by Segment
Customer Count = 
COUNTROWS('customer_rfm')

// Top Product Revenue
Top Product Revenue = 
CALCULATE(
    SUM('product_performance'[TotalRevenue]),
    TOPN(10, 'product_performance', 'product_performance'[RevenueRank], ASC)
)
```

### Configure Relationships

```python
# Define relationships programmatically (using Tabular Object Model API)
# Note: This is typically done in Fabric UI, but can be scripted

relationships = {
    "revenue_trends_to_date": {
        "from_table": "revenue_trends",
        "from_column": "Year, Month",
        "to_table": "dim_date",
        "to_column": "Year, Month",
        "cardinality": "many_to_one"
    },
    "orders_to_customers": {
        "from_table": "orders_clean",
        "from_column": "CustomerID",
        "to_table": "customer_rfm",
        "to_column": "CustomerID",
        "cardinality": "many_to_one"
    }
}
```

## Power BI Integration

### Connect Power BI to Semantic Model

1. **Open Power BI Desktop**
2. **Get Data** → **Power Platform** → **Microsoft Fabric**
3. **Select your workspace** and semantic model
4. **Import or DirectQuery mode**

### Create Visuals Using Semantic Model

```dax
-- Use pre-defined measures from semantic model
-- Revenue card visual
Total Revenue (Card) = [Total Revenue]

-- Revenue trend line chart
Revenue by Month (Line Chart):
X-axis: 'revenue_trends'[Month]
Y-axis: [Total Revenue]
Legend: 'revenue_trends'[Year]

-- Product performance table
Top 10 Products (Table):
Columns: 
  'product_performance'[Description]
  'product_performance'[TotalRevenue]
  'product_performance'[RevenueRank]
Filter: 'product_performance'[RevenueRank] <= 10

-- Customer segmentation donut chart
Customers by Segment (Donut):
Values: [Customer Count]
Legend: 'customer_rfm'[Segment]
```

## Complete Notebook Pipeline Example

```python
# Fabric Notebook: End-to-End Gold Layer Processing

from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.window import Window

# Initialize Spark (automatically configured in Fabric)
spark = SparkSession.builder.getOrCreate()

# Define paths
silver_path = "abfss://workspace@onelake.dfs.fabric.microsoft.com/lakehouse/Files/silver/orders_clean/"
gold_path = "abfss://workspace@onelake.dfs.fabric.microsoft.com/lakehouse/Files/gold/"

# Read Silver data
print("Reading Silver layer...")
silver_orders = spark.read.format("delta").load(silver_path)
print(f"Loaded {silver_orders.count()} records from Silver")

# Filter valid orders
valid_orders = silver_orders.filter((col("IsReturn") == False) & (col("CustomerID") != "GUEST"))

# === Gold 1: Revenue Trends ===
print("\nGenerating Revenue Trends...")
revenue_trends = valid_orders \
    .groupBy("Year", "Month") \
    .agg(
        round(sum("LineTotal"), 2).alias("TotalRevenue"),
        count("InvoiceNo").alias("OrderCount"),
        round(avg("LineTotal"), 2).alias("AvgOrderValue"),
        countDistinct("CustomerID").alias("UniqueCustomers")
    ) \
    .orderBy("Year", "Month")

revenue_trends.write.mode("overwrite").format("delta").save(f"{gold_path}revenue_trends/")
print(f"Revenue Trends: {revenue_trends.count()} records")

# === Gold 2: Product Performance ===
print("\nGenerating Product Performance...")
product_performance = valid_orders \
    .groupBy("StockCode", "Description") \
    .agg(
        round(sum("LineTotal"), 2).alias("TotalRevenue"),
        sum("Quantity").alias("TotalQuantitySold"),
        count("InvoiceNo").alias("OrderFrequency"),
        countDistinct("CustomerID").alias("UniqueCustomers")
    )

window_spec = Window.orderBy(desc("TotalRevenue"))
product_ranked = product_performance.withColumn("RevenueRank", rank().over(window_spec))

product_ranked.write.mode("overwrite").format("delta").save(f"{gold_path}product_performance/")
print(f"Product Performance: {product_ranked.count()} products")

# === Gold 3: Customer Analytics ===
print("\nGenerating Customer Analytics...")
customer_analytics = valid_orders \
    .groupBy("CustomerID", "Country") \
    .agg(
        round(sum("LineTotal"), 2).alias("TotalSpent"),
        count("InvoiceNo").alias("OrderCount"),
        round(avg("LineTotal"), 2).alias("AvgOrderValue"),
        min("InvoiceDate").alias("FirstPurchase"),
        max("InvoiceDate").alias("LastPurchase")
    ) \
    .withColumn("DaysSinceLastPurchase", datediff(current_date(), col("LastPurchase")))

customer_analytics.write.mode("overwrite").format("delta").save(f"{gold_path}customer_analytics/")
print(f"Customer Analytics: {customer_analytics.count()} customers")

# === Gold 4: RFM Segmentation ===
print("\nGenerating RFM Segmentation...")
customer_rfm = valid_orders \
    .groupBy("CustomerID") \
    .agg(
        datediff(current_date(), max("InvoiceDate")).alias("Recency"),
        count("InvoiceNo").alias("Frequency"),
        round(sum("LineTotal"), 2).alias("Monetary")
    )

# Calculate quartiles
r_quartiles = customer_rfm.approxQuantile("Recency", [0.25, 0.5, 0.75], 0.01)
f_quartiles = customer_rfm.approxQuantile("Frequency", [0.25, 0.5, 0.75], 0.01)
m_quartiles = customer_rfm.approxQuantile("Monetary", [0.25, 0.5, 0.75], 0.01)

# Assign RFM scores and segments
customer_segmented = customer_rfm \
    .withColumn("R_Score", 
        when(col("Recency") <= r_quartiles[0], 4)
        .when(col("Recency") <= r_quartiles[1], 3)
        .when(col("Recency") <= r_quartiles[2], 2)
        .otherwise(1)
    ) \
    .withColumn("F_Score",
        when(col("Frequency") >= f_quartiles[2], 4)
        .when(col("Frequency") >= f_quartiles[1], 3)
        .when(col("Frequency") >= f_quartiles[0], 2)
        .otherwise(1)
    ) \
    .withColumn("M_Score",
        when(col("Monetary") >= m_quartiles[2], 4)
        .when(col("Monetary") >= m_quartiles[1], 3)
        .when(col("Monetary") >= m_quartiles[0], 2)
        .otherwise(1)
    ) \
    .withColumn("Segment",
        when((col("R_Score") >= 3) & (col("F_Score") >= 3) & (col("M_Score") >= 3), "Champions")
        .when((col("R_Score") >= 3) & (col("F_Score") <= 2), "Potential Loyalists")
        .when((col("R_Score") <= 2) & (col("F_Score") >= 3), "At Risk")
        .when((col("R_Score") <= 2) & (col("F_Score") <= 2), "Lost")
        .otherwise("Regular")
    )

customer_segmented.write.mode("overwrite").format("delta").save(f"{gold_path}customer_rfm/")
print(f"RFM Segmentation: {customer_segmented.count()} customers")

# Summary statistics
print("\n=== Pipeline Summary ===")
print(f"Revenue Trends: {revenue_trends.count()} time periods")
print(f"Products Analyzed: {product_ranked.count()}")
print(f"Customers Segmented: {customer_segmented.count()}")
print("\nGold layer processing complete!")
```

## Delta Lake Optimization

### Optimize Delta Tables

```python
from delta.tables import DeltaTable

# Optimize Gold tables for query performance
gold_tables = [
    "revenue_trends",
    "product_performance",
    "customer_analytics",
    "customer_rfm"
]

for table_name in gold_tables:
    table_path = f"{gold_path}{table_name}/"
    print(f"Optimizing {table_name}...")
    
    # Load Delta table
    delta_table = DeltaTable.forPath(spark, table_path)
    
    # Optimize and Z-order
    delta_table.optimize().executeCompaction()
    
    # Z-order by commonly filtered columns
    if table_name == "revenue_trends":
        delta_table.optimize().executeZOrderBy("Year", "Month")
    elif table_name == "customer_rfm":
        delta_table.optimize().executeZOrderBy("Segment")
    
    print(f"{table_name} optimized")
```

### Vacuum Old Files

```python
# Clean up old Delta versions (older than 7 days)
for table_name in gold_tables:
    table_path = f"{gold_path}{table_name}/"
    delta_table = DeltaTable.forPath(spark, table_path)
    
    # Vacuum (remove files older than retention period)
    delta_table.vacuum(168)  # 168 hours = 7 days
    print(f"{table_name} vacuumed")
```

## Monitoring and Data Quality

### Data Quality Checks

```python
from pyspark.sql.functions import col, count, when, isnan, isnull

def data_quality_report(df, layer_name):
    """Generate data quality report for a DataFrame"""
    
    print(f"\n=== Data Quality Report: {layer_name} ===")
    print(f"Total Records: {df.count()}")
    
    # Null counts per column
    print("\nNull Counts:")
    null_counts = df.select([
        count(when(isnull(c) | isnan(c), c)).alias(c) 
        for c in df.columns
    ])
    null_counts.show()
    
    # Duplicate check (example: InvoiceNo + StockCode)
    if "InvoiceNo" in df.columns and "StockCode" in df.columns:
        duplicate_count = df.count() - df.dropDuplicates(["InvoiceNo", "StockCode"]).count()
        print(f"\nDuplicate Records: {duplicate_count}")
    
    # Data type validation
    print("\nSchema:")
    df.printSchema()

# Run quality checks
silver_orders = spark.read.format("delta").load(silver_orders_path)
data_quality_report(silver_orders, "Silver Orders")
```

### Pipeline Logging

```python
import logging
from datetime import datetime

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("FabricPipeline")

def log_pipeline_step(step_name, record_count, status="SUCCESS"):
    """Log pipeline execution step"""
    timestamp = datetime.now().isoformat()
    logger.info(f"[{timestamp}] {step_name} | Records: {record_count} | Status: {status}")

# Use in pipeline
log_pipeline_step("Bronze Ingestion", raw_orders_df.count())
log_pipeline_step("Silver Cleansing", silver_orders.count())
log_pipeline_step("Gold Aggregation", revenue_trends.count())
```

## Troubleshooting

### Common Issues

**Issue: Spark session not initialized**
```python
# Solution: Fabric notebooks auto-initialize Spark
# If needed, manually create session
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("FabricPipeline").getOrCreate()
```

**Issue: OneLake path not accessible**
```python
# Solution: Use correct ABFSS format
# Correct format:
path = "abfss://workspace@onelake.dfs.fabric.microsoft.com/lakehouse/Files/gold/"

# Check workspace name and lakehouse name
# Verify permissions in Fabric workspace
```

**Issue: Delta table version conflicts**
```python
# Solution: Optimize and checkpoint
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, table_path)
delta_table.optimize().executeCompaction()

# Or overwrite instead of append
df.write.mode("overwrite").format("delta").save(path)
```

**Issue: Memory errors with large datasets**
```python
# Solution: Partition data and increase executors
df.write \
    .mode("overwrite") \
    .partitionBy("Year", "Month") \
    .format("delta") \
    .save(path)

# Increase Spark config (in Fabric notebook)
spark.conf.set("spark.sql.shuffle.partitions", "200")
```

**Issue: Dataflow Gen2 timeout**
```
Solution: 
- Reduce data volume per refresh
- Optimize source queries
- Increase Fabric capacity tier
- Use incremental refresh patterns
```

## Best Practices

### Medallion Architecture Guidelines

1. **Bronze**: Immutable, preserve raw source schema
2. **Silver**: Apply data quality rules, standardize types
3. **Gold**: Business-ready aggregations, optimized for reporting

### Naming Conventions

```python
# Use descriptive, consistent names
# Bronze: {source_system}_{entity}_raw
bronze_erp_orders_raw

# Silver: {entity}_clean / {entity}_enriched
orders_clean
customers_enriched

# Gold: {business_domain}_{metric}
revenue_trends
customer_rfm_segmentation
product_performance_kpis
```

### Performance Optimization

```python
# 1. Partition large tables
df.write.partitionBy("Year", "Month").format("delta").save(path)

# 2. Cache frequently used DataFrames
df.cache()

# 3. Use broadcast for small dimension joins
from pyspark.sql.functions import broadcast
fact_df.join(broadcast(dim_df), "key")

# 4. Persist intermediate results
df.persist(StorageLevel.MEMORY_AND_DISK)
```

## Environment Variables

```python
import os

# If using external Azure resources
AZURE_STORAGE_ACCOUNT = os.getenv("AZURE_STORAGE_ACCOUNT")
AZURE_STORAGE_KEY = os.getenv("AZURE_STORAGE_KEY")

# Fabric workspace configuration
FABRIC_WORKSPACE_ID = os.getenv("FABRIC_WORKSPACE_ID")
FABRIC_LAKEHOUSE_NAME = os.getenv("FABRIC_LAKEHOUSE_NAME")

# Construct OneLake path dynamically
onelake_path = f"abfss://{FABRIC_WORKSPACE_ID}@onelake.dfs.fabric.microsoft.com/{FABRIC_LAKEHOUSE_NAME}/Files/"
```

## Additional Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/fabric/)
- [OneLake Overview](https://learn.microsoft.com/fabric/onelake/onelake-overview)
- [Dataflow Gen2 Guide](https://learn.microsoft.com/fabric/data-factory/dataflows-gen2-overview)
- [Fabric Notebooks](https://learn.microsoft.com/fabric/data-engineering/how-to-use-notebook)
- [Delta Lake Optimization](https://docs.delta.io/latest/optimizations-oss.html)

This skill provides comprehensive guidance for building unified analytics platforms with Microsoft Fabric following industry-standard Medallion Architecture patterns.
