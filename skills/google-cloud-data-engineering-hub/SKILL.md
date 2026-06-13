---
name: google-cloud-data-engineering-hub
description: Production-grade GCP data engineering projects using BigQuery, Dataflow, Beam, Composer, Pub/Sub, Dataproc, and Vertex AI
triggers:
  - build a GCP data pipeline with BigQuery
  - set up Google Cloud data engineering project
  - create Dataflow pipeline on GCP
  - use Apache Beam with BigQuery
  - implement Cloud Composer workflow
  - process data with Dataproc Serverless
  - build streaming pipeline with Pub/Sub
  - integrate Gemini AI with data pipeline
---

# Google Cloud Data Engineering Hub Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to help developers build production-grade Google Cloud Platform (GCP) data engineering solutions using this comprehensive reference repository of 54+ working projects covering BigQuery, Dataflow, Apache Beam, Cloud Composer, Pub/Sub, Dataproc, Gemini AI, and Vertex AI.

## What This Project Provides

A curated collection of complete, runnable GCP data engineering projects. Each project includes:
- Modular Python code (not scripts)
- ASCII architecture diagrams
- Sample data fixtures
- `deploy.sh` with GCP setup automation
- End-to-end working examples tested against live GCP environments

Built by Vishal Bulbule (Google Developer Expert, 12x GCP Certified).

## Installation & Setup

### Clone the Repository

```bash
git clone https://github.com/vishal-bulbule/google-cloud-data-engineering-hub.git
cd google-cloud-data-engineering-hub
```

### Prerequisites

- Python 3.10+
- Google Cloud SDK (`gcloud` CLI)
- Active GCP project with billing enabled
- Appropriate IAM permissions

### Environment Setup

```bash
# Set up Python virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies (per project)
cd <project-directory>
pip install -r requirements.txt

# Configure GCP credentials
export GOOGLE_CLOUD_PROJECT=your-project-id
export GOOGLE_CLOUD_LOCATION=us-central1
gcloud auth application-default login
```

## Project Categories & Key Examples

### BigQuery Projects (01-07)

#### CSV Ingestion Pipeline (01-bq-csv-ingestion-pipeline)

```python
from google.cloud import bigquery

def load_csv_to_bigquery(
    project_id: str,
    dataset_id: str,
    table_id: str,
    csv_file_path: str
):
    """Load CSV from local disk to BigQuery."""
    client = bigquery.Client(project=project_id)
    
    table_ref = f"{project_id}.{dataset_id}.{table_id}"
    
    job_config = bigquery.LoadJobConfig(
        source_format=bigquery.SourceFormat.CSV,
        skip_leading_rows=1,
        autodetect=True,
        write_disposition=bigquery.WriteDisposition.WRITE_TRUNCATE,
    )
    
    with open(csv_file_path, "rb") as source_file:
        job = client.load_table_from_file(
            source_file,
            table_ref,
            job_config=job_config
        )
    
    job.result()  # Wait for completion
    print(f"Loaded {job.output_rows} rows into {table_ref}")
```

#### UPSERT/MERGE Pattern (03-bq-upsert-merge-pattern)

```python
from google.cloud import bigquery

def upsert_data(project_id: str, dataset_id: str, table_id: str):
    """Perform MERGE operation for upsert pattern."""
    client = bigquery.Client(project=project_id)
    
    merge_query = f"""
    MERGE `{project_id}.{dataset_id}.{table_id}` AS target
    USING `{project_id}.{dataset_id}.staging_table` AS source
    ON target.id = source.id
    WHEN MATCHED THEN
      UPDATE SET 
        name = source.name,
        value = source.value,
        updated_at = CURRENT_TIMESTAMP()
    WHEN NOT MATCHED THEN
      INSERT (id, name, value, created_at, updated_at)
      VALUES (source.id, source.name, source.value, 
              CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP())
    """
    
    query_job = client.query(merge_query)
    result = query_job.result()
    print(f"MERGE completed. Rows modified: {result.total_rows}")
```

#### BigQuery ML (05-bq-ml-train-predict)

