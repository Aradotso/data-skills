---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with museum artifact data
  - extract and analyze Harvard Art Museums API data
  - set up data engineering pipeline with Streamlit
  - visualize museum collection data with SQL analytics
  - implement artifact data collection and transformation
  - query Harvard museum artifacts with Python
  - design relational database for art museum collections
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates end-to-end data engineering using the Harvard Art Museums API. It implements ETL pipelines to extract artifact data, transform it into relational structures, load into SQL databases, and visualize analytics through an interactive Streamlit dashboard.

## What It Does

- **Data Collection**: Fetches artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB with proper schema design (artifacts, media, colors)
- **Analytics Queries**: Runs 20+ predefined analytical SQL queries
- **Visualization**: Generates interactive Plotly charts from query results
- **Streamlit UI**: Provides web interface for data collection, querying, and visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file or use Streamlit secrets:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Streamlit Secrets

Create `.streamlit/secrets.toml`:

```toml
[api]
harvard_api_key = "your_api_key_here"

[database]
host = "your_db_host"
port = 3306
user = "your_db_user"
password = "your_db_password"
database = "harvard_artifacts"
```

Get your API key from: https://www.harvardartmuseums.org/collections/api

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(255),
    provenance TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
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

## Key Usage Patterns

### Running the Streamlit App

```bash
streamlit run app.py
```

The app typically has these main sections:
- Data Collection (fetch artifacts from API)
- ETL Execution (transform and load to database)
- Analytics Dashboard (run SQL queries)
- Visualization (display charts)

### ETL Pipeline Code Pattern

```python
import requests
import pandas as pd
import mysql.connector
from mysql.connector import Error

# 1. EXTRACT: Fetch data from Harvard API
def fetch_artifacts(api_key, page=1, size=100):
    """Extract artifact data from Harvard Art Museums API"""
    url = "https://api.harvardartmuseums.org/object"
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

# 2. TRANSFORM: Process nested JSON into flat structures
def transform_artifacts(raw_data):
    """Transform API response into structured dataframes"""
    artifacts = []
    media_list = []
    colors_list = []
    
    for record in raw_data.get('records', []):
        # Metadata
        artifact = {
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'dated': record.get('dated'),
            'medium': record.get('medium'),
            'technique': record.get('technique'),
            'period': record.get('period'),
            'provenance': record.get('provenance')
        }
        artifacts.append(artifact)
        
        # Media (images)
        artifact_id = record.get('id')
        for image in record.get('images', []):
            media_list.append({
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl'),
                'media_type': 'image'
            })
        
        # Colors
        for color in record.get('colors', []):
            colors_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return (
        pd.DataFrame(artifacts),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

# 3. LOAD: Insert into SQL database
def load_to_database(df_artifacts, df_media, df_colors, db_config):
    """Load transformed data into MySQL database"""
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        
        # Insert artifacts (batch)
        artifact_sql = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             dated, medium, technique, period, provenance)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(artifact_sql, df_artifacts.values.tolist())
        
        # Insert media
        media_sql = """
            INSERT INTO artifactmedia (artifact_id, image_url, media_type)
            VALUES (%s, %s, %s)
        """
        cursor.executemany(media_sql, df_media.values.tolist())
        
        # Insert colors
        color_sql = """
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
        """
        cursor.executemany(color_sql, df_colors.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(df_artifacts)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if conn.is_connected():
            cursor.close()
            conn.close()

# Complete ETL execution
def run_etl_pipeline(api_key, db_config, pages=5):
    """Execute complete ETL pipeline"""
    all_artifacts = []
    all_media = []
    all_colors = []
    
    for page in range(1, pages + 1):
        print(f"Processing page {page}...")
        raw_data = fetch_artifacts(api_key, page=page, size=100)
        
        df_art, df_med, df_col = transform_artifacts(raw_data)
        all_artifacts.append(df_art)
        all_media.append(df_med)
        all_colors.append(df_col)
    
    # Combine all pages
    final_artifacts = pd.concat(all_artifacts, ignore_index=True)
    final_media = pd.concat(all_media, ignore_index=True)
    final_colors = pd.concat(all_colors, ignore_index=True)
    
    # Load to database
    load_to_database(final_artifacts, final_media, final_colors, db_config)
```

### Analytics Query Patterns

