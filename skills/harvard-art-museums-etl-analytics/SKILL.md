---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API data with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and museum data
  - query Harvard Art Museums database with SQL
  - visualize art collection data with Plotly
  - set up data engineering project with Harvard API
  - extract and transform museum artifacts into SQL
  - analyze art museum collection data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: Run predefined analytical queries on structured museum data
- **Interactive Dashboards**: Visualize insights using Streamlit and Plotly

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from [https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dAd8CVqW1bvTQ/viewform](https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dAd8CVqW1bvTQ/viewform)

### Environment Variables

Create a `.env` file:
```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the database and tables:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accession_number VARCHAR(255),
    division VARCHAR(255)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    base_image_url VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The app will open in your browser at `http://localhost:8501`

## API Integration Patterns

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'

def fetch_artifacts(total_records=100, page_size=100):
    """Fetch artifacts from Harvard API with pagination"""
    artifacts = []
    pages_needed = (total_records + page_size - 1) // page_size
    
    for page in range(1, pages_needed + 1):
        params = {
            'apikey': API_KEY,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{pages_needed}")
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:total_records]
```

### Handling Rate Limits

```python
import time

def fetch_with_rate_limit(url, params, delay=0.5):
    """Fetch data with rate limiting"""
    response = requests.get(url, params=params)
    time.sleep(delay)  # Respect API rate limits
    return response
```

## ETL Pipeline Patterns

### Extract and Transform

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into structured DataFrames"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'accession_number': artifact.get('accessionyear', 'Unknown'),
            'division': artifact.get('division', 'Unknown')
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl', ''),
                'base_image_url': image.get('iiifbaseuri', '')
            }
            media_list.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', 'Unknown'),
                'spectrum': color.get('spectrum', 'Unknown'),
                'hue': color.get('hue', 'Unknown'),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load DataFrames to SQL database with batch inserts"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, dated, accession_number, division)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, base_image_url)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Top 10 cultures by artifact count
query_1 = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Artifacts by century
query_2 = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Color distribution analysis
query_3 = """
SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY usage_count DESC
LIMIT 15;
"""

# Department-wise classification
query_4 = """
SELECT department, classification, COUNT(*) as count
FROM artifactmetadata
WHERE department != 'Unknown' AND classification != 'Unknown'
GROUP BY department, classification
ORDER BY department, count DESC;
"""

# Artifacts with most images
query_5 = """
SELECT am.id, am.title, am.culture, COUNT(ame.id) as image_count
FROM artifactmetadata am
JOIN artifactmedia ame ON am.id = ame.artifact_id
GROUP BY am.id, am.title, am.culture
ORDER BY image_count DESC
LIMIT 10;
"""
```

### Execute Query Function

```python
def execute_query(query):
    """Execute SQL query and return DataFrame"""
    try:
        conn = get_db_connection()
        df = pd.read_sql(query, conn)
        conn.close()
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
```

## Streamlit Dashboard Patterns

### Basic Dashboard Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics")
    st.sidebar.title("Navigation")
    
    page = st.sidebar.radio(
        "Choose a page:",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    st.header("📥 Collect Artifacts")
    
    num_records = st.number_input(
        "Number of artifacts to fetch:",
        min_value=10,
        max_value=1000,
        value=100
    )
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(num_records)
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            load_to_database(metadata_df, media_df, colors_df)
            st.success(f"✅ Loaded {len(metadata_df)} artifacts")

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top 10 Cultures": query_1,
        "Artifacts by Century": query_2,
        "Color Distribution": query_3,
        "Department Classification": query_4,
        "Most Images": query_5
    }
    
    selected_query = st.selectbox("Select Query:", list(queries.keys()))
    
    if st.button("Execute Query"):
        df = execute_query(queries[selected_query])
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df) > 0 and len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Visualization Patterns

### Plotly Charts

```python
import plotly.express as px
import plotly.graph_objects as go

# Bar chart
def create_bar_chart(df, x_col, y_col, title):
    fig = px.bar(
        df,
        x=x_col,
        y=y_col,
        title=title,
        labels={x_col: x_col.replace('_', ' ').title()},
        color=y_col,
        color_continuous_scale='viridis'
    )
    return fig

# Pie chart for color distribution
def create_color_pie(df):
    fig = px.pie(
        df,
        names='color',
        values='usage_count',
        title='Color Distribution in Collection'
    )
    return fig

# Grouped bar chart
def create_grouped_bar(df):
    fig = px.bar(
        df,
        x='department',
        y='count',
        color='classification',
        title='Artifacts by Department and Classification',
        barmode='group'
    )
    return fig
```

## Troubleshooting

### API Issues

**Problem**: `401 Unauthorized` error
```python
# Verify API key is loaded correctly
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
print(f"API Key loaded: {bool(api_key)}")
```

**Problem**: Rate limiting errors
```python
# Add exponential backoff
import time

def fetch_with_backoff(url, params, max_retries=3):
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 200:
            return response
        elif response.status_code == 429:
            wait_time = 2 ** attempt
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
    return None
```

### Database Issues

**Problem**: Connection timeout
```python
# Increase timeout
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    connection_timeout=30
)
```

**Problem**: Duplicate key errors
```python
# Use INSERT IGNORE or ON DUPLICATE KEY UPDATE
query = """
INSERT INTO artifactmetadata (id, title, ...) 
VALUES (%s, %s, ...)
ON DUPLICATE KEY UPDATE title=VALUES(title)
"""
```

### Streamlit Issues

**Problem**: Session state not persisting
```python
# Initialize session state
if 'data_loaded' not in st.session_state:
    st.session_state.data_loaded = False

if st.button("Load Data"):
    # ... load data
    st.session_state.data_loaded = True
```

**Problem**: Large DataFrame display issues
```python
# Use pagination
st.dataframe(df.head(100))  # Show first 100 rows
st.download_button("Download Full Data", df.to_csv())
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, DB credentials)
2. **Implement proper error handling** in ETL pipelines
3. **Use batch inserts** for database operations (improves performance by 10-100x)
4. **Cache Streamlit data** with `@st.cache_data` for expensive operations
5. **Validate data** before loading to database
6. **Log ETL operations** for debugging and monitoring
