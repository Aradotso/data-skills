---
name: microsoft-fabric-unified-analytics
description: End-to-end unified analytics platform using Microsoft Fabric with Lakehouse, Medallion Architecture, Dataflow Gen2, PySpark notebooks, and Power BI for retail analytics.
triggers:
  - "build a microsoft fabric analytics pipeline"
  - "implement medallion architecture in fabric"
  - "create fabric lakehouse with bronze silver gold layers"
  - "use dataflow gen2 for data transformation"
  - "write pyspark notebooks in microsoft fabric"
  - "design unified analytics platform with onelake"
  - "connect power bi to fabric semantic model"
  - "set up fabric lakehouse architecture"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates how to build a production-grade unified analytics platform using **Microsoft Fabric**, implementing the **Medallion Architecture** (Bronze → Silver → Gold) with OneLake storage, Dataflow Gen2 for ETL, PySpark notebooks for transformations, Semantic Models, and Power BI for visualization.

**Key Features:**
- Unified SaaS analytics platform (no separate services to integrate)
- Lakehouse architecture with OneLake storage
- Multi-layer data processing (Bronze/Silver/Gold)
- Low-code transformations with Dataflow Gen2
- Code-based transformations with PySpark notebooks
- Semantic modeling for business KPIs
- Native Power BI integration

## Architecture

The solution follows a layered approach:

1. **Bronze Layer**: Raw data ingestion into OneLake
2. **Silver Layer**: Cleansed and enriched data via Dataflow Gen2
3. **Gold Layer**: Business-ready datasets via PySpark notebooks
4. **Semantic Model**: Centralized business metrics and KPIs
5. **Power BI**: Interactive dashboards and reports

## Prerequisites

- Microsoft Fabric workspace (Fabric trial or paid capacity)
- Power BI license
- Source data (CSV files or database connection)
- Basic knowledge of Python, PySpark, and Power Query

## Setup

### 1. Create Fabric Workspace

```text
1. Navigate to Microsoft Fabric portal (app.fabric.microsoft.com)
2. Create new workspace
3. Select Fabric capacity or trial
4. Enable required features (Lakehouse, Dataflow Gen2, Notebooks)
```

### 2. Create Lakehouse

```text
1. In workspace, click "+ New"
2. Select "Lakehouse"
3. Name it (e.g., "RetailAnalyticsLakehouse")
4. Create folder structure:
   - Bronze/
   - Silver/
   - Gold/
```

### 3. Set Up OneLake Storage

OneLake is automatically provisioned with your Lakehouse. Access it via:

```text
Lakehouse → Files → Bronze/Silver/Gold folders
```

## Data Ingestion (Bronze Layer)

### Upload Raw Data

**Option 1: Upload Files**
```text
1. Navigate to Lakehouse → Files → Bronze
2. Click "Upload" → "Upload files"
3. Select source CSV files
4. Data is stored in Delta Lake format
```

**Option 2: Dataflow Gen2 (for database sources)**
```text
1. Create new Dataflow Gen2
2. Get data from SQL Server, Azure SQL, etc.
3. No transformations at Bronze stage
4. Destination: Lakehouse → Bronze folder
5. Publish dataflow
```

## Data Cleansing (Silver Layer with Dataflow Gen2)

### Create Dataflow Gen2

```text
1. New → Dataflow Gen2
2. Get data → Lakehouse → Bronze tables
3. Apply Power Query transformations
```

### Common Transformations

**Remove Duplicates:**
```powerquery
= Table.Distinct(Source, {"InvoiceNo", "StockCode"})
```

**Handle Missing Values:**
```powerquery
= Table.RemoveRowsWithErrors(Source, {"Quantity", "UnitPrice"})
= Table.ReplaceValue(Source, null, 0, Replacer.ReplaceValue, {"CustomerID"})
```

**Change Data Types:**
```powerquery
= Table.TransformColumnTypes(Source, {
    {"InvoiceDate", type datetime},
    {"Quantity", Int64.Type},
    {"UnitPrice", type number}
})
```

