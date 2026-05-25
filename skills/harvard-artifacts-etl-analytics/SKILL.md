---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL/TiDB, and Plotly visualizations
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering project with museum artifacts data
  - set up analytics dashboard for Harvard art collection
  - extract and analyze Harvard museum data with Python
  - build Streamlit app for art museum analytics
  - configure TiDB database for artifact metadata
  - visualize museum collection data with Plotly
  - implement SQL analytics for art artifacts
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end ETL pipelines and analytics applications using the Harvard Art Museums API. The project demonstrates real-world data engineering patterns including API integration, data transformation, SQL database design, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection application provides:
- **API Integration**: Fetches artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts nested JSON, transforms to relational format, loads into SQL database
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations
- **Database Design**: Normalized schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

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

**Requirements (typical setup):**
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

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_from_harvard

# Database Configuration (MySQL/TiDB)
DB_HOST=your_db_host.tidbcloud.com
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_SSL_CA=/path/to/ca-cert.pem  # Optional for TiDB Cloud
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free tier available)
3. Add to your `.env` file

## Database Setup

### Create Database Schema

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

cursor = conn.cursor()

# Create artifactmetadata table
cursor.execute('''
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
''')

# Create artifactmedia table
cursor.execute('''
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri TEXT,
    primaryimageurl TEXT,
    baseimageurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
''')

# Create artifactcolors table
cursor.execute('''
CREATE TABLE IF NOT EXISTS artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
''')

conn.commit()
cursor.close()
conn.close()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        
        return {
            'records': data.get('records', []),
            'info': data.get('info', {}),
            'total_records': data.get('info', {}).get('totalrecords', 0)
        }
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

