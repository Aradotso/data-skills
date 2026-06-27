---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering and analytics application using Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with museum artifacts
  - fetch and analyze Harvard museum collection
  - set up data engineering pipeline for art collection
  - visualize museum artifact data with Streamlit
  - extract transform load pipeline for art museums API
  - query and analyze Harvard Art Museums metadata
  - build SQL analytics for museum collection data
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection application provides:
- **API Integration**: Fetches artifact metadata, media, and color data from Harvard Art Museums
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into relational database tables
- **SQL Analytics**: Runs 20+ predefined analytical queries on artifact collections
- **Interactive Dashboards**: Visualizes query results using Plotly and Streamlit
- **Production-Ready**: Handles pagination, rate limiting, batch inserts, and error handling

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

Create a `.env` file in the project root:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=gateway01.ap-southeast-1.prod.aws.tidbcloud.com
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
DB_SSL_CA=/path/to/ca-cert.pem
```

### Getting API Access

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Obtain your API key
3. Store in environment variable: `HARVARD_API_KEY`

### Database Setup

The application supports MySQL and TiDB Cloud. Schema includes three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    provenance TEXT,
    description TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url VARCHAR(500),
    format VARCHAR(50),
    width INT,
    height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

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

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Key Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(max_pages=10):
    """Collect multiple pages of artifacts"""
    all_records = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(page=page)
        records = data.get('records', [])
        all_records.extend(records)
        
        if len(records) == 0:
            break
    
    return all_records
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """Transform raw API data into structured tables"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionyear'),
            'provenance': artifact.get('provenance'),
            'description': artifact.get('description')
        }
        metadata_list.append(metadata)
        
        # Extract media
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact.get('id'),
                'base_url': img.get('baseimageurl'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height')
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(df_metadata, df_media, df_colors, connection):
    """Load transformed data into SQL database"""
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in df_metadata.iterrows():
        insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, dated, department, 
         classification, medium, dimensions, creditline, 
         accession_number, provenance, description)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.execute(insert_query, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, base_url, format, width, height)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(insert_query, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(insert_query, tuple(row))
    
    connection.commit()
    cursor.close()
```

### 3. Database Connection

```python
import os
import ssl

def create_db_connection():
    """Create MySQL/TiDB connection with SSL"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 4000)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            ssl_ca=os.getenv('DB_SSL_CA'),
            ssl_verify_cert=True
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
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
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(media.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.id = media.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Average Colors per Artifact": """
        SELECT 
            m.classification,
            AVG(color_count) as avg_colors
        FROM artifactmetadata m
        JOIN (
            SELECT artifact_id, COUNT(*) as color_count
            FROM artifactcolors
            GROUP BY artifact_id
        ) c ON m.id = c.artifact_id
        GROUP BY m.classification
        ORDER BY avg_colors DESC
    """
}

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar
    with st.sidebar:
        st.header("⚙️ Configuration")
        
        # Data Collection
        if st.button("Fetch New Data"):
            with st.spinner("Collecting artifacts..."):
                raw_data = collect_all_artifacts(max_pages=5)
                df_meta, df_media, df_colors = transform_artifacts(raw_data)
                
                conn = create_db_connection()
                if conn:
                    load_to_database(df_meta, df_media, df_colors, conn)
                    conn.close()
                    st.success(f"Loaded {len(df_meta)} artifacts!")
    
    # Main content
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        conn = create_db_connection()
        if conn:
            query = ANALYTICS_QUERIES[query_name]
            df_result = execute_query(conn, query)
            conn.close()
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df_result, use_container_width=True)
            
            # Visualization
            if len(df_result) > 0:
                st.subheader("Visualization")
                
                x_col = df_result.columns[0]
                y_col = df_result.columns[1]
                
                fig = px.bar(
                    df_result,
                    x=x_col,
                    y=y_col,
                    title=query_name,
                    color=y_col,
                    color_continuous_scale='Viridis'
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(connection, table, data_list, batch_size=1000):
    """Insert data in batches for better performance"""
    cursor = connection.cursor()
    
    for i in range(0, len(data_list), batch_size):
        batch = data_list[i:i + batch_size]
        # Execute batch insert
        cursor.executemany(insert_query, batch)
        connection.commit()
    
    cursor.close()
```

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(page, delay=1):
    """Fetch with delay to respect API rate limits"""
    data = fetch_artifacts(page=page)
    time.sleep(delay)  # Wait between requests
    return data
```

### Error Handling

```python
def safe_etl_pipeline(max_retries=3):
    """ETL pipeline with retry logic"""
    for attempt in range(max_retries):
        try:
            raw_data = collect_all_artifacts()
            df_meta, df_media, df_colors = transform_artifacts(raw_data)
            
            conn = create_db_connection()
            load_to_database(df_meta, df_media, df_colors, conn)
            conn.close()
            
            return True
        except Exception as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            time.sleep(5)
    
    return False
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` is set in `.env`
- Verify API key is valid at Harvard Art Museums portal
- Check rate limits (typically 2500 requests/day)

### Database Connection Errors
- Verify `DB_HOST`, `DB_USER`, `DB_PASSWORD` are correct
- For TiDB Cloud, ensure SSL certificate path is valid
- Check firewall rules allow connection to database port

### Data Loading Failures
- Use batch inserts for large datasets to avoid timeouts
- Implement ON DUPLICATE KEY UPDATE for idempotent loads
- Add connection pooling for concurrent requests

### Empty Query Results
- Verify data was loaded: `SELECT COUNT(*) FROM artifactmetadata`
- Check for NULL values in GROUP BY columns
- Ensure foreign key relationships are intact

### Streamlit Performance
- Cache database connections with `@st.cache_resource`
- Use `@st.cache_data` for query results
- Limit initial data fetch size during development
