---
name: microsoft-fabric-unified-analytics
description: End-to-end unified analytics platform using Microsoft Fabric, Lakehouse, Dataflow Gen2, PySpark, and Power BI with Medallion Architecture
triggers:
  - "how do I build a lakehouse in Microsoft Fabric"
  - "implement medallion architecture with fabric notebooks"
  - "create dataflow gen2 transformations"
  - "set up microsoft fabric analytics platform"
  - "process data with pyspark in fabric notebooks"
  - "build power bi semantic model from lakehouse"
  - "design unified analytics with onelake"
  - "transform bronze to gold layer in fabric"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in building end-to-end unified analytics platforms using Microsoft Fabric, implementing Medallion Architecture (Bronze → Silver → Gold), and orchestrating data workflows from ingestion to visualization using OneLake, Dataflow Gen2, Fabric Notebooks with PySpark, Semantic Models, and Power BI.

## What This Project Does

The Microsoft Fabric Unified Analytics Platform demonstrates:

- **Unified SaaS Analytics**: Single platform for ingestion, transformation, modeling, and visualization
- **Lakehouse Architecture**: OneLake-based storage with Medallion layers (Bronze, Silver, Gold)
- **Data Engineering Workflow**: Dataflow Gen2 for cleansing, PySpark notebooks for business logic
- **Semantic Modeling**: Centralized business layer with reusable KPIs and metrics
- **Native BI Integration**: Power BI reports connected directly to Fabric Lakehouse
- **Retail Analytics Scenario**: Complete implementation with customer, product, and sales data

## Architecture Overview

```
Raw Data (CSV/API)
    ↓
🟤 Bronze Layer (OneLake Lakehouse)
    ↓
Dataflow Gen2 (Cleansing & Enrichment)
    ↓
⚪ Silver Layer (Curated Data)
    ↓
Fabric Notebook (PySpark Business Logic)
    ↓
🟡 Gold Layer (Analytics-Ready)
    ↓
Semantic Model (Business Metrics)
    ↓
Power BI (Interactive Dashboards)
```

## Getting Started

### Prerequisites

- Microsoft Fabric workspace (Premium capacity or trial)
- Access to Microsoft Fabric Lakehouse
- Power BI Pro or Premium Per User license
- Python 3.x (for local notebook development)
- Basic knowledge of PySpark and SQL

### Initial Setup

1. **Create Fabric Workspace**:
   - Navigate to Microsoft Fabric portal
   - Create new workspace with Fabric capacity
   - Enable necessary features (Lakehouse, Data Engineering, Power BI)

2. **Create Lakehouse**:
   - In workspace, create new Lakehouse
   - Name it appropriately (e.g., `RetailAnalyticsLakehouse`)
   - OneLake storage is automatically provisioned

3. **Set Up Folder Structure**:
   - Create folders in Lakehouse: `bronze/`, `silver/`, `gold/`
   - Organize by entity: `bronze/sales/`, `bronze/customers/`, `bronze/products/`

## Medallion Architecture Implementation

### Bronze Layer (Raw Data Ingestion)

Upload raw CSV files or ingest via Dataflow Gen2:

```python
# Fabric Notebook: Load raw data to Bronze layer
from pyspark.sql import SparkSession

# Initialize Spark session (auto-configured in Fabric)
spark = SparkSession.builder.getOrCreate()

# Read raw CSV from Bronze layer
bronze_df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("Files/bronze/sales/online_retail.csv")

# Display schema and sample
bronze_df.printSchema()
bronze_df.show(5)

# Write to Bronze delta table (optional, for Delta Lake optimization)
bronze_df.write.format("delta") \
    .mode("overwrite") \
    .save("Tables/bronze_sales")

print(f"Bronze layer loaded: {bronze_df.count()} records")
```

### Silver Layer (Cleansing with Dataflow Gen2)

Create Dataflow Gen2 in Fabric workspace:

1. **Create New Dataflow Gen2**
2. **Connect to Bronze Layer** as source
3. **Apply Transformations**:
   - Remove duplicates
   - Handle missing values
   - Standardize data types
   - Add calculated columns

**Example Transformations (Power Query M)**:

```m
// Remove duplicates based on InvoiceNo and StockCode
#"Removed Duplicates" = Table.Distinct(Source, {"InvoiceNo", "StockCode"}),

// Filter out null CustomerID
#"Filtered Nulls" = Table.SelectRows(#"Removed Duplicates", each [CustomerID] <> null),

// Add line_total column
#"Added Line Total" = Table.AddColumn(#"Filtered Nulls", "line_total", 
    each [Quantity] * [UnitPrice], type number),

// Extract year and month
#"Added Year" = Table.AddColumn(#"Added Line Total", "year", 
    each Date.Year([InvoiceDate]), Int64.Type),
#"Added Month" = Table.AddColumn(#"Added Year", "month", 
    each Date.Month([InvoiceDate]), Int64.Type),

// Identify returns
#"Added IsReturn" = Table.AddColumn(#"Added Month", "is_return", 
    each if Text.Contains([InvoiceNo], "C") then true else false, type logical)
```

