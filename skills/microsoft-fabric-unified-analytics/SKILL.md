---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering with Microsoft Fabric using Lakehouse, Medallion Architecture, Dataflow Gen2, PySpark, and Power BI for unified analytics platforms.
triggers:
  - "How do I build a lakehouse in Microsoft Fabric?"
  - "Show me how to implement Medallion Architecture with Fabric"
  - "Help me create a Dataflow Gen2 transformation"
  - "How do I use PySpark in Fabric Notebooks?"
  - "Set up a semantic model in Microsoft Fabric"
  - "Build an end-to-end analytics platform with Fabric"
  - "Transform data from bronze to silver to gold layers"
  - "Create a unified analytics solution with OneLake"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end unified analytics platforms using Microsoft Fabric. It covers Lakehouse architecture, Medallion data organization (Bronze → Silver → Gold), Dataflow Gen2 transformations, PySpark processing in Fabric Notebooks, Semantic Models, and Power BI integration.

## What This Project Does

The Microsoft Fabric Unified Analytics Platform demonstrates how to build a production-grade analytics solution entirely within Microsoft Fabric's unified SaaS environment. It eliminates the complexity of managing multiple disconnected cloud services by providing:

- **Unified Storage**: OneLake as the single source of truth
- **Lakehouse Architecture**: Delta Lake-based storage optimized for analytics
- **Medallion Layers**: Bronze (raw), Silver (cleansed), Gold (business-ready)
- **Low-Code ETL**: Dataflow Gen2 for visual transformations
- **Code-First Processing**: Fabric Notebooks with PySpark for complex logic
- **Business Layer**: Semantic Models for consistent KPIs
- **Native BI**: Integrated Power BI for interactive dashboards

## Prerequisites

- **Microsoft Fabric Capacity**: Trial, F64, or higher SKU
- **Fabric Workspace**: With Lakehouse enabled
- **Data Source**: CSV, Parquet, or database connection
- **Power BI License**: Pro or PPU for report creation

## Platform Components

### 1. Lakehouse Setup

Create a Lakehouse to store data across Medallion layers:

```
Workspace → New → Lakehouse → Name: "RetailAnalytics"
```

**Folder Structure:**
```
RetailAnalytics/
├── Files/
│   ├── bronze/          # Raw ingested data
│   ├── silver/          # Cleansed data
│   └── gold/            # Business-ready datasets
└── Tables/              # Delta tables for SQL access
```

### 2. Dataflow Gen2 - Bronze to Silver

Dataflow Gen2 provides a visual Power Query interface for transformations.

**Creating a Dataflow Gen2:**

1. Navigate to Workspace → New → Dataflow Gen2
2. Get Data → Select source (Azure Blob, HTTP, Lakehouse)
3. Apply transformations using Power Query
4. Configure destination → Lakehouse table or file

**Common Transformations Pattern:**

```powerquery
let
    Source = Lakehouse.Contents(null),
    BronzeData = Source{[workspaceId="YOUR_WORKSPACE_ID"]}[Data],
    RawTable = BronzeData{[lakehouseId="YOUR_LAKEHOUSE_ID"]}[Data],
    BronzeFolder = RawTable{[Name="bronze",ItemKind="Folder"]}[Data],
    
    // Load raw CSV
    RawFile = BronzeFolder{[Name="transactions.csv"]}[Content],
    ParsedCSV = Csv.Document(RawFile,[Delimiter=",", Encoding=1252]),
    
    // Promote headers
    Headers = Table.PromoteHeaders(ParsedCSV, [PromoteAllScalars=true]),
    
    // Change data types
    TypedData = Table.TransformColumnTypes(Headers,{
        {"InvoiceNo", type text},
        {"StockCode", type text},
        {"Description", type text},
        {"Quantity", Int64.Type},
        {"InvoiceDate", type datetime},
        {"UnitPrice", type number},
        {"CustomerID", type text},
        {"Country", type text}
    }),
    
    // Remove duplicates
    RemoveDuplicates = Table.Distinct(TypedData, {"InvoiceNo", "StockCode"}),
    
    // Filter invalid records
    FilteredRows = Table.SelectRows(RemoveDuplicates, each [Quantity] <> null and [UnitPrice] <> null),
    
    // Add calculated columns
    AddLineTotal = Table.AddColumn(FilteredRows, "LineTotal", each [Quantity] * [UnitPrice], type number),
    AddYear = Table.AddColumn(AddLineTotal, "Year", each Date.Year([InvoiceDate]), Int64.Type),
    AddMonth = Table.AddColumn(AddYear, "Month", each Date.Month([InvoiceDate]), Int64.Type),
    AddIsReturn = Table.AddColumn(AddMonth, "IsReturn", each if [Quantity] < 0 then true else false, type logical)
in
    AddIsReturn
```

