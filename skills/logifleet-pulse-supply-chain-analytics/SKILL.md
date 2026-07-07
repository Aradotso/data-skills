---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform using MS SQL Server and Power BI for real-time supply chain orchestration and fleet optimization
triggers:
  - "set up LogiFleet Pulse analytics platform"
  - "deploy supply chain data warehouse with Power BI"
  - "configure multi-fact star schema for logistics"
  - "create warehouse gravity zone optimization"
  - "implement fleet telemetry tracking dashboard"
  - "build cross-modal supply chain KPI reports"
  - "analyze warehouse dwell time and fleet routing"
  - "integrate logistics data sources into unified model"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence engine that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic data model. Built on MS SQL Server with Power BI visualization, it enables:

- Multi-fact star schema data warehousing for logistics
- Real-time cross-modal KPI harmonization (warehouse + fleet + supplier)
- Predictive bottleneck detection and fleet triage
- Warehouse gravity zone optimization based on velocity and value
- Time-phased dimensional modeling with 15-minute granularity
- Role-based dashboards with natural language querying

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Admin access to create databases and users

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema creation script
-- Run the SQL files in this order:
-- 1. create_dimensions.sql
-- 2. create_facts.sql
-- 3. create_views.sql
-- 4. create_stored_procedures.sql
```

### Step 3: Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}",
    "port": 1433
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telemetry_feed": "${FLEET_TELEMETRY_URL}",
    "weather_api": "${WEATHER_API_URL}",
    "erp_connection": "${ERP_CONNECTION_STRING}"
  },
  "refresh_interval_minutes": 15
}
```

### Step 4: Open Power BI Template

```bash
# Open the .pbit template file
# File location: /dashboards/LogiFleet_Pulse_Master.pbit
# Connect to your SQL Server when prompted
# Server: ${SQL_SERVER_HOST}
# Database: LogiFleetPulse
# Authentication: Windows or SQL Server
```

## Core Data Model Structure

### Dimension Tables

```sql
-- DimTime: Time-phased dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    [Year] INT NOT NULL,
    [Quarter] INT NOT NULL,
    [Month] INT NOT NULL,
    [Day] INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    DayOfWeek VARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    INDEX IX_DimTime_DateTime (FullDateTime)
);

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite: velocity + value + fragility
    VelocityClass VARCHAR(20), -- Fast/Medium/Slow mover
    ValueTier VARCHAR(20), -- High/Medium/Low value
    FragilityIndex DECIMAL(3,2),
    OptimalZone VARCHAR(50), -- Calculated optimal storage zone
    LastRecalculated DATETIME2,
    INDEX IX_ProductGravity_Score (GravityScore DESC)
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, Distribution Center, Route Node
    ParentLocationID VARCHAR(50),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    INDEX IX_Geography_Type (LocationType)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    LeadTimeMean INT, -- Days
    LeadTimeVariance DECIMAL(8,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- Platinum/Gold/Silver/Bronze
    LastEvaluated DATETIME2,
    INDEX IX_Supplier_Reliability (ReliabilityTier, ComplianceScore DESC)
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT,
    DwellTimeMinutes INT, -- Time in storage before movement
    CycleTimeSeconds INT, -- Processing time for operation
    ZoneID VARCHAR(50),
    OperatorID VARCHAR(50),
    ErrorFlag BIT,
    INDEX IX_FactWarehouse_Time (TimeKey),
    INDEX IX_FactWarehouse_Product (ProductKey),
    INDEX IX_FactWarehouse_Operation (OperationType, TimeKey)
);

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID VARCHAR(50) NOT NULL,
    RouteID VARCHAR(50),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripDistanceKm DECIMAL(10,2),
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    OnTimeFlag BIT,
    DelayReasonCode VARCHAR(50), -- Weather, Traffic, Mechanical, Other
    MaintenanceAlertFlag BIT,
    INDEX IX_FactFleet_Time (TimeKey),
    INDEX IX_FactFleet_Vehicle (VehicleID, TimeKey),
    INDEX IX_FactFleet_Route (RouteID, TimeKey)
);

-- FactCrossDock: Transfer operations between inbound and outbound
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TransferTimeMinutes INT,
    Quantity INT,
    DockBayID VARCHAR(50),
    INDEX IX_FactCrossDock_Time (TimeKey),
    INDEX IX_FactCrossDock_Product (ProductKey)
);
```

