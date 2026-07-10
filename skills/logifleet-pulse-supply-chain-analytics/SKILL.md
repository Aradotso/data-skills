---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine with MS SQL Server data warehousing and Power BI dashboards for supply chain optimization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logistics data warehouse with Power BI"
  - "create multi-fact star schema for fleet and warehouse data"
  - "configure LogiFleet Pulse SQL database"
  - "build supply chain KPI dashboard"
  - "implement warehouse gravity zone analytics"
  - "integrate fleet telemetry with warehouse operations"
  - "set up predictive logistics bottleneck detection"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a single semantic layer using MS SQL Server and Power BI. It implements a multi-fact star schema with time-phased dimensions to enable cross-modal analytics like correlating warehouse dwell time with fleet idle costs.

**Key capabilities:**
- Multi-fact data warehousing (warehouse operations, fleet trips, cross-dock transfers)
- Time-aware dimensional modeling (15-minute granularity)
- Cross-fact KPI harmonization
- Warehouse gravity zone optimization
- Predictive bottleneck detection
- Real-time Power BI dashboards

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telemetry feeds)

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Create the database:**
```sql
-- Connect to your SQL Server instance
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO
```

3. **Deploy the schema:**
```sql
-- Execute the schema creation scripts in order
-- 1. Create dimensions first
-- 2. Create fact tables
-- 3. Create bridge tables
-- 4. Create views and stored procedures
```

### Core Schema Structure

#### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    Hour INT NOT NULL,
    Minute15Block INT NOT NULL, -- 0, 15, 30, 45
    IsWeekend BIT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    FiscalPeriod INT NOT NULL
);

-- Clustered index on primary key
CREATE CLUSTERED INDEX IX_DimTime_TimeKey ON DimTime(TimeKey);

-- Non-clustered index for date queries
CREATE NONCLUSTERED INDEX IX_DimTime_Date ON DimTime(Date);
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Hub, Route Node
    Address VARCHAR(500),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    IsActive BIT NOT NULL DEFAULT 1,
    ValidFrom DATETIME2 NOT NULL,
    ValidTo DATETIME2
);

-- Index for hierarchy queries
CREATE NONCLUSTERED INDEX IX_DimGeography_Hierarchy 
ON DimGeography(Continent, Country, StateProvince, City);
```

**DimProductGravity** - Product dimension with gravity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500) NOT NULL,
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT NOT NULL DEFAULT 0,
    IsPerishable BIT NOT NULL DEFAULT 0,
    ShelfLifeDays INT,
    UnitValue DECIMAL(12,2),
    -- Gravity scoring components
    PickFrequencyScore INT, -- 1-10 based on historical velocity
    ValueScore INT, -- 1-10 based on unit value
    FragilityScore INT, -- 1-10 based on handling requirements
    GravityIndex AS (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(20), -- High, Medium, Low gravity
    LastRecalculated DATETIME2
);

-- Index for gravity-based queries
CREATE NONCLUSTERED INDEX IX_DimProductGravity_GravityIndex 
ON DimProductGravity(GravityIndex DESC);
```

**DimSupplierReliability** - Supplier performance dimension:
```sql
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    SupplierType VARCHAR(50),
    Country VARCHAR(100),
    -- Reliability metrics (updated monthly)
    AvgLeadTimeDays DECIMAL(5,1),
    LeadTimeVarianceDays DECIMAL(5,1),
    OnTimeDeliveryPct DECIMAL(5,2),
    DefectRatePct DECIMAL(5,2),
    ComplianceScore INT, -- 1-100
    ReliabilityRating VARCHAR(20), -- Excellent, Good, Fair, Poor
    LastEvaluated DATETIME2
);
```

#### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID VARCHAR(100),
    BatchID VARCHAR(100),
    -- Measures
    QuantityUnits INT NOT NULL,
    OperationDurationMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(10,2), -- Time item spent in warehouse
    StorageZone VARCHAR(50),
    AssignedGravityZone VARCHAR(20),
    OperatorID VARCHAR(50),
    QualityScore INT, -- 1-100
    Notes VARCHAR(MAX)
);

-- Clustered columnstore for analytical queries
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations 
ON FactWarehouseOperations;

-- Non-clustered indexes for common filters
CREATE NONCLUSTERED INDEX IX_FactWarehouseOperations_TimeKey 
ON FactWarehouseOperations(TimeKey);

