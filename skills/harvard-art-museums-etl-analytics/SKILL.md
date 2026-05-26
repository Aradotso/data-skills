---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - show me how to query and visualize Harvard museum artifacts
  - help me set up a data engineering project with museum data
  - create an analytics dashboard for art museum collections
  - extract and transform Harvard Art Museums API data
  - build a Streamlit app with museum artifact data
  - set up SQL database for Harvard art collection analytics
  - process and visualize Harvard museum API responses
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App demonstrates a complete ETL (Extract, Transform, Load) pipeline that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON responses into normalized relational tables
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** data using predefined SQL queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

This is a real-world example of data engineering workflows used in analytics roles, handling API integration, data normalization, database design, and business intelligence visualization.

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
touch .env
```

### Configuration

Create a `.env` file with your credentials:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

**Get API Key:** Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)

## Key Components

### 1. API Data Collection

```python
import requests
import pandas as pd
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPICollector:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = 'https://api.harvardartmuseums.org/object'
    
    def fetch_artifacts(self, num_pages=5, page_size=100):
        """Fetch artifacts with pagination"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            params = {
                'apikey': self.api_key,
                'size': page_size,
                'page': page,
                'hasimage': 1  # Only artifacts with images
            }
            
            response = requests.get(self.base_url, params=params)
            
            if response.status_code == 200:
                data = response.json()
                all_artifacts.extend(data.get('records', []))
                print(f"Fetched page {page}/{num_pages}")
            else:
                print(f"Error: {response.status_code}")
                break
        
        return all_artifacts

# Usage
collector = HarvardAPICollector()
artifacts = collector.fetch_artifacts(num_pages=3)
print(f"Collected {len(artifacts)} artifacts")
```

### 2. ETL Transformation

```python
def transform_artifacts(raw_artifacts):
    """Transform nested JSON into relational tables"""
    
    # Artifact Metadata
    metadata = []
    media = []
    colors = []
    
    for artifact in raw_artifacts:
        artifact_id = artifact.get('id')
        
        # Main metadata table
        metadata.append({
            'artifact_id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance')
        })
        
        # Media table (images)
        for img in artifact.get('images', []):
            media.append({
                'artifact_id': artifact_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format')
            })
        
        # Colors table
        for color in artifact.get('colors', []):
            colors.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return {
        'metadata': pd.DataFrame(metadata),
        'media': pd.DataFrame(media),
        'colors': pd.DataFrame(colors)
    }

# Usage
transformed = transform_artifacts(artifacts)
print(f"Metadata rows: {len(transformed['metadata'])}")
print(f"Media rows: {len(transformed['media'])}")
print(f"Colors rows: {len(transformed['colors'])}")
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

class DatabaseLoader:
    def __init__(self):
        self.connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        self.cursor = self.connection.cursor()
    
    def create_tables(self):
        """Create database schema"""
        
        # Metadata table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                artifact_id INT PRIMARY KEY,
                title VARCHAR(500),
                culture VARCHAR(255),
                classification VARCHAR(255),
                century VARCHAR(100),
                dated VARCHAR(255),
                department VARCHAR(255),
                technique VARCHAR(500),
                medium VARCHAR(500),
                dimensions VARCHAR(500),
                creditline TEXT,
                description TEXT,
                provenance TEXT
            )
        """)
        
        # Media table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                image_id INT,
                base_url VARCHAR(500),
                width INT,
                height INT,
                format VARCHAR(50),
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        # Colors table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                color_hex VARCHAR(10),
                color_name VARCHAR(100),
                percentage FLOAT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        self.connection.commit()
    
    def load_dataframe(self, df, table_name):
        """Batch insert DataFrame into table"""
        
        if df.empty:
            return
        
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        values = [tuple(row) for row in df.values]
        self.cursor.executemany(query, values)
        self.connection.commit()
        print(f"Loaded {len(df)} rows into {table_name}")
    
    def close(self):
        self.cursor.close()
        self.connection.close()

