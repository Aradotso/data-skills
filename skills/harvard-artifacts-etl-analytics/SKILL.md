---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - create a data engineering project with Harvard artifacts collection
  - set up analytics dashboard for museum artifact data
  - extract and transform Harvard museum API data to SQL
  - build a Streamlit app for art museum data visualization
  - implement ETL for Harvard Art Museums with Python
  - analyze Harvard artifacts collection with SQL queries
  - create interactive dashboards for museum metadata
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- Fetches artifact data from the Harvard Art Museums API with pagination and rate limiting
- Performs ETL operations to transform nested JSON into relational database tables
- Stores structured data in MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata, media, and color data
- Provides interactive Streamlit dashboards with Plotly visualizations

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file or configure environment variables:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Schema

The application uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(300),
    period VARCHAR(200),
    dated VARCHAR(200),
    creditline TEXT,
    division VARCHAR(200)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
artifacts = data.get('records', [])
print(f"Fetched {len(artifacts)} artifacts")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.conn = None
        
    def connect_db(self):
        """Establish database connection"""
        self.conn = mysql.connector.connect(
            host=self.db_config['host'],
            user=self.db_config['user'],
            password=self.db_config['password'],
            database=self.db_config['database']
        )
        return self.conn.cursor()
    
    def extract_metadata(self, artifact: Dict) -> Dict:
        """Extract metadata from artifact JSON"""
        return {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'dated': artifact.get('dated'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division')
        }
    
    def extract_media(self, artifact: Dict) -> List[Dict]:
        """Extract media/image data from artifact"""
        media_list = []
        artifact_id = artifact.get('id')
        
        # Primary image
        if artifact.get('primaryimageurl'):
            media_list.append({
                'artifact_id': artifact_id,
                'image_url': artifact['primaryimageurl'],
                'media_type': 'primary_image',
                'caption': artifact.get('title', '')
            })
        
        # Additional images
        for image in artifact.get('images', []):
            media_list.append({
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl'),
                'media_type': 'additional_image',
                'caption': image.get('caption', '')
            })
        
        return media_list
    
    def extract_colors(self, artifact: Dict) -> List[Dict]:
        """Extract color data from artifact"""
        colors_list = []
        artifact_id = artifact.get('id')
        
        for color in artifact.get('colors', []):
            colors_list.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
        
        return colors_list
    
    def load_metadata(self, metadata_list: List[Dict]):
        """Load metadata into database"""
        cursor = self.connect_db()
        
        insert_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, century, classification, department, 
         technique, period, dated, creditline, division)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
        
        values = [
            (m['artifact_id'], m['title'], m['culture'], m['century'],
             m['classification'], m['department'], m['technique'],
             m['period'], m['dated'], m['creditline'], m['division'])
            for m in metadata_list
        ]
        
        cursor.executemany(insert_query, values)
        self.conn.commit()
        print(f"Loaded {cursor.rowcount} metadata records")
        cursor.close()
    
    def load_media(self, media_list: List[Dict]):
        """Load media data into database"""
        cursor = self.connect_db()
        
        insert_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, media_type, caption)
        VALUES (%s, %s, %s, %s)
        """
        
        values = [
            (m['artifact_id'], m['image_url'], m['media_type'], m['caption'])
            for m in media_list
        ]
        
        cursor.executemany(insert_query, values)
        self.conn.commit()
        print(f"Loaded {cursor.rowcount} media records")
        cursor.close()
    
    def run_etl(self, artifacts: List[Dict]):
        """Run full ETL pipeline"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in artifacts:
            metadata_list.append(self.extract_metadata(artifact))
            media_list.extend(self.extract_media(artifact))
            colors_list.extend(self.extract_colors(artifact))
        
        self.load_metadata(metadata_list)
        self.load_media(media_list)
        # Similar load for colors
        
        print(f"ETL completed: {len(metadata_list)} artifacts processed")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = HarvardETL(db_config)