```python
def execute_analytics_query(query, db_config):
    """Execute SQL analytics query and return results"""
    try:
        conn = mysql.connector.connect(**db_config)
        df = pd.read_sql_query(query, conn)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        if conn.is_connected():
            conn.close()

# Sample analytical queries
QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "media_availability": """
        SELECT 
            CASE 
                WHEN EXISTS (
                    SELECT 1 FROM artifactmedia m 
                    WHERE m.artifact_id = a.id
                ) THEN 'With Media'
                ELSE 'No Media'
            END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        GROUP BY media_status
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

# Execute and visualize
import plotly.express as px

def visualize_query(df, title, x_col, y_col):
    """Create bar chart from query results"""
    fig = px.bar(df, x=x_col, y=y_col, title=title)
    return fig

# Example usage
db_config = {
    'host': 'your_host',
    'user': 'your_user',
    'password': 'your_password',
    'database': 'harvard_artifacts'
}

df_culture = execute_analytics_query(QUERIES['artifacts_by_culture'], db_config)
fig = visualize_query(df_culture, 
                     "Top 15 Cultures by Artifact Count",
                     'culture', 'count')
```

### Streamlit Integration Pattern

```python
import streamlit as st
import os

# Page configuration
st.set_page_config(
    page_title="Harvard Art Museums Analytics",
    page_icon="🎨",
    layout="wide"
)

# Sidebar configuration
st.sidebar.title("Harvard Art Museums ETL")
api_key = st.sidebar.text_input("API Key", 
                                type="password",
                                value=os.getenv('HARVARD_API_KEY', ''))

# Database config from secrets
db_config = {
    'host': st.secrets.database.host,
    'port': st.secrets.database.port,
    'user': st.secrets.database.user,
    'password': st.secrets.database.password,
    'database': st.secrets.database.database
}

# Main app sections
tab1, tab2, tab3 = st.tabs(["Data Collection", "Analytics", "Visualization"])

with tab1:
    st.header("Collect Artifact Data")
    pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching and processing data..."):
            run_etl_pipeline(api_key, db_config, pages=pages)
            st.success(f"Successfully loaded {pages * 100} artifacts")

with tab2:
    st.header("SQL Analytics")
    query_name = st.selectbox("Select Query", list(QUERIES.keys()))
    
    if st.button("Execute Query"):
        df = execute_analytics_query(QUERIES[query_name], db_config)
        st.dataframe(df)

with tab3:
    st.header("Data Visualization")
    query_choice = st.selectbox("Choose Analysis", list(QUERIES.keys()))
    
    if st.button("Generate Chart"):
        df = execute_analytics_query(QUERIES[query_choice], db_config)
        fig = visualize_query(df, query_choice, df.columns[0], df.columns[1])
        st.plotly_chart(fig, use_container_width=True)
```

## Common Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch data with retry logic"""
    for attempt in range(max_retries):
        try:
            response = fetch_artifacts(api_key, page)
            return response
        except Exception as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            raise e
```

### Database Connection Issues

```python
def test_database_connection(db_config):
    """Test database connectivity"""
    try:
        conn = mysql.connector.connect(**db_config)
        if conn.is_connected():
            print("Database connection successful")
            conn.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Handling NULL Values

```python
def clean_dataframe(df):
    """Clean dataframe before database insertion"""
    # Replace NaN with None for SQL compatibility
    df = df.where(pd.notnull(df), None)
    
    # Truncate long strings
    for col in df.select_dtypes(include=['object']).columns:
        df[col] = df[col].apply(lambda x: str(x)[:500] if x else None)
    
    return df
```

### Pagination Handling

```python
def fetch_all_artifacts(api_key, max_records=1000):
    """Fetch all artifacts with automatic pagination"""
    page = 1
    size = 100
    all_records = []
    
    while len(all_records) < max_records:
        data = fetch_artifacts(api_key, page, size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_records.extend(records)
        page += 1
        
        if len(records) < size:  # Last page
            break
    
    return all_records[:max_records]
```

## Advanced Patterns

### Incremental ETL

```python
def get_last_artifact_id(db_config):
    """Get the last loaded artifact ID"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute(query)
    result = cursor.fetchone()
    conn.close()
    return result[0] if result[0] else 0

def incremental_load(api_key, db_config):
    """Load only new artifacts"""
    last_id = get_last_artifact_id(db_config)
    # Fetch artifacts with ID > last_id
    # Implementation depends on API filtering capabilities
```

### Data Quality Checks

```python
def validate_data_quality(df):
    """Perform data quality checks"""
    checks = {
        'null_ids': df['id'].isnull().sum(),
        'duplicate_ids': df['id'].duplicated().sum(),
        'missing_titles': df['title'].isnull().sum(),
        'total_records': len(df)
    }
    return checks
```

This skill provides comprehensive knowledge for building ETL pipelines and analytics applications using museum collection APIs with Python, SQL, and Streamlit.