CREATE NONCLUSTERED INDEX IX_FactWarehouseOperations_ProductKey 
ON FactWarehouseOperations(ProductKey);
```

**FactFleetTrips** - Fleet and route tracking:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripID VARCHAR(100) NOT NULL UNIQUE,
    DepartureTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ArrivalTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    -- Measures
    PlannedDistanceKm DECIMAL(10,2),
    ActualDistanceKm DECIMAL(10,2),
    PlannedDurationMinutes INT,
    ActualDurationMinutes INT,
    IdleTimeMinutes INT, -- Key KPI
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(12,2),
    LoadValue DECIMAL(15,2),
    OnTimeStatus BIT,
    DelayMinutes INT,
    DelayReason VARCHAR(200),
    -- Telemetry flags
    MaintenanceAlertFlag BIT DEFAULT 0,
    SpeedViolationCount INT DEFAULT 0,
    HarshBrakingCount INT DEFAULT 0
);

-- Columnstore for analytics
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON FactFleetTrips;

-- Indexes for common queries
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_DepartureTime 
ON FactFleetTrips(DepartureTimeKey);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Vehicle 
ON FactFleetTrips(VehicleID);
```

**FactCrossDock** - Cross-dock transfer operations:
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    -- Measures
    QuantityUnits INT NOT NULL,
    DockDwellMinutes INT, -- Time spent in cross-dock area
    TransferComplexity INT, -- 1-5 based on handling requirements
    QualityCheckPassed BIT
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactCrossDock 
ON FactCrossDock;
```

### Bridge Tables for Many-to-Many Relationships

**BridgeRouteProduct** - Links trips to multiple products:
```sql
CREATE TABLE BridgeRouteProduct (
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    QuantityUnits INT NOT NULL,
    WeightKg DECIMAL(10,2),
    VolumeM3 DECIMAL(10,3),
    PRIMARY KEY (TripKey, ProductKey)
);
```

### Key Stored Procedures

**Incremental Data Loading**:
```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OrderID, QuantityUnits, OperationDurationMinutes,
        DwellTimeHours, StorageZone, AssignedGravityZone
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OrderID,
        s.QuantityUnits,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS OperationDurationMinutes,
        CASE 
            WHEN s.OperationType = 'Shipping' 
            THEN DATEDIFF(HOUR, s.ReceiveTime, s.EndTime)
            ELSE NULL
        END AS DwellTimeHours,
        s.StorageZone,
        p.RecommendedZone AS AssignedGravityZone
    FROM StagingWarehouseOps s
    INNER JOIN DimTime t ON CAST(s.StartTime AS DATE) = t.Date 
        AND DATEPART(HOUR, s.StartTime) = t.Hour
        AND (DATEPART(MINUTE, s.StartTime) / 15) * 15 = t.Minute15Block
    INNER JOIN DimGeography g ON s.WarehouseID = g.LocationID
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE s.StartTime >= @StartDateTime 
        AND s.StartTime < @EndDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.OrderID = s.OrderID 
                AND f.OperationType = s.OperationType
        );
    
    SELECT @@ROWCOUNT AS RowsInserted;
END;
GO
```

**Gravity Score Recalculation**:
```sql
CREATE PROCEDURE usp_RecalculateProductGravity
    @LookbackDays INT = 90
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on recent activity
    UPDATE p
    SET 
        PickFrequencyScore = CASE 
            WHEN stats.PicksPerDay >= 50 THEN 10
            WHEN stats.PicksPerDay >= 30 THEN 8
            WHEN stats.PicksPerDay >= 15 THEN 6
            WHEN stats.PicksPerDay >= 5 THEN 4
            WHEN stats.PicksPerDay >= 1 THEN 2
            ELSE 1
        END,
        ValueScore = CASE 
            WHEN p.UnitValue >= 1000 THEN 10
            WHEN p.UnitValue >= 500 THEN 8
            WHEN p.UnitValue >= 200 THEN 6
            WHEN p.UnitValue >= 50 THEN 4
            ELSE 2
        END,
        FragilityScore = CASE 
            WHEN p.IsFragile = 1 OR p.IsPerishable = 1 THEN 10
            ELSE 1
        END,
        RecommendedZone = CASE 
            WHEN (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) >= 7 THEN 'High'
            WHEN (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) >= 4 THEN 'Medium'
            ELSE 'Low'
        END,
        LastRecalculated = GETDATE()
    FROM DimProductGravity p
    LEFT JOIN (
        SELECT 
            ProductKey,
            COUNT(*) / @LookbackDays AS PicksPerDay
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE Date >= DATEADD(DAY, -@LookbackDays, GETDATE()))
        GROUP BY ProductKey
    ) stats ON p.ProductKey = stats.ProductKey;
    
    SELECT @@ROWCOUNT AS ProductsUpdated;