**Publishing to Silver Layer:**

- Data destination: Lakehouse
- Destination path: `Files/silver/transactions_cleansed.parquet`
- Update method: Replace

### 3. Fabric Notebook - Silver to Gold (PySpark)

Fabric Notebooks run on Apache Spark for scalable data processing.

**Creating a Notebook:**

```
Workspace → New → Notebook → Name: "silver_to_gold_transformations"
```

**Connecting to Lakehouse:**

```python
# Fabric automatically mounts the default lakehouse
# Access data using relative paths or spark.read

from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.window import Window
from datetime import datetime

# Read Silver data
silver_path = "Files/silver/transactions_cleansed.parquet"
df_silver = spark.read.parquet(silver_path)

print(f"Silver records loaded: {df_silver.count()}")
df_silver.printSchema()
```

**Gold Layer: Revenue Metrics**

```python
# Calculate revenue metrics by month
revenue_metrics = df_silver.filter(col("IsReturn") == False) \
    .groupBy("Year", "Month") \
    .agg(
        sum("LineTotal").alias("TotalRevenue"),
        countDistinct("InvoiceNo").alias("TotalOrders"),
        countDistinct("CustomerID").alias("UniqueCustomers"),
        avg("LineTotal").alias("AvgOrderValue")
    ) \
    .orderBy("Year", "Month")

# Save to Gold layer
gold_revenue_path = "Tables/gold_revenue_metrics"
revenue_metrics.write.mode("overwrite").format("delta").saveAsTable(gold_revenue_path)

print(f"Gold revenue metrics created: {revenue_metrics.count()} records")
```

**Gold Layer: Product Performance**

```python
# Analyze product performance
product_performance = df_silver.filter(col("IsReturn") == False) \
    .groupBy("StockCode", "Description") \
    .agg(
        sum("Quantity").alias("TotalQuantitySold"),
        sum("LineTotal").alias("TotalRevenue"),
        countDistinct("InvoiceNo").alias("OrderCount"),
        countDistinct("CustomerID").alias("UniqueCustomers"),
        avg("UnitPrice").alias("AvgUnitPrice")
    ) \
    .withColumn("RevenuePerOrder", col("TotalRevenue") / col("OrderCount")) \
    .orderBy(col("TotalRevenue").desc())

# Save to Gold layer
gold_product_path = "Tables/gold_product_performance"
product_performance.write.mode("overwrite").format("delta").saveAsTable(gold_product_path)

print(f"Gold product performance created: {product_performance.count()} products")
```

**Gold Layer: Customer Segmentation (RFM)**

