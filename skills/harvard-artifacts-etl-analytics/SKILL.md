---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build etl pipeline for harvard art museums
  - analyze harvard artifacts collection data
  - create streamlit dashboard for museum data
  - extract data from harvard art museums api
  - set up sql database for artifact metadata
  - visualize museum collection analytics
  - implement batch data loading from api
  - query and analyze art museum collections
---

# Harvard Artifacts Collection ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an end-to-end data engineering and analytics application that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards using Streamlit and Plotly.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables (metadata, media, colors)
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics**: Executes 20+ predefined SQL queries for artifact insights
- **Visualization**: Creates interactive dashboards with Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Requirements typically include:**
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

Create a `.env` file or use Streamlit secrets:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Cloud Connection
DB_HOST=your_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Streamlit Secrets (`.streamlit/secrets.toml`)

```toml
[api]
harvard_api_key = "your_api_key_here"

[database]
host = "your_host"
port = 4000
user = "your_username"
password = "your_password"
database = "harvard_artifacts"
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    url VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
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

## Key Components

### 1. API Data Extraction

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    pages_needed = (num_records + page_size - 1) // page_size
    
    for page in range(1, pages_needed + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts[:num_records]
```

### 2. ETL Transformation

```python
def transform_artifacts(artifacts):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
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
            'description': artifact.get('description'),
            'url': artifact.get('url')
        })
        
        # Extract media/images
        if artifact.get('images'):
            for image in artifact['images']:
                media_list.append({
                    'artifact_id': artifact.get('id'),
                    'baseimageurl': image.get('baseimageurl'),
                    'iiifbaseuri': image.get('iiifbaseuri'),
                    'publiccaption': image.get('publiccaption')
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
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

### 3. SQL Loading

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, db_config):
    """
    Batch insert dataframes into MySQL/TiDB
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata
        insert_metadata = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, division, dated, description, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(insert_metadata, df_metadata.values.tolist())
        
        # Insert media
        insert_media = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, publiccaption)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(insert_media, df_media.values.tolist())
        
        # Insert colors
        insert_colors = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(insert_colors, df_colors.values.tolist())
        
        connection.commit()
        print(f"Inserted {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 4. Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "Top Colors in Collection": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY frequency DESC 
        LIMIT 10
    """,
    
    "Artifacts with Images": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as with_images,
            COUNT(DISTINCT a.id) as total_artifacts,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as percentage
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE department IS NOT NULL 
        GROUP BY department 
        ORDER BY count DESC
    """
}

def execute_analytics_query(query, db_config):
    """
    Execute analytics query and return DataFrame
    """
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=st.secrets.get("api", {}).get("harvard_api_key", ""))
        
        num_records = st.slider("Number of Records to Fetch", 10, 1000, 100)
        
        if st.button("Fetch & Load Data"):
            with st.spinner("Fetching from API..."):
                artifacts = fetch_artifacts(api_key, num_records)
                df_meta, df_media, df_colors = transform_artifacts(artifacts)
                
                db_config = {
                    'host': st.secrets["database"]["host"],
                    'port': st.secrets["database"]["port"],
                    'user': st.secrets["database"]["user"],
                    'password': st.secrets["database"]["password"],
                    'database': st.secrets["database"]["database"]
                }
                
                load_to_database(df_meta, df_media, df_colors, db_config)
                st.success(f"Loaded {len(df_meta)} artifacts!")
    
    # Analytics section
    st.header("📊 Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        db_config = {
            'host': st.secrets["database"]["host"],
            'port': st.secrets["database"]["port"],
            'user': st.secrets["database"]["user"],
            'password': st.secrets["database"]["password"],
            'database': st.secrets["database"]["database"]
        }
        
        result_df = execute_analytics_query(ANALYTICS_QUERIES[query_name], db_config)
        
        st.subheader("Query Results")
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) >= 2:
            fig = px.bar(result_df, x=result_df.columns[0], y=result_df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_max_artifact_id(db_config):
    """Get the highest artifact ID already in database"""
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    connection.close()
    return max_id

def fetch_new_artifacts(api_key, last_id):
    """Fetch only artifacts newer than last_id"""
    # Implementation depends on API capabilities
    pass
```

### Error Handling for API Rate Limits

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Retry API calls with exponential backoff"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:  # Rate limit
            wait_time = 2 ** attempt
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
        else:
            raise Exception(f"API error: {response.status_code}")
    return None
```

## Troubleshooting

**Database Connection Issues**
- Verify TiDB/MySQL credentials in secrets
- Check firewall/security group settings for remote connections
- Ensure database exists before running ETL

**API Key Errors**
- Get free API key from: https://harvardartmuseums.org/collections/api
- Check key is correctly set in environment variables

**Memory Issues with Large Datasets**
- Use pagination and batch processing
- Process data in chunks rather than loading all at once
- Adjust `page_size` parameter

**Missing Data in Queries**
- Some artifacts may not have all fields (culture, period, etc.)
- Use `WHERE field IS NOT NULL` in queries
- Handle NULL values in transformations