**Add Calculated Columns:**
```powerquery
= Table.AddColumn(Source, "line_total", each [Quantity] * [UnitPrice], type number)
= Table.AddColumn(#"Added line_total", "year", each Date.Year([InvoiceDate]), Int64.Type)
= Table.AddColumn(#"Added year", "month", each Date.Month([InvoiceDate]), Int64.Type)
= Table.AddColumn(#"Added month", "is_return", each if [Quantity] < 0 then true else false, type logical)
```

**Data Destination:**
```text
1. Click on query → "+" icon → "Data destination"
2. Select "Lakehouse"
3. Choose Silver folder
4. Set table name (e.g., "online_retail_cleaned")
5. Update method: Replace
6. Publish dataflow
```

## Business Transformations (Gold Layer with PySpark)

### Create Fabric Notebook

```text
1. New → Notebook
2. Attach to Lakehouse
3. Runtime: Spark 3.4 or later
```

### PySpark Code Examples

**Read Silver Data:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

# Read from Silver layer
df = spark.read.format("delta").load("Tables/online_retail_cleaned")
df.show(5)
```

**Revenue Aggregation:**
```python
# Daily revenue trends
revenue_daily = df.groupBy("InvoiceDate") \
    .agg(
        sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("order_count"),
        countDistinct("CustomerID").alias("unique_customers")
    ) \
    .orderBy("InvoiceDate")

# Write to Gold layer
revenue_daily.write.format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("Tables/gold_revenue_daily")
```

**Product Performance Analysis:**
```python
# Top products by revenue
product_performance = df.groupBy("StockCode", "Description") \
    .agg(
        sum("line_total").alias("total_revenue"),
        sum("Quantity").alias("total_quantity"),
        countDistinct("InvoiceNo").alias("order_count")
    ) \
    .withColumn("avg_order_value", col("total_revenue") / col("order_count")) \
    .orderBy(desc("total_revenue"))

# Save to Gold
product_performance.write.format("delta") \
    .mode("overwrite") \
    .save("Tables/gold_product_performance")
```

**Customer Segmentation (RFM Analysis):**
```python
from pyspark.sql.window import Window

# Calculate Recency, Frequency, Monetary
reference_date = df.agg(max("InvoiceDate")).collect()[0][0]

customer_rfm = df.filter(col("Quantity") > 0) \
    .groupBy("CustomerID") \
    .agg(
        datediff(lit(reference_date), max("InvoiceDate")).alias("recency"),
        countDistinct("InvoiceNo").alias("frequency"),
        sum("line_total").alias("monetary")
    )

# RFM scoring (1-5 scale)
rfm_quantiles = customer_rfm.approxQuantile(
    ["recency", "frequency", "monetary"],
    [0.2, 0.4, 0.6, 0.8],
    0.01
)

def score_rfm(value, quantiles, reverse=False):
    if reverse:
        quantiles = quantiles[::-1]
    if value <= quantiles[0]:
        return 5
    elif value <= quantiles[1]:
        return 4
    elif value <= quantiles[2]:
        return 3
    elif value <= quantiles[3]:
        return 2
    else:
        return 1

from pyspark.sql.types import IntegerType
score_udf = udf(lambda v, q: score_rfm(v, q), IntegerType())
score_udf_rev = udf(lambda v, q: score_rfm(v, q, True), IntegerType())

customer_segments = customer_rfm \
    .withColumn("r_score", score_udf_rev(col("recency"), lit(rfm_quantiles[0]))) \
    .withColumn("f_score", score_udf(col("frequency"), lit(rfm_quantiles[1]))) \
    .withColumn("m_score", score_udf(col("monetary"), lit(rfm_quantiles[2]))) \
    .withColumn("rfm_score", concat(col("r_score"), col("f_score"), col("m_score")))

# Segment classification
customer_segments = customer_segments \
    .withColumn("segment", 
        when((col("r_score") >= 4) & (col("f_score") >= 4), "Champions")
        .when((col("r_score") >= 4) & (col("f_score") >= 2), "Loyal Customers")
        .when((col("r_score") >= 3) & (col("m_score") >= 3), "Potential Loyalists")
        .when(col("r_score") >= 4, "Recent Customers")
        .when(col("r_score") <= 2, "At Risk")
        .otherwise("Others")
    )