```python
# Calculate RFM (Recency, Frequency, Monetary) metrics
from pyspark.sql.functions import datediff, max as spark_max, current_date

# Reference date for recency calculation
reference_date = df_silver.agg(spark_max("InvoiceDate")).collect()[0][0]

rfm_data = df_silver.filter(col("IsReturn") == False) \
    .groupBy("CustomerID") \
    .agg(
        datediff(lit(reference_date), spark_max("InvoiceDate")).alias("Recency"),
        countDistinct("InvoiceNo").alias("Frequency"),
        sum("LineTotal").alias("MonetaryValue")
    )

# Create RFM score using quantiles
rfm_quantiles = rfm_data.approxQuantile(
    ["Recency", "Frequency", "MonetaryValue"],
    [0.25, 0.5, 0.75],
    0.01
)

def score_rfm(value, quantiles, reverse=False):
    """Assign RFM score based on quantiles"""
    if reverse:  # For Recency (lower is better)
        if value <= quantiles[0]:
            return 4
        elif value <= quantiles[1]:
            return 3
        elif value <= quantiles[2]:
            return 2
        else:
            return 1
    else:  # For Frequency and Monetary (higher is better)
        if value >= quantiles[2]:
            return 4
        elif value >= quantiles[1]:
            return 3
        elif value >= quantiles[0]:
            return 2
        else:
            return 1

# Create UDF for scoring
from pyspark.sql.types import IntegerType

score_recency_udf = udf(lambda x: score_rfm(x, rfm_quantiles[0], reverse=True), IntegerType())
score_frequency_udf = udf(lambda x: score_rfm(x, rfm_quantiles[1]), IntegerType())
score_monetary_udf = udf(lambda x: score_rfm(x, rfm_quantiles[2]), IntegerType())

customer_segments = rfm_data \
    .withColumn("RecencyScore", score_recency_udf(col("Recency"))) \
    .withColumn("FrequencyScore", score_frequency_udf(col("Frequency"))) \
    .withColumn("MonetaryScore", score_monetary_udf(col("MonetaryValue"))) \
    .withColumn("RFMScore", concat(col("RecencyScore"), col("FrequencyScore"), col("MonetaryScore"))) \
    .withColumn("CustomerSegment", 
        when((col("RecencyScore") >= 3) & (col("FrequencyScore") >= 3) & (col("MonetaryScore") >= 3), "Champions")
        .when((col("RecencyScore") >= 3) & (col("FrequencyScore") >= 2), "Loyal Customers")
        .when((col("RecencyScore") >= 3), "Potential Loyalists")
        .when((col("FrequencyScore") >= 3) & (col("MonetaryScore") >= 3), "At Risk")
        .when(col("RecencyScore") <= 2, "Lost Customers")
        .otherwise("Needs Attention")
    )

# Save to Gold layer
gold_customer_path = "Tables/gold_customer_segments"
customer_segments.write.mode("overwrite").format("delta").saveAsTable(gold_customer_path)

print(f"Gold customer segments created: {customer_segments.count()} customers")
```

**Data Quality Checks**

```python
# Implement data quality validation
def validate_gold_layer(table_name):
    """Validate Gold layer data quality"""
    df = spark.read.table(table_name)
    
    checks = {
        "total_records": df.count(),
        "null_records": df.filter(df.columns[0].isNull()).count(),
        "duplicate_keys": df.count() - df.dropDuplicates([df.columns[0]]).count()
    }
    
    print(f"\n=== Data Quality Check: {table_name} ===")
    for check, value in checks.items():
        print(f"{check}: {value}")
    
    return checks

# Run validations
validate_gold_layer("gold_revenue_metrics")
validate_gold_layer("gold_product_performance")
validate_gold_layer("gold_customer_segments")
```

### 4. Semantic Model Configuration

The Semantic Model (formerly Power BI Dataset) creates a business layer on top of Gold tables.

**Creating a Semantic Model:**

1. Navigate to Lakehouse → New semantic model
2. Select Gold tables: `gold_revenue_metrics`, `gold_product_performance`, `gold_customer_segments`
3. Define relationships between tables
4. Create measures and KPIs

**DAX Measures Examples:**

