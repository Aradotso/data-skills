---
name: spark-data-engineering-pipeline
description: Production PySpark ETL pipeline with AWS S3, PostgreSQL, schema validation, data quality checks, and automated testing
triggers:
  - build a pyspark data pipeline
  - create an etl pipeline with spark and s3
  - set up spark data engineering workflow
  - validate and transform data using pyspark
  - load parquet data from s3 to postgres
  - implement data quality checks in spark
  - create production spark pipeline
  - set up spark with aws s3 and postgresql
---

# Spark Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

A production-grade ETL pipeline built with PySpark that extracts JSON data from AWS S3, performs schema validation and data quality checks, transforms the data, writes it as Parquet, and loads it into PostgreSQL. This project demonstrates end-to-end data engineering best practices including logging, testing, and CI/CD.

## Project Structure

```
spark-data-engineering-pipeline/
├── configs/           # Configuration files
├── data/             # Sample data
├── drivers/          # JDBC drivers
├── logs/             # Application logs
├── notebooks/        # Jupyter notebooks
├── sql/              # SQL scripts
├── src/
│   ├── extract/      # Data extraction modules
│   ├── validation/   # Schema and quality validation
│   ├── transform/    # Data transformation logic
│   ├── load/         # Data loading to PostgreSQL
│   └── utils/        # Utility functions
├── tests/            # Unit tests
├── main.py           # Pipeline entry point
└── requirements.txt
```

## Installation

```bash
# Clone the repository
git clone https://github.com/Giri-25/spark-data-engineering-pipeline.git
cd spark-data-engineering-pipeline

# Create and activate virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

## Configuration

Create a `.env` file in the project root:

```bash
# AWS Configuration
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_REGION=us-east-1
S3_BUCKET=your-bucket-name

# PostgreSQL Configuration
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=data_warehouse
POSTGRES_USER=your_user
POSTGRES_PASSWORD=your_password
```

## Running the Pipeline

```bash
# Run the complete pipeline
python main.py

# Run with custom configuration
python main.py --config configs/pipeline_config.yaml

# Run unit tests
pytest tests/

# Run tests with coverage
pytest --cov=src tests/
```

## Core Components

### 1. Data Extraction from S3

```python
# src/extract/s3_extractor.py
from pyspark.sql import SparkSession
import os

def extract_from_s3(spark, s3_path):
    """
    Extract JSON data from S3
    
    Args:
        spark: SparkSession instance
        s3_path: S3 path to JSON files (s3a://bucket/path/)
    
    Returns:
        DataFrame with raw data
    """
    # Configure Spark for S3 access
    spark._jsc.hadoopConfiguration().set(
        "fs.s3a.access.key", 
        os.getenv("AWS_ACCESS_KEY_ID")
    )
    spark._jsc.hadoopConfiguration().set(
        "fs.s3a.secret.key", 
        os.getenv("AWS_SECRET_ACCESS_KEY")
    )
    spark._jsc.hadoopConfiguration().set(
        "fs.s3a.endpoint", 
        f"s3.{os.getenv('AWS_REGION')}.amazonaws.com"
    )
    
    # Read JSON data
    df = spark.read.json(s3_path)
    return df

# Usage
spark = SparkSession.builder \
    .appName("DataPipeline") \
    .config("spark.jars.packages", "org.apache.hadoop:hadoop-aws:3.3.1") \
    .getOrCreate()

s3_bucket = os.getenv("S3_BUCKET")
raw_df = extract_from_s3(spark, f"s3a://{s3_bucket}/raw/data/*.json")
```

### 2. Schema Validation

```python
# src/validation/schema_validator.py
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, TimestampType
from pyspark.sql import DataFrame
import logging

logger = logging.getLogger(__name__)

def define_expected_schema():
    """Define the expected schema for incoming data"""
    return StructType([
        StructField("user_id", StringType(), nullable=False),
        StructField("transaction_id", StringType(), nullable=False),
        StructField("amount", IntegerType(), nullable=False),
        StructField("currency", StringType(), nullable=False),
        StructField("timestamp", TimestampType(), nullable=False),
        StructField("status", StringType(), nullable=True),
        StructField("merchant_id", StringType(), nullable=True)
    ])

