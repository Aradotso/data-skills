---
name: microsoft-fabric-unified-analytics
description: End-to-end data engineering on Microsoft Fabric using Lakehouse, Dataflow Gen2, PySpark, and Power BI with Medallion Architecture
triggers:
  - "how do I build a lakehouse in Microsoft Fabric"
  - "implement medallion architecture with fabric"
  - "create dataflow gen2 transformations"
  - "write pyspark notebooks for fabric lakehouse"
  - "build semantic models in microsoft fabric"
  - "set up bronze silver gold layers in onelake"
  - "integrate power bi with fabric lakehouse"
  - "design unified analytics platform with fabric"
---

# Microsoft Fabric Unified Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This skill enables AI agents to guide developers in building production-grade unified analytics platforms using **Microsoft Fabric**. The project demonstrates end-to-end data engineering following the **Medallion Architecture** (Bronze → Silver → Gold) with OneLake storage, Dataflow Gen2 transformations, PySpark processing, Semantic Models, and native Power BI integration.

Microsoft Fabric unifies data ingestion, transformation, storage, semantic modeling, and visualization in a single SaaS platform, eliminating the need for multiple disconnected services.

## Architecture Pattern

The solution follows a layered Lakehouse approach:

- **Bronze Layer**: Raw source data preservation
- **Silver Layer**: Cleansed and enriched data
- **Gold Layer**: Business-ready analytical datasets

## Key Components

1. **OneLake**: Unified storage foundation (similar to Azure Data Lake)
2. **Lakehouse**: Data lake + warehouse hybrid architecture
3. **Dataflow Gen2**: Low-code ETL transformations
4. **Fabric Notebooks**: PySpark-based data processing
5. **Semantic Model**: Business logic and KPI layer
6. **Power BI**: Native visualization and reporting

## Project Setup

### Prerequisites

- Microsoft Fabric capacity or trial workspace
- Fabric Lakehouse created
- Source data (CSV, Parquet, or streaming)
- Python 3.8+ for local development (optional)

### Creating a Lakehouse

1. Navigate to Fabric workspace
2. Click **+ New** → **Lakehouse**
3. Name: `RetailAnalyticsLakehouse`
4. Create folder structure:
   ```
   Files/
   ├── bronze/
   ├── silver/
   └── gold/
   ```

### Initial Data Upload

Upload raw files to Bronze layer via:
- Fabric UI drag-and-drop
- Dataflow Gen2
- Fabric Pipelines
- Azure Data Factory

## Bronze Layer: Raw Data Ingestion

Store raw data as-is without transformations:

```python
# Read raw CSV data in Fabric Notebook
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("BronzeIngestion").getOrCreate()

# Read from OneLake bronze layer
bronze_path = "Files/bronze/online_retail.csv"
df_bronze = spark.read.csv(bronze_path, header=True, inferSchema=True)

# Write to bronze table
df_bronze.write.mode("overwrite").format("delta").saveAsTable("bronze_online_retail")

print(f"Bronze records loaded: {df_bronze.count()}")
```

## Dataflow Gen2: Silver Transformations

Dataflow Gen2 provides low-code transformations using Power Query:

### Common Transformations

1. **Handle Missing Values**:
   ```
   - Remove rows with null CustomerID
   - Fill missing Description with "Unknown"
   ```

2. **Remove Duplicates**:
   ```
   - Based on InvoiceNo + StockCode + InvoiceDate
   ```

3. **Add Business Columns**:
   ```
   - line_total = Quantity × UnitPrice
   - year = YEAR(InvoiceDate)
   - month = MONTH(InvoiceDate)
   - is_return = IF(Quantity < 0, true, false)
   ```

4. **Data Type Standardization**:
   ```
   - InvoiceDate → DateTime
   - Quantity → Integer
   - UnitPrice → Decimal
   ```

### Dataflow Gen2 Configuration

1. Create Dataflow Gen2 in workspace
2. Add data source: Bronze table
3. Apply transformations in Power Query
4. Set destination: Silver layer table
5. Schedule refresh (optional)

## Fabric Notebook: Gold Layer Processing

Use PySpark to create business-ready analytical datasets:

### Example: Revenue Analytics

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Read silver data
df_silver = spark.table("silver_online_retail")

# Revenue by Product
df_revenue_product = (
    df_silver
    .filter(F.col("is_return") == False)
    .groupBy("StockCode", "Description")
    .agg(
        F.sum("line_total").alias("total_revenue"),
        F.sum("Quantity").alias("total_quantity"),
        F.countDistinct("InvoiceNo").alias("order_count")
    )
    .orderBy(F.desc("total_revenue"))
)