```dax
// Total Revenue
Total Revenue = SUM(gold_revenue_metrics[TotalRevenue])

// Year-over-Year Growth
Revenue YoY % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD('gold_revenue_metrics'[Year], -1, YEAR)
    )
RETURN
    DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0)

// Average Customer Value
Avg Customer Lifetime Value = 
DIVIDE(
    SUM(gold_customer_segments[MonetaryValue]),
    DISTINCTCOUNT(gold_customer_segments[CustomerID]),
    0
)

// Top Products by Revenue
Top 10 Products Revenue = 
CALCULATE(
    SUM(gold_product_performance[TotalRevenue]),
    TOPN(
        10,
        ALLSELECTED(gold_product_performance[Description]),
        gold_product_performance[TotalRevenue],
        DESC
    )
)

// Customer Retention Rate
Customer Retention % = 
VAR CustomersThisMonth = DISTINCTCOUNT(gold_revenue_metrics[UniqueCustomers])
VAR CustomersPreviousMonth = 
    CALCULATE(
        DISTINCTCOUNT(gold_revenue_metrics[UniqueCustomers]),
        DATEADD('gold_revenue_metrics'[Month], -1, MONTH)
    )
RETURN
    DIVIDE(CustomersThisMonth, CustomersPreviousMonth, 0)
```

### 5. Power BI Report Development

Connect Power BI Desktop to the Semantic Model:

1. Open Power BI Desktop
2. Get Data → Power BI semantic models
3. Select your Fabric semantic model
4. Build visualizations using Gold tables and measures

**Key Visualizations:**
- Line chart: Revenue trends over time
- Bar chart: Top products by revenue
- Pie chart: Customer segment distribution
- KPI cards: Total revenue, orders, customers
- Matrix: Product performance metrics

## Common Patterns

### Pattern 1: Incremental Load (Bronze to Silver)

```python
from delta.tables import DeltaTable

# Define paths
bronze_path = "Files/bronze/transactions"
silver_table = "Tables/silver_transactions"

# Read new bronze data (e.g., filtered by date)
new_data = spark.read.parquet(bronze_path) \
    .filter(col("ProcessedDate") > lit("2024-01-01"))

# Upsert into Silver Delta table
if DeltaTable.isDeltaTable(spark, silver_table):
    silver_table_delta = DeltaTable.forPath(spark, silver_table)
    
    silver_table_delta.alias("target").merge(
        new_data.alias("source"),
        "target.InvoiceNo = source.InvoiceNo AND target.StockCode = source.StockCode"
    ).whenMatchedUpdateAll() \
     .whenNotMatchedInsertAll() \
     .execute()
else:
    new_data.write.format("delta").mode("overwrite").saveAsTable(silver_table)
```

### Pattern 2: Slowly Changing Dimension (SCD Type 2)

```python
from pyspark.sql.functions import current_timestamp, lit

def apply_scd_type2(source_df, target_table, key_columns, compare_columns):
    """Apply SCD Type 2 logic for dimension tables"""
    
    # Add metadata columns to source
    source_with_metadata = source_df \
        .withColumn("ValidFrom", current_timestamp()) \
        .withColumn("ValidTo", lit(None).cast("timestamp")) \
        .withColumn("IsCurrent", lit(True))
    
    if DeltaTable.isDeltaTable(spark, target_table):
        target = DeltaTable.forPath(spark, target_table)
        
        # Identify changed records
        changed_records = source_with_metadata.alias("source") \
            .join(
                target.toDF().alias("target"),
                key_columns,
                "inner"
            ).filter(
                " OR ".join([f"source.{col} != target.{col}" for col in compare_columns])
            ).select("source.*")
        
        # Close old records
        target.alias("target").merge(
            changed_records.alias("source"),
            " AND ".join([f"target.{col} = source.{col}" for col in key_columns]) + " AND target.IsCurrent = true"
        ).whenMatchedUpdate(
            set={
                "ValidTo": current_timestamp(),
                "IsCurrent": lit(False)
            }
        ).execute()
        
        # Insert new versions
        changed_records.write.format("delta").mode("append").saveAsTable(target_table)
    else:
        source_with_metadata.write.format("delta").mode("overwrite").saveAsTable(target_table)

# Example usage for product dimension
product_dim = df_silver.select("StockCode", "Description", "UnitPrice").distinct()
apply_scd_type2(product_dim, "Tables/dim_product", ["StockCode"], ["Description", "UnitPrice"])
```