def validate_schema(df: DataFrame, expected_schema: StructType) -> bool:
    """
    Validate DataFrame schema against expected schema
    
    Args:
        df: Input DataFrame
        expected_schema: Expected StructType schema
    
    Returns:
        True if schema matches, False otherwise
    """
    df_fields = set(df.schema.fieldNames())
    expected_fields = set(expected_schema.fieldNames())
    
    # Check for missing fields
    missing_fields = expected_fields - df_fields
    if missing_fields:
        logger.error(f"Missing required fields: {missing_fields}")
        return False
    
    # Check for extra fields
    extra_fields = df_fields - expected_fields
    if extra_fields:
        logger.warning(f"Extra fields found: {extra_fields}")
    
    # Validate data types
    for field in expected_schema.fields:
        if field.name in df.schema.fieldNames():
            df_field = df.schema[field.name]
            if df_field.dataType != field.dataType:
                logger.error(
                    f"Type mismatch for {field.name}: "
                    f"expected {field.dataType}, got {df_field.dataType}"
                )
                return False
    
    logger.info("Schema validation passed")
    return True

# Usage
expected_schema = define_expected_schema()
if validate_schema(raw_df, expected_schema):
    validated_df = raw_df
else:
    raise ValueError("Schema validation failed")
```

### 3. Data Quality Checks

```python
# src/validation/quality_checker.py
from pyspark.sql import DataFrame
from pyspark.sql import functions as F
import logging

logger = logging.getLogger(__name__)

class DataQualityChecker:
    """Perform data quality checks on DataFrames"""
    
    def __init__(self, df: DataFrame):
        self.df = df
        self.quality_report = {}
    
    def check_null_values(self, columns: list) -> dict:
        """Check for null values in specified columns"""
        null_counts = {}
        for col in columns:
            null_count = self.df.filter(F.col(col).isNull()).count()
            null_counts[col] = null_count
            if null_count > 0:
                logger.warning(f"Column '{col}' has {null_count} null values")
        
        self.quality_report['null_counts'] = null_counts
        return null_counts
    
    def check_duplicates(self, key_columns: list) -> int:
        """Check for duplicate records based on key columns"""
        total_count = self.df.count()
        distinct_count = self.df.dropDuplicates(key_columns).count()
        duplicates = total_count - distinct_count
        
        if duplicates > 0:
            logger.warning(f"Found {duplicates} duplicate records")
        
        self.quality_report['duplicates'] = duplicates
        return duplicates
    
    def check_value_ranges(self, column: str, min_val, max_val) -> int:
        """Check if values are within expected range"""
        out_of_range = self.df.filter(
            (F.col(column) < min_val) | (F.col(column) > max_val)
        ).count()
        
        if out_of_range > 0:
            logger.warning(
                f"Column '{column}' has {out_of_range} values outside range "
                f"[{min_val}, {max_val}]"
            )
        
        self.quality_report[f'{column}_out_of_range'] = out_of_range
        return out_of_range
    
    def check_categorical_values(self, column: str, allowed_values: list) -> int:
        """Check if categorical values are in allowed list"""
        invalid_count = self.df.filter(
            ~F.col(column).isin(allowed_values)
        ).count()
        
        if invalid_count > 0:
            logger.warning(
                f"Column '{column}' has {invalid_count} invalid values"
            )
        
        self.quality_report[f'{column}_invalid'] = invalid_count
        return invalid_count
    
    def get_report(self) -> dict:
        """Return complete quality report"""
        return self.quality_report

# Usage
qc = DataQualityChecker(validated_df)

# Check for nulls in required fields
qc.check_null_values(['user_id', 'transaction_id', 'amount'])

# Check for duplicates
qc.check_duplicates(['transaction_id'])

# Check value ranges
qc.check_value_ranges('amount', min_val=0, max_val=1000000)

# Check categorical values
qc.check_categorical_values('status', ['pending', 'completed', 'failed'])

# Get quality report
quality_report = qc.get_report()
logger.info(f"Quality report: {quality_report}")
```

### 4. Data Transformation

```python
# src/transform/transformer.py
from pyspark.sql import DataFrame
from pyspark.sql import functions as F
from pyspark.sql.window import Window
import logging

logger = logging.getLogger(__name__)

