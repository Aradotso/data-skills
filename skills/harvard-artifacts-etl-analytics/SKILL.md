---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, MySQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - set up analytics dashboard for museum artifact data
  - create a data engineering project with Harvard API
  - extract and transform Harvard museum data
  - build streamlit app for artifact analytics
  - implement SQL analytics for art museum collections
  - design database schema for museum artifacts
  - visualize Harvard Art Museums data with Plotly
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build production-ready ETL pipelines and interactive analytics dashboards using the Harvard Art Museums API. The project demonstrates real-world data engineering patterns including API integration, data transformation, relational database design, SQL analytics, and visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Data Collection**: Fetch artifact metadata, media details, and color information from Harvard Art Museums API
- **ETL Pipeline**: Extract JSON data, transform nested structures into relational tables, load into MySQL/TiDB
- **Database Schema**: Normalized tables (`artifactmetadata`, `artifactmedia`, `artifactcolors`) with proper relationships
- **SQL Analytics**: 20+ predefined analytical queries for insights on cultures, centuries, media, colors
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# MySQL or TiDB Cloud account
# Harvard Art Museums API key (get from https://harvardartmuseums.org/collections/api)
```

### Setup

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

### Requirements File

Typical `requirements.txt`:
```
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.1.0
plotly>=5.17.0
python-dotenv>=1.0.0
```

## Project Structure

```
.
├── app.py                  # Main Streamlit application
├── etl_pipeline.py         # ETL logic for data extraction and transformation
├── database.py             # Database connection and operations
├── queries.py              # Predefined SQL analytics queries
├── config.py               # Configuration management
└── requirements.txt        # Python dependencies
```

## Core Components

### 1. API Integration

```python
import requests
import os
from typing import Dict, List, Optional