### Pattern 3: Data Lineage Tracking

```python
# Add metadata for data lineage
def add_lineage_metadata(df, source_layer, target_layer, transformation_name):
    """Add lineage tracking columns"""
    return df \
        .withColumn("SourceLayer", lit(source_layer)) \
        .withColumn("TargetLayer", lit(target_layer)) \
        .withColumn("TransformationName", lit(transformation_name)) \
        .withColumn("ProcessedTimestamp", current_timestamp()) \
        .withColumn("ProcessedBy", lit("fabric_notebook"))

# Example usage
df_with_lineage = add_lineage_metadata(
    revenue_metrics,
    source_layer="Silver",
    target_layer="Gold",
    transformation_name="revenue_aggregation"
)
```

### Pattern 4: Parameterized Notebook Execution

```python
# Accept parameters from Fabric Pipeline
import sys

# Define default parameters
default_params = {
    "source_date": "2024-01-01",
    "target_layer": "gold",
    "overwrite_mode": "false"
}

# Read parameters (set via Pipeline or notebook parameters)
try:
    source_date = spark.conf.get("source_date", default_params["source_date"])
    target_layer = spark.conf.get("target_layer", default_params["target_layer"])
    overwrite_mode = spark.conf.get("overwrite_mode", default_params["overwrite_mode"]) == "true"
except:
    source_date = default_params["source_date"]
    target_layer = default_params["target_layer"]
    overwrite_mode = default_params["overwrite_mode"] == "true"

print(f"Processing data for date: {source_date}")
print(f"Target layer: {target_layer}")
print(f"Overwrite mode: {overwrite_mode}")

# Use parameters in transformation
df_filtered = df_silver.filter(col("InvoiceDate") >= lit(source_date))
```

## Configuration Best Practices

### Lakehouse Configuration

```python
# Configure Spark session for optimal performance
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.files.maxPartitionBytes", "134217728")  # 128MB
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")
```

### Delta Table Optimization

```python
# Optimize Delta tables for query performance
from delta.tables import DeltaTable

def optimize_gold_tables():
    """Run OPTIMIZE and VACUUM on Gold tables"""
    gold_tables = ["gold_revenue_metrics", "gold_product_performance", "gold_customer_segments"]
    
    for table_name in gold_tables:
        print(f"Optimizing {table_name}...")
        
        # Run OPTIMIZE with Z-ORDER
        spark.sql(f"OPTIMIZE {table_name} ZORDER BY (Year, Month)")
        
        # Clean up old versions (retain 7 days)
        delta_table = DeltaTable.forName(spark, table_name)
        delta_table.vacuum(168)  # 7 days in hours
        
        print(f"{table_name} optimized successfully")

# Run optimization (schedule this periodically)
optimize_gold_tables()
```

### Environment Variables

```python
# Use environment variables for sensitive configuration
import os

# Avoid hardcoding workspace or lakehouse IDs
WORKSPACE_ID = os.environ.get("FABRIC_WORKSPACE_ID")
LAKEHOUSE_ID = os.environ.get("FABRIC_LAKEHOUSE_ID")

# Use in connection strings
lakehouse_path = f"abfss://{WORKSPACE_ID}@onelake.dfs.fabric.microsoft.com/{LAKEHOUSE_ID}"
```

## Troubleshooting

### Issue: Dataflow Gen2 Fails to Publish

**Symptoms:** Error when publishing Dataflow to Lakehouse

**Solutions:**
```
1. Check Lakehouse permissions (Contributor or higher)
2. Verify destination path format: Files/silver/filename.parquet
3. Ensure data types are compatible with Parquet format
4. Check for null values in key columns
5. Validate Power Query M syntax in Advanced Editor
```

### Issue: Fabric Notebook Spark Memory Errors

**Symptoms:** `OutOfMemoryError` or `Container killed by YARN`