## Key Stored Procedures

### Incremental Data Load

```sql
-- Stored procedure for incremental warehouse data load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LoadFromDateTime DATETIME2,
    @LoadToDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        Quantity, DwellTimeMinutes, CycleTimeSeconds,
        ZoneID, OperatorID, ErrorFlag
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.OperationType,
        s.Quantity,
        s.DwellTimeMinutes,
        s.CycleTimeSeconds,
        s.ZoneID,
        s.OperatorID,
        s.ErrorFlag
    FROM StagingWarehouseOps s
    INNER JOIN DimTime t ON CAST(s.OperationDateTime AS DATETIME2) = t.FullDateTime
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    INNER JOIN DimGeography g ON s.LocationID = g.LocationID
    WHERE s.OperationDateTime >= @LoadFromDateTime
      AND s.OperationDateTime < @LoadToDateTime
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations f
          WHERE f.TimeKey = t.TimeKey
            AND f.ProductKey = p.ProductKey
            AND f.OperationType = s.OperationType
            AND f.OperatorID = s.OperatorID
      );
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' warehouse operations';
END;
GO
```

### Gravity Zone Recalculation

```sql
-- Calculate and update product gravity scores
CREATE PROCEDURE usp_RecalculateProductGravity
    @DaysToAnalyze INT = 90
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AnalysisStartDate DATETIME2 = DATEADD(DAY, -@DaysToAnalyze, GETDATE());
    
    -- Update gravity scores based on recent activity
    UPDATE p
    SET 
        p.GravityScore = (
            -- Velocity component (40% weight)
            (ISNULL(metrics.PicksPerDay, 0) * 0.4) +
            -- Value component (35% weight)
            (CASE 
                WHEN p.ValueTier = 'High' THEN 35
                WHEN p.ValueTier = 'Medium' THEN 20
                ELSE 10
            END) +
            -- Fragility component (25% weight, inverted)
            ((1 - ISNULL(p.FragilityIndex, 0.5)) * 25)
        ),
        p.VelocityClass = CASE
            WHEN ISNULL(metrics.PicksPerDay, 0) >= 50 THEN 'Fast'
            WHEN ISNULL(metrics.PicksPerDay, 0) >= 10 THEN 'Medium'
            ELSE 'Slow'
        END,
        p.OptimalZone = CASE
            WHEN (ISNULL(metrics.PicksPerDay, 0) * 0.4) + 
                 (CASE WHEN p.ValueTier = 'High' THEN 35 ELSE 10 END) >= 50 
            THEN 'High-Gravity-Zone-A'
            WHEN (ISNULL(metrics.PicksPerDay, 0) * 0.4) + 
                 (CASE WHEN p.ValueTier = 'High' THEN 35 ELSE 10 END) >= 25 
            THEN 'Medium-Gravity-Zone-B'
            ELSE 'Low-Gravity-Zone-C'
        END,
        p.LastRecalculated = GETDATE()
    FROM DimProductGravity p
    LEFT JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / @DaysToAnalyze AS PicksPerDay
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
          AND TimeKey IN (
              SELECT TimeKey FROM DimTime 
              WHERE FullDateTime >= @AnalysisStartDate
          )
        GROUP BY ProductKey
    ) metrics ON p.ProductKey = metrics.ProductKey;
    
    PRINT 'Recalculated gravity for ' + CAST(@@ROWCOUNT AS VARCHAR) + ' products';
END;
GO
```

## Cross-Fact KPI Queries

### Dwell Time vs Fleet Efficiency