class DataTransformer:
    """Transform and clean data"""
    
    @staticmethod
    def clean_data(df: DataFrame) -> DataFrame:
        """Clean data by removing nulls and duplicates"""
        # Remove duplicates
        df_cleaned = df.dropDuplicates(['transaction_id'])
        
        # Fill nulls in optional fields
        df_cleaned = df_cleaned.fillna({
            'status': 'unknown',
            'merchant_id': 'unknown'
        })
        
        # Remove rows with nulls in required fields
        df_cleaned = df_cleaned.dropna(
            subset=['user_id', 'transaction_id', 'amount']
        )
        
        logger.info(f"Cleaned data: {df_cleaned.count()} rows")
        return df_cleaned
    
    @staticmethod
    def add_derived_columns(df: DataFrame) -> DataFrame:
        """Add derived columns for analytics"""
        # Extract date components
        df_transformed = df.withColumn('date', F.to_date('timestamp'))
        df_transformed = df_transformed.withColumn('year', F.year('timestamp'))
        df_transformed = df_transformed.withColumn('month', F.month('timestamp'))
        df_transformed = df_transformed.withColumn('day', F.dayofmonth('timestamp'))
        df_transformed = df_transformed.withColumn('hour', F.hour('timestamp'))
        
        # Add day of week
        df_transformed = df_transformed.withColumn(
            'day_of_week', 
            F.dayofweek('timestamp')
        )
        
        # Convert amount to USD (assuming conversion logic)
        df_transformed = df_transformed.withColumn(
            'amount_usd',
            F.when(F.col('currency') == 'EUR', F.col('amount') * 1.1)
             .when(F.col('currency') == 'GBP', F.col('amount') * 1.3)
             .otherwise(F.col('amount'))
        )
        
        return df_transformed
    
    @staticmethod
    def add_aggregations(df: DataFrame) -> DataFrame:
        """Add user-level aggregations"""
        # Window specification for user-level aggregations
        user_window = Window.partitionBy('user_id')
        
        df_agg = df.withColumn(
            'user_total_transactions',
            F.count('transaction_id').over(user_window)
        )
        
        df_agg = df_agg.withColumn(
            'user_total_amount',
            F.sum('amount_usd').over(user_window)
        )
        
        df_agg = df_agg.withColumn(
            'user_avg_amount',
            F.avg('amount_usd').over(user_window)
        )
        
        return df_agg

# Usage
transformer = DataTransformer()

# Clean data
cleaned_df = transformer.clean_data(validated_df)

# Add derived columns
enriched_df = transformer.add_derived_columns(cleaned_df)

# Add aggregations
final_df = transformer.add_aggregations(enriched_df)

logger.info(f"Transformation complete: {final_df.count()} rows")
```

### 5. Write to Parquet

```python
# src/load/parquet_writer.py
from pyspark.sql import DataFrame
import os
import logging

logger = logging.getLogger(__name__)

def write_to_parquet(df: DataFrame, output_path: str, partition_cols: list = None):
    """
    Write DataFrame to Parquet format
    
    Args:
        df: DataFrame to write
        output_path: S3 or local path for Parquet files
        partition_cols: Columns to partition by (optional)
    """
    writer = df.write.mode('overwrite').format('parquet')
    
    if partition_cols:
        writer = writer.partitionBy(*partition_cols)
    
    # Write with compression
    writer.option('compression', 'snappy').save(output_path)
    
    logger.info(f"Data written to Parquet at {output_path}")

# Usage
s3_bucket = os.getenv("S3_BUCKET")
output_path = f"s3a://{s3_bucket}/processed/transactions/"

write_to_parquet(
    final_df,
    output_path,
    partition_cols=['year', 'month', 'day']
)
```

### 6. Load to PostgreSQL

```python
# src/load/postgres_loader.py
from pyspark.sql import DataFrame
import os
import logging

logger = logging.getLogger(__name__)

def load_to_postgres(df: DataFrame, table_name: str, mode: str = 'overwrite'):
    """
    Load DataFrame to PostgreSQL
    
    Args:
        df: DataFrame to load
        table_name: Target table name
        mode: Write mode ('overwrite', 'append')
    """
    postgres_url = (
        f"jdbc:postgresql://{os.getenv('POSTGRES_HOST')}:"
        f"{os.getenv('POSTGRES_PORT')}/{os.getenv('POSTGRES_DB')}"
    )
    
    connection_properties = {
        "user": os.getenv("POSTGRES_USER"),
        "password": os.getenv("POSTGRES_PASSWORD"),
        "driver": "org.postgresql.Driver"
    }
    
    # Write to PostgreSQL
    df.write.jdbc(
        url=postgres_url,
        table=table_name,
        mode=mode,
        properties=connection_properties
    )
    
    logger.info(f"Data loaded to PostgreSQL table '{table_name}'")

