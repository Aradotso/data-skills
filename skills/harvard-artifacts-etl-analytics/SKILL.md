---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I fetch Harvard Art Museum artifacts with Python
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and SQL
  - extract and transform Harvard API data to database
  - query and visualize art museum collections
  - set up data engineering pipeline for artifacts
  - analyze Harvard Art Museums data with SQL
  - visualize artifact metadata with Plotly
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational tables, loading into SQL databases (MySQL/TiDB), and building interactive analytics dashboards with Streamlit.

## What It Does

- **ETL Pipeline**: Extracts artifact metadata, media, and color data from Harvard API with pagination
- **Database Design**: Stores data in normalized SQL tables with foreign key relationships
- **SQL Analytics**: Runs 20+ analytical queries for insights (culture distribution, media availability, color patterns)
- **Interactive Dashboard**: Visualizes query results with Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the Streamlit app
streamlit run app.py
```

## Configuration

### API Key Setup

Get a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
# In your Streamlit app or config
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector
import os

# MySQL/TiDB connection
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    primaryimageurl TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    idsid VARCHAR(100),
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    artifacts = []
    pages = (num_records + page_size - 1) // page_size
    
    for page in range(1, pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'fields': 'id,title,culture,period,century,classification,department,division,dated,primaryimageurl,totalpageviews,totaluniquepageviews,images,colors'
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{pages}")
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Metadata
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'totalpageviews': artifact.get('totalpageviews'),
            'totaluniquepageviews': artifact.get('totaluniquepageviews')
        })
        
        # Media (images)
        images = artifact.get('images', [])
        for img in images:
            media_list.append({
                'artifact_id': artifact.get('id'),
                'idsid': img.get('idsid'),
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
            })
        
        # Colors
        colors = artifact.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert into SQL

```python
def load_to_sql(connection, df_metadata, df_media, df_colors):
    """
    Load transformed data into SQL database
    """
    cursor = connection.cursor()
    
    # Load metadata
    metadata_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, 
     division, dated, primaryimageurl, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, df_metadata.values.tolist())
    
    # Load media
    media_query = """
    INSERT INTO artifactmedia 
    (artifact_id, idsid, baseimageurl, format, height, width)
    VALUES (%s, %s, %s, %s, %s, %s)
    """
    cursor.executemany(media_query, df_media.values.tolist())
    
    # Load colors
    colors_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    cursor.executemany(colors_query, df_colors.values.tolist())
    
    connection.commit()
    print(f"Loaded {len(df_metadata)} artifacts, {len(df_media)} media, {len(df_colors)} colors")
```

## Analytics Queries

### Sample SQL Analytics

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
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
    
    "Media Availability": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' ELSE 'No Image' END as status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}
```

### Execute and Visualize

```python
import plotly.express as px
import streamlit as st

def execute_query_and_visualize(connection, query, title):
    """
    Execute SQL query and create visualization
    """
    df = pd.read_sql(query, connection)
    
    st.subheader(title)
    st.dataframe(df)
    
    if len(df.columns) >= 2:
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=title,
            labels={df.columns[0]: df.columns[0].title(), df.columns[1]: 'Count'}
        )
        st.plotly_chart(fig, use_container_width=True)

# Usage in Streamlit
st.title("Harvard Artifacts Analytics Dashboard")

query_choice = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
if st.button("Run Analysis"):
    execute_query_and_visualize(
        connection, 
        ANALYTICS_QUERIES[query_choice],
        query_choice
    )
```

## Complete Streamlit App Structure

```python
import streamlit as st
import os
import mysql.connector
import pandas as pd

st.set_page_config(page_title="Harvard Artifacts ETL", layout="wide")

# Sidebar configuration
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
    num_records = st.number_input("Records to Fetch", min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            artifacts = fetch_artifacts(api_key, num_records)
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        
        with st.spinner("Loading to database..."):
            connection = mysql.connector.connect(**db_config)
            load_to_sql(connection, df_metadata, df_media, df_colors)
            st.success("ETL Complete!")

# Main analytics dashboard
st.title("🎨 Harvard Art Museums Analytics")

tab1, tab2, tab3 = st.tabs(["Analytics", "Raw Data", "Schema"])

with tab1:
    st.header("SQL Analytics Dashboard")
    # Query execution code here

with tab2:
    st.header("Browse Raw Data")
    table = st.selectbox("Select Table", ["artifactmetadata", "artifactmedia", "artifactcolors"])
    df = pd.read_sql(f"SELECT * FROM {table} LIMIT 100", connection)
    st.dataframe(df)

with tab3:
    st.header("Database Schema")
    st.code("""
    artifactmetadata (id, title, culture, century, ...)
    artifactmedia (media_id, artifact_id, baseimageurl, ...)
    artifactcolors (color_id, artifact_id, color, percent, ...)
    """)
```

## Troubleshooting

**API Rate Limits**: Add `time.sleep()` between requests (0.5-1 second recommended)

**Large Dataset Memory Issues**: Process in smaller batches or use chunking:
```python
for chunk in pd.read_sql(query, connection, chunksize=1000):
    process(chunk)
```

**Database Connection Timeouts**: Use connection pooling or reconnect:
```python
try:
    cursor.execute(query)
except mysql.connector.errors.OperationalError:
    connection.reconnect()
    cursor = connection.cursor()
```

**NULL Values in Analytics**: Always filter in SQL:
```sql
WHERE column_name IS NOT NULL AND column_name != ''
```

**Streamlit Performance**: Cache database queries:
```python
@st.cache_data(ttl=3600)
def get_analytics_data(query):
    return pd.read_sql(query, connection)
```