```sql
-- Analyze relationship between warehouse dwell and fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        dt.FullDateTime,
        dg.Region,
        dp.Category,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(fwo.Quantity) AS TotalQuantity
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE dt.FullDateTime >= DATEADD(DAY, -30, GETDATE())
      AND fwo.OperationType = 'Shipping'
    GROUP BY dt.FullDateTime, dg.Region, dp.Category
),
FleetPerformance AS (
    SELECT 
        dt.FullDateTime,
        dg.Region,
        AVG(fft.IdleTimeMinutes * 1.0 / NULLIF(fft.TripDurationMinutes, 0)) AS AvgIdleRatio,
        AVG(fft.FuelConsumedLiters) AS AvgFuelConsumption,
        SUM(CASE WHEN fft.OnTimeFlag = 0 THEN 1 ELSE 0 END) AS DelayedTrips
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fft.OriginGeographyKey = dg.GeographyKey
    WHERE dt.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY dt.FullDateTime, dg.Region
)
SELECT 
    wd.Region,
    wd.Category,
    AVG(wd.AvgDwellMinutes) AS AvgWarehouseDwell,
    AVG(fp.AvgIdleRatio) * 100 AS AvgFleetIdlePercent,
    AVG(fp.AvgFuelConsumption) AS AvgFuelPerTrip,
    SUM(fp.DelayedTrips) AS TotalDelays,
    -- Correlation metric: higher dwell + higher idle = potential bottleneck
    (AVG(wd.AvgDwellMinutes) * AVG(fp.AvgIdleRatio)) AS BottleneckIndex
FROM WarehouseDwell wd
INNER JOIN FleetPerformance fp 
    ON wd.FullDateTime = fp.FullDateTime 
    AND wd.Region = fp.Region
GROUP BY wd.Region, wd.Category
ORDER BY BottleneckIndex DESC;
```

### Predictive Maintenance Priority Queue

```sql
-- Generate fleet maintenance priority based on telemetry and cargo value
SELECT TOP 20
    fft.VehicleID,
    COUNT(*) AS TripsLast7Days,
    AVG(fft.IdleTimeMinutes * 1.0 / NULLIF(fft.TripDurationMinutes, 0)) AS AvgIdleRatio,
    SUM(fft.FuelConsumedLiters) AS TotalFuel,
    SUM(fft.TripDistanceKm) AS TotalDistance,
    SUM(CASE WHEN fft.MaintenanceAlertFlag = 1 THEN 1 ELSE 0 END) AS AlertCount,
    -- Calculate revenue at risk (approximate via high-value cargo correlation)
    SUM(CASE 
        WHEN dp.ValueTier = 'High' AND fcd.Quantity IS NOT NULL 
        THEN fcd.Quantity * 100 -- Placeholder: $100 per high-value unit
        ELSE 0 
    END) AS EstimatedCargoValueAtRisk,
    -- Priority score: alerts * idle ratio * cargo value
    (SUM(CASE WHEN fft.MaintenanceAlertFlag = 1 THEN 1 ELSE 0 END) * 
     AVG(fft.IdleTimeMinutes * 1.0 / NULLIF(fft.TripDurationMinutes, 0)) * 
     SUM(CASE WHEN dp.ValueTier = 'High' AND fcd.Quantity IS NOT NULL 
              THEN fcd.Quantity * 100 ELSE 0 END)) AS MaintenancePriorityScore
FROM FactFleetTrips fft
LEFT JOIN FactCrossDock fcd ON fft.TripKey = fcd.OutboundTripKey
LEFT JOIN DimProductGravity dp ON fcd.ProductKey = dp.ProductKey
INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
WHERE dt.FullDateTime >= DATEADD(DAY, -7, GETDATE())
GROUP BY fft.VehicleID
HAVING SUM(CASE WHEN fft.MaintenanceAlertFlag = 1 THEN 1 ELSE 0 END) > 0
ORDER BY MaintenancePriorityScore DESC;
```

## Power BI DAX Measures

### Cross-Fact Composite KPIs

```dax
// Measure: Average Dwell Time (Hours)
Avg Dwell Time (Hours) = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60

// Measure: Fleet Idle Time Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// Measure: On-Time Delivery Rate
On-Time Delivery Rate = 
DIVIDE(
    CALCULATE(
        COUNT(FactFleetTrips[TripKey]),
        FactFleetTrips[OnTimeFlag] = TRUE()
    ),
    COUNT(FactFleetTrips[TripKey]),
    0
) * 100

// Measure: Warehouse Efficiency Score (Composite)
Warehouse Efficiency Score = 
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeSeconds])
VAR ErrorRate = DIVIDE(
    CALCULATE(COUNT(FactWarehouseOperations[OperationKey]), 
              FactWarehouseOperations[ErrorFlag] = TRUE()),
    COUNT(FactWarehouseOperations[OperationKey])
)
VAR DwellScore = 1 - (AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 240)
RETURN
    (DwellScore * 0.5 + (1 - ErrorRate) * 0.3 + (1 - AvgCycleTime/300) * 0.2) * 100

// Measure: Gravity Zone Compliance Rate
Gravity Zone Compliance = 
VAR ProductsInOptimalZone = 
    CALCULATE(
        DISTINCTCOUNT(FactWarehouseOperations[ProductKey]),
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[ZoneID] = 
            RELATED(DimProductGravity[OptimalZone])
        )
    )
VAR TotalProducts = DISTINCTCOUNT(FactWarehouseOperations[ProductKey])
RETURN DIVIDE(ProductsInOptimalZone, TotalProducts, 0) * 100
```