# Usage
load_to_postgres(
    final_df,
    table_name='transactions_processed',
    mode='overwrite'
)
```

### 7. Complete Pipeline

```python
# main.py
from pyspark.sql import SparkSession
from src.extract.s3_extractor import extract_from_s3
from src.validation.schema_validator import define_expected_schema, validate_schema
from src.validation.quality_checker import DataQualityChecker
from src.transform.transformer import DataTransformer
from src.load.parquet_writer import write_to_parquet
from src.load.postgres_loader import load_to_postgres
import logging
import os
from dotenv import load_dotenv

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('logs/pipeline.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

def main():
    """Main pipeline execution"""
    load_dotenv()
    
    # Initialize Spark
    spark = SparkSession.builder \
        .appName("DataEngineeringPipeline") \
        .config("spark.jars.packages", 
                "org.apache.hadoop:hadoop-aws:3.3.1,"
                "org.postgresql:postgresql:42.5.0") \
        .config("spark.sql.adaptive.enabled", "true") \
        .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
        .getOrCreate()
    
    try:
        # Step 1: Extract
        logger.info("Step 1: Extracting data from S3")
        s3_bucket = os.getenv("S3_BUCKET")
        raw_df = extract_from_s3(spark, f"s3a://{s3_bucket}/raw/data/")
        
        # Step 2: Validate schema
        logger.info("Step 2: Validating schema")
        expected_schema = define_expected_schema()
        if not validate_schema(raw_df, expected_schema):
            raise ValueError("Schema validation failed")
        
        # Step 3: Data quality checks
        logger.info("Step 3: Running data quality checks")
        qc = DataQualityChecker(raw_df)
        qc.check_null_values(['user_id', 'transaction_id', 'amount'])
        qc.check_duplicates(['transaction_id'])
        qc.check_value_ranges('amount', 0, 1000000)
        qc.check_categorical_values('status', ['pending', 'completed', 'failed'])
        
        # Step 4: Transform
        logger.info("Step 4: Transforming data")
        transformer = DataTransformer()
        cleaned_df = transformer.clean_data(raw_df)
        enriched_df = transformer.add_derived_columns(cleaned_df)
        final_df = transformer.add_aggregations(enriched_df)
        
        # Step 5: Write to Parquet
        logger.info("Step 5: Writing to Parquet")
        parquet_path = f"s3a://{s3_bucket}/processed/transactions/"
        write_to_parquet(final_df, parquet_path, ['year', 'month'])
        
        # Step 6: Load to PostgreSQL
        logger.info("Step 6: Loading to PostgreSQL")
        load_to_postgres(final_df, 'transactions_processed', mode='overwrite')
        
        logger.info("Pipeline completed successfully")
        
    except Exception as e:
        logger.error(f"Pipeline failed: {str(e)}", exc_info=True)
        raise
    finally:
        spark.stop()

if __name__ == "__main__":
    main()
```

## Testing

```python
# tests/test_transformer.py
import pytest
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, TimestampType
from src.transform.transformer import DataTransformer
from datetime import datetime

@pytest.fixture(scope="session")
def spark():
    """Create Spark session for testing"""
    return SparkSession.builder \
        .appName("TestPipeline") \
        .master("local[2]") \
        .getOrCreate()

@pytest.fixture
def sample_data(spark):
    """Create sample test data"""
    schema = StructType([
        StructField("user_id", StringType()),
        StructField("transaction_id", StringType()),
        StructField("amount", IntegerType()),
        StructField("currency", StringType()),
        StructField("timestamp", TimestampType())
    ])
    
    data = [
        ("user1", "txn1", 100, "USD", datetime(2024, 1, 15, 10, 30)),
        ("user1", "txn2", 200, "EUR", datetime(2024, 1, 16, 14, 20)),
        ("user2", "txn3", 150, "GBP", datetime(2024, 1, 17, 9, 15))
    ]
    
    return spark.createDataFrame(data, schema)

def test_add_derived_columns(sample_data):
    """Test derived column creation"""
    transformer = DataTransformer()
    result = transformer.add_derived_columns(sample_data)
    
    # Check new columns exist
    assert 'date' in result.columns
    assert 'year' in result.columns
    assert 'month' in result.columns
    assert 'amount_usd' in result.columns
    
    # Verify transformations
    first_row = result.filter(result.transaction_id == 'txn1').first()
    assert first_row['year'] == 2024
    assert first_row['month'] == 1
    assert first_row['amount_usd'] == 100

def test_clean_data_removes_duplicates(spark):
    """Test duplicate removal"""
    data = [
        ("user1", "txn1", 100, "USD", datetime(2024, 1, 15)),
        ("user1", "txn1", 100, "USD", datetime(2024, 1, 15)),  # duplicate
        ("user2", "txn2", 200, "EUR", datetime(2024, 1, 16))
    ]
    
    schema = StructType([
        StructField("user_id", StringType()),
        StructField("transaction_id", StringType()),
        StructField("amount", IntegerType()),
        StructField("currency", StringType()),
        StructField("timestamp", TimestampType())
    ])
    
    df = spark.createDataFrame(data, schema)
    transformer = DataTransformer()
    result = transformer.clean_data(df)
    
    assert result.count() == 2  # One duplicate removed
```

## Common Patterns

### Pattern 1: Incremental Data Processing

```python
# Process only new data based on timestamp
from datetime import datetime, timedelta

last_processed_date = datetime(2024, 1, 1)

incremental_df = raw_df.filter(
    F.col('timestamp') > F.lit(last_processed_date)
)

# Process and append to existing data
load_to_postgres(incremental_df, 'transactions_processed', mode='append')
```

### Pattern 2: Error Handling and Data Quarantine

```python
# Separate valid and invalid records
valid_df = raw_df.filter(F.col('amount') > 0)
invalid_df = raw_df.filter(F.col('amount') <= 0)

# Write invalid records to quarantine
invalid_df.write.mode('append').parquet(
    f"s3a://{s3_bucket}/quarantine/invalid_amounts/"
)

# Continue processing with valid data
process(valid_df)
```

### Pattern 3: Data Versioning

```python
# Add version and processing metadata
from datetime import datetime

versioned_df = final_df.withColumn(
    'processing_date', F.lit(datetime.now())
).withColumn(
    'version', F.lit('v1.0')
)

# Write with timestamp partition
write_to_parquet(
    versioned_df,
    f"s3a://{s3_bucket}/processed/transactions/",
    partition_cols=['year', 'month', 'version']
)
```

## Troubleshooting

### Issue: S3 Connection Errors

```python
# Ensure AWS credentials are set correctly
spark._jsc.hadoopConfiguration().set("fs.s3a.aws.credentials.provider", 
    "org.apache.hadoop.fs.s3a.SimpleAWSCredentialsProvider")

# For debugging, enable S3A logging
spark.sparkContext.setLogLevel("DEBUG")
```

### Issue: PostgreSQL Connection Timeout

```python
# Add connection timeout settings
connection_properties = {
    "user": os.getenv("POSTGRES_USER"),
    "password": os.getenv("POSTGRES_PASSWORD"),
    "driver": "org.postgresql.Driver",
    "connectTimeout": "60",
    "socketTimeout": "60"
}
```

### Issue: Out of Memory Errors

```python
# Repartition large DataFrames
df_repartitioned = large_df.repartition(100)

# Or use coalesce for reducing partitions
df_coalesced = df_repartitioned.coalesce(10)

# Configure Spark memory
spark = SparkSession.builder \
    .config("spark.executor.memory", "4g") \
    .config("spark.driver.memory", "2g") \
    .getOrCreate()
```

### Issue: Schema Evolution

```python
# Handle schema changes gracefully
from pyspark.sql.utils import AnalysisException

try:
    df = spark.read.parquet(path)
except AnalysisException:
    # Schema mismatch - use mergeSchema option
    df = spark.read.option("mergeSchema", "true").parquet(path)
```

## Performance Optimization

### Optimize Parquet Writes

```python
# Configure optimal file sizes
df.write \
    .option("maxRecordsPerFile", 100000) \
    .option("compression", "snappy") \
    .partitionBy("year", "month") \
    .parquet(output_path)
```

### Cache Intermediate Results

```python
# Cache DataFrames used multiple times
cleaned_df = transformer.clean_data(raw_df)
cleaned_df.cache()

# Use cached data
enriched_df = transformer.add_derived_columns(cleaned_df)
aggregated_df = transformer.add_aggregations(cleaned_df)

# Unpersist when done
cleaned_df.unpersist()
```

### Broadcast Small Lookup Tables

```python
# Broadcast small dimension tables for joins
from pyspark.sql.functions import broadcast

merchants_df = spark.read.parquet("s3a://bucket/merchants/")

# Join with broadcast hint
result = transactions_df.join(
    broadcast(merchants_df),
    on='merchant_id',
    how='left'
)
```
