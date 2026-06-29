---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard Art Museums API data
  - set up data engineering pipeline with Streamlit
  - query and visualize museum artifact collections
  - implement SQL analytics for art museum data
  - build Harvard artifacts data pipeline
  - create interactive museum data visualization
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases, and visualizing insights through interactive Streamlit dashboards.

## What It Does

- **API Integration**: Fetches artifact metadata, media, and color information from Harvard Art Museums API
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB with proper schema design
- **Analytics**: Executes 20+ predefined analytical queries
- **Visualization**: Generates interactive charts using Plotly in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### Environment Variables

Set up your credentials using environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### Database Setup

The project uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url TEXT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Collection

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    url = f"https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts_data = fetch_artifacts(api_key, page=1, size=50)
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector

def extract_metadata(artifacts):
    """Extract metadata from API response"""
    metadata_list = []
    
    for artifact in artifacts.get('records', []):
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'url': artifact.get('url', '')
        })
    
    return pd.DataFrame(metadata_list)

def extract_media(artifacts):
    """Extract media information"""
    media_list = []
    
    for artifact in artifacts.get('records', []):
        artifact_id = artifact.get('id')
        for media in artifact.get('images', []):
            media_list.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'baseimageurl': media.get('baseimageurl', '')
            })
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """Extract color information"""
    color_list = []
    
    for artifact in artifacts.get('records', []):
        artifact_id = artifact.get('id')
        for color in artifact.get('colors', []):
            color_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color', 'Unknown'),
                'percentage': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_list)
```

### 3. Loading to SQL

```python
def load_to_sql(df, table_name, connection_config):
    """Load DataFrame to SQL database"""
    conn = mysql.connector.connect(**connection_config)
    cursor = conn.cursor()
    
    # Prepare batch insert
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    sql = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Execute batch insert
    data = [tuple(row) for row in df.values]
    cursor.executemany(sql, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    return len(df)

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

metadata_df = extract_metadata(artifacts_data)
load_to_sql(metadata_df, 'artifactmetadata', db_config)
```

### 4. SQL Analytics Queries

```python
def execute_query(query, connection_config):
    """Execute SQL query and return DataFrame"""
    conn = mysql.connector.connect(**connection_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Example analytical queries
queries = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Images": """
        SELECT 
            m.department,
            COUNT(DISTINCT m.id) as total_artifacts,
            COUNT(DISTINCT media.artifact_id) as with_images,
            ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as image_percentage
        FROM artifactmetadata m
        LEFT JOIN artifactmedia media ON m.id = media.artifact_id
        GROUP BY m.department
        ORDER BY total_artifacts DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """
}
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_name = st.sidebar.selectbox("Choose Query", list(queries.keys()))
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Fetching data..."):
            df = execute_query(queries[query_name], db_config)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Full ETL Workflow

```python
def run_etl_pipeline(api_key, db_config, num_pages=5):
    """Complete ETL pipeline"""
    all_metadata = []
    all_media = []
    all_colors = []
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        artifacts = fetch_artifacts(api_key, page=page)
        
        # Transform
        metadata = extract_metadata(artifacts)
        media = extract_media(artifacts)
        colors = extract_colors(artifacts)
        
        all_metadata.append(metadata)
        all_media.append(media)
        all_colors.append(colors)
    
    # Combine all pages
    metadata_df = pd.concat(all_metadata, ignore_index=True)
    media_df = pd.concat(all_media, ignore_index=True)
    colors_df = pd.concat(all_colors, ignore_index=True)
    
    # Load
    load_to_sql(metadata_df, 'artifactmetadata', db_config)
    load_to_sql(media_df, 'artifactmedia', db_config)
    load_to_sql(colors_df, 'artifactcolors', db_config)
    
    print(f"✅ Loaded {len(metadata_df)} artifacts")
```

### Error Handling and Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = fetch_artifacts(api_key, page)
            time.sleep(1)  # Rate limiting
            return response
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s...")
            time.sleep(wait_time)
```

## Troubleshooting

### API Rate Limits
If you encounter rate limiting errors:
```python
# Add delay between requests
import time
time.sleep(2)  # Wait 2 seconds between API calls
```

### Database Connection Issues
```python
# Test database connection
try:
    conn = mysql.connector.connect(**db_config)
    print("✅ Database connected successfully")
    conn.close()
except mysql.connector.Error as err:
    print(f"❌ Database error: {err}")
```

### Memory Issues with Large Datasets
```python
# Process in smaller batches
def batch_process(api_key, db_config, batch_size=10):
    for i in range(0, 100, batch_size):
        # Process and load batch
        # Clear memory
        import gc
        gc.collect()
```

### Missing Data Fields
```python
# Use safe extraction with defaults
def safe_get(dictionary, key, default='Unknown'):
    return dictionary.get(key, default) or default
```

## Advanced Usage

### Custom Analytics Query

```python
# Add custom query to dashboard
custom_query = st.text_area("Enter Custom SQL Query")
if st.button("Execute Custom Query"):
    df = execute_query(custom_query, db_config)
    st.dataframe(df)
```

### Export Results

```python
# Export query results to CSV
df = execute_query(queries[query_name], db_config)
csv = df.to_csv(index=False)
st.download_button("Download CSV", csv, "results.csv", "text/csv")
```

This skill enables AI agents to help developers build complete data engineering pipelines using the Harvard Art Museums API, from data collection through analytics and visualization.