```python
from google.cloud import bigquery

def train_ml_model(project_id: str, dataset_id: str):
    """Train a logistic regression model using BigQuery ML."""
    client = bigquery.Client(project=project_id)
    
    training_query = f"""
    CREATE OR REPLACE MODEL `{project_id}.{dataset_id}.classification_model`
    OPTIONS(
      model_type='LOGISTIC_REG',
      input_label_cols=['label'],
      max_iterations=10
    ) AS
    SELECT
      feature1,
      feature2,
      feature3,
      label
    FROM `{project_id}.{dataset_id}.training_data`
    """
    
    job = client.query(training_query)
    job.result()
    print("Model training completed")

def predict_with_model(project_id: str, dataset_id: str):
    """Make predictions using trained BQML model."""
    client = bigquery.Client(project=project_id)
    
    prediction_query = f"""
    SELECT
      *
    FROM ML.PREDICT(
      MODEL `{project_id}.{dataset_id}.classification_model`,
      (SELECT feature1, feature2, feature3 
       FROM `{project_id}.{dataset_id}.prediction_data`)
    )
    """
    
    results = client.query(prediction_query).to_dataframe()
    return results
```

### Cloud Storage Projects (08-10)

#### File Management (08-gcs-file-management)

```python
from google.cloud import storage

def upload_blob(bucket_name: str, source_file: str, destination_blob: str):
    """Upload a file to GCS."""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(destination_blob)
    
    blob.upload_from_filename(source_file)
    print(f"File {source_file} uploaded to {destination_blob}")

def list_blobs_with_prefix(bucket_name: str, prefix: str):
    """List all blobs with a specific prefix."""
    storage_client = storage.Client()
    blobs = storage_client.list_blobs(bucket_name, prefix=prefix)
    
    return [blob.name for blob in blobs]

def copy_blob(bucket_name: str, blob_name: str, 
              destination_bucket: str, destination_blob: str):
    """Copy a blob within or across buckets."""
    storage_client = storage.Client()
    source_bucket = storage_client.bucket(bucket_name)
    source_blob = source_bucket.blob(blob_name)
    dest_bucket = storage_client.bucket(destination_bucket)
    
    source_bucket.copy_blob(source_blob, dest_bucket, destination_blob)
    print(f"Blob {blob_name} copied to {destination_blob}")
```

#### Signed URLs & Lifecycle (09-gcs-signed-urls-lifecycle)

```python
from google.cloud import storage
from datetime import timedelta

def generate_signed_url(bucket_name: str, blob_name: str, 
                        expiration_minutes: int = 15):
    """Generate a v4 signed URL for secure access."""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    
    url = blob.generate_signed_url(
        version="v4",
        expiration=timedelta(minutes=expiration_minutes),
        method="GET"
    )
    
    return url

def set_lifecycle_policy(bucket_name: str):
    """Set lifecycle rules for storage cost optimization."""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    
    lifecycle_rules = [
        {
            "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
            "condition": {"age": 30, "matchesPrefix": ["archive/"]}
        },
        {
            "action": {"type": "Delete"},
            "condition": {"age": 365, "matchesPrefix": ["temp/"]}
        }
    ]
    
    bucket.lifecycle_rules = lifecycle_rules
    bucket.patch()
    print(f"Lifecycle policy set for bucket {bucket_name}")
```

### Pub/Sub Streaming (31-pubsub-streaming-pipeline)

```python
from google.cloud import pubsub_v1
import json

def publish_messages(project_id: str, topic_name: str, messages: list):
    """Publish messages to Pub/Sub topic."""
    publisher = pubsub_v1.PublisherClient()
    topic_path = publisher.topic_path(project_id, topic_name)
    
    futures = []
    for message in messages:
        message_json = json.dumps(message)
        future = publisher.publish(
            topic_path,
            message_json.encode("utf-8"),
            origin="data-pipeline",
            priority="high"
        )
        futures.append(future)
    
    # Wait for all messages to publish
    for future in futures:
        future.result()
    
    print(f"Published {len(messages)} messages to {topic_name}")

def subscribe_messages(project_id: str, subscription_name: str, 
                       callback_fn, timeout: int = None):
    """Subscribe and process messages from Pub/Sub."""
    subscriber = pubsub_v1.SubscriberClient()
    subscription_path = subscriber.subscription_path(
        project_id, subscription_name
    )
    
    flow_control = pubsub_v1.types.FlowControl(
        max_messages=100,
        max_bytes=10 * 1024 * 1024,  # 10MB
    )
    
    streaming_pull_future = subscriber.subscribe(
        subscription_path,
        callback=callback_fn,
        flow_control=flow_control
    )
    
    print(f"Listening for messages on {subscription_path}...")
    
    try:
        streaming_pull_future.result(timeout=timeout)
    except KeyboardInterrupt:
        streaming_pull_future.cancel()

# Example callback
def message_callback(message):
    """Process received message."""
    print(f"Received: {message.data.decode('utf-8')}")
    message.ack()
```

