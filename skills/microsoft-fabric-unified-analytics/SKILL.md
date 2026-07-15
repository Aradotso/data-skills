---
name: microsoft-fabric-unified-analytics
description: End-to-end unified analytics platform using Microsoft Fabric, Lakehouse, Dataflow Gen2, PySpark, and Power BI with Medallion Architecture.
triggers:
  - "set up microsoft fabric lakehouse"
  - "implement medallion architecture in fabric"
  - "create dataflow gen2 transformations"
  - "process data with fabric notebooks"
  - "build semantic model in fabric"
  - "configure onelake storage layers"
  - "create bronze silver gold layers"
  - "design unified analytics platform"
---

# Microsoft Fabric Unified Analytics Platform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end unified analytics platforms using **Microsoft Fabric**. The project demonstrates a production-inspired data engineering solution following the **Medallion Architecture** (Bronze → Silver → Gold) with OneLake, Dataflow Gen2, Fabric Notebooks (PySpark), Semantic Models, and Power BI.

## What This Project Does

Microsoft Fabric Unified Analytics Platform provides:

- **Unified SaaS Platform**: Single environment for data engineering, analytics, and BI
- **Lakehouse Architecture**: OneLake-based storage with Medallion layers
- **Data Integration**: Dataflow Gen2 for low-code transformations
- **Data Processing**: PySpark notebooks for business logic
- **Semantic Modeling**: Centralized business metrics layer
- **Business Intelligence**: Native Power BI integration
- **Retail Analytics Scenario**: Real-world e-commerce use case

## Project Structure

```
project/
├── architecture/          # Architecture diagrams and screenshots
├── notebooks/            # PySpark notebooks for transformations
├── case-study/           # Technical documentation
├── images/               # Project assets
└── README.md
```

## Prerequisites

- **Microsoft Fabric Workspace**: Trial or licensed workspace
- **OneLake Access**: Storage provisioned automatically with Fabric
- **Power BI License**: Included with Fabric capacity
- **Python 3.10+**: For local notebook development
- **PySpark**: Included in Fabric runtime

## Core Architecture Concepts

### Medallion Architecture Layers

**Bronze Layer** (Raw Zone)
- Stores unmodified source data
- Maintains data lineage
- Preserves audit trail
- Typically in Delta or Parquet format

**Silver Layer** (Cleansed Zone)
- Cleaned and standardized data
- Duplicate removal
- Data type consistency
- Business enrichment

**Gold Layer** (Curated Zone)
- Business-ready datasets
- Aggregated metrics
- Optimized for analytics
- Semantic model sources

### OneLake Storage Structure

```
OneLake/
└── Workspace/
    └── Lakehouse/
        ├── Files/
        │   ├── bronze/
        │   │   ├── customers/
        │   │   ├── products/
        │   │   └── transactions/
        │   ├── silver/
        │   │   └── online_retail_enriched/
        │   └── gold/
        │       ├── revenue_summary/
        │       ├── product_performance/
        │       └── customer_rfm/
        └── Tables/
            └── (managed Delta tables)
```

## Setting Up Microsoft Fabric Lakehouse

### 1. Create Lakehouse

```python
# In Fabric Workspace UI:
# 1. Click "+ New" → "Lakehouse"
# 2. Name: "RetailAnalyticsLakehouse"
# 3. Click "Create"

# Access lakehouse programmatically in notebooks:
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("RetailAnalytics").getOrCreate()

# Lakehouse is automatically mounted at:
# /lakehouse/default/Files/
# /lakehouse/default/Tables/
```

### 2. Create Folder Structure

```python
# Create medallion architecture folders
import notebookutils

# Bronze layer folders
notebookutils.fs.mkdirs("/lakehouse/default/Files/bronze/customers")
notebookutils.fs.mkdirs("/lakehouse/default/Files/bronze/products")
notebookutils.fs.mkdirs("/lakehouse/default/Files/bronze/transactions")

# Silver layer
notebookutils.fs.mkdirs("/lakehouse/default/Files/silver")

# Gold layer
notebookutils.fs.mkdirs("/lakehouse/default/Files/gold")
```

## Dataflow Gen2 Transformations

### Creating a Dataflow Gen2

1. **In Fabric Workspace** → "+ New" → "Dataflow Gen2"
2. **Get Data** → Choose source (CSV, Excel, Database, etc.)
3. **Apply Transformations** → Use Power Query M
4. **Set Destination** → OneLake Lakehouse (Silver layer)