4. **Set Destination**: Silver layer folder in Lakehouse
5. **Publish and Refresh**

### Gold Layer (Business Logic with PySpark)

Fabric Notebook for creating analytics-ready datasets:

```python
# Fabric Notebook: Transform Silver to Gold layer
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Read Silver layer data
silver_df = spark.read.format("delta").load("Tables/silver_sales")

# ====================
# Gold: Revenue KPIs
# ====================
revenue_kpis = silver_df.filter(~F.col("is_return")) \
    .groupBy("year", "month") \
    .agg(
        F.sum("line_total").alias("total_revenue"),
        F.count("InvoiceNo").alias("total_orders"),
        F.countDistinct("CustomerID").alias("unique_customers"),
        F.sum("Quantity").alias("total_quantity")
    ) \
    .withColumn("avg_order_value", F.col("total_revenue") / F.col("total_orders")) \
    .orderBy("year", "month")

# Write to Gold layer
revenue_kpis.write.format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Tables/gold_revenue_kpis")

print("Gold: Revenue KPIs created")

# ====================
# Gold: Product Performance
# ====================
product_performance = silver_df.filter(~F.col("is_return")) \
    .groupBy("StockCode", "Description") \
    .agg(
        F.sum("line_total").alias("total_revenue"),
        F.sum("Quantity").alias("total_quantity_sold"),
        F.countDistinct("CustomerID").alias("unique_customers")
    ) \
    .withColumn("avg_price", F.col("total_revenue") / F.col("total_quantity_sold")) \
    .orderBy(F.col("total_revenue").desc())

product_performance.write.format("delta") \
    .mode("overwrite") \
    .save("Tables/gold_product_performance")

print("Gold: Product Performance created")

# ====================
# Gold: Customer RFM Segmentation
# ====================
from datetime import datetime

# Calculate reference date (last date in dataset + 1 day)
max_date = silver_df.agg(F.max("InvoiceDate")).collect()[0][0]
reference_date = max_date + pd.Timedelta(days=1)

# Calculate RFM metrics
rfm_df = silver_df.filter(~F.col("is_return")) \
    .groupBy("CustomerID") \
    .agg(
        F.datediff(F.lit(reference_date), F.max("InvoiceDate")).alias("recency"),
        F.countDistinct("InvoiceNo").alias("frequency"),
        F.sum("line_total").alias("monetary")
    )

# Assign RFM scores (1-5 scale, 5 is best)
rfm_window = Window.orderBy("recency")
frequency_window = Window.orderBy(F.col("frequency").desc())
monetary_window = Window.orderBy(F.col("monetary").desc())

rfm_scored = rfm_df \
    .withColumn("R_score", F.ntile(5).over(rfm_window)) \
    .withColumn("F_score", F.ntile(5).over(frequency_window)) \
    .withColumn("M_score", F.ntile(5).over(monetary_window)) \
    .withColumn("RFM_score", F.concat(F.col("R_score"), F.col("F_score"), F.col("M_score")))

# Segment customers
def segment_customer(rfm_score):
    r, f, m = int(rfm_score[0]), int(rfm_score[1]), int(rfm_score[2])
    if r >= 4 and f >= 4 and m >= 4:
        return "Champions"
    elif r >= 3 and f >= 3:
        return "Loyal Customers"
    elif r >= 4:
        return "Promising"
    elif r <= 2 and f >= 3:
        return "At Risk"
    elif r <= 2 and f <= 2:
        return "Lost"
    else:
        return "Need Attention"

segment_udf = F.udf(segment_customer)
rfm_segmented = rfm_scored.withColumn("customer_segment", segment_udf(F.col("RFM_score")))

rfm_segmented.write.format("delta") \
    .mode("overwrite") \
    .save("Tables/gold_customer_rfm")

print("Gold: Customer RFM Segmentation created")

# ====================
# Gold: Sales Trends
# ====================
sales_trends = silver_df.filter(~F.col("is_return")) \
    .groupBy("year", "month", "Country") \
    .agg(
        F.sum("line_total").alias("revenue"),
        F.count("InvoiceNo").alias("order_count")
    ) \
    .orderBy("year", "month")

sales_trends.write.format("delta") \
    .mode("overwrite") \
    .save("Tables/gold_sales_trends")

print("Gold: Sales Trends created")

# Display summary
print("\n=== Gold Layer Summary ===")
print(f"Revenue KPIs: {revenue_kpis.count()} records")
print(f"Product Performance: {product_performance.count()} records")
print(f"Customer RFM: {rfm_segmented.count()} records")
print(f"Sales Trends: {sales_trends.count()} records")
```