### Apache Beam / Dataflow (47-50)

#### Basic Beam Pipeline (47-beam-data-transformation)

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

def run_word_count_pipeline(input_path: str, output_path: str):
    """Classic WordCount example with Beam."""
    options = PipelineOptions()
    
    with beam.Pipeline(options=options) as pipeline:
        (pipeline
         | 'Read' >> beam.io.ReadFromText(input_path)
         | 'Split' >> beam.FlatMap(lambda line: line.split())
         | 'PairWithOne' >> beam.Map(lambda word: (word, 1))
         | 'GroupAndSum' >> beam.CombinePerKey(sum)
         | 'Format' >> beam.Map(lambda kv: f"{kv[0]}: {kv[1]}")
         | 'Write' >> beam.io.WriteToText(output_path)
        )

def csv_transform_pipeline(input_file: str, output_file: str):
    """Transform CSV data with Beam."""
    
    def parse_csv(line):
        import csv
        from io import StringIO
        reader = csv.DictReader(StringIO(line))
        return next(reader)
    
    def transform_record(record):
        return {
            'id': record['id'],
            'name': record['name'].upper(),
            'value': float(record['value']) * 1.1,
            'processed': True
        }
    
    options = PipelineOptions()
    
    with beam.Pipeline(options=options) as pipeline:
        (pipeline
         | 'Read CSV' >> beam.io.ReadFromText(input_file, skip_header_lines=1)
         | 'Parse' >> beam.Map(parse_csv)
         | 'Transform' >> beam.Map(transform_record)
         | 'Format JSON' >> beam.Map(lambda x: json.dumps(x))
         | 'Write' >> beam.io.WriteToText(output_file)
        )
```

#### Beam to BigQuery (48-beam-csv-to-bigquery-load)

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
from apache_beam.io.gcp.bigquery import WriteToBigQuery

def csv_to_bigquery_pipeline(
    input_file: str,
    project_id: str,
    dataset_id: str,
    table_id: str
):
    """Load CSV to BigQuery using Beam."""
    
    table_spec = f"{project_id}:{dataset_id}.{table_id}"
    
    table_schema = {
        'fields': [
            {'name': 'id', 'type': 'INTEGER', 'mode': 'REQUIRED'},
            {'name': 'name', 'type': 'STRING', 'mode': 'REQUIRED'},
            {'name': 'value', 'type': 'FLOAT', 'mode': 'NULLABLE'},
            {'name': 'timestamp', 'type': 'TIMESTAMP', 'mode': 'REQUIRED'}
        ]
    }
    
    def parse_csv_row(line):
        parts = line.split(',')
        return {
            'id': int(parts[0]),
            'name': parts[1],
            'value': float(parts[2]),
            'timestamp': parts[3]
        }
    
    options = PipelineOptions()
    
    with beam.Pipeline(options=options) as pipeline:
        (pipeline
         | 'Read CSV' >> beam.io.ReadFromText(input_file, skip_header_lines=1)
         | 'Parse Rows' >> beam.Map(parse_csv_row)
         | 'Write to BigQuery' >> WriteToBigQuery(
             table_spec,
             schema=table_schema,
             write_disposition=beam.io.BigQueryDisposition.WRITE_TRUNCATE,
             create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED
         )
        )
```

#### Advanced Beam Patterns (49-beam-advanced-patterns)

