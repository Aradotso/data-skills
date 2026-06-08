---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - set up analytics dashboard with Harvard artifacts API
  - create data engineering project with museum artifacts
  - extract and analyze Harvard art collections data
  - build Streamlit app for Harvard museums API
  - set up SQL database for artifact metadata
  - visualize Harvard art museums data with Plotly
  - create end-to-end data pipeline for museum collections
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media details, and color information
- **SQL Database**: Store structured data in MySQL/TiDB with proper relational design
- **Analytics**: Execute 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly dashboards in Streamlit

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)

Create a `.env` file or configure in your app:
```bash
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud connection:
```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os

class HarvardAPIClient:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, size=100, page=1):
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_all_artifacts(self, max_records=1000):
        """Fetch multiple pages of artifacts"""
        all_artifacts = []
        page = 1
        size = 100
        
        while len(all_artifacts) < max_records:
            data = self.fetch_artifacts(size=size, page=page)
            records = data.get('records', [])
            
            if not records:
                break
            
            all_artifacts.extend(records)
            page += 1
            
            # Respect rate limits
            import time
            time.sleep(0.5)
        
        return all_artifacts[:max_records]

# Usage
api_key = os.getenv('HARVARD_API_KEY')
client = HarvardAPIClient(api_key)
artifacts = client.fetch_all_artifacts(max_records=500)
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifacts into metadata DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division'),
            'objectnumber': artifact.get('objectnumber')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """Transform media information"""
    media_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_data.append({
                'artifact_id': artifact_id,
                'media_id': img.get('imageid'),
                'media_url': img.get('baseimageurl'),
                'media_type': 'image',
                'width': img.get('width'),
                'height': img.get('height')
            })
        
        # Handle primary image
        primary_image = artifact.get('primaryimageurl')
        if primary_image:
            media_data.append({
                'artifact_id': artifact_id,
                'media_id': f"{artifact_id}_primary",
                'media_url': primary_image,
                'media_type': 'primary_image',
                'width': None,
                'height': None
            })
    
    return pd.DataFrame(media_data)

def transform_artifact_colors(artifacts):
    """Transform color information"""
    color_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'color_percent': color.get('percent'),
                'color_spectrum': color.get('spectrum')
            })
    
    return pd.DataFrame(color_data)
```

### 3. Database Schema and Loading

```python
import mysql.connector
from mysql.connector import Error