etl.run_etl(artifacts)
```

### 3. SQL Analytics Queries

```python
class ArtifactAnalytics:
    def __init__(self, db_config):
        self.db_config = db_config
    
    def execute_query(self, query: str) -> pd.DataFrame:
        """Execute SQL query and return DataFrame"""
        conn = mysql.connector.connect(**self.db_config)
        df = pd.read_sql(query, conn)
        conn.close()
        return df
    
    def artifacts_by_culture(self):
        """Get artifact count by culture"""
        query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
        """
        return self.execute_query(query)
    
    def artifacts_by_century(self):
        """Get artifact distribution by century"""
        query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        """
        return self.execute_query(query)
    
    def artifacts_with_images(self):
        """Count artifacts with images"""
        query = """
        SELECT 
            COUNT(DISTINCT a.artifact_id) as total_artifacts,
            COUNT(DISTINCT m.artifact_id) as artifacts_with_images,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.artifact_id), 2) as percentage
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.artifact_id = m.artifact_id
        """
        return self.execute_query(query)
    
    def top_colors(self):
        """Get most common colors across artifacts"""
        query = """
        SELECT color_hex, COUNT(*) as frequency, 
               ROUND(AVG(color_percent), 2) as avg_percentage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY frequency DESC
        LIMIT 15
        """
        return self.execute_query(query)
    
    def department_distribution(self):
        """Get artifact count by department"""
        query = """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
        """
        return self.execute_query(query)

# Usage
analytics = ArtifactAnalytics(db_config)
culture_df = analytics.artifacts_by_culture()
print(culture_df.head())
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    st.markdown("---")
    
    # Initialize analytics
    analytics = ArtifactAnalytics(db_config)
    
    # Sidebar for query selection
    st.sidebar.header("📊 Analytics Queries")
    query_option = st.sidebar.selectbox(
        "Select Analysis",
        [
            "Artifacts by Culture",
            "Artifacts by Century",
            "Image Availability",
            "Color Distribution",
            "Department Distribution"
        ]
    )
    
    # Execute selected query
    if query_option == "Artifacts by Culture":
        st.subheader("Artifact Distribution by Culture")
        df = analytics.artifacts_by_culture()
        
        # Display table
        st.dataframe(df, use_container_width=True)
        
        # Create visualization
        fig = px.bar(df, x='culture', y='artifact_count',
                     title='Top 20 Cultures by Artifact Count',
                     labels={'culture': 'Culture', 'artifact_count': 'Count'})
        st.plotly_chart(fig, use_container_width=True)
    
    elif query_option == "Artifacts by Century":
        st.subheader("Artifact Distribution by Century")
        df = analytics.artifacts_by_century()
        
        st.dataframe(df, use_container_width=True)
        
        fig = px.bar(df, x='century', y='count',
                     title='Artifacts by Century')
        st.plotly_chart(fig, use_container_width=True)
    
    elif query_option == "Color Distribution":
        st.subheader("Most Common Colors in Artifacts")
        df = analytics.top_colors()
        
        st.dataframe(df, use_container_width=True)
        
        # Create color visualization
        fig = px.bar(df, x='color_hex', y='frequency',
                     color='color_hex',
                     title='Top 15 Colors by Frequency')
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_dashboard()
```

Run the dashboard:
```bash
streamlit run app.py
```

## Common Patterns

### Pagination for Large Datasets

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=100)
        
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        
        # Check if more pages exist
        info = data.get('info', {})
        if page >= info.get('pages', 0):
            break
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts
```

### Incremental ETL Updates

```python
def incremental_etl(api_key, db_config, last_update_date=None):
    """Run ETL only for new/updated artifacts"""
    etl = HarvardETL(db_config)
    
    # Fetch only recent artifacts
    params = {'apikey': api_key, 'size': 100}
    if last_update_date:
        params['after'] = last_update_date
    
    response = requests.get("https://api.harvardartmuseums.org/object", params=params)
    artifacts = response.json().get('records', [])
    
    etl.run_etl(artifacts)
    
    # Update last sync timestamp
    return pd.Timestamp.now()
```

### Error Handling for API Requests

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch artifacts with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            print(f"Retry {attempt + 1}/{max_retries} after error: {e}")
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting
**Issue:** Getting 429 Too Many Requests errors
**Solution:** Add delays between requests
```python
import time
time.sleep(0.5)  # 500ms delay between requests
```

### Database Connection Issues
**Issue:** Lost connection to MySQL server
**Solution:** Implement connection pooling
```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

conn = db_pool.get_connection()
```

### Large Batch Insert Failures
**Issue:** Memory errors or timeouts with large datasets
**Solution:** Process in smaller batches
```python
def batch_insert(data_list, batch_size=1000):
    for i in range(0, len(data_list), batch_size):
        batch = data_list[i:i + batch_size]
        etl.load_metadata(batch)
```

### Missing or Null Data
**Issue:** KeyError or NULL values in database
**Solution:** Use safe extraction with defaults
```python
def safe_get(data, key, default=''):
    return data.get(key) if data.get(key) is not None else default

metadata = {
    'artifact_id': artifact.get('id'),
    'culture': safe_get(artifact, 'culture'),
    'century': safe_get(artifact, 'century')
}
```