class HarvardAPIClient:
    """Client for Harvard Art Museums API"""
    
    def __init__(self, api_key: Optional[str] = None):
        self.api_key = api_key or os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org"
        
    def fetch_objects(self, size: int = 100, page: int = 1) -> Dict:
        """Fetch artifact objects with pagination"""
        url = f"{self.base_url}/object"
        params = {
            'apikey': self.api_key,
            'size': size,
            'page': page
        }
        
        response = requests.get(url, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_all_objects(self, total_records: int = 1000) -> List[Dict]:
        """Fetch multiple pages of objects"""
        all_objects = []
        page = 1
        size = 100
        
        while len(all_objects) < total_records:
            data = self.fetch_objects(size=size, page=page)
            records = data.get('records', [])
            
            if not records:
                break
                
            all_objects.extend(records)
            page += 1
            
        return all_objects[:total_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
from typing import List, Dict, Tuple

class ArtifactETL:
    """ETL pipeline for Harvard Art Museums data"""
    
    def extract(self, api_client: HarvardAPIClient, num_records: int = 1000) -> List[Dict]:
        """Extract data from API"""
        return api_client.fetch_all_objects(total_records=num_records)
    
    def transform(self, raw_data: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """Transform raw JSON into normalized dataframes"""
        
        # Transform artifact metadata
        metadata_records = []
        for obj in raw_data:
            metadata_records.append({
                'objectid': obj.get('objectid'),
                'title': obj.get('title'),
                'culture': obj.get('culture'),
                'period': obj.get('period'),
                'century': obj.get('century'),
                'classification': obj.get('classification'),
                'technique': obj.get('technique'),
                'department': obj.get('department'),
                'division': obj.get('division'),
                'dated': obj.get('dated'),
                'accessionyear': obj.get('accessionyear')
            })
        
        df_metadata = pd.DataFrame(metadata_records)
        
        # Transform media information
        media_records = []
        for obj in raw_data:
            objectid = obj.get('objectid')
            images = obj.get('images', [])
            
            if images:
                media_records.append({
                    'objectid': objectid,
                    'has_images': 1,
                    'image_count': len(images),
                    'primary_image_url': obj.get('primaryimageurl')
                })
            else:
                media_records.append({
                    'objectid': objectid,
                    'has_images': 0,
                    'image_count': 0,
                    'primary_image_url': None
                })
        
        df_media = pd.DataFrame(media_records)
        
        # Transform color information
        color_records = []
        for obj in raw_data:
            objectid = obj.get('objectid')
            colors = obj.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'objectid': objectid,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        
        df_colors = pd.DataFrame(color_records) if color_records else pd.DataFrame()
        
        return df_metadata, df_media, df_colors
    
    def load(self, db_connection, df_metadata: pd.DataFrame, 
             df_media: pd.DataFrame, df_colors: pd.DataFrame):
        """Load dataframes into database"""
        
        cursor = db_connection.cursor()
        
        # Insert metadata
        for _, row in df_metadata.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (objectid, title, culture, period, century, classification, 
                 technique, department, division, dated, accessionyear)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                title=VALUES(title), culture=VALUES(culture)
            """, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (objectid, has_images, image_count, primary_image_url)
                VALUES (%s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                has_images=VALUES(has_images), image_count=VALUES(image_count)
            """, tuple(row))
        
        # Insert colors
        if not df_colors.empty:
            for _, row in df_colors.iterrows():
                cursor.execute("""
                    INSERT INTO artifactcolors 
                    (objectid, color, spectrum, hue, percent)
                    VALUES (%s, %s, %s, %s, %s)
                """, tuple(row))
        
        db_connection.commit()
```

### 3. Database Schema

```python
import mysql.connector
from mysql.connector import Error
import os

class DatabaseManager:
    """Manage database connections and schema"""
    
    def __init__(self):
        self.host = os.getenv('DB_HOST')
        self.user = os.getenv('DB_USER')
        self.password = os.getenv('DB_PASSWORD')
        self.database = os.getenv('DB_NAME')
        self.connection = None
    
    def connect(self):
        """Create database connection"""
        try:
            self.connection = mysql.connector.connect(
                host=self.host,
                user=self.user,
                password=self.password,
                database=self.database
            )
            return self.connection
        except Error as e:
            print(f"Database connection error: {e}")
            raise
    
    def create_tables(self):
        """Create database schema"""
        cursor = self.connection.cursor()
        
        # Artifact metadata table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                objectid INT PRIMARY KEY,
                title VARCHAR(500),
                culture VARCHAR(255),
                period VARCHAR(255),
                century VARCHAR(255),
                classification VARCHAR(255),
                technique VARCHAR(255),
                department VARCHAR(255),
                division VARCHAR(255),
                dated VARCHAR(255),
                accessionyear INT
            )
        """)
        
        # Artifact media table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                objectid INT PRIMARY KEY,
                has_images TINYINT,
                image_count INT,
                primary_image_url TEXT,
                FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
            )
        """)
        
        # Artifact colors table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                id INT AUTO_INCREMENT PRIMARY KEY,
                objectid INT,
                color VARCHAR(50),
                spectrum VARCHAR(50),
                hue VARCHAR(50),
                percent FLOAT,
                FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
            )
        """)
        
        self.connection.commit()
```

### 4. SQL Analytics Queries

```python
class AnalyticsQueries:
    """Predefined analytical queries"""
    
    @staticmethod
    def get_query(query_name: str) -> str:
        queries = {
            'artifacts_by_culture': """
                SELECT culture, COUNT(*) as artifact_count
                FROM artifactmetadata
                WHERE culture IS NOT NULL
                GROUP BY culture
                ORDER BY artifact_count DESC
                LIMIT 10
            """,
            
            'artifacts_by_century': """
                SELECT century, COUNT(*) as artifact_count
                FROM artifactmetadata
                WHERE century IS NOT NULL
                GROUP BY century
                ORDER BY artifact_count DESC
            """,
            
            'media_availability': """
                SELECT 
                    CASE WHEN has_images = 1 THEN 'Has Images' ELSE 'No Images' END as media_status,
                    COUNT(*) as count
                FROM artifactmedia
                GROUP BY has_images
            """,
            
            'top_colors': """
                SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
                FROM artifactcolors
                WHERE color IS NOT NULL
                GROUP BY color
                ORDER BY usage_count DESC
                LIMIT 10
            """,
            
            'artifacts_by_department': """
                SELECT department, COUNT(*) as artifact_count
                FROM artifactmetadata
                WHERE department IS NOT NULL
                GROUP BY department
                ORDER BY artifact_count DESC
            """,
            
            'classification_distribution': """
                SELECT classification, COUNT(*) as count
                FROM artifactmetadata
                WHERE classification IS NOT NULL
                GROUP BY classification
                ORDER BY count DESC
                LIMIT 15
            """
        }
        
        return queries.get(query_name, "")
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Initialize components
    api_client = HarvardAPIClient()
    db_manager = DatabaseManager()
    db_connection = db_manager.connect()
    
    # ETL Pipeline Section
    st.header("📥 ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_records = st.number_input("Number of records to fetch", 
                                       min_value=100, max_value=5000, 
                                       value=1000, step=100)
    
    with col2:
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL pipeline..."):
                etl = ArtifactETL()
                
                # Extract
                st.info("Extracting data from API...")
                raw_data = etl.extract(api_client, num_records)
                
                # Transform
                st.info("Transforming data...")
                df_metadata, df_media, df_colors = etl.transform(raw_data)
                
                # Load
                st.info("Loading data into database...")
                etl.load(db_connection, df_metadata, df_media, df_colors)
                
                st.success(f"✅ Successfully processed {len(raw_data)} artifacts!")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Media Availability": "media_availability",
        "Top Colors": "top_colors",
        "Artifacts by Department": "artifacts_by_department",
        "Classification Distribution": "classification_distribution"
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Analysis"):
        query_key = query_options[selected_query]
        sql_query = AnalyticsQueries.get_query(query_key)
        
        # Execute query
        df_result = pd.read_sql(sql_query, db_connection)
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df_result)
        
        # Visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, 
                        x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Configuration

### Environment Variables

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Config Management

```python
from dotenv import load_dotenv
import os

load_dotenv()

class Config:
    HARVARD_API_KEY = os.getenv('HARVARD_API_KEY')
    DB_HOST = os.getenv('DB_HOST')
    DB_USER = os.getenv('DB_USER')
    DB_PASSWORD = os.getenv('DB_PASSWORD')
    DB_NAME = os.getenv('DB_NAME', 'harvard_artifacts')
    
    # API Settings
    API_BASE_URL = "https://api.harvardartmuseums.org"
    DEFAULT_PAGE_SIZE = 100
    MAX_RETRIES = 3
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Initialize database schema
python database.py --init-schema
```

## Common Patterns

### Batch Processing

```python
def batch_insert(cursor, table_name: str, df: pd.DataFrame, batch_size: int = 1000):
    """Insert dataframe in batches for better performance"""
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch logic
        print(f"Inserted batch {i//batch_size + 1}")
```

### Error Handling

```python
def safe_api_call(func, max_retries: int = 3):
    """Wrapper for API calls with retry logic"""
    import time
    
    for attempt in range(max_retries):
        try:
            return func()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Data Validation

```python
def validate_artifact_data(df: pd.DataFrame) -> pd.DataFrame:
    """Validate and clean artifact data"""
    # Remove duplicates
    df = df.drop_duplicates(subset=['objectid'])
    
    # Handle missing values
    df['title'] = df['title'].fillna('Unknown')
    df['culture'] = df['culture'].fillna('Unknown Culture')
    
    # Data type validation
    df['objectid'] = df['objectid'].astype(int)
    df['accessionyear'] = pd.to_numeric(df['accessionyear'], errors='coerce')
    
    return df
```

## Troubleshooting

**API Rate Limiting**
```python
import time

def fetch_with_rate_limit(api_client, delay: float = 0.5):
    """Add delay between API calls"""
    time.sleep(delay)
    return api_client.fetch_objects()
```

**Database Connection Issues**
```python
def reconnect_db(db_manager, max_attempts: int = 3):
    """Attempt to reconnect to database"""
    for attempt in range(max_attempts):
        try:
            return db_manager.connect()
        except Error as e:
            if attempt == max_attempts - 1:
                raise
            time.sleep(5)
```

**Memory Management for Large Datasets**
```python
def process_in_chunks(data: List[Dict], chunk_size: int = 100):
    """Process large datasets in chunks"""
    for i in range(0, len(data), chunk_size):
        chunk = data[i:i+chunk_size]
        yield chunk
```

This skill equips AI agents to help developers build robust ETL pipelines and analytics dashboards using the Harvard Art Museums API with professional data engineering practices.