def fetch_all_artifacts(api_key, max_records=1000):
    """
    Fetch multiple pages of artifacts with rate limiting
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        print(f"Fetching page {page}...")
        result = fetch_artifacts(api_key, page=page, size=size)
        
        if not result or not result['records']:
            break
            
        all_artifacts.extend(result['records'])
        
        # Rate limiting: 1 request per second
        time.sleep(1)
        
        page += 1
        
        if len(all_artifacts) >= max_records:
            all_artifacts = all_artifacts[:max_records]
            break
    
    return all_artifacts
```

### Transform: Flatten Nested JSON

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform artifact data into three separate dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata = {
            'id': artifact_id,
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'dated': artifact.get('dated', ''),
            'medium': artifact.get('medium', ''),
            'technique': artifact.get('technique', ''),
            'dimensions': artifact.get('dimensions', ''),
            'creditline': artifact.get('creditline', ''),
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact_id,
            'iiifbaseuri': artifact.get('iiifbaseuri', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'baseimageurl': artifact.get('baseimageurl', '')
        }
        media_list.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return {
        'metadata': pd.DataFrame(metadata_list),
        'media': pd.DataFrame(media_list),
        'colors': pd.DataFrame(colors_list)
    }
```

### Load: Insert into Database

```python
def load_to_database(dataframes, conn):
    """
    Load transformed data into MySQL/TiDB database
    """
    cursor = conn.cursor()
    
    # Load metadata
    metadata_df = dataframes['metadata']
    for _, row in metadata_df.iterrows():
        cursor.execute('''
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             dated, medium, technique, dimensions, creditline, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ''', tuple(row))
    
    # Load media
    media_df = dataframes['media']
    for _, row in media_df.iterrows():
        cursor.execute('''
            INSERT INTO artifactmedia 
            (artifact_id, iiifbaseuri, primaryimageurl, baseimageurl)
            VALUES (%s, %s, %s, %s)
        ''', tuple(row))
    
    # Load colors
    colors_df = dataframes['colors']
    if not colors_df.empty:
        for _, row in colors_df.iterrows():
            cursor.execute('''
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            ''', tuple(row))
    
    conn.commit()
    cursor.close()
    print(f"Loaded {len(metadata_df)} artifacts into database")
```

## Streamlit Application

### Main Application Structure

```python
import streamlit as st
import mysql.connector
import pandas as pd
import plotly.express as px
from dotenv import load_dotenv
import os

load_dotenv()

# Page configuration
st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🎨",
    layout="wide"
)

# Database connection
@st.cache_resource
def get_database_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def run_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_database_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Sidebar navigation
st.sidebar.title("📊 Navigation")
page = st.sidebar.radio(
    "Select Page",
    ["Home", "ETL Pipeline", "SQL Analytics", "Data Visualization"]
)

if page == "Home":
    st.title("🎨 Harvard Artifacts Analytics Dashboard")
    st.markdown("""
    ### Welcome to the Harvard Art Museums Analytics Platform
    
    This application provides:
    - **ETL Pipeline**: Extract, Transform, Load artifact data
    - **SQL Analytics**: Run predefined analytical queries
    - **Visualizations**: Interactive charts and insights
    """)
    
    # Display database statistics
    conn = get_database_connection()
    cursor = conn.cursor()
    
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    total_artifacts = cursor.fetchone()[0]
    
    cursor.execute("SELECT COUNT(DISTINCT culture) FROM artifactmetadata WHERE culture != ''")
    total_cultures = cursor.fetchone()[0]
    
    cursor.execute("SELECT COUNT(*) FROM artifactmedia WHERE primaryimageurl != ''")
    artifacts_with_images = cursor.fetchone()[0]
    
    conn.close()
    
    col1, col2, col3 = st.columns(3)
    col1.metric("Total Artifacts", total_artifacts)
    col2.metric("Unique Cultures", total_cultures)
    col3.metric("With Images", artifacts_with_images)
```

### ETL Pipeline Page

```python
elif page == "ETL Pipeline":
    st.title("🔄 ETL Pipeline")
    
    st.markdown("### Extract Data from Harvard API")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    num_records = st.number_input("Number of records to fetch", 
                                  min_value=10, max_value=5000, value=100)
    
    if st.button("Run ETL Pipeline"):
        if not api_key:
            st.error("Please provide API key")
        else:
            with st.spinner("Fetching artifacts..."):
                artifacts = fetch_all_artifacts(api_key, max_records=num_records)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                dataframes = transform_artifacts(artifacts)
                st.success("Data transformation complete")
            
            with st.spinner("Loading to database..."):
                conn = get_database_connection()
                load_to_database(dataframes, conn)
                conn.close()
                st.success("Data loaded successfully!")
            
            st.balloons()
```

### SQL Analytics Page

```python
elif page == "SQL Analytics":
    st.title("📊 SQL Analytics Dashboard")
    
    # Predefined analytical queries
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century != ''
            GROUP BY century
            ORDER BY count DESC
            LIMIT 15
        """,
        "Most Common Classifications": """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification != ''
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 10
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department != ''
            GROUP BY department
            ORDER BY count DESC
        """,
        "Top Colors Used": """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 10
        """,
        "Artifacts with Media": """
            SELECT 
                CASE 
                    WHEN primaryimageurl != '' THEN 'Has Image'
                    ELSE 'No Image'
                END as image_status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY image_status
        """,
        "Artifacts by Medium": """
            SELECT medium, COUNT(*) as count
            FROM artifactmetadata
            WHERE medium != ''
            GROUP BY medium
            ORDER BY count DESC
            LIMIT 10
        """,
        "Color Spectrum Analysis": """
            SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_coverage
            FROM artifactcolors
            WHERE spectrum != ''
            GROUP BY spectrum
            ORDER BY count DESC
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = run_query(queries[selected_query])
            
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) == 2 and 'count' in df.columns:
                fig = px.bar(df, x=df.columns[0], y='count',
                           title=f"{selected_query}")
                st.plotly_chart(fig, use_container_width=True)
```

## Common Analytics Patterns

### Time-based Analysis

```python
# Artifacts by time period
query = """
SELECT 
    CASE 
        WHEN century LIKE '%BC%' THEN 'Ancient'
        WHEN century LIKE '1%' THEN 'Renaissance/Early Modern'
        WHEN century LIKE '19%' THEN '19th Century'
        WHEN century LIKE '20%' THEN '20th Century'
        WHEN century LIKE '21%' THEN '21st Century'
        ELSE 'Unknown'
    END as period,
    COUNT(*) as count