# Usage
loader = DatabaseLoader()
loader.create_tables()
loader.load_dataframe(transformed['metadata'], 'artifactmetadata')
loader.load_dataframe(transformed['media'], 'artifactmedia')
loader.load_dataframe(transformed['colors'], 'artifactcolors')
loader.close()
```

### 4. SQL Analytics Queries

```python
def run_analytics_query(query_name):
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
        
        'top_colors': """
            SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percent
            FROM artifactcolors
            WHERE color_name IS NOT NULL
            GROUP BY color_name
            ORDER BY frequency DESC
            LIMIT 15
        """,
        
        'media_statistics': """
            SELECT 
                COUNT(DISTINCT artifact_id) as artifacts_with_images,
                COUNT(*) as total_images,
                AVG(width) as avg_width,
                AVG(height) as avg_height
            FROM artifactmedia
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        
        'artifacts_with_provenance': """
            SELECT 
                COUNT(CASE WHEN provenance IS NOT NULL THEN 1 END) as with_provenance,
                COUNT(CASE WHEN provenance IS NULL THEN 1 END) as without_provenance
            FROM artifactmetadata
        """
    }
    
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(queries[query_name], connection)
    connection.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar navigation
page = st.sidebar.selectbox(
    "Select Module",
    ["Data Collection", "SQL Analytics", "Visualizations"]
)

if page == "Data Collection":
    st.header("📥 Collect Artifact Data")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=20, value=5)
    
    if st.button("Start Collection"):
        with st.spinner("Fetching data from API..."):
            collector = HarvardAPICollector()
            artifacts = collector.fetch_artifacts(num_pages=num_pages)
            
            st.success(f"Collected {len(artifacts)} artifacts")
            
            # Transform
            transformed = transform_artifacts(artifacts)
            st.write("### Transformed Data Preview")
            st.dataframe(transformed['metadata'].head())
            
            # Load to database
            if st.button("Load to Database"):
                loader = DatabaseLoader()
                loader.create_tables()
                loader.load_dataframe(transformed['metadata'], 'artifactmetadata')
                loader.load_dataframe(transformed['media'], 'artifactmedia')
                loader.load_dataframe(transformed['colors'], 'artifactcolors')
                loader.close()
                st.success("Data loaded successfully!")

elif page == "SQL Analytics":
    st.header("📊 SQL Query Analytics")
    
    query_options = [
        "Artifacts by Culture",
        "Artifacts by Century",
        "Top Colors",
        "Media Statistics",
        "Department Distribution"
    ]
    
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Query"):
        query_map = {
            "Artifacts by Culture": "artifacts_by_culture",
            "Artifacts by Century": "artifacts_by_century",
            "Top Colors": "top_colors",
            "Media Statistics": "media_statistics",
            "Department Distribution": "department_distribution"
        }
        
        result_df = run_analytics_query(query_map[selected_query])
        st.dataframe(result_df)
        
        # Auto-generate chart
        if len(result_df.columns) >= 2 and len(result_df) > 1:
            fig = px.bar(result_df, x=result_df.columns[0], y=result_df.columns[1])
            st.plotly_chart(fig, use_container_width=True)

elif page == "Visualizations":
    st.header("📈 Interactive Visualizations")
    
    # Example: Color distribution
    colors_df = run_analytics_query("top_colors")
    fig = px.bar(colors_df, x='color_name', y='frequency', 
                 title='Top Colors in Harvard Art Collection')
    st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(url, params, delay=1):
    """Add delay between API calls"""
    response = requests.get(url, params=params)
    time.sleep(delay)
    return response
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default=None):
    """Safely extract values from nested dictionaries"""
    try:
        return dictionary.get(key, default)
    except AttributeError:
        return default
```

### Upsert Operations

```python
def upsert_artifact(cursor, artifact_data):
    """Insert or update artifact data"""
    query = """
        INSERT INTO artifactmetadata (artifact_id, title, culture, ...)
        VALUES (%s, %s, %s, ...)
        ON DUPLICATE KEY UPDATE
        title = VALUES(title),
        culture = VALUES(culture)
    """
    cursor.execute(query, artifact_data)
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')

if not api_key:
    raise ValueError("HARVARD_API_KEY not found in .env file")
```

### Database Connection Errors

```python
# Test connection
try:
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    print("Connection successful")
    connection.close()
except Error as e:
    print(f"Connection failed: {e}")
```

### Empty API Responses

```python
# Check API response status
response = requests.get(url, params=params)
print(f"Status: {response.status_code}")
print(f"Response: {response.json()}")

# Verify 'records' key exists
data = response.json()
if 'records' not in data:
    print("No records in response")
```

### Memory Issues with Large Datasets

```python
# Process in chunks
def batch_process(artifacts, batch_size=1000):
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        transformed = transform_artifacts(batch)
        loader.load_dataframe(transformed['metadata'], 'artifactmetadata')
```

## Advanced Usage

### Custom SQL Queries

```python
def custom_query(sql):
    """Execute custom SQL query"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    df = pd.read_sql(sql, connection)
    connection.close()
    return df

# Example
result = custom_query("""
    SELECT m.title, c.color_name, c.percentage
    FROM artifactmetadata m
    JOIN artifactcolors c ON m.artifact_id = c.artifact_id
    WHERE c.percentage > 50
    ORDER BY c.percentage DESC
""")
```

### Export Results

```python
# Export to CSV
result_df.to_csv('analysis_results.csv', index=False)

# Export to Excel
result_df.to_excel('analysis_results.xlsx', index=False)
```

This skill provides comprehensive knowledge for building production-ready ETL pipelines with museum data, including API integration, data transformation, SQL analytics, and interactive visualization.
