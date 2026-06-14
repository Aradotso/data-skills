---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - create analytics dashboard with Harvard Art Museums API
  - set up data engineering project with Streamlit
  - extract and transform art collection data
  - build SQL analytics for museum artifacts
  - create interactive visualization for art data
  - design relational database for museum collections
  - implement batch data pipeline with API pagination
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build and work with the Harvard Art Museums data engineering and analytics application. The project demonstrates a complete ETL pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases, and provides interactive analytics through Streamlit dashboards.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a comprehensive data pipeline that:

- Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables (metadata, media, colors)
- Loads data into MySQL/TiDB Cloud databases with optimized batch inserts
- Executes 20+ analytical SQL queries for insights
- Visualizes results through interactive Plotly charts in Streamlit

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation & Setup

### Prerequisites

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```python
# requirements.txt should include:
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

### Environment Configuration

Create a `.env` file or configure Streamlit secrets:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

For Streamlit deployment, use `.streamlit/secrets.toml`:

```toml
HARVARD_API_KEY = "your_api_key_here"
DB_HOST = "your_database_host"
DB_USER = "your_database_user"
DB_PASSWORD = "your_database_password"
DB_NAME = "harvard_artifacts"
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# With custom port
streamlit run app.py --server.port 8080
```

## Core Components

### 1. API Integration

Fetch data from Harvard Art Museums API with pagination:

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts with pagination support"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
total_records = data['info']['totalrecords']
artifacts = data['records']
```

### 2. ETL Pipeline

#### Extract and Transform

```python
import pandas as pd

def transform_artifacts(records):
    """Transform API records into relational tables"""
    metadata = []
    media = []
    colors = []
    
    for record in records:
        artifact_id = record.get('id')
        
        # Metadata table
        metadata.append({
            'artifact_id': artifact_id,
            'title': record.get('title', 'Untitled'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'dated': record.get('dated'),
            'department': record.get('department'),
            'division': record.get('division'),
            'classification': record.get('classification'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'creditline': record.get('creditline'),
            'accession_number': record.get('accessionyear'),
            'object_number': record.get('objectnumber')
        })
        
        # Media table
        if record.get('images'):
            for img in record['images']:
                media.append({
                    'artifact_id': artifact_id,
                    'image_id': img.get('imageid'),
                    'base_url': img.get('baseimageurl'),
                    'width': img.get('width'),
                    'height': img.get('height'),
                    'format': img.get('format')
                })
        
        # Colors table
        if record.get('colors'):
            for color in record['colors']:
                colors.append({
                    'artifact_id': artifact_id,
                    'color_hex': color.get('hex'),
                    'color_name': color.get('color'),
                    'percentage': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media),
        pd.DataFrame(colors)
    )
```

#### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def create_connection(host, user, password, database):
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        print(f"Error: {e}")
        return None

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            dated VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            classification VARCHAR(200),
            medium TEXT,
            dimensions TEXT,
            creditline TEXT,
            accession_number VARCHAR(50),
            object_number VARCHAR(100)
        )
    """)
    
    # Media table
    cursor.execute("""
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
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_name VARCHAR(50),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def batch_insert_data(connection, df, table_name):
    """Batch insert DataFrame into table"""
    cursor = connection.cursor()
    
    # Prepare insert statement
    cols = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    sql = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
    
    # Convert DataFrame to list of tuples
    data = [tuple(row) for row in df.values]
    
    # Execute batch insert
    cursor.executemany(sql, data)
    connection.commit()
    cursor.close()
```

### 3. SQL Analytics Queries

```python
def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    cursor.close()
    return pd.DataFrame(results, columns=columns)

