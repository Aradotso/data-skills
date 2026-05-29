---
name: harvard-art-museum-etl-analytics
description: ETL pipeline and analytics application for Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - analyze Harvard museum artifacts with SQL
  - create a data engineering pipeline for art museum collections
  - visualize Harvard Art Museums API data
  - set up artifact metadata analytics dashboard
  - extract and transform museum collection data
  - build a Streamlit app for museum data analytics
  - query Harvard art collection database
---

# Harvard Art Museum ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering solution that:

- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database schemas
- Loads data into MySQL/TiDB Cloud databases
- Provides 20+ pre-built SQL analytics queries
- Visualizes results through an interactive Streamlit dashboard

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Getting Harvard Art Museums API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to your `.env` file

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The application will launch at `http://localhost:8501`

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Your Harvard API key
        size: Number of records per page (max 100)
        page: Page number for pagination
    
    Returns:
        JSON response with artifact data
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=50, page=1)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardETL:
    """ETL Pipeline for Harvard Art Museums data"""
    
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(**self.db_config)
        return self.connection
    
    def extract_metadata(self, records: List[Dict]) -> pd.DataFrame:
        """Extract and transform artifact metadata"""
        metadata = []
        
        for record in records:
            metadata.append({
                'id': record.get('id'),
                'title': record.get('title', '')[:500],
                'culture': record.get('culture', '')[:200],
                'century': record.get('century', '')[:100],
                'classification': record.get('classification', '')[:200],
                'department': record.get('department', '')[:200],
                'division': record.get('division', '')[:200],
                'dated': record.get('dated', '')[:200],
                'period': record.get('period', '')[:200],
                'technique': record.get('technique', '')[:500]
            })
        
        return pd.DataFrame(metadata)
    
    def extract_media(self, records: List[Dict]) -> pd.DataFrame:
        """Extract media/image information"""
        media_data = []
        
        for record in records:
            media_data.append({
                'artifact_id': record.get('id'),
                'baseimageurl': record.get('baseimageurl', ''),
                'primaryimageurl': record.get('primaryimageurl', ''),
                'iiifbaseuri': record.get('iiifbaseuri', '')
            })
        
        return pd.DataFrame(media_data)
    
    def extract_colors(self, records: List[Dict]) -> pd.DataFrame:
        """Extract color information from artifacts"""
        color_data = []
        
        for record in records:
            if 'colors' in record and record['colors']:
                for color in record['colors']:
                    color_data.append({
                        'artifact_id': record.get('id'),
                        'color': color.get('color', ''),
                        'spectrum': color.get('spectrum', ''),
                        'percent': color.get('percent', 0.0)
                    })
        
        return pd.DataFrame(color_data)
    
    def load_to_db(self, df: pd.DataFrame, table_name: str):
        """Load dataframe to database table"""
        cursor = self.connection.cursor()
        
        # Get column names
        cols = ','.join(df.columns)
        placeholders = ','.join(['%s'] * len(df.columns))
        
        # Prepare insert query
        query = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
        query += " ON DUPLICATE KEY UPDATE " + ','.join([f"{col}=VALUES({col})" for col in df.columns if col != 'id'])
        
        # Batch insert
        data = [tuple(row) for row in df.values]
        cursor.executemany(query, data)
        self.connection.commit()
        
        print(f"Loaded {len(df)} records into {table_name}")

# Usage Example
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = HarvardETL(db_config)
etl.connect_db()

# Fetch data
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=100)

# ETL Process
metadata_df = etl.extract_metadata(data['records'])
media_df = etl.extract_media(data['records'])
colors_df = etl.extract_colors(data['records'])