## Semantic Model Configuration

1. **Create Semantic Model** from Lakehouse:
   - In Fabric Lakehouse, select "New Semantic Model"
   - Select Gold layer tables
   - Define relationships

2. **Define Relationships**:

```dax
// In Semantic Model, create relationships between tables
// Example: gold_revenue_kpis[year] → dim_date[year]
// Example: gold_product_performance[StockCode] → dim_products[StockCode]
```

3. **Create Measures**:

```dax
// Total Revenue Measure
Total Revenue = 
CALCULATE(
    SUM(gold_revenue_kpis[total_revenue])
)

// Year-over-Year Growth
YoY Revenue Growth = 
VAR CurrentYearRevenue = [Total Revenue]
VAR PreviousYearRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD(dim_date[Date], -1, YEAR)
    )
RETURN
DIVIDE(
    CurrentYearRevenue - PreviousYearRevenue,
    PreviousYearRevenue
)

// Average Order Value
Average Order Value = 
DIVIDE(
    [Total Revenue],
    SUM(gold_revenue_kpis[total_orders])
)

// Customer Lifetime Value
Customer LTV = 
AVERAGEX(
    gold_customer_rfm,
    gold_customer_rfm[monetary]
)
```

## Power BI Integration

Connect Power BI Desktop or Service to Semantic Model:

1. **Open Power BI Desktop**
2. **Get Data** → **Power Platform** → **Microsoft Fabric**
3. **Select Semantic Model** from workspace
4. **Build Reports** using Gold layer tables and measures

**Example DAX for Report Calculations**:

```dax
// Customer Segment Distribution
Customer Count by Segment = 
COUNTROWS(gold_customer_rfm)

// Top 10 Products by Revenue
Top 10 Products = 
TOPN(
    10,
    gold_product_performance,
    gold_product_performance[total_revenue],
    DESC
)

// Monthly Revenue Trend
Monthly Revenue = 
CALCULATE(
    SUM(gold_revenue_kpis[total_revenue]),
    ALLEXCEPT(gold_revenue_kpis, gold_revenue_kpis[year], gold_revenue_kpis[month])
)
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
# Fabric Notebook: Incremental load to Silver layer
from delta.tables import DeltaTable

# Define checkpoint for incremental load
checkpoint_path = "Files/checkpoints/silver_sales"

# Read new data from Bronze with watermark
new_data = spark.read.format("delta") \
    .load("Tables/bronze_sales") \
    .filter(F.col("load_timestamp") > last_checkpoint)

# Merge into Silver (upsert)
if DeltaTable.isDeltaTable(spark, "Tables/silver_sales"):
    silver_table = DeltaTable.forPath(spark, "Tables/silver_sales")
    
    silver_table.alias("target").merge(
        new_data.alias("source"),
        "target.InvoiceNo = source.InvoiceNo AND target.StockCode = source.StockCode"
    ).whenMatchedUpdateAll() \
     .whenNotMatchedInsertAll() \
     .execute()
else:
    new_data.write.format("delta").save("Tables/silver_sales")

print(f"Incremental load completed: {new_data.count()} records processed")
```

### Pattern 2: Data Quality Checks

```python
# Fabric Notebook: Data quality validation
def validate_data_quality(df, table_name):
    """Run data quality checks on DataFrame"""
    
    checks = {
        "total_records": df.count(),
        "null_customer_ids": df.filter(F.col("CustomerID").isNull()).count(),
        "negative_quantities": df.filter(F.col("Quantity") < 0).count(),
        "negative_prices": df.filter(F.col("UnitPrice") < 0).count(),
        "duplicate_records": df.count() - df.dropDuplicates(["InvoiceNo", "StockCode"]).count()
    }
    
    print(f"\n=== Data Quality Report: {table_name} ===")
    for check, value in checks.items():
        status = "✓ PASS" if value == 0 or check == "total_records" else "✗ FAIL"
        print(f"{check}: {value:,} {status}")
    
    return checks

# Run validation
quality_report = validate_data_quality(silver_df, "silver_sales")

# Write quality report to logs
quality_df = spark.createDataFrame([{
    "table_name": "silver_sales",
    "check_timestamp": datetime.now(),
    **quality_report
}])

quality_df.write.format("delta") \
    .mode("append") \
    .save("Tables/data_quality_logs")
```

### Pattern 3: Parameterized Notebook Execution

