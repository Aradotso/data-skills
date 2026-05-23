---
name: harvard-artifacts-data-engineering-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline for Harvard Art Museums API
  - create ETL workflow for art museum artifacts
  - set up Harvard artifacts analytics dashboard
  - extract and analyze Harvard museum collection data
  - build Streamlit app for art museum data
  - query Harvard Art Museums API with pagination
  - create SQL schema for artifact metadata
  - visualize museum artifact analytics
---

# Harvard Artifacts Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

An end-to-end data engineering and analytics application that:
- Extracts artifact data from Harvard Art Museums API with pagination handling
- Transforms nested JSON into relational database tables
- Loads structured data into MySQL/TiDB Cloud
- Provides 20+ predefined SQL analytics queries
- Visualizes insights through interactive Streamlit dashboards

Architecture: **API → ETL → SQL → Analytics → Visualization**

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

### 1. Harvard Art Museums API Key

Get your API key from: https://harvardartmuseums.org/collections/api

```python
# Set as environment variable
export HARVARD_API_KEY="your_api_key_here"

# Or configure in app
api_key = os.getenv('HARVARD_API_KEY')
```

### 2. Database Configuration

```python
# MySQL/TiDB Cloud connection
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

### 3. Environment Variables Setup

```bash
# .env file
HARVARD_API_KEY=your_api_key
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    credit_line TEXT,
    accession_number VARCHAR(100),
    provenance TEXT,
    copyright TEXT,
    url VARCHAR(500),
    last_updated TIMESTAMP
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    base_url VARCHAR(500),
    alt_text TEXT,
    caption TEXT,
    technique VARCHAR(255),
    width INT,
    height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    color_percent FLOAT,
    css_color VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## API Integration

### Extract Artifacts with Pagination

```python
import requests
import time

def fetch_artifacts(api_key, total_records=100, page_size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < total_records:
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            records = data.get('records', [])
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
            
            # Rate limiting - respect API limits
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts[:total_records]
```

### Handle Nested JSON

```python
def extract_artifact_metadata(artifact):
    """Extract metadata from artifact JSON"""
    return {
        'artifact_id': artifact.get('id'),
        'title': artifact.get('title'),
        'culture': artifact.get('culture'),
        'period': artifact.get('period'),
        'century': artifact.get('century'),
        'classification': artifact.get('classification'),
        'department': artifact.get('department'),
        'division': artifact.get('division'),
        'dated': artifact.get('dated'),
        'medium': artifact.get('medium'),
        'dimensions': artifact.get('dimensions'),
        'credit_line': artifact.get('creditline'),
        'accession_number': artifact.get('accessionmethod'),
        'provenance': artifact.get('provenance'),
        'copyright': artifact.get('copyright'),
        'url': artifact.get('url'),
        'last_updated': artifact.get('lastupdate')
    }

def extract_artifact_media(artifact):
    """Extract media/images from artifact JSON"""
    media_list = []
    artifact_id = artifact.get('id')
    
    for image in artifact.get('images', []):
        media_list.append({
            'artifact_id': artifact_id,
            'image_url': image.get('iiifbaseuri'),
            'base_url': image.get('baseimageurl'),
            'alt_text': image.get('alttext'),
            'caption': image.get('caption'),
            'technique': image.get('technique'),
            'width': image.get('width'),
            'height': image.get('height')
        })
    
    return media_list

def extract_artifact_colors(artifact):
    """Extract color data from artifact JSON"""
    colors_list = []
    artifact_id = artifact.get('id')
    
    for color in artifact.get('colors', []):
        colors_list.append({
            'artifact_id': artifact_id,
            'color_hex': color.get('hex'),
            'color_name': color.get('color'),
            'color_percent': color.get('percent'),
            'css_color': color.get('css3')
        })
    
    return colors_list
```

## ETL Pipeline

### Complete ETL Workflow

```python
import pandas as pd
import mysql.connector
from sqlalchemy import create_engine

class HarvardArtifactsETL:
    def __init__(self, api_key, db_config):
        self.api_key = api_key
        self.db_config = db_config
        self.engine = create_engine(
            f"mysql+mysqlconnector://{db_config['user']}:{db_config['password']}@"
            f"{db_config['host']}:{db_config['port']}/{db_config['database']}"
        )
    
    def extract(self, num_records=100):
        """Extract data from API"""
        print(f"Extracting {num_records} artifacts...")
        artifacts = fetch_artifacts(self.api_key, num_records)
        return artifacts
    
    def transform(self, artifacts):
        """Transform nested JSON to relational format"""
        print("Transforming data...")
        
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in artifacts:
            metadata_list.append(extract_artifact_metadata(artifact))
            media_list.extend(extract_artifact_media(artifact))
            colors_list.extend(extract_artifact_colors(artifact))
        
        df_metadata = pd.DataFrame(metadata_list)
        df_media = pd.DataFrame(media_list)
        df_colors = pd.DataFrame(colors_list)
        
        # Clean data
        df_metadata = df_metadata.fillna('')
        df_media = df_media.fillna('')
        df_colors = df_colors.fillna(0)
        
        return df_metadata, df_media, df_colors
    
    def load(self, df_metadata, df_media, df_colors):
        """Load data into SQL database"""
        print("Loading data to database...")
        
        try:
            # Load with batch inserts
            df_metadata.to_sql('artifactmetadata', self.engine, 
                             if_exists='append', index=False)
            df_media.to_sql('artifactmedia', self.engine, 
                          if_exists='append', index=False)
            df_colors.to_sql('artifactcolors', self.engine, 
                           if_exists='append', index=False)
            
            print(f"✓ Loaded {len(df_metadata)} artifacts")
            print(f"✓ Loaded {len(df_media)} media records")
            print(f"✓ Loaded {len(df_colors)} color records")
            
        except Exception as e:
            print(f"Error loading data: {e}")
            raise
    
    def run(self, num_records=100):
        """Execute full ETL pipeline"""
        artifacts = self.extract(num_records)
        df_metadata, df_media, df_colors = self.transform(artifacts)
        self.load(df_metadata, df_media, df_colors)
        return True

# Usage
etl = HarvardArtifactsETL(
    api_key=os.getenv('HARVARD_API_KEY'),
    db_config=db_config
)
etl.run(num_records=500)
```