# Load to database
etl.load_to_db(metadata_df, 'artifactmetadata')
etl.load_to_db(media_df, 'artifactmedia')
etl.load_to_db(colors_df, 'artifactcolors')
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries included in the project

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "top_departments": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
        LIMIT 10
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE 
                WHEN primaryimageurl IS NOT NULL THEN 'Has Primary Image'
                ELSE 'No Primary Image'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "artifacts_by_classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_analytics_query(connection, query_name):
    """Execute a predefined analytics query"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(ANALYTICS_QUERIES[query_name])
    results = cursor.fetchall()
    return pd.DataFrame(results)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Main Streamlit dashboard"""
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", value=os.getenv('HARVARD_API_KEY'))
    
    # Data collection section
    st.header("1. Data Collection")
    col1, col2 = st.columns(2)
    
    with col1:
        num_records = st.number_input("Number of records to fetch", min_value=1, max_value=100, value=50)
    
    with col2:
        page_num = st.number_input("Page number", min_value=1, value=1)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            data = fetch_artifacts(api_key, size=num_records, page=page_num)
            
            # ETL process
            etl = HarvardETL(db_config)
            etl.connect_db()
            
            metadata_df = etl.extract_metadata(data['records'])
            media_df = etl.extract_media(data['records'])
            colors_df = etl.extract_colors(data['records'])
            
            etl.load_to_db(metadata_df, 'artifactmetadata')
            etl.load_to_db(media_df, 'artifactmedia')
            etl.load_to_db(colors_df, 'artifactcolors')
            
            st.success(f"Successfully loaded {len(metadata_df)} artifacts!")
    
    # Analytics section
    st.header("2. SQL Analytics")
    
    query_choice = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Analysis"):
        connection = mysql.connector.connect(**db_config)
        results_df = execute_analytics_query(connection, query_choice)
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(results_df)
        
        # Visualization
        if len(results_df) > 0:
            st.subheader("Visualization")
            
            # Auto-detect columns for plotting
            x_col = results_df.columns[0]
            y_col = results_df.columns[1]
            
            fig = px.bar(
                results_df,
                x=x_col,
                y=y_col,
                title=f"{query_choice.replace('_', ' ').title()}"
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Pagination for Large Datasets

```python
def fetch_all_artifacts(api_key, total_records=500):
    """Fetch multiple pages of artifacts"""
    all_records = []
    page = 1
    page_size = 100
    
    while len(all_records) < total_records:
        data = fetch_artifacts(api_key, size=page_size, page=page)
        all_records.extend(data['records'])
        
        if len(data['records']) < page_size:
            break  # No more data
        
        page += 1
        time.sleep(0.5)  # Rate limiting
    
    return all_records[:total_records]
```

### Incremental Data Loading

```python
def get_max_artifact_id(connection):
    """Get the latest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def load_new_artifacts_only(api_key, db_config):
    """Load only new artifacts not in database"""
    connection = mysql.connector.connect(**db_config)
    max_id = get_max_artifact_id(connection)
    
    # Fetch recent artifacts
    data = fetch_artifacts(api_key, size=100, page=1)
    new_records = [r for r in data['records'] if r['id'] > max_id]
    
    if new_records:
        etl = HarvardETL(db_config)
        etl.connection = connection
        
        metadata_df = etl.extract_metadata(new_records)
        etl.load_to_db(metadata_df, 'artifactmetadata')
        
        print(f"Loaded {len(new_records)} new artifacts")
    else:
        print("No new artifacts found")
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, size=100, page=1, max_retries=3):
    """Fetch with automatic retry on rate limit"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, size, page)
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def create_connection_with_retry(db_config, max_attempts=3):
    """Create database connection with retry logic"""
    for attempt in range(max_attempts):
        try:
            connection = mysql.connector.connect(**db_config)
            print("Database connected successfully")
            return connection
        except mysql.connector.Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            if attempt < max_attempts - 1:
                time.sleep(2)
            else:
                raise
```

### Handling Missing Data

```python
def safe_extract_metadata(records):
    """Extract metadata with robust null handling"""
    metadata = []
    
    for record in records:
        try:
            metadata.append({
                'id': record.get('id'),
                'title': (record.get('title') or 'Untitled')[:500],
                'culture': (record.get('culture') or 'Unknown')[:200],
                'century': (record.get('century') or 'Unknown')[:100],
                'classification': (record.get('classification') or '')[:200],
                'department': (record.get('department') or '')[:200]
            })
        except Exception as e:
            print(f"Error processing record {record.get('id')}: {e}")
            continue
    
    return pd.DataFrame(metadata)
```

### Streamlit Caching

```python
@st.cache_data(ttl=3600)  # Cache for 1 hour
def fetch_artifacts_cached(api_key, size, page):
    """Cached version of API fetch"""
    return fetch_artifacts(api_key, size, page)

@st.cache_resource
def get_db_connection():
    """Singleton database connection"""
    return mysql.connector.connect(**db_config)
```

## Performance Optimization

```python
# Use batch inserts for better performance
def batch_insert(connection, table_name, df, batch_size=1000):
    """Insert data in batches"""
    cursor = connection.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # ... insert logic
        connection.commit()
        print(f"Inserted batch {i//batch_size + 1}")

# Create indexes for faster queries
"""
CREATE INDEX idx_culture ON artifactmetadata(culture);
CREATE INDEX idx_century ON artifactmetadata(century);
CREATE INDEX idx_department ON artifactmetadata(department);
CREATE INDEX idx_artifact_id ON artifactcolors(artifact_id);
"""
```
