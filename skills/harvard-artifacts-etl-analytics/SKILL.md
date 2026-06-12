---
name: harvard-artifacts-etl-analytics
description: End-to-end data engineering and analytics application for Harvard Art Museums API with ETL pipelines, SQL storage, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - create analytics dashboard for Harvard Art Museums
  - set up artifact collection data pipeline
  - extract and analyze museum artifact metadata
  - visualize Harvard Art Museums collection data
  - build Streamlit app for museum analytics
  - create SQL database for artifact collections
  - process Harvard API museum data
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What It Does

This project is a complete data engineering solution that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database structures
- Loads data into MySQL/TiDB Cloud databases
- Provides 20+ predefined SQL analytics queries
- Visualizes results through interactive Streamlit dashboards with Plotly charts

The application demonstrates production-grade ETL patterns, API pagination handling, batch SQL operations, and real-time analytics visualization.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
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

Create a `.env` file or configure environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application expects three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    accession_number VARCHAR(100)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url TEXT,
    primary_image_url TEXT,
    total_images INT DEFAULT 0,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percent DECIMAL(5,2),
    color_css VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {info['totalrecords']}")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(**self.db_config)
        return self.connection
    
    def extract_metadata(self, records: List[Dict]) -> pd.DataFrame:
        """Extract and transform artifact metadata"""
        metadata = []
        for record in records:
            metadata.append({
                'id': record.get('id'),
                'title': record.get('title', ''),
                'culture': record.get('culture', ''),
                'century': record.get('century', ''),
                'classification': record.get('classification', ''),
                'department': record.get('department', ''),
                'division': record.get('division', ''),
                'technique': record.get('technique', ''),
                'period': record.get('period', ''),
                'dated': record.get('dated', ''),
                'url': record.get('url', ''),
                'accession_number': record.get('accessionnumber', '')
            })
        return pd.DataFrame(metadata)
    
    def extract_media(self, records: List[Dict]) -> pd.DataFrame:
        """Extract media information"""
        media = []
        for record in records:
            media.append({
                'artifact_id': record.get('id'),
                'base_image_url': record.get('baseimageurl', ''),
                'primary_image_url': record.get('primaryimageurl', ''),
                'total_images': record.get('totalpageviews', 0)
            })
        return pd.DataFrame(media)
    
    def extract_colors(self, records: List[Dict]) -> pd.DataFrame:
        """Extract color data from artifacts"""
        colors = []
        for record in records:
            artifact_id = record.get('id')
            color_data = record.get('colors', [])
            
            for color in color_data:
                colors.append({
                    'artifact_id': artifact_id,
                    'color_hex': color.get('hex', ''),
                    'color_percent': color.get('percent', 0),
                    'color_css': color.get('css3', '')
                })
        return pd.DataFrame(colors)
    
    def load_data(self, df: pd.DataFrame, table_name: str, batch_size=1000):
        """Batch load data into database"""
        cursor = self.connection.cursor()
        
        for i in range(0, len(df), batch_size):
            batch = df.iloc[i:i+batch_size]
            
            # Prepare insert statement
            columns = ', '.join(batch.columns)
            placeholders = ', '.join(['%s'] * len(batch.columns))
            insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
            
            # Execute batch insert
            values = [tuple(row) for row in batch.values]
            cursor.executemany(insert_query, values)
            self.connection.commit()
        
        cursor.close()
    
    def run_pipeline(self, records: List[Dict]):
        """Execute complete ETL pipeline"""
        self.connect()
        
        # Extract
        metadata_df = self.extract_metadata(records)
        media_df = self.extract_media(records)
        colors_df = self.extract_colors(records)
        
        # Load
        self.load_data(metadata_df, 'artifactmetadata')
        self.load_data(media_df, 'artifactmedia')
        self.load_data(colors_df, 'artifactcolors')
        
        print(f"Loaded {len(metadata_df)} artifacts")
        print(f"Loaded {len(media_df)} media records")
        print(f"Loaded {len(colors_df)} color records")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = ArtifactETL(db_config)
etl.run_pipeline(artifacts)
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "top_departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "media_availability": """
        SELECT 
            COUNT(*) as total_artifacts,
            SUM(CASE WHEN primary_image_url != '' THEN 1 ELSE 0 END) as with_images,
            ROUND(SUM(CASE WHEN primary_image_url != '' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as image_percentage
        FROM artifactmedia
    """,
    
    "top_colors": """
        SELECT color_hex, color_css, COUNT(*) as usage_count,
               ROUND(AVG(color_percent), 2) as avg_percent
        FROM artifactcolors
        WHERE color_hex IS NOT NULL
        GROUP BY color_hex, color_css
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "artifacts_with_technique": """
        SELECT technique, COUNT(*) as count
        FROM artifactmetadata
        WHERE technique IS NOT NULL AND technique != ''
        GROUP BY technique
        ORDER BY count DESC
        LIMIT 20
    """
}

def run_analytics_query(connection, query_name):
    """Execute analytics query and return results"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(ANALYTICS_QUERIES[query_name])
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_options = list(ANALYTICS_QUERIES.keys())
    selected_query = st.sidebar.selectbox("Choose Query", query_options)
    
    # Database connection
    connection = mysql.connector.connect(**db_config)
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = run_analytics_query(connection, selected_query)
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df) > 0:
                st.subheader("Visualization")
                
                # Determine columns for plotting
                x_col = df.columns[0]
                y_col = df.columns[1] if len(df.columns) > 1 else df.columns[0]
                
                # Create bar chart
                fig = px.bar(df, x=x_col, y=y_col, 
                            title=f"{selected_query.replace('_', ' ').title()}")
                st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

# Run the app
if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Handling API Pagination

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            records, info = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(records)
            
            total_pages = info['pages']
            if page >= total_pages:
                break
                
            # Rate limiting
            import time
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Incremental Data Loading

```python
def get_latest_artifact_id(connection):
    """Get the most recent artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) as max_id FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_load(etl, api_key):
    """Load only new artifacts since last update"""
    latest_id = get_latest_artifact_id(etl.connection)
    
    # Fetch new artifacts with filter
    new_artifacts = []
    # Implementation depends on API filtering capabilities
    
    if new_artifacts:
        etl.run_pipeline(new_artifacts)
```

## Troubleshooting

**API Rate Limiting:**
```python
# Add delay between requests
import time
time.sleep(1)  # Wait 1 second between API calls
```

**Database Connection Errors:**
```python
# Test connection
try:
    connection = mysql.connector.connect(**db_config)
    print("Connection successful")
    connection.close()
except mysql.connector.Error as e:
    print(f"Database error: {e}")
```

**Missing Data Handling:**
```python
# Safe extraction with defaults
def safe_get(record, key, default=''):
    return record.get(key, default) if record.get(key) is not None else default
```

**Streamlit Caching:**
```python
@st.cache_data(ttl=3600)
def load_cached_data(query):
    """Cache query results for 1 hour"""
    return run_analytics_query(connection, query)
```