# Write to gold layer
df_revenue_product.write.mode("overwrite").format("delta").saveAsTable("gold_revenue_by_product")
```

### Example: Customer RFM Segmentation

```python
from datetime import datetime

# Define analysis date
analysis_date = datetime(2011, 12, 31)

# Calculate RFM metrics
df_rfm = (
    df_silver
    .filter(F.col("is_return") == False)
    .groupBy("CustomerID")
    .agg(
        F.datediff(F.lit(analysis_date), F.max("InvoiceDate")).alias("recency"),
        F.countDistinct("InvoiceNo").alias("frequency"),
        F.sum("line_total").alias("monetary")
    )
)

# Assign RFM scores (1-5)
window_spec = Window.orderBy(F.col("recency").desc())
df_rfm = df_rfm.withColumn("R_score", F.ntile(5).over(window_spec))

window_spec = Window.orderBy(F.col("frequency"))
df_rfm = df_rfm.withColumn("F_score", F.ntile(5).over(window_spec))

window_spec = Window.orderBy(F.col("monetary"))
df_rfm = df_rfm.withColumn("M_score", F.ntile(5).over(window_spec))

# Create RFM segment
df_rfm = df_rfm.withColumn("RFM_segment", 
    F.concat(F.col("R_score"), F.col("F_score"), F.col("M_score"))
)

# Customer segmentation logic
df_rfm = df_rfm.withColumn("customer_segment",
    F.when((F.col("R_score") >= 4) & (F.col("F_score") >= 4), "Champions")
    .when((F.col("R_score") >= 3) & (F.col("F_score") >= 3), "Loyal Customers")
    .when((F.col("R_score") >= 4) & (F.col("F_score") <= 2), "Potential Loyalists")
    .when(F.col("R_score") <= 2, "At Risk")
    .otherwise("Regular")
)

# Save to gold layer
df_rfm.write.mode("overwrite").format("delta").saveAsTable("gold_customer_rfm")
```

### Example: Time Series Aggregation

```python
# Monthly revenue trends
df_monthly_revenue = (
    df_silver
    .filter(F.col("is_return") == False)
    .groupBy("year", "month")
    .agg(
        F.sum("line_total").alias("total_revenue"),
        F.countDistinct("CustomerID").alias("unique_customers"),
        F.countDistinct("InvoiceNo").alias("order_count")
    )
    .withColumn("avg_order_value", F.col("total_revenue") / F.col("order_count"))
    .orderBy("year", "month")
)

df_monthly_revenue.write.mode("overwrite").format("delta").saveAsTable("gold_monthly_trends")
```

## Semantic Model Creation

Create a Semantic Model to define business logic and relationships:

### Steps

1. In Fabric Lakehouse, click **New Semantic Model**
2. Select Gold tables:
   - `gold_revenue_by_product`
   - `gold_customer_rfm`
   - `gold_monthly_trends`
3. Define relationships (if applicable)
4. Create measures using DAX:

```dax
Total Revenue = SUM([total_revenue])

Total Orders = SUM([order_count])

Average Order Value = DIVIDE([Total Revenue], [Total Orders], 0)

YoY Revenue Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PreviousRevenue = CALCULATE([Total Revenue], SAMEPERIODLASTYEAR('gold_monthly_trends'[date]))
RETURN DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0)

Customer Count = DISTINCTCOUNT([CustomerID])

Revenue per Customer = DIVIDE([Total Revenue], [Customer Count], 0)
```

## Power BI Integration

Connect Power BI directly to the Semantic Model:

1. Open Power BI Desktop
2. Get Data → **Power BI Semantic Models**
3. Select your Fabric Semantic Model
4. Create visualizations using defined measures

### Example Report Structure

- **Overview Page**: KPI cards (Revenue, Orders, Customers)
- **Product Performance**: Bar chart of top products by revenue
- **Customer Segmentation**: Pie chart of RFM segments
- **Trends**: Line chart of monthly revenue with YoY comparison

## Common Patterns

### Pattern: Incremental Load

```python
# Load only new records since last run
from delta.tables import DeltaTable

# Get max date from silver layer
max_date = spark.sql("SELECT MAX(InvoiceDate) FROM silver_online_retail").collect()[0][0]

# Read new bronze records
df_new = (
    spark.table("bronze_online_retail")
    .filter(F.col("InvoiceDate") > max_date)
)

# Append to silver
df_new.write.mode("append").format("delta").saveAsTable("silver_online_retail")
```

### Pattern: Data Quality Checks

```python
# Validate silver data before gold processing
quality_checks = {
    "null_customerid": df_silver.filter(F.col("CustomerID").isNull()).count(),
    "negative_revenue": df_silver.filter(F.col("line_total") < 0).count(),
    "duplicate_records": df_silver.groupBy("InvoiceNo", "StockCode").count().filter(F.col("count") > 1).count()
}