```python
import apache_beam as beam
from apache_beam import window
from apache_beam.transforms.trigger import AfterWatermark, AfterCount

def windowing_pipeline(input_subscription: str, output_table: str):
    """Event-time windowing with late data handling."""
    
    class ParseEvent(beam.DoFn):
        def process(self, element):
            import json
            from datetime import datetime
            
            data = json.loads(element)
            timestamp = datetime.fromisoformat(data['timestamp'])
            
            yield beam.window.TimestampedValue(
                data,
                timestamp.timestamp()
            )
    
    options = PipelineOptions()
    
    with beam.Pipeline(options=options) as pipeline:
        events = (pipeline
                  | 'Read from Pub/Sub' >> beam.io.ReadFromPubSub(
                      subscription=input_subscription)
                  | 'Parse Events' >> beam.ParDo(ParseEvent())
                 )
        
        windowed = (events
                    | 'Apply Window' >> beam.WindowInto(
                        window.FixedWindows(60),  # 1-minute windows
                        trigger=AfterWatermark(early=AfterCount(10)),
                        allowed_lateness=300,  # 5-minute late data
                        accumulation_mode=beam.trigger.AccumulationMode.ACCUMULATING
                    )
                   )
        
        (windowed
         | 'Count by Key' >> beam.CombinePerKey(sum)
         | 'Format Output' >> beam.Map(lambda kv: {'key': kv[0], 'count': kv[1]})
         | 'Write to BigQuery' >> beam.io.WriteToBigQuery(output_table)
        )

def branching_pipeline_with_dead_letter():
    """Branching with side outputs and dead-letter queue."""
    
    class ValidateAndRoute(beam.DoFn):
        OUTPUT_TAG_VALID = 'valid'
        OUTPUT_TAG_INVALID = 'invalid'
        
        def process(self, element):
            try:
                if self.is_valid(element):
                    yield element
                else:
                    yield beam.pvalue.TaggedOutput(
                        self.OUTPUT_TAG_INVALID, 
                        {'error': 'validation_failed', 'data': element}
                    )
            except Exception as e:
                yield beam.pvalue.TaggedOutput(
                    self.OUTPUT_TAG_INVALID,
                    {'error': str(e), 'data': element}
                )
        
        def is_valid(self, element):
            return 'id' in element and 'value' in element
    
    options = PipelineOptions()
    
    with beam.Pipeline(options=options) as pipeline:
        results = (pipeline
                   | 'Read' >> beam.io.ReadFromText('input.txt')
                   | 'Validate' >> beam.ParDo(ValidateAndRoute()).with_outputs(
                       ValidateAndRoute.OUTPUT_TAG_INVALID,
                       main=ValidateAndRoute.OUTPUT_TAG_VALID
                   )
                  )
        
        # Main path
        (results[ValidateAndRoute.OUTPUT_TAG_VALID]
         | 'Process Valid' >> beam.Map(lambda x: x)
         | 'Write Valid' >> beam.io.WriteToText('valid_output')
        )
        
        # Dead-letter queue
        (results[ValidateAndRoute.OUTPUT_TAG_INVALID]
         | 'Format Errors' >> beam.Map(lambda x: json.dumps(x))
         | 'Write DLQ' >> beam.io.WriteToText('dead_letter_queue')
        )
```

### Cloud Composer / Airflow (51-54)

#### Basic DAG (51-cloud-composer-dag-basics)

```python
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.dummy import DummyOperator
from airflow.utils.task_group import TaskGroup
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-eng-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

def extract_data(**context):
    """Extract data from source."""
    data = {'records': 1000, 'source': 'api'}
    context['ti'].xcom_push(key='extract_result', value=data)
    return data

def decide_branch(**context):
    """Branch based on extracted data volume."""
    ti = context['ti']
    data = ti.xcom_pull(key='extract_result', task_ids='extract')
    
    if data['records'] > 500:
        return 'process_large'
    else:
        return 'process_small'

with DAG(
    'data_pipeline_basic',
    default_args=default_args,
    schedule_interval='@daily',
    catchup=False,
    tags=['data-engineering', 'example']
) as dag:
    
    start = DummyOperator(task_id='start')
    
    extract = PythonOperator(
        task_id='extract',
        python_callable=extract_data,
        provide_context=True
    )
    
    branch = BranchPythonOperator(
        task_id='branch',
        python_callable=decide_branch,
        provide_context=True
    )
    
    process_large = PythonOperator(
        task_id='process_large',
        python_callable=lambda: print("Processing large dataset")
    )
    
    process_small = PythonOperator(
        task_id='process_small',
        python_callable=lambda: print("Processing small dataset")
    )
    
    end = DummyOperator(
        task_id='end',
        trigger_rule='none_failed_min_one_success'
    )
    
    start >> extract >> branch >> [process_large, process_small] >> end
```