# Save to Gold
customer_segments.write.format("delta") \
    .mode("overwrite") \
    .save("Tables/gold_customer_segments")
```

**Monthly KPIs:**
```python
# Business KPIs aggregated monthly
monthly_kpis = df.groupBy("year", "month") \
    .agg(
        sum("line_total").alias("revenue"),
        count("InvoiceNo").alias("total_orders"),
        countDistinct("CustomerID").alias("unique_customers"),
        sum(when(col("is_return") == True, 1).otherwise(0)).alias("return_count")
    ) \
    .withColumn("avg_order_value", col("revenue") / col("total_orders")) \
    .withColumn("return_rate", col("return_count") / col("total_orders")) \
    .orderBy("year", "month")

monthly_kpis.write.format("delta") \
    .mode("overwrite") \
    .save("Tables/gold_monthly_kpis")
```

## Semantic Model Configuration

### Create Semantic Model

```text
1. Navigate to Lakehouse
2. Go to "Semantic model (default)"
3. Or create new Semantic Model
4. Add Gold layer tables
```

### Define Relationships

```text
1. Open Semantic Model in Model view
2. Create relationships between fact and dimension tables
   - gold_revenue_daily → date dimension
   - gold_product_performance → product dimension
   - gold_customer_segments → customer dimension
```

### Create Measures (DAX)

**Total Revenue:**
```dax
Total Revenue = SUM(gold_revenue_daily[total_revenue])
```

**Average Order Value:**
```dax
Avg Order Value = DIVIDE([Total Revenue], [Total Orders])
```

**Customer Retention Rate:**
```dax
Retention Rate = 
DIVIDE(
    CALCULATE(DISTINCTCOUNT(gold_customer_segments[CustomerID]), 
              gold_customer_segments[segment] <> "At Risk"),
    DISTINCTCOUNT(gold_customer_segments[CustomerID])
)
```

**Month-over-Month Growth:**
```dax
MoM Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD('Date'[Date], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue)
```

## Power BI Report Creation

### Connect to Semantic Model

```text
1. Open Power BI Desktop
2. Get Data → Power BI semantic models
3. Select your Fabric Semantic Model
4. Import or DirectQuery mode
```

### Create Visualizations

**Revenue Trend Chart:**
```text
1. Add Line Chart
2. Axis: InvoiceDate
3. Values: Total Revenue
4. Add trend line
```

**Top Products Table:**
```text
1. Add Table visual
2. Columns: Description, total_revenue, total_quantity
3. Sort by total_revenue descending
4. Add conditional formatting
```

**Customer Segmentation Pie Chart:**
```text
1. Add Pie Chart
2. Legend: segment
3. Values: Count of CustomerID
```

**KPI Cards:**
```text
1. Add Card visual for each KPI
   - Total Revenue
   - Unique Customers
   - Avg Order Value
   - Return Rate
2. Apply number formatting
```

### Publish to Fabric

```text
1. File → Publish → Publish to Power BI
2. Select Fabric workspace
3. Report is now accessible in Fabric portal
```

## Common Patterns

### Incremental Loading Pattern

```python
# Read last processed timestamp
last_timestamp = spark.sql("""
    SELECT MAX(InvoiceDate) as max_date 
    FROM gold_revenue_daily
""").collect()[0][0]

# Process only new records
new_data = df.filter(col("InvoiceDate") > last_timestamp)

# Append to Gold
new_data.write.format("delta") \
    .mode("append") \
    .save("Tables/gold_revenue_daily")
```

### Slowly Changing Dimension (Type 2)

```python
from delta.tables import DeltaTable

# Merge for SCD Type 2
gold_table = DeltaTable.forPath(spark, "Tables/gold_customers")