class ArtifactDatabase:
    def __init__(self, config):
        self.config = config
        self.connection = None
    
    def connect(self):
        """Create database connection"""
        try:
            self.connection = mysql.connector.connect(**self.config)
            return self.connection
        except Error as e:
            print(f"Error connecting to database: {e}")
            return None
    
    def create_tables(self):
        """Create database schema"""
        cursor = self.connection.cursor()
        
        # Metadata table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                artifact_id INT PRIMARY KEY,
                title VARCHAR(500),
                culture VARCHAR(255),
                century VARCHAR(100),
                classification VARCHAR(255),
                department VARCHAR(255),
                dated VARCHAR(255),
                medium TEXT,
                dimensions VARCHAR(500),
                creditline TEXT,
                division VARCHAR(255),
                objectnumber VARCHAR(100)
            )
        """)
        
        # Media table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                media_id VARCHAR(255),
                media_url TEXT,
                media_type VARCHAR(50),
                width INT,
                height INT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        # Colors table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                color_hex VARCHAR(10),
                color_name VARCHAR(100),
                color_percent FLOAT,
                color_spectrum VARCHAR(100),
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        self.connection.commit()
    
    def load_metadata(self, df):
        """Bulk insert metadata"""
        cursor = self.connection.cursor()
        
        insert_query = """
            INSERT INTO artifactmetadata 
            (artifact_id, title, culture, century, classification, department, 
             dated, medium, dimensions, creditline, division, objectnumber)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        data = df.values.tolist()
        cursor.executemany(insert_query, data)
        self.connection.commit()
    
    def load_media(self, df):
        """Bulk insert media data"""
        cursor = self.connection.cursor()
        
        insert_query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_id, media_url, media_type, width, height)
            VALUES (%s, %s, %s, %s, %s, %s)
        """
        
        data = df.values.tolist()
        cursor.executemany(insert_query, data)
        self.connection.commit()
    
    def execute_query(self, query):
        """Execute analytical query"""
        cursor = self.connection.cursor(dictionary=True)
        cursor.execute(query)
        return cursor.fetchall()

# Usage
db = ArtifactDatabase(DB_CONFIG)
db.connect()
db.create_tables()

# Load transformed data
metadata_df = transform_artifact_metadata(artifacts)
media_df = transform_artifact_media(artifacts)
colors_df = transform_artifact_colors(artifacts)

db.load_metadata(metadata_df)
db.load_media(media_df)
```

### 4. Analytical SQL Queries

```python
# Sample analytical queries
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top Colors Used": """
        SELECT color_name, COUNT(*) as usage_count,
               AVG(color_percent) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_items,
            AVG(width) as avg_width,
            AVG(height) as avg_height
        FROM artifactmedia
    """,
    
    "Classification Analysis": """
        SELECT classification, 
               COUNT(*) as count,
               COUNT(DISTINCT culture) as cultures
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def create_dashboard():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    db = ArtifactDatabase(DB_CONFIG)
    db.connect()
    
    # Query selection
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            query = ANALYTICAL_QUERIES[query_name]
            results = db.execute_query(query)
            df = pd.DataFrame(results)
            
            # Display results
            st.subheader(f"📊 {query_name}")
            
            col1, col2 = st.columns([2, 1])
            
            with col1:
                st.dataframe(df, use_container_width=True)
            
            with col2:
                st.code(query, language="sql")
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(15),
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name,
                    labels={df.columns[0]: df.columns[0].replace('_', ' ').title()}
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL section
    st.sidebar.header("ETL Pipeline")
    
    num_records = st.sidebar.number_input("Records to fetch", 100, 5000, 500)
    
    if st.sidebar.button("Run ETL"):
        with st.spinner("Fetching data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            client = HarvardAPIClient(api_key)
            artifacts = client.fetch_all_artifacts(max_records=num_records)
            
            st.success(f"Fetched {len(artifacts)} artifacts")
            
            # Transform
            metadata_df = transform_artifact_metadata(artifacts)
            media_df = transform_artifact_media(artifacts)
            colors_df = transform_artifact_colors(artifacts)
            
            # Load
            db.load_metadata(metadata_df)
            db.load_media(media_df)
            
            st.success("ETL completed successfully!")

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_last_artifact_id(db):
    """Get the last loaded artifact ID"""
    query = "SELECT MAX(artifact_id) as max_id FROM artifactmetadata"
    result = db.execute_query(query)
    return result[0]['max_id'] if result else 0

def incremental_load(db, api_key):
    """Load only new artifacts"""
    last_id = get_last_artifact_id(db)
    client = HarvardAPIClient(api_key)
    
    # Fetch artifacts with ID > last_id
    # Implementation depends on API filtering capabilities
    pass
```

### Pattern 2: Data Quality Checks

```python
def validate_artifacts(df):
    """Validate artifact data quality"""
    issues = []
    
    # Check for missing IDs
    if df['artifact_id'].isnull().any():
        issues.append("Missing artifact IDs found")
    
    # Check for duplicates
    if df['artifact_id'].duplicated().any():
        issues.append("Duplicate artifact IDs found")
    
    # Check required fields
    required = ['title', 'artifact_id']
    for field in required:
        null_count = df[field].isnull().sum()
        if null_count > 0:
            issues.append(f"{null_count} records missing {field}")
    
    return issues
```

### Pattern 3: Batch Processing

```python
def batch_load_artifacts(db, artifacts, batch_size=100):
    """Load artifacts in batches for better performance"""
    metadata_df = transform_artifact_metadata(artifacts)
    
    for i in range(0, len(metadata_df), batch_size):
        batch = metadata_df.iloc[i:i+batch_size]
        db.load_metadata(batch)
        print(f"Loaded batch {i//batch_size + 1}")
```

## Troubleshooting

### API Rate Limiting
```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(client, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.fetch_artifacts()
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
def reconnect_database(db, max_attempts=3):
    for attempt in range(max_attempts):
        try:
            db.connect()
            return True
        except Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            time.sleep(5)
    return False
```

### Memory Management for Large Datasets
```python
def stream_large_dataset(client, db, total_records=10000, chunk_size=500):
    """Process large datasets in chunks"""
    for offset in range(0, total_records, chunk_size):
        artifacts = client.fetch_artifacts(size=chunk_size, page=offset//chunk_size + 1)
        
        # Process and load immediately
        metadata_df = transform_artifact_metadata(artifacts)
        db.load_metadata(metadata_df)
        
        # Free memory
        del artifacts, metadata_df
```

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards using museum collection APIs.
