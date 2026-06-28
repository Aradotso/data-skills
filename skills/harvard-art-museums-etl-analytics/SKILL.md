---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - "build an ETL pipeline for Harvard Art Museums data"
  - "create analytics dashboard from Harvard museum API"
  - "extract and transform art museum data into SQL"
  - "visualize Harvard artifacts collection with Streamlit"
  - "set up data engineering pipeline for museum artifacts"
  - "query and analyze Harvard Art Museums data"
  - "build museum data warehouse with Python"
  - "create art collection analytics app"
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete end-to-end data engineering and analytics solution for the Harvard Art Museums API. It demonstrates professional ETL pipelines, SQL database design, and interactive visualization using Streamlit.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into relational database schema
- **Loads** data into MySQL/TiDB Cloud with batch optimization
- **Analyzes** data using 20+ predefined SQL analytical queries
- **Visualizes** insights through interactive Plotly charts in Streamlit

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

Create a `.env` file or configure Streamlit secrets:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
MYSQL_HOST=your_database_host
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=harvard_artifacts
```

### Streamlit Secrets

Alternative configuration via `.streamlit/secrets.toml`:

```toml
HARVARD_API_KEY = "your_api_key_here"
MYSQL_HOST = "your_database_host"
MYSQL_PORT = 3306
MYSQL_USER = "your_username"
MYSQL_PASSWORD = "your_password"
MYSQL_DATABASE = "harvard_artifacts"
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Store in environment variables (never commit to code)

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Database Schema

The ETL pipeline creates three main tables:

### ArtifactMetadata Table

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    department VARCHAR(255)
);
```

### ArtifactMedia Table

```sql
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    primaryimageurl VARCHAR(1000),
    baseimageurl VARCHAR(1000),
    imagecount INT,
    mediacount INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### ArtifactColors Table

```sql
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch from API

```python
import requests
import os

def fetch_artifacts(api_key, num_records=100):
    """Extract artifact data from Harvard API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # API max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': size
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
        else:
            raise Exception(f"API Error: {response.status_code}")
    
    return artifacts[:num_records]
```

### Transform: JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON into relational format"""
    
    # Metadata transformation
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'division': artifact.get('division', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'department': artifact.get('department', '')[:255]
        }
        metadata_list.append(metadata)
        
        # Extract media
        media = {
            'artifact_id': artifact.get('id'),
            'primaryimageurl': artifact.get('primaryimageurl', '')[:1000],
            'baseimageurl': artifact.get('baseimageurl', '')[:1000],
            'imagecount': artifact.get('imagecount', 0),
            'mediacount': artifact.get('mediacount', 0)
        }
        media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_entry)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert to SQL

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, config):
    """Load transformed data into MySQL with batch inserts"""
    
    try:
        connection = mysql.connector.connect(
            host=config['host'],
            port=config['port'],
            user=config['user'],
            password=config['password'],
            database=config['database']
        )
        
        cursor = connection.cursor()
        
        # Load metadata (batch insert)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, division, 
             dated, accessionyear, technique, medium, department)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Load media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, primaryimageurl, baseimageurl, imagecount, mediacount)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Load colors
        if not df_colors.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, percent)
                VALUES (%s, %s, %s)
            """
            cursor.executemany(colors_query, df_colors.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        raise
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Query 1: Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10;
"""

# Query 2: Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC;
"""

# Query 3: Department distribution
query_departments = """
    SELECT department, COUNT(*) as total_artifacts
    FROM artifactmetadata
    GROUP BY department
    ORDER BY total_artifacts DESC;
"""

# Query 4: Color usage analysis
query_colors = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 15;
"""

# Query 5: Media availability
query_media = """
    SELECT 
        CASE 
            WHEN imagecount > 0 THEN 'Has Images'
            ELSE 'No Images'
        END as image_status,
        COUNT(*) as count
    FROM artifactmedia
    GROUP BY image_status;
"""

# Query 6: Classification breakdown
query_classification = """
    SELECT classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE classification IS NOT NULL
    GROUP BY classification
    ORDER BY count DESC
    LIMIT 10;
"""
```

### Execute Queries in Streamlit

```python
import streamlit as st
import pandas as pd

