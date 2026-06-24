---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - show me how to use the Harvard artifacts data engineering app
  - help me create a data pipeline for museum artifacts
  - how to set up SQL analytics for Harvard Art Museums data
  - build a Streamlit dashboard for art museum data
  - extract and analyze Harvard Art Museums collection data
  - create an end-to-end data engineering project with museum API
  - how to visualize Harvard Art Museums data with Plotly
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Stores data in MySQL/TiDB Cloud with proper schema design and foreign key relationships
- **Interactive Dashboards**: Streamlit-based UI for running SQL queries and visualizing results with Plotly
- **Real-world Simulation**: Demonstrates patterns used in production data engineering roles

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### Environment Variables

Set up your configuration using environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### API Key Setup

1. Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Store it securely in your environment or `.env` file
3. Never commit API keys to version control

### Database Setup

```python
import mysql.connector
from sqlalchemy import create_engine

# Using mysql-connector-python
connection = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

# Using SQLAlchemy
engine = create_engine(
    f"mysql+mysqlconnector://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
)
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Commands

### Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Run with custom port
streamlit run app.py --server.port 8501
```

## Code Examples

### Fetching Data from Harvard API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector

def extract_artifacts(api_key, num_pages=5):
    """Extract artifact data with pagination"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(api_key, page=page, size=100)
        all_artifacts.extend(data['records'])
        
    return all_artifacts

def transform_artifacts(artifacts):
    """Transform nested JSON to flat structure"""
    metadata_list = []
    media_list = []
    color_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('images'):
            for img in artifact['images']:
                media_list.append({
                    'artifact_id': artifact['id'],
                    'image_url': img.get('baseimageurl'),
                    'media_type': 'image'
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_list.append({
                    'artifact_id': artifact['id'],
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(color_list)
    )

def load_to_database(df, table_name, connection):
    """Load dataframe to SQL database"""
    cursor = connection.cursor()
    
    # Create insert query
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data = [tuple(row) for row in df.values]
    cursor.executemany(query, data)
    connection.commit()
    
    print(f"Loaded {len(df)} records into {table_name}")
```

### Complete ETL Execution

```python
import os

# Configuration
api_key = os.getenv('HARVARD_API_KEY')
connection = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

# Execute ETL
artifacts = extract_artifacts(api_key, num_pages=5)
metadata_df, media_df, colors_df = transform_artifacts(artifacts)

# Load to database
load_to_database(metadata_df, 'artifactmetadata', connection)
load_to_database(media_df, 'artifactmedia', connection)
load_to_database(colors_df, 'artifactcolors', connection)

connection.close()
```

### SQL Analytics Queries

```python
def run_analytical_query(connection, query_name):
    """Execute predefined analytical queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'media_availability': """
            SELECT 
                CASE WHEN media_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia a ON m.id = a.artifact_id
            GROUP BY media_status
        """,
        
        'top_colors': """
            SELECT color, SUM(percentage) as total_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY total_percentage DESC
            LIMIT 10
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """
    }
    
    cursor = connection.cursor()
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    
    return pd.DataFrame(results, columns=columns)

# Usage
df_culture = run_analytical_query(connection, 'artifacts_by_culture')
print(df_culture)
```

### Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

st.title("Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Analytics Queries")
query_options = {
    "Artifacts by Culture": "artifacts_by_culture",
    "Artifacts by Century": "artifacts_by_century",
    "Media Availability": "media_availability",
    "Top Colors": "top_colors",
    "Department Distribution": "department_distribution"
}

selected_query = st.sidebar.selectbox(
    "Select Query",
    list(query_options.keys())
)

# Execute query
if st.button("Run Query"):
    with st.spinner("Executing query..."):
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        df = run_analytical_query(connection, query_options[selected_query])
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Visualization
        if len(df.columns) >= 2:
            fig = px.bar(
                df,
                x=df.columns[0],
                y=df.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig)
        
        connection.close()
```

### Data Collection with Rate Limiting