### Example Transformations (Power Query M)

```m
// Remove duplicates
let
    Source = Csv.Document(Web.Contents("source_url")),
    PromotedHeaders = Table.PromoteHeaders(Source),
    RemovedDuplicates = Table.Distinct(PromotedHeaders, {"InvoiceNo", "StockCode"})
in
    RemovedDuplicates

// Handle missing values
let
    Source = Bronze_Transactions,
    ReplacedNulls = Table.ReplaceValue(Source, null, 0, Replacer.ReplaceValue, {"Quantity"}),
    FilteredRows = Table.SelectRows(ReplacedNulls, each [CustomerID] <> null)
in
    FilteredRows

// Add calculated columns
let
    Source = Cleaned_Data,
    AddedLineTotal = Table.AddColumn(Source, "line_total", each [Quantity] * [UnitPrice]),
    AddedYear = Table.AddColumn(AddedLineTotal, "year", each Date.Year([InvoiceDate])),
    AddedMonth = Table.AddColumn(AddedYear, "month", each Date.Month([InvoiceDate])),
    AddedIsReturn = Table.AddColumn(AddedMonth, "is_return", each if [Quantity] < 0 then true else false)
in
    AddedIsReturn
```

### Dataflow Destination Configuration

```python
# Configure dataflow to write to Silver layer
# In Dataflow Gen2 UI:
# 1. Click "Add data destination" → "Lakehouse"
# 2. Select workspace and lakehouse
# 3. Destination path: Files/silver/online_retail_enriched
# 4. File format: Delta or Parquet
# 5. Update method: Replace or Append
```

## Fabric Notebooks with PySpark

### Reading Silver Data

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

# Initialize Spark session (auto-configured in Fabric)
spark = SparkSession.builder.getOrCreate()

# Read Silver layer data
silver_path = "/lakehouse/default/Files/silver/online_retail_enriched"
df_silver = spark.read.format("parquet").load(silver_path)

# Display schema
df_silver.printSchema()

# Show sample data
display(df_silver.limit(10))
```

### Creating Gold Layer: Revenue Summary

```python
from pyspark.sql.functions import col, sum as _sum, count, avg, round as _round

# Calculate revenue metrics by year and month
df_revenue_summary = df_silver.groupBy("year", "month") \
    .agg(
        _sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("total_transactions"),
        count(col("CustomerID").distinct()).alias("unique_customers"),
        avg("line_total").alias("avg_order_value")
    ) \
    .withColumn("total_revenue", _round(col("total_revenue"), 2)) \
    .withColumn("avg_order_value", _round(col("avg_order_value"), 2)) \
    .orderBy("year", "month")

# Write to Gold layer
gold_revenue_path = "/lakehouse/default/Files/gold/revenue_summary"
df_revenue_summary.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save(gold_revenue_path)

# Create managed table for semantic model
df_revenue_summary.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold_revenue_summary")

print(f"✓ Revenue summary saved to {gold_revenue_path}")
```

### Creating Gold Layer: Product Performance

```python
# Analyze product performance
df_product_performance = df_silver \
    .filter(col("is_return") == False) \
    .groupBy("StockCode", "Description") \
    .agg(
        _sum("Quantity").alias("total_quantity_sold"),
        _sum("line_total").alias("total_revenue"),
        count("InvoiceNo").alias("number_of_orders"),
        avg("UnitPrice").alias("avg_unit_price")
    ) \
    .withColumn("total_revenue", _round(col("total_revenue"), 2)) \
    .withColumn("avg_unit_price", _round(col("avg_unit_price"), 2)) \
    .orderBy(col("total_revenue").desc())

# Write to Gold layer
gold_product_path = "/lakehouse/default/Files/gold/product_performance"
df_product_performance.write \
    .format("delta") \
    .mode("overwrite") \
    .save(gold_product_path)

# Create managed table
df_product_performance.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold_product_performance")

# Show top 10 products
display(df_product_performance.limit(10))
```

### Creating Gold Layer: Customer RFM Segmentation

```python
from pyspark.sql.window import Window
from datetime import datetime

# Calculate RFM metrics
max_date = df_silver.select(max("InvoiceDate")).collect()[0][0]