END;
GO
```

**Cross-Fact KPI Query**:
```sql
CREATE PROCEDURE usp_GetCrossFactKPIs
    @StartDate DATE,
    @EndDate DATE,
    @WarehouseID VARCHAR(50) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Correlate warehouse dwell time with fleet idle time
    SELECT 
        t.Date,
        g.LocationName AS Warehouse,
        p.Category AS ProductCategory,
        -- Warehouse metrics
        AVG(w.DwellTimeHours) AS AvgDwellTimeHours,
        SUM(w.QuantityUnits) AS TotalUnitsProcessed,
        -- Fleet metrics from same products
        AVG(f.IdleTimeMinutes) AS AvgFleetIdleMinutes,
        AVG(f.ActualDurationMinutes - f.PlannedDurationMinutes) AS AvgDelayMinutes,
        -- Composite KPI: Dwell-to-Idle Correlation Score
        CASE 
            WHEN AVG(w.DwellTimeHours) > 48 AND AVG(f.IdleTimeMinutes) > 60 
            THEN 'High Risk'
            WHEN AVG(w.DwellTimeHours) > 24 OR AVG(f.IdleTimeMinutes) > 30 
            THEN 'Moderate Risk'
            ELSE 'Normal'
        END AS RiskLevel
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    LEFT JOIN BridgeRouteProduct brp ON w.ProductKey = brp.ProductKey
    LEFT JOIN FactFleetTrips f ON brp.TripKey = f.TripKey
        AND f.OriginGeographyKey = w.GeographyKey
    WHERE t.Date BETWEEN @StartDate AND @EndDate
        AND (@WarehouseID IS NULL OR g.LocationID = @WarehouseID)
    GROUP BY t.Date, g.LocationName, p.Category
    ORDER BY t.Date, RiskLevel DESC;
END;
GO
```

## Power BI Configuration

### Connecting to SQL Server

1. **Open Power BI Desktop**
2. **Get Data > SQL Server**
3. **Connection settings:**
```
Server: your-sql-server.database.windows.net
Database: LogiFleetPulse
Data Connectivity mode: Import (for smaller datasets) or DirectQuery (for real-time)
```

4. **Import tables:**
```
- All Dim* tables (Dimensions)
- All Fact* tables (Fact tables)
- All Bridge* tables (Bridge tables)
```

### Model Relationships Setup

```dax
// Relationships are typically auto-detected, but verify:
// DimTime[TimeKey] -> FactWarehouseOperations[TimeKey] (Many-to-One)
// DimTime[TimeKey] -> FactFleetTrips[DepartureTimeKey] (Many-to-One)
// DimGeography[GeographyKey] -> FactWarehouseOperations[GeographyKey] (Many-to-One)
// DimProductGravity[ProductKey] -> FactWarehouseOperations[ProductKey] (Many-to-One)
// FactFleetTrips[TripKey] -> BridgeRouteProduct[TripKey] (One-to-Many)
// DimProductGravity[ProductKey] -> BridgeRouteProduct[ProductKey] (One-to-Many)
```

### Key DAX Measures

**Average Dwell Time**:
```dax
Avg Dwell Time (Hours) = 
AVERAGE(FactWarehouseOperations[DwellTimeHours])
```

**Fleet Utilization Rate**:
```dax
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[ActualDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[ActualDurationMinutes]),
    0
) * 100
```

**Cross-Fact Efficiency Score**:
```dax
Efficiency Score = 
VAR WarehouseScore = 
    DIVIDE(
        COUNTROWS(FILTER(FactWarehouseOperations, FactWarehouseOperations[DwellTimeHours] <= 24)),
        COUNTROWS(FactWarehouseOperations)
    ) * 50

VAR FleetScore = 
    DIVIDE(
        COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[OnTimeStatus] = TRUE())),
        COUNTROWS(FactFleetTrips)
    ) * 50