## Configuration Examples

### Scheduled Data Refresh (SQL Agent Job)

```sql
-- Create SQL Agent job for 15-minute refresh cycle
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_IncrementalRefresh',
    @enabled = 1,
    @description = N'Incremental data load every 15 minutes';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalRefresh',
    @step_name = N'Load Warehouse Operations',
    @subsystem = N'TSQL',
    @database_name = N'LogiFleetPulse',
    @command = N'
        DECLARE @LoadFrom DATETIME2 = DATEADD(MINUTE, -15, GETDATE());
        DECLARE @LoadTo DATETIME2 = GETDATE();
        EXEC usp_LoadWarehouseOperations @LoadFrom, @LoadTo;
    ';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalRefresh',
    @step_name = N'Load Fleet Trips',
    @subsystem = N'TSQL',
    @database_name = N'LogiFleetPulse',
    @command = N'
        DECLARE @LoadFrom DATETIME2 = DATEADD(MINUTE, -15, GETDATE());
        DECLARE @LoadTo DATETIME2 = GETDATE();
        EXEC usp_LoadFleetTrips @LoadFrom, @LoadTo;
    ';

-- Schedule to run every 15 minutes
EXEC sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalRefresh',
    @schedule_name = N'Every15Minutes';

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_IncrementalRefresh';
GO
```

### Alert Threshold Configuration

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    ThresholdID INT PRIMARY KEY IDENTITY(1,1),
    MetricName VARCHAR(100) NOT NULL,
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    NotificationChannel VARCHAR(50), -- Email, SMS, Teams
    IsActive BIT DEFAULT 1
);

-- Insert common thresholds
INSERT INTO AlertThresholds (MetricName, ThresholdValue, ComparisonOperator, NotificationChannel)
VALUES 
    ('Fleet_Idle_Percentage', 15.0, '>', 'Email'),
    ('Warehouse_Dwell_Hours', 72.0, '>', 'Teams'),
    ('On_Time_Delivery_Rate', 95.0, '<', 'SMS'),
    ('Gravity_Zone_Compliance', 80.0, '<', 'Email'),
    ('Fuel_Consumption_Variance', 20.0, '>', 'Email');

-- Stored procedure to check thresholds and send alerts
CREATE PROCEDURE usp_CheckAlertsAndNotify
AS
BEGIN
    -- Check fleet idle time
    DECLARE @CurrentFleetIdle DECIMAL(5,2) = (
        SELECT AVG(IdleTimeMinutes * 100.0 / NULLIF(TripDurationMinutes, 0))
        FROM FactFleetTrips fft
        INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
        WHERE dt.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
    );
    
    IF @CurrentFleetIdle > (SELECT ThresholdValue FROM AlertThresholds 
                            WHERE MetricName = 'Fleet_Idle_Percentage' AND IsActive = 1)
    BEGIN
        -- Log alert (implement notification via sp_send_dbmail or external API)
        INSERT INTO AlertLog (AlertTime, MetricName, CurrentValue, ThresholdValue)
        VALUES (GETDATE(), 'Fleet_Idle_Percentage', @CurrentFleetIdle, 
                (SELECT ThresholdValue FROM AlertThresholds 
                 WHERE MetricName = 'Fleet_Idle_Percentage'));
    END;
    
    -- Additional threshold checks...
END;
GO
```

## Common Integration Patterns

### Connecting External APIs (Python)

```python
import pyodbc
import requests
import json
from datetime import datetime, timedelta
import os

# SQL Server connection
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_USER')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)

