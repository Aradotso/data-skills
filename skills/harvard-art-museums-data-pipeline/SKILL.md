---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and museum data
  - query Harvard Art Museums API and store in SQL
  - build artifact collection data engineering project
  - visualize museum data with Plotly and SQL
  - implement ETL for Harvard artifacts collection
  - create data analytics app with museum API
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates production-ready ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON data into relational SQL tables
- **Database Design**: Structures data across three normalized tables (metadata, media, colors)
- **SQL Analytics**: Provides 20+ predefined analytical queries for insights
- **Visualization**: Interactive Streamlit dashboard with Plotly charts

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

**Dependencies** (typical requirements.txt):
```
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.1.0
plotly>=5.17.0
python-dotenv>=1.0.0
```

## Getting Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Store in environment variable: `HARVARD_API_KEY`

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Core Components

### 1. API Integration

**Fetch artifacts with pagination:**

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key from environment
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        JSON response with artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.environ.get('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

**Handle pagination for large datasets:**

```python
def fetch_all_artifacts(api_key, max_records=1000):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        try:
            response = fetch_artifacts(api_key, page=page, size=size)
            records = response.get('records', [])
            
            if not records:
                break
            
            all_artifacts.extend(records)
            page += 1
            
            # Rate limiting
            import time
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts[:max_records]
```

### 2. ETL Pipeline

**Extract and transform artifact data:**

```python
import pandas as pd

def extract_metadata(artifacts):
    """
    Extract metadata from artifact records
    
    Returns:
        DataFrame with normalized metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        })
    
    return pd.DataFrame(metadata_list)

def extract_media(artifacts):
    """Extract media/image information"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'media_id': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'iiifbaseuri': img.get('iiifbaseuri'),
                'height': img.get('height'),
                'width': img.get('width')
            })
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """Extract color information"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent'),
                'hue': color.get('hue'),
                'css3': color.get('css3')
            })
    
    return pd.DataFrame(color_list)
```

**Complete ETL workflow:**

```python
def run_etl_pipeline(api_key, num_records=500):
    """
    Complete ETL pipeline: Extract, Transform, Load
    """
    print("Starting ETL Pipeline...")
    
    # EXTRACT
    print(f"Extracting {num_records} artifacts from API...")
    artifacts = fetch_all_artifacts(api_key, max_records=num_records)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # TRANSFORM
    print("Transforming data...")
    df_metadata = extract_metadata(artifacts)
    df_media = extract_media(artifacts)
    df_colors = extract_colors(artifacts)
    
    print(f"Metadata records: {len(df_metadata)}")
    print(f"Media records: {len(df_media)}")
    print(f"Color records: {len(df_colors)}")
    
    return df_metadata, df_media, df_colors
```

### 3. Database Setup

**SQL schema for MySQL/TiDB:**

```sql
-- Metadata table
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title TEXT,
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique TEXT,
    medium TEXT,
    dimensions TEXT,
    creditline TEXT,
    accessionyear INT,
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_department (department)
);

-- Media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_id INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id),
    INDEX idx_artifact_id (artifact_id)
);

-- Colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    hue VARCHAR(50),
    css3 VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id),
    INDEX idx_artifact_id (artifact_id),
    INDEX idx_color (color)
);
```

**Database connection and loading:**

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.environ.get('DB_HOST'),
            user=os.environ.get('DB_USER'),
            password=os.environ.get('DB_PASSWORD'),
            database=os.environ.get('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(df_metadata, df_media, df_colors):
    """Load dataframes to SQL database"""
    connection = create_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Load metadata
        for _, row in df_metadata.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (artifact_id, title, culture, century, classification, 
                 department, division, dated, period, technique, 
                 medium, dimensions, creditline, accessionyear)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Load media
        for _, row in df_media.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, media_id, baseimageurl, iiifbaseuri, height, width)
                VALUES (%s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Load colors
        for _, row in df_colors.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, percent, hue, css3)
                VALUES (%s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        connection.commit()
        print("Data loaded successfully!")
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. SQL Analytics Queries

**Example analytical queries:**

```python
def get_artifacts_by_culture(limit=10):
    """Query: Top cultures by artifact count"""
    query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT %s
    """
    return execute_query(query, (limit,))

def get_artifacts_by_century():
    """Query: Distribution by century"""
    query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """
    return execute_query(query)

def get_color_distribution(top_n=10):
    """Query: Most common colors"""
    query = """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT %s
    """
    return execute_query(query, (top_n,))

def get_media_availability():
    """Query: Artifacts with/without images"""
    query = """
        SELECT 
            CASE WHEN media_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as status,
            COUNT(DISTINCT m.artifact_id) as count
        FROM artifactmetadata m
        LEFT JOIN artifactmedia am ON m.artifact_id = am.artifact_id
        GROUP BY status
    """
    return execute_query(query)

def execute_query(query, params=None):
    """Execute SQL query and return results as DataFrame"""
    connection = create_db_connection()
    if not connection:
        return pd.DataFrame()
    
    try:
        df = pd.read_sql(query, connection, params=params)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        connection.close()
```

### 5. Streamlit Dashboard

**Main application structure:**

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        
        # API Key input
        api_key = st.text_input(
            "Harvard API Key",
            type="password",
            value=os.environ.get('HARVARD_API_KEY', '')
        )
        
        # Data collection
        num_records = st.slider("Records to fetch", 100, 2000, 500, 100)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL..."):
                df_metadata, df_media, df_colors = run_etl_pipeline(
                    api_key, num_records
                )
                load_to_database(df_metadata, df_media, df_colors)
                st.success("ETL completed!")
    
    # Main content area
    tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🗄️ Data Tables", "📈 Custom Query"])
    
    with tab1:
        show_analytics_dashboard()
    
    with tab2:
        show_data_tables()
    
    with tab3:
        show_custom_query_interface()

def show_analytics_dashboard():
    """Display predefined analytics"""
    st.header("Analytical Insights")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("Top Cultures")
        df = get_artifacts_by_culture(10)
        if not df.empty:
            fig = px.bar(df, x='culture', y='artifact_count',
                        title="Artifacts by Culture")
            st.plotly_chart(fig, use_container_width=True)
            st.dataframe(df)
    
    with col2:
        st.subheader("Century Distribution")
        df = get_artifacts_by_century()
        if not df.empty:
            fig = px.bar(df, x='century', y='count',
                        title="Artifacts by Century")
            st.plotly_chart(fig, use_container_width=True)
    
    st.subheader("Color Analysis")
    df_colors = get_color_distribution(15)
    if not df_colors.empty:
        fig = px.pie(df_colors, names='color', values='frequency',
                    title="Color Distribution in Artifacts")
        st.plotly_chart(fig, use_container_width=True)

def show_custom_query_interface():
    """Allow users to run custom SQL queries"""
    st.header("Custom SQL Query")
    
    query = st.text_area(
        "Enter SQL Query",
        "SELECT * FROM artifactmetadata LIMIT 10",
        height=150
    )
    
    if st.button("Execute Query"):
        try:
            df = execute_query(query)
            st.success(f"Query returned {len(df)} rows")
            st.dataframe(df)
            
            # Auto-generate chart if numeric columns exist
            numeric_cols = df.select_dtypes(include=['number']).columns
            if len(numeric_cols) > 0:
                st.subheader("Visualization")
                x_col = st.selectbox("X-axis", df.columns)
                y_col = st.selectbox("Y-axis", numeric_cols)
                fig = px.bar(df.head(20), x=x_col, y=y_col)
                st.plotly_chart(fig)
        except Exception as e:
            st.error(f"Query error: {e}")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_etl(api_key, last_processed_id=0):
    """Load only new artifacts since last run"""
    query = f"SELECT MAX(artifact_id) FROM artifactmetadata"
    result = execute_query(query)
    last_id = result.iloc[0, 0] if not result.empty else 0
    
    # Fetch artifacts with ID > last_id
    # (Requires API support for filtering)
    pass
```

### Pattern 2: Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_records):
    """ETL with comprehensive error handling"""
    try:
        artifacts = fetch_all_artifacts(api_key, num_records)
        
        # Validate data
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform with error logging
        df_metadata = extract_metadata(artifacts)
        
        # Check for duplicates
        duplicates = df_metadata.duplicated(subset=['artifact_id'])
        if duplicates.any():
            print(f"Warning: {duplicates.sum()} duplicate artifact IDs")
            df_metadata = df_metadata.drop_duplicates(subset=['artifact_id'])
        
        # Load to database
        success = load_to_database(df_metadata, df_media, df_colors)
        
        return success
        
    except requests.RequestException as e:
        print(f"API Error: {e}")
        return False
    except pd.errors.EmptyDataError as e:
        print(f"Data Error: {e}")
        return False
    except Exception as e:
        print(f"Unexpected error: {e}")
        return False
```

### Pattern 3: Batch Processing

```python
def batch_load(df, table_name, batch_size=1000):
    """Load large dataframes in batches"""
    connection = create_db_connection()
    cursor = connection.cursor()
    
    total_rows = len(df)
    for start in range(0, total_rows, batch_size):
        end = min(start + batch_size, total_rows)
        batch = df.iloc[start:end]
        
        # Generate bulk insert
        values = [tuple(row) for _, row in batch.iterrows()]
        placeholders = ', '.join(['%s'] * len(batch.columns))
        query = f"INSERT INTO {table_name} VALUES ({placeholders})"
        
        cursor.executemany(query, values)
        connection.commit()
        
        print(f"Loaded {end}/{total_rows} rows")
    
    cursor.close()
    connection.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed
            if wait_time > 0:
                time.sleep(wait_time)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifacts_rate_limited(api_key, page):
    return fetch_artifacts(api_key, page)
```

### Database Connection Pool

```python
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.environ.get('DB_HOST'),
    user=os.environ.get('DB_USER'),
    password=os.environ.get('DB_PASSWORD'),
    database=os.environ.get('DB_NAME')
)

def get_connection():
    """Get connection from pool"""
    return connection_pool.get_connection()
```

### Handling Missing Data

```python
def clean_artifact_data(df):
    """Clean and handle missing values"""
    # Fill numeric nulls with 0
    numeric_cols = df.select_dtypes(include=['number']).columns
    df[numeric_cols] = df[numeric_cols].fillna(0)
    
    # Fill string nulls with 'Unknown'
    string_cols = df.select_dtypes(include=['object']).columns
    df[string_cols] = df[string_cols].fillna('Unknown')
    
    # Remove rows with critical missing data
    df = df.dropna(subset=['artifact_id', 'title'])
    
    return df
```

### Memory Optimization

```python
def optimize_dataframe_memory(df):
    """Reduce memory usage of DataFrame"""
    for col in df.columns:
        col_type = df[col].dtype
        
        if col_type == 'object':
            # Convert to category if few unique values
            if df[col].nunique() / len(df) < 0.5:
                df[col] = df[col].astype('category')
        
        elif 'int' in str(col_type):
            # Downcast integers
            df[col] = pd.to_numeric(df[col], downcast='integer')
        
        elif 'float' in str(col_type):
            # Downcast floats
            df[col] = pd.to_numeric(df[col], downcast='float')
    
    return df
```

## Environment Variables

Required environment variables for production:

```bash
# Harvard API
HARVARD_API_KEY=your_api_key_here

# Database
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306

# Optional: TiDB Cloud
TIDB_HOST=gateway01.us-west-2.prod.aws.tidbcloud.com
TIDB_PORT=4000
```

Use `.env` file with `python-dotenv`:

```python
from dotenv import load_dotenv
load_dotenv()
```

## Key Commands Summary

```bash
# Run the full application
streamlit run app.py

# Run specific components
python etl_pipeline.py          # Run ETL only
python analytics_queries.py     # Test SQL queries
python api_collector.py         # Fetch data only

# Database setup
mysql -u $DB_USER -p$DB_PASSWORD < schema.sql
```

This skill provides comprehensive guidance for building production-ready data engineering pipelines with museum data, SQL analytics, and interactive visualizations.