FROM artifactmetadata
GROUP BY period
ORDER BY count DESC
"""
```

### Color Dominance Analysis

```python
# Find artifacts with dominant color percentage > 50%
query = """
SELECT 
    a.title,
    a.culture,
    c.color,
    c.percent
FROM artifactmetadata a
JOIN artifactcolors c ON a.id = c.artifact_id
WHERE c.percent > 50
ORDER BY c.percent DESC
LIMIT 20
"""
```

### Cross-table Insights

```python
# Cultures with most colorful artifacts
query = """
SELECT 
    a.culture,
    COUNT(DISTINCT c.artifact_id) as artifacts_with_colors,
    COUNT(c.id) as total_colors,
    AVG(c.percent) as avg_color_coverage
FROM artifactmetadata a
JOIN artifactcolors c ON a.id = c.artifact_id
WHERE a.culture != ''
GROUP BY a.culture
HAVING artifacts_with_colors > 5
ORDER BY total_colors DESC
LIMIT 15
"""
```

## Troubleshooting

### API Rate Limiting

```python
# Implement exponential backoff
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time} seconds...")
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues

```python
# Add connection pooling and error handling
from mysql.connector import pooling

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'pool_name': 'mypool',
    'pool_size': 5
}

try:
    connection_pool = mysql.connector.pooling.MySQLConnectionPool(**db_config)
    conn = connection_pool.get_connection()
except mysql.connector.Error as e:
    print(f"Database connection error: {e}")
```

### Large Dataset Memory Management

```python
# Use batch processing for large datasets
def load_in_batches(dataframes, conn, batch_size=100):
    cursor = conn.cursor()
    
    metadata_df = dataframes['metadata']
    total_rows = len(metadata_df)
    
    for i in range(0, total_rows, batch_size):
        batch = metadata_df.iloc[i:i+batch_size]
        for _, row in batch.iterrows():
            cursor.execute('''
                INSERT IGNORE INTO artifactmetadata 
                (id, title, culture, century, classification, department, 
                 dated, medium, technique, dimensions, creditline, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ''', tuple(row))
        conn.commit()
        print(f"Processed {min(i+batch_size, total_rows)}/{total_rows} rows")
    
    cursor.close()
```

### Missing or Null Values

```python
# Handle missing data during transformation
def safe_get(dictionary, key, default=''):
    """Safely extract values with null handling"""
    value = dictionary.get(key, default)
    return value if value is not None else default

# Apply in transform function
metadata = {
    'id': artifact.get('id'),
    'title': safe_get(artifact, 'title'),
    'culture': safe_get(artifact, 'culture'),
    # ... rest of fields
}
```

## Advanced Usage

### Custom Query Builder

```python
def build_filter_query(filters):
    """Build dynamic SQL query based on user filters"""
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    params = []
    
    if filters.get('culture'):
        base_query += " AND culture = %s"
        params.append(filters['culture'])
    
    if filters.get('century'):
        base_query += " AND century = %s"
        params.append(filters['century'])
    
    if filters.get('classification'):
        base_query += " AND classification = %s"
        params.append(filters['classification'])
    
    return base_query, params
```

### Export Results

```python
# Export query results to CSV
def export_to_csv(df, filename):
    """Export DataFrame to CSV file"""
    df.to_csv(filename, index=False)
    st.success(f"Data exported to {filename}")

# In Streamlit app
if st.button("Export Results"):
    export_to_csv(df, "query_results.csv")
```

This skill enables comprehensive ETL pipeline development and analytics dashboard creation using Harvard's museum data with modern data engineering practices.