**Solutions:**
```python
# Reduce partition size
df = df.repartition(100)  # Increase partition count

# Enable adaptive query execution
spark.conf.set("spark.sql.adaptive.enabled", "true")

# Persist intermediate results
df_intermediate.cache()
df_intermediate.count()  # Trigger caching

# Use broadcast for small dimension tables
from pyspark.sql.functions import broadcast
result = large_table.join(broadcast(small_dim), "key")
```

### Issue: Delta Table Version Conflicts

**Symptoms:** `ConcurrentAppendException` or `MetadataChangedException`

**Solutions:**
```python
# Enable optimistic concurrency control
spark.conf.set("spark.databricks.delta.optimisticTransaction.enabled", "true")

# Retry logic for concurrent writes
from time import sleep
from py4j.protocol import Py4JJavaError

def write_with_retry(df, table_name, max_retries=3):
    for attempt in range(max_retries):
        try:
            df.write.format("delta").mode("append").saveAsTable(table_name)
            print(f"Write successful on attempt {attempt + 1}")
            break
        except Py4JJavaError as e:
            if "ConcurrentAppendException" in str(e) and attempt < max_retries - 1:
                print(f"Conflict detected, retrying... (attempt {attempt + 1})")
                sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

### Issue: Semantic Model Refresh Failures

**Symptoms:** Dataset refresh fails with timeout or data source errors

**Solutions:**
```
1. Check Gateway connectivity (if using on-premises sources)
2. Verify credentials in data source settings
3. Optimize Gold table queries (add indexes, partitions)
4. Reduce dataset size using incremental refresh
5. Enable query folding in Dataflow Gen2
```

### Issue: Power BI Report Performance

**Symptoms:** Slow report loading or visual rendering

**Solutions:**
```dax
// Use variables to avoid recalculating expressions
Total Revenue Optimized = 
VAR _Revenue = SUM(gold_revenue_metrics[TotalRevenue])
RETURN _Revenue

// Replace CALCULATE with simpler aggregations when possible
// Instead of:
Revenue All Time = CALCULATE(SUM(gold_revenue_metrics[TotalRevenue]), ALL(gold_revenue_metrics))

// Use:
Revenue All Time = SUMX(ALL(gold_revenue_metrics), gold_revenue_metrics[TotalRevenue])

// Avoid iterators on large tables
// Use measure references instead of direct column references
```

## Monitoring and Logging

### Notebook Execution Logging

```python
import logging
from datetime import datetime

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

def log_transformation_metrics(stage, record_count, duration_seconds):
    """Log transformation metrics for monitoring"""
    metrics = {
        "timestamp": datetime.utcnow().isoformat(),
        "stage": stage,
        "record_count": record_count,
        "duration_seconds": duration_seconds,
        "records_per_second": record_count / duration_seconds if duration_seconds > 0 else 0
    }
    logger.info(f"Transformation metrics: {metrics}")
    
    # Optionally write to monitoring table
    metrics_df = spark.createDataFrame([metrics])
    metrics_df.write.format("delta").mode("append").saveAsTable("monitoring_metrics")

# Example usage
start_time = datetime.now()
# ... transformation logic ...
end_time = datetime.now()
duration = (end_time - start_time).total_seconds()

log_transformation_metrics(
    stage="silver_to_gold_revenue",
    record_count=revenue_metrics.count(),
    duration_seconds=duration
)
```

## Additional Resources

- **Microsoft Fabric Documentation**: https://learn.microsoft.com/en-us/fabric/
- **Dataflow Gen2 Guide**: https://learn.microsoft.com/en-us/fabric/data-factory/dataflows-gen2
- **Fabric Notebooks**: https://learn.microsoft.com/en-us/fabric/data-engineering/how-to-use-notebook
- **Delta Lake on Fabric**: https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview
- **Power BI Semantic Models**: https://learn.microsoft.com/en-us/power-bi/connect-data/service-datasets-understand

This skill equips AI agents to guide developers through building complete unified analytics platforms using Microsoft Fabric's integrated services and modern data engineering best practices.