def ingest_telemetry_data():
    """Fetch fleet telemetry from API and load into staging"""
    
    # Fetch from telemetry API
    api_url = os.getenv('FLEET_TELEMETRY_URL')
    api_key = os.getenv('FLEET_API_KEY')
    
    response = requests.get(
        f"{api_url}/telemetry/recent",
        headers={"Authorization": f"Bearer {api_key}"},
        params={"minutes": 15}
    )
    
    telemetry_data = response.json()
    
    # Connect to SQL Server
    conn = pyodbc.connect(conn_str)
    cursor = conn.cursor()
    
    # Insert into staging table
    insert_query = """
        INSERT INTO StagingFleetTelemetry 
        (VehicleID, TripDateTime, RouteID, DistanceKm, DurationMinutes, 
         IdleMinutes, FuelLiters, LoadKg, OnTimeFlag, DelayReason)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """
    
    for record in telemetry_data.get('trips', []):
        cursor.execute(insert_query, (
            record['vehicle_id'],
            datetime.fromisoformat(record['timestamp']),
            record['route_id'],
            record['distance_km'],
            record['duration_minutes'],
            record['idle_minutes'],
            record['fuel_liters'],
            record['load_kg'],
            1 if record['on_time'] else 0,
            record.get('delay_reason', None)
        ))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Ingested {len(telemetry_data.get('trips', []))} telemetry records")

def enrich_weather_delays():
    """Correlate weather data with delayed trips"""
    
    weather_api = os.getenv('WEATHER_API_URL')
    weather_key = os.getenv('WEATHER_API_KEY')
    
    conn = pyodbc.connect(conn_str)
    cursor = conn.cursor()
    
    # Get recent delayed trips
    cursor.execute("""
        SELECT TripKey, OriginGeographyKey, TimeKey
        FROM FactFleetTrips
        WHERE OnTimeFlag = 0 
          AND DelayReasonCode IS NULL
          AND TimeKey IN (
              SELECT TimeKey FROM DimTime 
              WHERE FullDateTime >= DATEADD(HOUR, -2, GETDATE())
          )
    """)
    
    delayed_trips = cursor.fetchall()
    
    for trip in delayed_trips:
        trip_key, geo_key, time_key = trip
        
        # Get location coordinates
        cursor.execute("""
            SELECT Latitude, Longitude 
            FROM DimGeography WHERE GeographyKey = ?
        """, geo_key)
        lat, lon = cursor.fetchone()
        
        # Get timestamp
        cursor.execute("""
            SELECT FullDateTime FROM DimTime WHERE TimeKey = ?
        """, time_key)
        trip_time = cursor.fetchone()[0]
        
        # Fetch weather data
        weather_response = requests.get(
            f"{weather_api}/historical",
            params={
                "lat": lat,
                "lon": lon,
                "dt": int(trip_time.timestamp()),
                "appid": weather_key
            }
        )
        
        weather = weather_response.json()
        
        # Check for adverse conditions
        if 'rain' in weather or 'snow' in weather or weather.get('wind_speed', 0) > 50:
            cursor.execute("""
                UPDATE FactFleetTrips 
                SET DelayReasonCode = 'Weather'
                WHERE TripKey = ?
            """, trip_key)
    
    conn.commit()
    cursor.close()
    conn.close()

if __name__ == "__main__":
    ingest_telemetry_data()
    enrich_weather_delays()
```

### Power BI Report Automation (PowerShell)

```powershell
# Refresh Power BI dataset via REST API

$TenantId = $env:POWERBI_TENANT_ID
$ClientId = $env:POWERBI_CLIENT_ID
$ClientSecret = $env:POWERBI_CLIENT_SECRET
$WorkspaceId = $env:POWERBI_WORKSPACE_ID
$DatasetId = $env:POWERBI_DATASET_ID

# Get access token
$TokenUrl = "https://login.microsoftonline.com/$TenantId/oauth2/v2.0/token"
$TokenBody = @{
    client_id     = $ClientId
    scope         = "https://analysis.windows.net/powerbi/api/.default"
    client_secret = $ClientSecret
    grant_type    = "client_credentials"
}

$TokenResponse = Invoke-RestMethod -Method Post -Uri $TokenUrl -Body $TokenBody
$AccessToken = $TokenResponse.access_token

# Trigger dataset refresh
$RefreshUrl = "https://api.powerbi.com/v1.0/myorg/groups/$WorkspaceId/datasets/$DatasetId/refreshes"
$Headers = @{
    "Authorization" = "Bearer $AccessToken"
    "Content-Type"  = "application/json"
}

$RefreshBody = @{
    