df_customer_rfm = df_silver \
    .filter(col("is_return") == False) \
    .groupBy("CustomerID") \
    .agg(
        datediff(lit(max_date), max("InvoiceDate")).alias("recency"),
        count("InvoiceNo").alias("frequency"),
        _sum("line_total").alias("monetary")
    ) \
    .withColumn("monetary", _round(col("monetary"), 2))

# Calculate RFM scores (1-5 scale)
def calculate_rfm_score(df, metric, ascending=True):
    """Calculate quintile-based scores for RFM metrics"""
    percentiles = df.approxQuantile(metric, [0.2, 0.4, 0.6, 0.8], 0.01)
    
    score_col = f"{metric}_score"
    if ascending:
        return df.withColumn(
            score_col,
            when(col(metric) <= percentiles[0], 1)
            .when(col(metric) <= percentiles[1], 2)
            .when(col(metric) <= percentiles[2], 3)
            .when(col(metric) <= percentiles[3], 4)
            .otherwise(5)
        )
    else:
        return df.withColumn(
            score_col,
            when(col(metric) <= percentiles[0], 5)
            .when(col(metric) <= percentiles[1], 4)
            .when(col(metric) <= percentiles[2], 3)
            .when(col(metric) <= percentiles[3], 2)
            .otherwise(1)
        )

# Apply RFM scoring (recency is reverse scored)
df_customer_rfm = calculate_rfm_score(df_customer_rfm, "recency", ascending=False)
df_customer_rfm = calculate_rfm_score(df_customer_rfm, "frequency", ascending=True)
df_customer_rfm = calculate_rfm_score(df_customer_rfm, "monetary", ascending=True)

# Calculate overall RFM score
df_customer_rfm = df_customer_rfm.withColumn(
    "rfm_score",
    concat(col("recency_score"), col("frequency_score"), col("monetary_score"))
)

# Segment customers
df_customer_rfm = df_customer_rfm.withColumn(
    "customer_segment",
    when((col("recency_score") >= 4) & (col("frequency_score") >= 4) & (col("monetary_score") >= 4), "Champions")
    .when((col("recency_score") >= 3) & (col("frequency_score") >= 3), "Loyal Customers")
    .when((col("recency_score") >= 4) & (col("frequency_score") <= 2), "New Customers")
    .when((col("recency_score") <= 2) & (col("frequency_score") >= 3), "At Risk")
    .when((col("recency_score") <= 2) & (col("frequency_score") <= 2), "Lost Customers")
    .otherwise("Regular Customers")
)

# Write to Gold layer
gold_rfm_path = "/lakehouse/default/Files/gold/customer_rfm"
df_customer_rfm.write \
    .format("delta") \
    .mode("overwrite") \
    .save(gold_rfm_path)

# Create managed table
df_customer_rfm.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("gold_customer_rfm")

# Show segment distribution
display(df_customer_rfm.groupBy("customer_segment").count().orderBy(col("count").desc()))
```

### Data Quality Checks

```python
def validate_data_quality(df, table_name):
    """Perform basic data quality checks"""
    print(f"\n=== Data Quality Report: {table_name} ===")
    
    # Record count
    total_records = df.count()
    print(f"Total records: {total_records:,}")
    
    # Null checks
    print("\nNull value counts:")
    for column in df.columns:
        null_count = df.filter(col(column).isNull()).count()
        if null_count > 0:
            print(f"  {column}: {null_count} ({null_count/total_records*100:.2f}%)")
    
    # Duplicate checks
    distinct_count = df.distinct().count()
    if distinct_count != total_records:
        print(f"\nDuplicates found: {total_records - distinct_count}")
    
    # Data range for numeric columns
    numeric_cols = [f.name for f in df.schema.fields if str(f.dataType) in ['IntegerType', 'DoubleType', 'FloatType']]
    if numeric_cols:
        print("\nNumeric column ranges:")
        for col_name in numeric_cols:
            stats = df.select(min(col_name), max(col_name), avg(col_name)).collect()[0]
            print(f"  {col_name}: min={stats[0]}, max={stats[1]}, avg={stats[2]:.2f}")
    
    print("=" * 50)

# Run validation
validate_data_quality(df_silver, "Silver Layer")
validate_data_quality(df_revenue_summary, "Revenue Summary")
```

## Semantic Model Configuration

### Creating Semantic Model

1. **In Lakehouse** → Click "New semantic model"
2. **Select Tables** → Choose Gold layer tables:
   - `gold_revenue_summary`
   - `gold_product_performance`
   - `gold_customer_rfm`
3. **Define Relationships** (if multiple tables with keys)
4. **Create Measures** using DAX

### DAX Measures for Business KPIs

```dax
// Total Revenue
Total Revenue = SUM(gold_revenue_summary[total_revenue])