for check, count in quality_checks.items():
    if count > 0:
        print(f"⚠️ Warning: {check} = {count}")
    else:
        print(f"✅ Pass: {check}")
```

### Pattern: Delta Lake Time Travel

```python
# Query previous version of data
df_previous = spark.read.format("delta").option("versionAsOf", 0).table("gold_revenue_by_product")

# Query data as of specific timestamp
df_historical = spark.read.format("delta").option("timestampAsOf", "2024-01-01").table("gold_revenue_by_product")
```

## Configuration Best Practices

### Notebook Configuration

```python
# Set Spark configuration for optimization
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")
```

### Environment Variables

```python
import os

# Use environment variables for configuration
LAKEHOUSE_NAME = os.getenv("FABRIC_LAKEHOUSE_NAME", "RetailAnalyticsLakehouse")
WORKSPACE_ID = os.getenv("FABRIC_WORKSPACE_ID")
```

## Troubleshooting

### Issue: "Table not found" error

**Solution**: Ensure table is registered in Lakehouse metastore:

```python
# Refresh tables
spark.sql("REFRESH TABLE bronze_online_retail")

# List available tables
spark.sql("SHOW TABLES").show()
```

### Issue: Dataflow Gen2 fails with memory error

**Solution**: 
- Reduce data volume with filtering
- Use incremental refresh
- Increase Fabric capacity tier

### Issue: PySpark job runs slowly

**Solution**: Optimize with partitioning and caching:

```python
# Partition by commonly filtered column
df.write.partitionBy("year", "month").format("delta").saveAsTable("gold_partitioned")

# Cache frequently used DataFrames
df_silver.cache()
df_silver.count()  # Trigger cache
```

### Issue: Semantic Model doesn't show latest data

**Solution**: Schedule or trigger refresh:

```python
# Via Fabric REST API
import requests

url = f"https://api.fabric.microsoft.com/v1/workspaces/{WORKSPACE_ID}/semanticModels/{MODEL_ID}/refreshes"
headers = {"Authorization": f"Bearer {os.getenv('FABRIC_ACCESS_TOKEN')}"}

response = requests.post(url, headers=headers)
print(f"Refresh triggered: {response.status_code}")
```

### Issue: Power BI report shows old data

**Solution**:
1. Check Semantic Model last refresh time
2. Trigger manual refresh in Fabric workspace
3. Verify notebook completed successfully
4. Check Dataflow Gen2 execution history

## Advanced Techniques

### Delta Lake Optimization

```python
from delta.tables import DeltaTable

# Optimize table layout
spark.sql("OPTIMIZE gold_revenue_by_product")

# Z-order by frequently filtered columns
spark.sql("OPTIMIZE gold_revenue_by_product ZORDER BY (year, month)")

# Vacuum old versions (keep 7 days)
spark.sql("VACUUM gold_revenue_by_product RETAIN 168 HOURS")
```

### Parametrized Notebooks

```python
# Use widgets for parameters
dbutils.widgets.text("processing_date", "2024-01-01")
dbutils.widgets.dropdown("environment", "dev", ["dev", "test", "prod"])

processing_date = dbutils.widgets.get("processing_date")
environment = dbutils.widgets.get("environment")

print(f"Processing date: {processing_date}")
print(f"Environment: {environment}")
```

## CI/CD Integration

While Fabric doesn't yet have full CI/CD support, you can version control notebooks:

```bash
# Export notebook as .ipynb
# Commit to Git repository
git add notebooks/gold_transformations.ipynb
git commit -m "Update gold layer transformations"
git push origin main
```

## Monitoring and Logging

```python
import logging

# Set up logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

try:
    df_gold = spark.table("silver_online_retail").groupBy("Country").count()
    df_gold.write.mode("overwrite").saveAsTable("gold_country_summary")
    logger.info(f"✅ Gold table created successfully with {df_gold.count()} rows")
except Exception as e:
    logger.error(f"❌ Failed to create gold table: {str(e)}")
    raise
```

## Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric/)
- [OneLake Storage](https://learn.microsoft.com/en-us/fabric/onelake/)
- [Dataflow Gen2 Guide](https://learn.microsoft.com/en-us/fabric/data-factory/dataflow-gen2)
- [Fabric Notebooks](https://learn.microsoft.com/en-us/fabric/data-engineering/how-to-use-notebook)
- [Delta Lake on Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview)