```python
# Fabric Notebook: Parameterized processing
import sys

# Get parameters (can be passed from Fabric Pipeline)
layer = sys.argv[1] if len(sys.argv) > 1 else "gold"
entity = sys.argv[2] if len(sys.argv) > 2 else "sales"
refresh_mode = sys.argv[3] if len(sys.argv) > 3 else "full"

print(f"Processing: layer={layer}, entity={entity}, mode={refresh_mode}")

# Dynamic path construction
source_path = f"Tables/{layer}_{entity}"
target_path = f"Tables/{layer}_processed_{entity}"

# Process based on parameters
if refresh_mode == "full":
    df = spark.read.format("delta").load(source_path)
    # Full refresh logic
elif refresh_mode == "incremental":
    # Incremental logic
    pass

print(f"Processing completed for {entity}")
```

## Configuration

### Environment Variables

Set in Fabric workspace or notebook:

```python
# Access Fabric environment settings
import os

# Lakehouse connection (auto-configured in Fabric)
LAKEHOUSE_ID = os.getenv("LAKEHOUSE_ID")
WORKSPACE_ID = os.getenv("WORKSPACE_ID")

# Custom configurations
BATCH_SIZE = int(os.getenv("BATCH_SIZE", "1000000"))
PROCESSING_MODE = os.getenv("PROCESSING_MODE", "batch")
```

### Delta Table Optimization

```python
# Optimize Delta tables for performance
from delta.tables import DeltaTable

# Optimize Bronze table
deltaTable = DeltaTable.forPath(spark, "Tables/bronze_sales")
deltaTable.optimize().executeCompaction()

# Z-order optimization for common filter columns
deltaTable.optimize().executeZOrderBy("InvoiceDate", "CustomerID")

# Vacuum old files (remove files older than retention period)
deltaTable.vacuum(retentionHours=168)  # 7 days

print("Delta table optimization completed")
```

## Troubleshooting

### Issue: Dataflow Gen2 Refresh Fails

**Symptoms**: Dataflow refresh errors, timeout issues

**Solutions**:
- Check source data connectivity
- Reduce data volume per refresh (add incremental load)
- Verify destination Lakehouse permissions
- Review Power Query M code for errors

```m
// Add error handling in Dataflow
try 
    #"Final Output"
otherwise 
    #table(type table [Error = text], {{"Data load failed"}})
```

### Issue: PySpark Notebook Memory Errors

**Symptoms**: `OutOfMemoryError`, executor failures

**Solutions**:
- Increase Fabric capacity or upgrade workspace
- Use `.repartition()` for large datasets
- Implement batch processing
- Cache intermediate results strategically

```python
# Repartition for large datasets
large_df = spark.read.format("delta").load("Tables/bronze_large") \
    .repartition(100)

# Cache frequently used DataFrames
silver_df.cache()
silver_df.count()  # Trigger caching

# Process in batches
batch_size = 1000000
for batch in range(0, total_records, batch_size):
    batch_df = silver_df.limit(batch_size).offset(batch)
    # Process batch
```

### Issue: Semantic Model Refresh Slow

**Symptoms**: Long refresh times, timeout errors

**Solutions**:
- Optimize Gold layer queries
- Use DirectQuery for large tables
- Implement incremental refresh
- Add indexes on key columns

```python
# Optimize Gold tables with indexing
spark.sql("""
    CREATE INDEX idx_revenue_date 
    ON gold_revenue_kpis (year, month)
""")
```

### Issue: Power BI Report Performance

**Symptoms**: Slow visuals, long load times

**Solutions**:
- Use aggregations in Semantic Model
- Implement calculated tables for complex calculations
- Optimize DAX measures
- Use composite models with aggregations

```dax
// Create aggregation table in Semantic Model
Revenue Summary = 
SUMMARIZE(
    gold_revenue_kpis,
    gold_revenue_kpis[year],
    gold_revenue_kpis[month],
    "Total Revenue", SUM(gold_revenue_kpis[total_revenue]),
    "Total Orders", SUM(gold_revenue_kpis[total_orders])
)
```

## Best Practices

1. **Medallion Architecture Discipline**: Keep Bronze immutable, Silver cleansed, Gold business-ready
2. **Delta Lake Everywhere**: Use Delta format for ACID transactions and time travel
3. **Incremental Processing**: Implement watermarks to avoid full table scans
4. **Data Quality Gates**: Validate data at each layer boundary
5. **Semantic Model Governance**: Centralize business logic and metrics
6. **Optimize for BI**: Design Gold layer specifically for reporting needs
7. **Monitor Performance**: Track notebook execution times and optimize bottlenecks
8. **Version Control**: Store notebook code in Git for collaboration

## Additional Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/fabric/)
- [Delta Lake Documentation](https://docs.delta.io/)
- [PySpark API Reference](https://spark.apache.org/docs/latest/api/python/)
- [Power Query M Reference](https://learn.microsoft.com/powerquery-m/)
- [DAX Reference](https://dax.guide/)

---

This skill equips AI coding agents with comprehensive knowledge to implement unified analytics platforms using Microsoft Fabric, from raw data ingestion through to interactive business intelligence dashboards.