## Analytics SQL Queries

### Common Analytical Queries

```python
# Sample analytics queries
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "media_availability": """
        SELECT 
            CASE WHEN m.artifact_id IS NULL THEN 'No Media' ELSE 'Has Media' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.artifact_id = m.artifact_id
        GROUP BY media_status
    """,
    
    "top_colors": """
        SELECT color_name, COUNT(*) as count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        WHERE color_name != ''
        GROUP BY color_name
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department != ''
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "classification_distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification != ''
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(query, db_config):
    """Execute SQL query and return DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def main():
    st.title("🎨 Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Database connection
    db_host = st.sidebar.text_input("Database Host", value="localhost")
    db_user = st.sidebar.text_input("Database User", value="root")
    db_password = st.sidebar.text_input("Database Password", type="password")
    
    # ETL Section
    st.header("📥 Data Collection")
    num_records = st.number_input("Number of artifacts to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            try:
                db_config = {
                    'host': db_host,
                    'user': db_user,
                    'password': db_password,
                    'database': 'harvard_artifacts'
                }
                etl = HarvardArtifactsETL(api_key, db_config)
                etl.run(num_records)
                st.success(f"✓ Successfully processed {num_records} artifacts!")
            except Exception as e:
                st.error(f"Error: {e}")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_options = list(ANALYTICS_QUERIES.keys())
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            try:
                df_result = execute_query(
                    ANALYTICS_QUERIES[selected_query], 
                    db_config
                )
                
                # Display table
                st.dataframe(df_result)
                
                # Auto-generate visualization
                if len(df_result.columns) >= 2:
                    fig = px.bar(df_result, 
                                x=df_result.columns[0], 
                                y=df_result.columns[1],
                                title=selected_query.replace('_', ' ').title())
                    st.plotly_chart(fig, use_container_width=True)
                    
            except Exception as e:
                st.error(f"Query error: {e}")

if __name__ == "__main__":
    main()
```

### Run the App

```bash
streamlit run app.py
```

## Common Patterns

### Batch Processing Large Datasets

```python
def batch_insert(df, table_name, engine, batch_size=1000):
    """Insert data in batches for better performance"""
    total_rows = len(df)
    
    for start_idx in range(0, total_rows, batch_size):
        end_idx = min(start_idx + batch_size, total_rows)
        batch = df.iloc[start_idx:end_idx]
        batch.to_sql(table_name, engine, if_exists='append', index=False)
        print(f"Inserted batch {start_idx}-{end_idx} of {total_rows}")
```

### Error Handling for API Calls

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    """Create requests session with retry logic"""
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

### Data Quality Validation

```python
def validate_artifacts(df):
    """Validate artifact data quality"""
    issues = []
    
    # Check for required fields
    if df['artifact_id'].isnull().any():
        issues.append("Missing artifact IDs detected")
    
    # Check for duplicates
    duplicates = df['artifact_id'].duplicated().sum()
    if duplicates > 0:
        issues.append(f"{duplicates} duplicate artifact IDs found")
    
    # Check data types
    if not pd.api.types.is_numeric_dtype(df['artifact_id']):
        issues.append("artifact_id should be numeric")
    
    return issues
```

## Troubleshooting

### API Rate Limiting

```python
# Add delays between requests
time.sleep(0.5)  # 500ms delay

# Monitor rate limit headers
if 'X-RateLimit-Remaining' in response.headers:
    remaining = int(response.headers['X-RateLimit-Remaining'])
    if remaining < 10:
        time.sleep(2)
```

### Database Connection Issues

```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("✓ Database connection successful")
    conn.close()
except mysql.connector.Error as err:
    print(f"✗ Connection failed: {err}")
```

### Memory Management for Large Datasets

```python
# Use chunking for large API responses
def fetch_artifacts_chunked(api_key, total_records, chunk_size=100):
    for offset in range(0, total_records, chunk_size):
        yield fetch_artifacts(api_key, chunk_size, page=offset//chunk_size)
```

### Empty API Responses

```python
# Handle missing data gracefully
records = data.get('records', [])
if not records:
    print("No more records available")
    break

# Validate response structure
if 'info' not in data or 'records' not in data:
    raise ValueError("Invalid API response structure")
```