RETURN WarehouseScore + FleetScore
```

**Gravity Zone Compliance**:
```dax
Gravity Compliance % = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR CompliantOps = 
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[StorageZone] = FactWarehouseOperations[AssignedGravityZone]
        )
    )
RETURN DIVIDE(CompliantOps, TotalOps, 0) * 100
```

**Time Intelligence - Dwell Time YoY**:
```dax
Dwell Time YoY Change % = 
VAR CurrentPeriod = [Avg Dwell Time (Hours)]
VAR PreviousYear = 
    CALCULATE(
        [Avg Dwell Time (Hours)],
        DATEADD(DimTime[Date], -1, YEAR)
    )
RETURN 
    DIVIDE(CurrentPeriod - PreviousYear, PreviousYear, 0) * 100
```

**Bottleneck Probability Index**:
```dax
Bottleneck Index = 
VAR DwellScore = 
    IF([Avg Dwell Time (Hours)] > 48, 3, 
       IF([Avg Dwell Time (Hours)] > 24, 2, 1))
VAR IdleScore = 
    IF(AVERAGE(FactFleetTrips[IdleTimeMinutes]) > 60, 3,
       IF(AVERAGE(FactFleetTrips[IdleTimeMinutes]) > 30, 2, 1))
VAR DelayScore = 
    IF(AVERAGE(FactFleetTrips[DelayMinutes]) > 30, 3,
       IF(AVERAGE(FactFleetTrips[DelayMinutes]) > 15, 2, 1))

RETURN (DwellScore + IdleScore + DelayScore) / 3
```

### Dashboard Best Practices

**Performance Optimization:**
```dax
// Use variables to avoid recalculating sub-expressions
Optimized Metric = 
VAR BaseValue = SUM(FactWarehouseOperations[QuantityUnits])
VAR TargetValue = 10000
VAR Variance = BaseValue - TargetValue
RETURN 
    IF(Variance >= 0, "On Track", "Below Target")

// Use SUMMARIZE for complex aggregations
Summarized Data = 
SUMMARIZE(
    FactWarehouseOperations,
    DimTime[Date],
    DimProductGravity[Category],
    "Total Units", SUM(FactWarehouseOperations[QuantityUnits]),
    "Avg Dwell", AVERAGE(FactWarehouseOperations[DwellTimeHours])
)
```

## Data Integration Patterns

### ETL from WMS (Warehouse Management System)

**Python script for CSV ingestion:**
```python
import pyodbc
import pandas as pd
import os
from datetime import datetime

# Connection string (use environment variables)
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.environ['SQL_SERVER']};"
    f"DATABASE={os.environ['SQL_DATABASE']};"
    f"UID={os.environ['SQL_USER']};"
    f"PWD={os.environ['SQL_PASSWORD']}"
)