def execute_query(query, config):
    """Execute SQL query and return DataFrame"""
    connection = mysql.connector.connect(**config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df

# In Streamlit app
st.title("Harvard Art Museums Analytics")

query_choice = st.selectbox("Select Analysis", [
    "Top Cultures",
    "Century Distribution",
    "Department Breakdown",
    "Color Analysis"
])

if st.button("Run Analysis"):
    df_result = execute_query(selected_query, db_config)
    st.dataframe(df_result)
    
    # Auto-generate visualization
    import plotly.express as px
    fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
    st.plotly_chart(fig)
```

## Visualization Patterns

### Interactive Dashboard with Plotly

```python
import plotly.express as px
import plotly.graph_objects as go

def create_culture_chart(df):
    """Create interactive bar chart for culture distribution"""
    fig = px.bar(
        df, 
        x='culture', 
        y='artifact_count',
        title='Top Cultures by Artifact Count',
        labels={'culture': 'Culture', 'artifact_count': 'Number of Artifacts'},
        color='artifact_count',
        color_continuous_scale='Viridis'
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_century_timeline(df):
    """Create timeline visualization"""
    fig = px.line(
        df,
        x='century',
        y='count',
        title='Artifacts by Century',
        markers=True
    )
    return fig

def create_color_distribution(df):
    """Create pie chart for color usage"""
    fig = px.pie(
        df,
        values='usage_count',
        names='color',
        title='Color Distribution in Artifacts'
    )
    return fig
```

## Common Patterns

### Full ETL Workflow

```python
def run_etl_pipeline(api_key, db_config, num_records=500):
    """Complete ETL workflow"""
    
    # Extract
    st.info("Extracting data from API...")
    artifacts = fetch_artifacts(api_key, num_records)
    
    # Transform
    st.info("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(artifacts)
    
    # Load
    st.info("Loading to database...")
    load_to_database(df_metadata, df_media, df_colors, db_config)
    
    st.success(f"ETL complete! Processed {num_records} artifacts")
    
    return df_metadata, df_media, df_colors
```

### Streamlit App Structure

```python
import streamlit as st
import os

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    # Sidebar configuration
    st.sidebar.title("Configuration")
    api_key = os.getenv('HARVARD_API_KEY') or st.sidebar.text_input("API Key", type="password")
    
    # Database config from environment
    db_config = {
        'host': os.getenv('MYSQL_HOST'),
        'port': int(os.getenv('MYSQL_PORT', 3306)),
        'user': os.getenv('MYSQL_USER'),
        'password': os.getenv('MYSQL_PASSWORD'),
        'database': os.getenv('MYSQL_DATABASE')
    }
    
    # Main tabs
    tab1, tab2 = st.tabs(["ETL Pipeline", "Analytics Dashboard"])
    
    with tab1:
        st.header("Data Collection & ETL")
        num_records = st.slider("Number of records", 100, 1000, 500)
        
        if st.button("Run ETL Pipeline"):
            run_etl_pipeline(api_key, db_config, num_records)
    
    with tab2:
        st.header("SQL Analytics")
        # Query selection and execution
        # Visualization display

if __name__ == "__main__":
    main()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        
        if response.status_code == 200:
            return response
        elif response.status_code == 429:  # Rate limit
            wait_time = 2 ** attempt
            time.sleep(wait_time)
        else:
            raise Exception(f"API Error: {response.status_code}")
    
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_database_connection(config):
    """Verify database connectivity"""
    try:
        connection = mysql.connector.connect(**config)
        if connection.is_connected():
            st.success("Database connected successfully")
            connection.close()
            return True
    except Error as e:
        st.error(f"Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def clean_artifact_data(artifact):
    """Handle null/missing values in API response"""
    return {
        'id': artifact.get('id', 0),
        'title': artifact.get('title', 'Unknown')[:500] or 'Unknown',
        'culture': artifact.get('culture', 'Unknown')[:255] or 'Unknown',
        'century': artifact.get('century', 'Unknown')[:100] or 'Unknown',
        'accessionyear': artifact.get('accessionyear') or None
    }
```

### Memory Optimization for Large Datasets

```python
def process_in_batches(artifacts, batch_size=100):
    """Process large datasets in batches"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        df_meta, df_media, df_colors = transform_artifacts(batch)
        load_to_database(df_meta, df_media, df_colors, db_config)
        st.progress((i + batch_size) / len(artifacts))
```

This skill enables AI agents to build complete data engineering pipelines using the Harvard Art Museums API with production-ready ETL processes, SQL analytics, and interactive visualizations.