```python
import time

def collect_artifacts_with_limit(api_key, total_records=1000, rate_limit=1):
    """
    Collect artifacts with rate limiting to respect API limits
    """
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < total_records:
        try:
            data = fetch_artifacts(api_key, page=page, size=size)
            artifacts.extend(data['records'])
            
            print(f"Fetched page {page}: {len(artifacts)} total artifacts")
            
            page += 1
            time.sleep(rate_limit)  # Rate limiting
            
            # Check if we've reached the end
            if len(data['records']) < size:
                break
                
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return artifacts[:total_records]
```

## Common Patterns

### Pattern 1: Incremental ETL Updates

```python
def incremental_update(api_key, connection, last_id):
    """
    Only fetch artifacts newer than last_id
    """
    cursor = connection.cursor()
    
    # Fetch new artifacts
    new_artifacts = []
    page = 1
    
    while True:
        data = fetch_artifacts(api_key, page=page)
        records = [r for r in data['records'] if r['id'] > last_id]
        
        if not records:
            break
            
        new_artifacts.extend(records)
        page += 1
    
    # Transform and load
    metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
    load_to_database(metadata_df, 'artifactmetadata', connection)
    load_to_database(media_df, 'artifactmedia', connection)
    load_to_database(colors_df, 'artifactcolors', connection)
    
    return len(new_artifacts)
```

### Pattern 2: Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(api_key, connection):
    """ETL with comprehensive error handling"""
    try:
        logger.info("Starting ETL pipeline")
        
        # Extract
        artifacts = extract_artifacts(api_key)
        logger.info(f"Extracted {len(artifacts)} artifacts")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        logger.info("Transformation complete")
        
        # Load
        load_to_database(metadata_df, 'artifactmetadata', connection)
        load_to_database(media_df, 'artifactmedia', connection)
        load_to_database(colors_df, 'artifactcolors', connection)
        logger.info("Load complete")
        
        return True
        
    except Exception as e:
        logger.error(f"ETL pipeline failed: {e}")
        connection.rollback()
        return False
```

### Pattern 3: Data Quality Checks

```python
def validate_data(df, table_name):
    """Validate data before loading"""
    issues = []
    
    # Check for nulls in required fields
    if table_name == 'artifactmetadata':
        if df['id'].isnull().any():
            issues.append("NULL values in id column")
    
    # Check data types
    if table_name == 'artifactcolors':
        if not pd.api.types.is_numeric_dtype(df['percentage']):
            issues.append("percentage column must be numeric")
    
    # Check duplicates
    if df.duplicated().any():
        issues.append(f"{df.duplicated().sum()} duplicate rows found")
    
    return issues

# Usage
issues = validate_data(metadata_df, 'artifactmetadata')
if issues:
    print("Data quality issues:", issues)
else:
    load_to_database(metadata_df, 'artifactmetadata', connection)
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time
from functools import wraps

def retry_with_backoff(retries=3, backoff_in_seconds=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            x = 0
            while True:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if x == retries:
                        raise
                    sleep_time = backoff_in_seconds * 2 ** x
                    time.sleep(sleep_time)
                    x += 1
        return wrapper
    return decorator

@retry_with_backoff(retries=3)
def fetch_artifacts_safe(api_key, page, size):
    return fetch_artifacts(api_key, page, size)
```

### Database Connection Issues

```python
def get_db_connection(max_retries=3):
    """Get database connection with retry logic"""
    for attempt in range(max_retries):
        try:
            connection = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connect_timeout=10
            )
            return connection
        except mysql.connector.Error as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

### Memory Management for Large Datasets

```python
def batch_load(df, table_name, connection, batch_size=1000):
    """Load large dataframes in batches"""
    total_rows = len(df)
    
    for start in range(0, total_rows, batch_size):
        end = min(start + batch_size, total_rows)
        batch = df.iloc[start:end]
        load_to_database(batch, table_name, connection)
        print(f"Loaded batch {start}-{end} of {total_rows}")
```

### Streamlit Caching

```python
import streamlit as st

@st.cache_data(ttl=3600)
def load_cached_query(query_name):
    """Cache query results for 1 hour"""
    connection = get_db_connection()
    df = run_analytical_query(connection, query_name)
    connection.close()
    return df

# Usage in Streamlit
df = load_cached_query(selected_query)
st.dataframe(df)
```

This skill enables AI agents to guide developers through building complete ETL pipelines, designing SQL schemas, creating analytics dashboards, and visualizing museum artifact data using industry-standard tools and best practices.