def load_warehouse_data(csv_path):
    """Load warehouse operations from CSV into staging table"""
    try:
        # Read CSV
        df = pd.read_csv(csv_path)
        
        # Data transformations
        df['StartTime'] = pd.to_datetime(df['StartTime'])
        df['EndTime'] = pd.to_datetime(df['EndTime'])
        df['ReceiveTime'] = pd.to_datetime(df['ReceiveTime'])
        
        # Connect to SQL Server
        conn = pyodbc.connect(conn_str)
        cursor = conn.cursor()
        
        # Truncate staging table
        cursor.execute("TRUNCATE TABLE StagingWarehouseOps")
        
        # Bulk insert
        for _, row in df.iterrows():
            cursor.execute("""
                INSERT INTO StagingWarehouseOps (
                    OperationType, OrderID, SKU, WarehouseID,
                    QuantityUnits, StartTime, EndTime, ReceiveTime,
                    StorageZone
                ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, 
            row['OperationType'], row['OrderID'], row['SKU'], 
            row['WarehouseID'], row['QuantityUnits'], row['StartTime'],
            row['EndTime'], row['ReceiveTime'], row['StorageZone'])
        
        conn.commit()
        
        # Call stored procedure to move to fact table
        start_time = df['StartTime'].min()
        end_time = df['EndTime'].max()
        cursor.execute(
            "EXEC usp_LoadWarehouseOperations ?, ?",
            start_time, end_time
        )
        result = cursor.fetchone()
        
        print(f"Loaded {result[0]} rows into FactWarehouseOperations")
        
        cursor.close()
        conn.close()
        
    except Exception as e:
        print(f"Error loading data: {str(e)}")
        raise

# Usage
load_warehouse_data('/data/warehouse_ops_2026_07_10.csv')
```

### Real-Time Telemetry Ingestion (REST API)

**Python script for fleet telemetry:**
```python
import requests
import pyodbc
import os
from datetime import datetime, timedelta

API_ENDPOINT = os.environ['TELEMETRY_API_ENDPOINT']
API_KEY = os.environ['TELEMETRY_API_KEY']

def fetch_and_load_telemetry():
    """Fetch telemetry data and load into FactFleetTrips"""
    
    # Get trips from last 24 hours
    start_time = (datetime.now() - timedelta(hours=24)).isoformat()
    
    headers = {
        'Authorization': f'Bearer {API_KEY}',
        'Content-Type': 'application/json'
    }
    
    params = {
        'start_time': start_time,
        'status': 'completed'
    }
    
    # Fetch from API
    response = requests.get(
        f"{API_ENDPOINT}/trips", 
        headers=headers, 
        params=params
    )
    response.raise_for_status()
    trips = response.json()['data']
    
    # Connect to database
    conn_str = (
        f"DRIVER={{ODBC Driver 17 for SQL Server}};"
        f"SERVER={os.environ['SQL_SERVER']};"
        f"DATABASE={os.environ['SQL_DATABASE']};"
        f"UID={os.environ['SQL_USER']};"
        f"PWD={os.environ['SQL_PASSWORD']}"
    )
    conn = pyodbc.connect(conn_str)
    cursor = conn.cursor()
    
    for trip in trips:
        # Get time keys
        cursor.execute("""
            SELECT TimeKey FROM DimTime 
            WHERE FullDateTime = ?
        """, trip['departure_time'])
        departure_key = cursor.fetchone()[0]
        
        cursor.execute("""
            SELECT TimeKey FROM DimTime 
            WHERE FullDateTime = ?
        """, trip['arrival_time'])
        arrival_key = cursor.fetchone()[0]
        
        # Get geography keys
        cursor.execute("""
            SELECT GeographyKey FROM DimGeography 
            WHERE LocationID = ?
        """, trip['origin_id'])
        origin_key = cursor.fetchone()[0]
        
        cursor.execute("""
            SELECT GeographyKey FROM DimGeography 
            WHERE LocationID = ?
        """, trip['destination_id'])
        dest_key = cursor.fetchone()[0]
        
        # Insert trip
        cursor.execute("""
            INSERT INTO FactFleetTrips (
                TripID, DepartureTimeKey, ArrivalTimeKey,
                OriginGeographyKey, DestinationGeographyKey,
                VehicleID, DriverID, ActualDistanceKm,
                ActualDurationMinutes, IdleTimeMinutes,
                FuelConsumedLiters, LoadWeightKg, OnTimeStatus,
                DelayMinutes, DelayReason
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """,
        trip['trip_id'], departure_key, arrival_key,
        origin_key, dest_key, trip['vehicle_id'], trip['driver_id'],
        trip['actual_distance_km'], trip['actual_duration_minutes'],
        trip['idle_time_minutes'], trip['fuel_consumed_liters'],
        trip['load_weight_kg'], trip['on_time'],
        trip['delay_minutes'], trip.get('delay_reason', '')
        )
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(trips)} trips into FactFleetTrips")

# Run as scheduled job
if __name__ == "__main__":
    fetch_and_load_telemetry()
```

## Common Analysis Patterns

### Pattern 1: Warehouse Gravity Zone Analysis

```sql
-- Identify products in wrong gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityIndex,
    p.RecommendedZone,
    w.StorageZone AS CurrentZone,
    COUNT(*) AS OperationCount,
    AVG(w.OperationDurationMinutes) AS AvgPickTimeMinutes
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE w.OperationType = 'Picking'
    AND w.StorageZone != p.RecommendedZone
    AND w.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE Date >= DATEADD(DAY, -30, GETDATE()))
GROUP BY p.SKU, p.ProductName, p.GravityIndex, p.RecommendedZone, w.StorageZone
HAVING COUNT(*) > 10
ORDER BY COUNT(*) DESC;
```

### Pattern 2: Fleet Idle Time Correlation

```sql
-- Correlate fleet idle time with origin warehouse dwell time
WITH WarehouseDwell AS (
    SELECT 
        w.GeographyKey,
        AVG(w.DwellTimeHours) AS 