// Total Transactions
Total Transactions = SUM(gold_revenue_summary[total_transactions])

// Average Order Value
Average Order Value = AVERAGE(gold_revenue_summary[avg_order_value])

// Revenue Growth (Month-over-Month)
Revenue MoM Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = 
    CALCULATE(
        [Total Revenue],
        DATEADD(gold_revenue_summary[year], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0) * 100

// Customer Retention Rate
Customer Retention Rate = 
DIVIDE(
    CALCULATE(DISTINCTCOUNT(gold_customer_rfm[CustomerID]), gold_customer_rfm[recency] <= 90),
    DISTINCTCOUNT(gold_customer_rfm[CustomerID]),
    0
) * 100

// Top Product by Revenue
Top Product = 
CALCULATE(
    FIRSTNONBLANK(gold_product_performance[Description], 1),
    TOPN(1, ALL(gold_product_performance), gold_product_performance[total_revenue], DESC)
)

// Customer Lifetime Value
Customer LTV = 
DIVIDE(
    SUM(gold_customer_rfm[monetary]),
    DISTINCTCOUNT(gold_customer_rfm[CustomerID]),
    0
)
```

### Time Intelligence Setup

```dax
// Create date table for time intelligence
Date Table = 
ADDCOLUMNS(
    CALENDAR(DATE(2020, 1, 1), DATE(2023, 12, 31)),
    "Year", YEAR([Date]),
    "Month", MONTH([Date]),
    "Month Name", FORMAT([Date], "MMM"),
    "Quarter", "Q" & QUARTER([Date]),
    "Year-Month", FORMAT([Date], "YYYY-MM")
)

// Mark as date table
// In Model view → right-click Date Table → Mark as date table

// Year-to-Date Revenue
YTD Revenue = 
TOTALYTD(
    [Total Revenue],
    'Date Table'[Date]
)

// Previous Year Revenue
PY Revenue = 
CALCULATE(
    [Total Revenue],
    SAMEPERIODLASTYEAR('Date Table'[Date])
)
```

## Power BI Integration

### Connecting to Semantic Model

```python
# Power BI automatically connects to Fabric semantic models
# In Power BI Desktop:
# 1. Get Data → Power Platform → Power BI semantic models
# 2. Select your workspace and semantic model
# 3. Build visuals using semantic model measures and tables

# No additional configuration needed - native integration
```

### Creating Report Visuals

Common visual configurations for retail analytics:

**KPI Cards**
- Total Revenue
- Total Transactions
- Average Order Value
- Customer Retention Rate

**Time Series**
- Revenue trend by month/year
- Transaction volume over time

**Bar/Column Charts**
- Top 10 products by revenue
- Customer segments distribution

**Matrix/Table**
- Product performance details
- Customer RFM scores

## Fabric Pipeline Orchestration

### Creating Data Pipeline

```python
# In Fabric Workspace → "+ New" → "Data pipeline"
# Add activities:

# 1. Dataflow Gen2 activity
# - Bronze to Silver transformation
# - Schedule: Daily at 2 AM

# 2. Notebook activity
# - Silver to Gold transformations
# - Runs after Dataflow completes
# - Notebook: GoldLayerTransformations

# 3. Refresh Semantic Model activity
# - Refreshes Power BI semantic model
# - Runs after Notebook completes

# Pipeline definition (conceptual JSON):
{
  "name": "RetailAnalyticsPipeline",
  "activities": [
    {
      "name": "BronzeToSilver",
      "type": "DataflowGen2",
      "dataflow": "RetailDataTransformation"
    },
    {
      "name": "SilverToGold",
      "type": "Notebook",
      "notebook": "GoldLayerTransformations",
      "dependsOn": ["BronzeToSilver"]
    },
    {
      "name": "RefreshSemanticModel",
      "type": "RefreshSemanticModel",
      "semanticModel": "RetailAnalytics",
      "dependsOn": ["SilverToGold"]
    }
  ],
  "schedule": {
    "frequency": "Day",
    "interval": 1,
    "startTime": "02:00:00"
  }
}
```

## Environment Configuration

### Fabric Workspace Settings

```python
# Environment variables for notebooks
import os

# Workspace and lakehouse (auto-configured in Fabric)
WORKSPACE_ID = os.getenv("FABRIC_WORKSPACE_ID", "auto-detected")
LAKEHOUSE_ID = os.getenv("FABRIC_LAKEHOUSE_ID", "auto-detected")

# Custom configuration
CONFIG = {
    "bronze_path": "/lakehouse/default/Files/bronze",
    "silver_path": "/lakehouse/default/Files/silver",
    "gold_path": "/lakehouse/default/Files/gold",
    "data_retention_days": 365,
    "delta_optimize_enabled": True
}

# Storage optimization settings
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")
```

### Notebook Parameters

```python
# Use Fabric notebook parameters for dynamic execution
# In notebook: Add parameters cell

# Parameters (can be overridden by pipeline)
source_layer = "silver"  # default
target_layer = "gold"    # default
processing_date = "2024-01-01"  # default

# Use in code
print(f"Processing {source_layer} → {target_layer} for {processing_date}")

source_path = f"/lakehouse/default/Files/{source_layer}"
target_path = f"/lakehouse/default/Files/{target_layer}"
```

## Delta Lake Optimization

### Optimize and Vacuum

```python
from delta.tables import DeltaTable

# Optimize Gold tables
gold_tables = [
    "gold_revenue_summary",
    "gold_product_performance", 
    "gold_customer_rfm"
]

for table_name in gold_tables:
    print(f"Optimizing {table_name}...")
    
    # Get Delta table
    delta_table = DeltaTable.forName(spark, table_name)
    
    # Optimize (compact small files)
    delta_table.optimize().executeCompaction()
    
    # Z-order by common filter columns (if applicable)
    if table_name == "gold_revenue_summary":
        delta_table.optimize().executeZOrderBy("year", "month")
    elif table_name == "gold_product_performance":
        delta_table.optimize().executeZOrderBy("StockCode")
    elif table_name == "gold_customer_rfm":
        delta_table.optimize().executeZOrderBy("customer_segment")
    
    # Vacuum old files (7 days retention)
    delta_table.vacuum(168)  # hours
    
    print(f"✓ {table_name} optimized")
```

### Time Travel Queries

```python
# Query historical versions of Delta tables
from delta.tables import DeltaTable

# View table history
delta_table = DeltaTable.forPath(spark, "/lakehouse/default/Files/gold/revenue_summary")
display(delta_table.history())

# Query specific version
df_version = spark.read.format("delta").option("versionAsOf", 5).load("/lakehouse/default/Files/gold/revenue_summary")
display(df_version)

# Query at timestamp
df_timestamp = spark.read.format("delta").option("timestampAsOf", "2024-01-15").load("/lakehouse/default/Files/gold/revenue_summary")
display(df_timestamp)

# Restore to previous version
delta_table.restoreToVersion(5)
```

## Common Patterns

### Pattern 1: Incremental Processing

```python
from datetime import datetime, timedelta

def process_incremental_data(source_path, target_path, watermark_column="InvoiceDate"):
    """Process only new data since last run"""
    
    # Get last processed date
    try:
        df_existing = spark.read.format("delta").load(target_path)
        last_processed = df_existing.select(max(watermark_column)).collect()[0][0]
    except:
        last_processed = datetime(2020, 1, 1)  # default start date
    
    print(f"Processing data since: {last_processed}")
    
    # Read only new data
    df_new = spark.read.format("parquet").load(source_path) \
        .filter(col(watermark_column) > last_processed)
    
    new_count = df_new.count()
    print(f"New records to process: {new_count:,}")
    
    if new_count > 0:
        # Apply transformations
        df_transformed = df_new  # Add your transformations here
        
        # Append to target
        df_transformed.write \
            .format("delta") \
            .mode("append") \
            .save(target_path)
        
        print(f"✓ Processed {new_count:,} new records")
    else:
        print("No new records to process")

# Usage
process_incremental_data(
    source_path="/lakehouse/default/Files/silver/online_retail_enriched",
    target_path="/lakehouse/default/Files/gold/revenue_summary"
)
```

### Pattern 2: Data Validation Framework

```python
class DataValidator:
    """Framework for data quality validation"""
    
    def __init__(self, df, table_name):
        self.df = df
        self.table_name = table_name
        self.issues = []
    
    def check_nulls(self, columns, threshold=0.05):
        """Check null percentage in critical columns"""
        total = self.df.count()
        for col_name in columns:
            null_count = self.df.filter(col(col_name).isNull()).count()
            null_pct = null_count / total
            if null_pct > threshold:
                self.issues.append(f"{col_name}: {null_pct*100:.2f}% nulls (threshold: {threshold*100}%)")
        return self
    
    def check_duplicates(self, key_columns):
        """Check for duplicate keys"""
        total = self.df.count()
        distinct = self.df.select(key_columns).distinct().count()
        if total != distinct:
            self.issues.append(f"Found {total - distinct} duplicate records on {key_columns}")
        return self
    
    def check_range(self, column, min_val, max_val):
        """Check if numeric values are within expected range"""
        out_of_range = self.df.filter((col(column) < min_val) | (col(column) > max_val)).count()
        if out_of_range > 0:
            self.issues.append(f"{column}: {out_of_range} values outside range [{min_val}, {max_val}]")
        return self
    
    def check_referential_integrity(self, foreign_key, reference_df, reference_key):
        """Check foreign key relationships"""
        missing = self.df.join(reference_df, self.df[foreign_key] == reference_df[reference_key], "left_anti").count()
        if missing > 0:
            self.issues.append(f"{foreign_key}: {missing} orphaned records")
        return self
    
    def report(self):
        """Generate validation report"""
        print(f"\n=== Validation Report: {self.table_name} ===")
        if self.issues:
            print("⚠️  Issues found:")
            for issue in self.issues:
                print(f"  - {issue}")
            return False
        else:
            print("✓ All validation checks passed")
            return True

# Usage
validator = DataValidator(df_silver, "Silver Layer")
is_valid = validator \
    .check_nulls(["CustomerID", "StockCode", "InvoiceDate"]) \
    .check_duplicates(["InvoiceNo", "StockCode"]) \
    .check_range("Quantity", -1000, 10000) \
    .check_range("UnitPrice", 0, 100000) \
    .report()

if not is_valid:
    raise Exception("Data validation failed")
```

### Pattern 3: Slowly Changing Dimensions (SCD Type 2)

```python
from pyspark.sql.functions import current_timestamp, lit

def apply_scd_type2(new_df, existing_path, key_columns, compare_columns):
    """Implement SCD Type 2 for dimension tables"""
    
    try:
        # Read existing data
        existing_df = spark.read.format("delta").load(existing_path)
        
        # Add metadata columns if not present
        if "effective_start_date" not in existing_df.columns:
            existing_df = existing_df \
                .withColumn("effective_start_date", current_timestamp()) \
                .withColumn("effective_end_date", lit(None).cast("timestamp")) \
                .withColumn("is_current", lit(True))
        
        # Identify changed records
        changed_df = new_df.join(
            existing_df.filter(col("is_current") == True),
            key_columns,
            "inner"
        )
        
        # Filter for actual changes
        change_condition = None
        for col_name in compare_columns:
            condition = col(f"new_df.{col_name}") != col(f"existing_df.{col_name}")
            change_condition = condition if change_condition is None else (change_condition | condition)
        
        changed_df = changed_df.filter(change_condition)
        
        # Expire old records
        expired_df = existing_df.join(changed_df.select(key_columns), key_columns, "inner") \
            .withColumn("effective_end_date", current_timestamp()) \
            .withColumn("is_current", lit(False))
        
        # Add new versions
        new_versions_df = new_df.join(changed_df.select(key_columns), key_columns, "inner") \
            .withColumn("effective_start_date", current_timestamp()) \
            .withColumn("effective_end_date", lit(None).cast("timestamp")) \
            .withColumn("is_current", lit(True))
        
        # Identify truly new records
        truly_new_df = new_df.join(existing_df.select(key_columns), key_columns, "left_anti") \
            .withColumn("effective_start_date", current_timestamp()) \
            .withColumn("effective_end_date", lit(None).cast("timestamp")) \
            .withColumn("is_current", lit(True))
        
        # Combine and write
        result_df = existing_df \
            .join(changed_df.select(key_columns), key_columns, "left_anti") \
            .union(expired_df) \
            .union(new_versions_df) \
            .union(truly_new_df)
        
        result_df.write.format("delta").mode("overwrite").save(existing_path)
        
        print(f"✓ SCD Type 2 applied: {changed_df.count()} changed, {truly_new_df.count()} new")
        
    except Exception as e:
        # First load - just write with metadata
        new_df