#### BigQuery Pipeline (52-composer-bigquery-pipeline)

```python
from airflow import DAG
from airflow.providers.google.cloud.sensors.gcs import GCSObjectExistenceSensor
from airflow.providers.google.cloud.transfers.gcs_to_bigquery import GCSToBigQueryOperator
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator
from airflow.operators.python import PythonOperator
from datetime import datetime

GCS_BUCKET = 'your-data-bucket'
PROJECT_ID = 'your-project-id'
DATASET_ID = 'analytics'

def validate_data(**context):
    """Validate loaded data quality."""
    from google.cloud import bigquery
    
    client = bigquery.Client()
    query = f"""
    SELECT 
        COUNT(*) as total_rows,
        COUNTIF(id IS NULL) as null_ids,
        COUNTIF(value < 0) as negative_values
    FROM `{PROJECT_ID}.{DATASET_ID}.staging_table`
    """
    
    results = client.query(query).result()
    for row in results:
        if row.null_ids > 0 or row.negative_values > 0:
            raise ValueError(f"Data quality check failed: {dict(row)}")
    
    print(f"Validation passed: {row.total_rows} rows")

with DAG(
    'bigquery_etl_pipeline',
    start_date=datetime(2024, 1, 1),
    schedule_interval='0 2 * * *',  # 2 AM daily
    catchup=False
) as dag:
    
    wait_for_file = GCSObjectExistenceSensor(
        task_id='wait_for_file',
        bucket=GCS_BUCKET,
        object='data/input_{{ ds_nodash }}.csv',
        timeout=3600,
        poke_interval=60
    )
    
    load_to_staging = GCSToBigQueryOperator(
        task_id='load_to_staging',
        bucket=GCS_BUCKET,
        source_objects=['data/input_{{ ds_nodash }}.csv'],
        destination_project_dataset_table=f'{PROJECT_ID}.{DATASET_ID}.staging_table',
        source_format='CSV',
        skip_leading_rows=1,
        write_disposition='WRITE_TRUNCATE',
        autodetect=True
    )
    
    transform_data = BigQueryInsertJobOperator(
        task_id='transform_data',
        configuration={
            'query': {
                'query': f"""
                    INSERT INTO `{PROJECT_ID}.{DATASET_ID}.final_table`
                    SELECT 
                        id,
                        UPPER(name) as name,
                        value * 1.1 as adjusted_value,
                        CURRENT_TIMESTAMP() as processed_at
                    FROM `{PROJECT_ID}.{DATASET_ID}.staging_table`
                    WHERE value > 0
                """,
                'useLegacySql': False
            }
        }
    )
    
    validate = PythonOperator(
        task_id='validate_data',
        python_callable=validate_data,
        provide_context=True
    )
    
    wait_for_file >> load_to_staging >> transform_data >> validate
```

### Gemini AI Integration (11-13)

#### Text Generation (11-gemini-text-generation-basics)

```python
import vertexai
from vertexai.generative_models import GenerativeModel, GenerationConfig

def generate_text(project_id: str, location: str, prompt: str):
    """Generate text using Gemini."""
    vertexai.init(project=project_id, location=location)
    
    model = GenerativeModel("gemini-1.5-pro")
    
    generation_config = GenerationConfig(
        temperature=0.7,
        top_p=0.95,
        top_k=40,
        max_output_tokens=1024,
    )
    
    response = model.generate_content(
        prompt,
        generation_config=generation_config,
        stream=False
    )
    
    return response.text

def stream_generation(project_id: str, location: str, prompt: str):
    """Stream text generation for real-time output."""
    vertexai.init(project=project_id, location=location)
    
    model = GenerativeModel("gemini-1.5-pro")
    
    responses = model.generate_content(prompt, stream=True)
    
    for response in responses:
        print(response.text, end='')
```

#### Multimodal Analysis (12-gemini-multimodal-analysis)