# Example analytical queries
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count 
        FROM artifactmetadata 
        WHERE department IS NOT NULL 
        GROUP BY department 
        ORDER BY artifact_count DESC
    """,
    
    "Most Common Colors": """
        SELECT color_name, COUNT(*) as frequency 
        FROM artifactcolors 
        WHERE color_name IS NOT NULL 
        GROUP BY color_name 
        ORDER BY frequency DESC 
        LIMIT 15
    """,
    
    "Artifacts with Multiple Images": """
        SELECT m.artifact_id, m.title, COUNT(med.image_id) as image_count 
        FROM artifactmetadata m 
        JOIN artifactmedia med ON m.artifact_id = med.artifact_id 
        GROUP BY m.artifact_id, m.title 
        HAVING image_count > 1 
        ORDER BY image_count DESC 
        LIMIT 20
    """,
    
    "Color Diversity by Artifact": """
        SELECT m.title, COUNT(DISTINCT c.color_name) as color_count 
        FROM artifactmetadata m 
        JOIN artifactcolors c ON m.artifact_id = c.artifact_id 
        GROUP BY m.artifact_id, m.title 
        ORDER BY color_count DESC 
        LIMIT 20
    """
}
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    conn = create_connection(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    # Query selection
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            query = ANALYTICAL_QUERIES[query_name]
            df_result = execute_query(conn, query)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(df_result)
            
            # Visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Section
    st.sidebar.header("ETL Operations")
    
    if st.sidebar.button("Run ETL Pipeline"):
        api_key = os.getenv('HARVARD_API_KEY')
        
        with st.spinner("Fetching data from API..."):
            data = fetch_artifacts(api_key, page=1, size=100)
            records = data['records']
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(records)
        
        with st.spinner("Loading to database..."):
            create_tables(conn)
            batch_insert_data(conn, df_metadata, 'artifactmetadata')
            batch_insert_data(conn, df_media, 'artifactmedia')
            batch_insert_data(conn, df_colors, 'artifactcolors')
        
        st.success(f"✅ Loaded {len(df_metadata)} artifacts successfully!")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pagination Handler

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_records = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page, size=100)
        all_records.extend(data['records'])
        
        # Check if more pages exist
        if page >= data['info']['pages']:
            break
    
    return all_records
```

### Error Handling for API Calls

```python
import time

def safe_api_call(api_key, page, retries=3):
    """API call with retry logic"""
    for attempt in range(retries):
        try:
            response = fetch_artifacts(api_key, page)
            return response
        except requests.exceptions.RequestException as e:
            if attempt < retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            else:
                raise e
```

## Troubleshooting

### Issue: Database Connection Failed

**Solution**: Verify environment variables and network access:

```python
import os

# Check if all required env vars are set
required_vars = ['DB_HOST', 'DB_USER', 'DB_PASSWORD', 'DB_NAME']
missing = [var for var in required_vars if not os.getenv(var)]

if missing:
    print(f"Missing environment variables: {missing}")
```

### Issue: API Rate Limiting

**Solution**: Implement rate limiting:

```python
import time

def rate_limited_fetch(api_key, pages):
    """Fetch with rate limiting"""
    all_records = []
    
    for page in range(1, pages + 1):
        data = fetch_artifacts(api_key, page)
        all_records.extend(data['records'])
        time.sleep(1)  # 1 second delay between requests
    
    return all_records
```

### Issue: Duplicate Primary Keys

**Solution**: Use `INSERT IGNORE` or `REPLACE INTO`:

```python
def upsert_data(connection, df, table_name):
    """Insert or update data"""
    cursor = connection.cursor()
    cols = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    sql = f"REPLACE INTO {table_name} ({cols}) VALUES ({placeholders})"
    data = [tuple(row) for row in df.values]
    cursor.executemany(sql, data)
    connection.commit()
    cursor.close()
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement pagination** for large datasets to avoid memory issues
3. **Use batch inserts** for better database performance
4. **Add data validation** before loading to database
5. **Log ETL operations** for debugging and monitoring
6. **Cache API responses** to reduce redundant calls
7. **Index foreign keys** for query performance

## Advanced Usage

### Custom Query Builder

```python
def build_custom_query(filters):
    """Build dynamic SQL query based on filters"""
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        base_query += f" AND culture = '{filters['culture']}'"
    
    if filters.get('century'):
        base_query += f" AND century = '{filters['century']}'"
    
    if filters.get('department'):
        base_query += f" AND department = '{filters['department']}'"
    
    return base_query
```

### Export Results

```python
def export_to_csv(df, filename):
    """Export query results to CSV"""
    df.to_csv(filename, index=False)
    return filename

# In Streamlit
if st.button("Export Results"):
    csv = df_result.to_csv(index=False)
    st.download_button(
        label="Download CSV",
        data=csv,
        file_name="analytics_results.csv",
        mime="text/csv"
    )
```

This skill provides comprehensive guidance for building and extending the Harvard Art Museums ETL analytics pipeline with production-ready patterns and best practices.