gold_table.alias("target").merge(
    new_customers.alias("source"),
    "target.CustomerID = source.CustomerID AND target.is_current = true"
).whenMatchedUpdate(
    condition="target.email != source.email OR target.country != source.country",
    set={
        "is_current": "false",
        "end_date": "current_date()"
    }
).whenNotMatchedInsert(
    values={
        "CustomerID": "source.CustomerID",
        "email": "source.email",
        "country": "source.country",
        "is_current": "true",
        "start_date": "current_date()",
        "end_date": "null"
    }
).execute()
```

### Data Quality Checks

```python
# Validate data quality before writing to Gold
def validate_data(df, table_name):
    row_count = df.count()
    null_count = df.filter(col("CustomerID").isNull()).count()
    duplicate_count = df.count() - df.dropDuplicates(["InvoiceNo", "StockCode"]).count()
    
    print(f"Quality Report for {table_name}:")
    print(f"Total Rows: {row_count}")
    print(f"Null CustomerIDs: {null_count}")
    print(f"Duplicates: {duplicate_count}")
    
    if null_count / row_count > 0.1:
        raise Exception("Too many null values!")
    
    return df

validated_df = validate_data(df, "online_retail")
```

## Scheduling and Automation

### Schedule Dataflow Gen2

```text
1. Open Dataflow Gen2
2. Click "Refresh settings"
3. Configure refresh schedule
   - Frequency: Daily, Weekly, or Custom
   - Time zones and specific times
4. Enable automatic refresh
```

### Schedule Notebook Execution

```text
1. Navigate to Notebook
2. Click "..." → "Schedule"
3. Set recurrence pattern
4. Configure time zone
5. Add notification emails
```

## Troubleshooting

### Dataflow Gen2 Refresh Failures

**Issue:** Dataflow times out or fails
**Solution:**
```text
1. Check source data size - break into smaller batches
2. Optimize Power Query steps - reduce complex calculations
3. Use query folding where possible
4. Check Fabric capacity limits
```

### PySpark Memory Errors

**Issue:** OutOfMemoryError in notebooks
**Solution:**
```python
# Use partitioning
df.repartition(10, "CustomerID") \
    .write.format("delta") \
    .mode("overwrite") \
    .save("Tables/output")

# Or use coalesce for smaller datasets
df.coalesce(1).write.format("delta").save("Tables/output")

# Increase session memory (requires capacity admin)
spark.conf.set("spark.executor.memory", "8g")
```

### Delta Lake Version Conflicts

**Issue:** Schema mismatch when writing to Delta
**Solution:**
```python
# Enable schema evolution
df.write.format("delta") \
    .mode("overwrite") \
    .option("mergeSchema", "true") \
    .option("overwriteSchema", "true") \
    .save("Tables/output")
```

### Power BI Performance Issues

**Issue:** Slow report loading
**Solution:**
```text
1. Use DirectQuery instead of Import for large datasets
2. Create aggregations in Semantic Model
3. Optimize DAX measures - avoid complex calculated columns
4. Use incremental refresh policies
5. Enable query reduction in report settings
```

### OneLake Access Issues

**Issue:** Cannot access Lakehouse tables
**Solution:**
```text
1. Check workspace roles (Admin, Member, Contributor)
2. Verify Lakehouse sharing settings
3. Refresh Lakehouse schema
4. Check Fabric capacity is running
```

## Best Practices

1. **Medallion Architecture**: Always maintain Bronze/Silver/Gold separation
2. **Schema Evolution**: Use `mergeSchema` for backward compatibility
3. **Partitioning**: Partition large tables by date or key column
4. **Documentation**: Comment notebooks and name tables descriptively
5. **Error Handling**: Implement try-except blocks in notebooks
6. **Monitoring**: Set up refresh notifications for data pipelines
7. **Security**: Use workspace roles and sharing policies appropriately
8. **Version Control**: Export notebooks to Git repositories when possible

## Reference

- [Microsoft Fabric Documentation](https://learn.microsoft.com/fabric/)
- [Dataflow Gen2 Guide](https://learn.microsoft.com/fabric/data-factory/dataflows-gen2-overview)
- [PySpark in Fabric](https://learn.microsoft.com/fabric/data-engineering/author-execute-notebook)
- [OneLake Storage](https://learn.microsoft.com/fabric/onelake/onelake-overview)