```python
import vertexai
from vertexai.generative_models import GenerativeModel, Part
from google.cloud import storage

def analyze_image(project_id: str, location: str, 
                  gcs_uri: str, prompt: str):
    """Analyze an image using Gemini."""
    vertexai.init(project=project_id, location=location)
    
    model = GenerativeModel("gemini-1.5-pro")
    
    image_part = Part.from_uri(gcs_uri, mime_type="image/jpeg")
    
    response = model.generate_content([prompt, image_part])
    
    return response.text

def analyze_pdf_document(project_id: str, location: str, pdf_gcs_uri: str):
    """Extract and analyze content from PDF."""
    vertexai.init(project=project_id, location=location)
    
    model = GenerativeModel("gemini-1.5-pro")
    
    pdf_part = Part.from_uri(pdf_gcs_uri, mime_type="application/pdf")
    
    prompt = """
    Analyze this document and provide:
    1. Main topics covered
    2. Key data points or statistics
    3. Summary of conclusions
    """
    
    response = model.generate_content([prompt, pdf_part])
    return response.text
```

#### Function Calling (13-gemini-function-calling-tools)

```python
import vertexai
from vertexai.generative_models import (
    GenerativeModel,
    FunctionDeclaration,
    Tool
)

def get_weather(location: str):
    """Simulated weather API."""
    return {
        "location": location,
        "temperature": 72,
        "conditions": "sunny"
    }

def setup_function_calling(project_id: str, location: str):
    """Set up Gemini with function calling."""
    vertexai.init(project=project_id, location=location)
    
    # Define function schema
    get_weather_func = FunctionDeclaration(
        name="get_weather",
        description="Get current weather for a location",
        parameters={
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "City name"
                }
            },
            "required": ["location"]
        }
    )
    
    weather_tool = Tool(function_declarations=[get_weather_func])
    
    model = GenerativeModel(
        "gemini-1.5-pro",
        tools=[weather_tool]
    )
    
    # User asks about weather
    chat = model.start_chat()
    response = chat.send_message("What's the weather in San Francisco?")
    
    # Check if function call is requested
    function_call = response.candidates[0].content.parts[0].function_call
    
    if function_call.name == "get_weather":
        # Execute function
        weather_data = get_weather(
            location=function_call.args["location"]
        )
        
        # Send result back to model
        response = chat.send_message(
            Part.from_function_response(
                name="get_weather",
                response={"content": weather_data}
            )
        )
        
        return response.text
```

## Common Deployment Patterns

### Project Deployment Script

Each project includes a `deploy.sh` script:

```bash
#!/bin/bash
set -e

PROJECT_ID=${GOOGLE_CLOUD_PROJECT}
LOCATION=${GOOGLE_CLOUD_LOCATION:-us-central1}

# Enable required APIs
gcloud services enable bigquery.googleapis.com \
    storage.googleapis.com \
    dataflow.googleapis.com \
    --project=${PROJECT_ID}

# Create resources
bq mk --dataset ${PROJECT_ID}:analytics
gsutil mb -l ${LOCATION} gs://${PROJECT_ID}-data

echo "✓ Deployment complete"
```

### Running a Project

```bash
cd <project-directory>

# Review and run deployment
chmod +x deploy.sh
./deploy.sh

# Run main pipeline
python main.py

# Or with arguments
python main.py --project-id=$GOOGLE_CLOUD_PROJECT \
               --location=us-central1 \
               --input-file=data/sample.csv
```

## Configuration Patterns

### Environment Variables

```bash
# Required for all projects
export GOOGLE_CLOUD_PROJECT=your-project-id
export GOOGLE_CLOUD_LOCATION=us-central1

# Optional performance tuning
export BEAM_MAX_NUM_WORKERS=10
export BQ_BATCH_SIZE=1000

# Credentials (use Application Default Credentials)
gcloud auth application-default login
```

### Config Files (config.yaml)

```yaml
project_id: ${GOOGLE_CLOUD_PROJECT}
location: us-central1

bigquery:
  dataset: analytics
  staging_dataset: staging
  
storage:
  bucket: ${GOOGLE_CLOUD_PROJECT}-data
  temp_location: gs://${GOOGLE_CLOUD_PROJECT}-data/temp

dataflow:
  runner: DataflowRunner
  num_workers: 2
  max_num_workers: 10
  machine_type: n1-standard-2
  
composer:
  environment: production-composer
  dag_folder: dags/
```

### Loading Configuration

```python
import os
import yaml
from string import Template

def load_config(config_path: str = 'config.yaml') -> dict:
    """Load configuration with environment variable substitution."""
    with open(config_path, 'r') as f:
        config_template = f.read()
