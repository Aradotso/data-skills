---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for real-time logistics intelligence and cross-modal supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - configure power bi logistics dashboard
  - create warehouse fleet data model
  - implement supply chain star schema
  - build logistics intelligence warehouse
  - deploy logifleet pulse sql schema
  - integrate warehouse and fleet analytics
  - setup cross-modal supply chain reporting
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to implement and configure LogiFleet Pulse, an advanced MS SQL Server and Power BI-based data warehouse for real-time logistics intelligence, warehouse operations, fleet management, and cross-modal supply chain analytics.

## What LogiFleet Pulse Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that:
- Integrates warehouse operations, fleet telemetry, inventory, and external data into a unified semantic layer
- Uses a custom multi-fact star schema with time-phased dimensions
- Provides cross-fact KPI harmonization (e.g., dwell time vs. fleet idling costs)
- Offers predictive bottleneck detection and adaptive fleet triage
- Implements "Warehouse Gravity Zones" for optimal spatial organization
- Delivers Power BI dashboards with 15-minute refresh intervals

## Installation & Setup

### Prerequisites
- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, telematics/GPS, supplier portals, weather/traffic APIs

### Initial Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL Schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Run the schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_bridges.sql
:r schema/05_create_views.sql
:r schema/06_create_stored_procedures.sql
```

3. **Configure Data Sources:**
```json
// config.json (copy from config_sample.json)
{
  "sql_server": {
    "server": "YOUR_SERVER_NAME",
    "database": "LogiFleetPulse",
    "authentication": "Windows" // or "SQL"
  },
  "data_sources": {
    "wms_connection": "${WMS_CONNECTION_STRING}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Open Power BI Template:**
```bash
# Open the .pbit template file
start LogiFleet_Pulse_Master.pbit
```

## Core Data Model Structure

### Key Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    DateKey INT,
    TimeSlot15Min TINYINT, -- 0-95 (96 slots per day)
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    FiscalPeriod VARCHAR(10),
    IsBusinessHour BIT
);

-- DimGeography: Hierarchical location data
CREATE TABLE dbo.DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(50),
    LocationName NVARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, RouteNode, CrossDock
    ParentGeographyKey INT,
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- DimProductGravity: Products with velocity-based gravity scoring
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(100),
    ProductName NVARCHAR(200),
    ProductCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- 0-100, higher = faster movement
    Fragility TINYINT, -- 1-5
    ValueTier VARCHAR(20), -- Low, Medium, High
    OptimalZoneType VARCHAR(50)
);
```

### Key Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES dbo.DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES dbo.DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES dbo.DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT,
    DwellTimeMinutes INT,
    ZoneDistance DECIMAL(10,2), -- Distance traveled in warehouse
    WorkerID INT,
    BatchID VARCHAR(50)
);

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES dbo.DimTime(TimeKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    OriginGeographyKey INT FOREIGN KEY REFERENCES dbo.DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES dbo.DimGeography(GeographyKey),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(100)
);

-- FactCrossDock: Cross-docking operations
CREATE TABLE dbo.FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES dbo.DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES dbo.DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES dbo.DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES dbo.FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES dbo.FactFleetTrips(TripKey),
    DwellTimeMinutes INT,
    Quantity INT
);
```

## Common Patterns & Queries

### Cross-Fact KPI Analysis

```sql
-- Correlate warehouse dwell time with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        p.ProductCategory,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        t.DateKey
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateKey >= DATEADD(day, -30, GETDATE())
    GROUP BY p.ProductCategory, t.DateKey
),
FleetIdle AS (
    SELECT 
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        t.DateKey
    FROM dbo.FactFleetTrips ft
    INNER JOIN dbo.DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.DateKey >= DATEADD(day, -30, GETDATE())
    GROUP BY t.DateKey
)
SELECT 
    wd.ProductCategory,
    wd.DateKey,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    -- Calculate correlation metric
    (wd.AvgDwellTime * fi.AvgIdleTime) AS CorrelationIndex
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.DateKey = fi.DateKey
ORDER BY wd.DateKey DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be relocated to different gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneType,
    g.LocationName AS CurrentZone,
    COUNT(wo.OperationKey) AS PickFrequency,
    AVG(wo.ZoneDistance) AS AvgTravelDistance,
    CASE 
        WHEN p.GravityScore > 80 AND g.LocationType <> 'HighGravityZone' THEN 'Move to High Gravity'
        WHEN p.GravityScore < 30 AND g.LocationType <> 'LowGravityZone' THEN 'Move to Low Gravity'
        ELSE 'Optimal'
    END AS RecommendedAction
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN dbo.DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
WHERE 
    wo.OperationType = 'Picking'
    AND t.DateKey >= DATEADD(day, -90, GETDATE())
GROUP BY 
    p.SKU, p.ProductName, p.GravityScore, p.OptimalZoneType, g.LocationName, g.LocationType
HAVING 
    COUNT(wo.OperationKey) > 10
ORDER BY 
    PickFrequency DESC;
```

### Predictive Bottleneck Detection

```sql
-- Stored procedure for bottleneck index calculation
CREATE PROCEDURE dbo.sp_CalculateBottleneckIndex
    @PredictionHorizonHours INT = 24
AS
BEGIN
    WITH HistoricalPattern AS (
        SELECT 
            t.HourOfDay,
            t.DayOfWeek,
            AVG(wo.DwellTimeMinutes) AS AvgHistoricalDwell,
            STDEV(wo.DwellTimeMinutes) AS StdDevDwell,
            COUNT(*) AS SampleSize
        FROM dbo.FactWarehouseOperations wo
        INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateKey >= DATEADD(day, -90, GETDATE())
        GROUP BY t.HourOfDay, t.DayOfWeek
    ),
    CurrentPattern AS (
        SELECT 
            t.HourOfDay,
            t.DayOfWeek,
            AVG(wo.DwellTimeMinutes) AS CurrentDwell
        FROM dbo.FactWarehouseOperations wo
        INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateKey >= DATEADD(hour, -2, GETDATE())
        GROUP BY t.HourOfDay, t.DayOfWeek
    )
    SELECT 
        hp.HourOfDay,
        hp.DayOfWeek,
        hp.AvgHistoricalDwell,
        cp.CurrentDwell,
        -- Bottleneck index: how many std devs above historical
        CASE 
            WHEN hp.StdDevDwell > 0 THEN 
                (cp.CurrentDwell - hp.AvgHistoricalDwell) / hp.StdDevDwell
            ELSE 0
        END AS BottleneckIndex,
        CASE 
            WHEN (cp.CurrentDwell - hp.AvgHistoricalDwell) / NULLIF(hp.StdDevDwell, 0) > 2 
            THEN 'High Risk'
            WHEN (cp.CurrentDwell - hp.AvgHistoricalDwell) / NULLIF(hp.StdDevDwell, 0) > 1 
            THEN 'Moderate Risk'
            ELSE 'Normal'
        END AS RiskLevel
    FROM HistoricalPattern hp
    LEFT JOIN CurrentPattern cp 
        ON hp.HourOfDay = cp.HourOfDay 
        AND hp.DayOfWeek = cp.DayOfWeek
    WHERE hp.SampleSize > 10
    ORDER BY BottleneckIndex DESC;
END;
```

### Fleet Adaptive Triage Scoring

```sql
-- Calculate weighted priority scores for fleet maintenance
WITH FleetMetrics AS (
    SELECT 
        ft.VehicleID,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) AS FuelEfficiency,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(ft.DelayMinutes) AS TotalDelayMinutes,
        COUNT(*) AS TripCount
    FROM dbo.FactFleetTrips ft
    INNER JOIN dbo.DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.DateKey >= DATEADD(day, -30, GETDATE())
    GROUP BY ft.VehicleID
),
CargoValue AS (
    SELECT 
        ft.VehicleID,
        SUM(p.ValueTier = 'High' THEN 1 ELSE 0 END) AS HighValueTrips
    FROM dbo.FactFleetTrips ft
    INNER JOIN dbo.FactWarehouseOperations wo ON ft.TimeKey = wo.TimeKey
    INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
    GROUP BY ft.VehicleID
)
SELECT 
    fm.VehicleID,
    fm.FuelEfficiency,
    fm.AvgIdleTime,
    fm.TotalDelayMinutes,
    cv.HighValueTrips,
    -- Weighted triage score (higher = more urgent)
    (
        (fm.AvgIdleTime * 0.3) + 
        (fm.TotalDelayMinutes * 0.4) + 
        (cv.HighValueTrips * 10 * 0.3)
    ) AS TriageScore,
    CASE 
        WHEN (fm.AvgIdleTime * 0.3 + fm.TotalDelayMinutes * 0.4 + cv.HighValueTrips * 10 * 0.3) > 100 
        THEN 'Critical'
        WHEN (fm.AvgIdleTime * 0.3 + fm.TotalDelayMinutes * 0.4 + cv.HighValueTrips * 10 * 0.3) > 50 
        THEN 'High Priority'
        ELSE 'Normal'
    END AS MaintenancePriority
FROM FleetMetrics fm
LEFT JOIN CargoValue cv ON fm.VehicleID = cv.VehicleID
ORDER BY TriageScore DESC;
```

## Data Loading & ETL Patterns

### Incremental Load Stored Procedure

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE dbo.sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get last load timestamp if not provided
    IF @LastLoadTimestamp IS NULL
    BEGIN
        SELECT @LastLoadTimestamp = ISNULL(MAX(LastUpdated), '1900-01-01')
        FROM dbo.ETL_LoadLog
        WHERE TableName = 'FactWarehouseOperations';
    END
    
    -- Insert new records from staging
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, 
        Quantity, DwellTimeMinutes, ZoneDistance, WorkerID, BatchID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        stg.ZoneDistance,
        stg.WorkerID,
        stg.BatchID
    FROM dbo.Staging_WarehouseOperations stg
    INNER JOIN dbo.DimTime t 
        ON CAST(stg.OperationTime AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, stg.OperationTime) = t.HourOfDay
        AND (DATEPART(MINUTE, stg.OperationTime) / 15) = (t.TimeSlot15Min / 4)
    INNER JOIN dbo.DimGeography g ON stg.LocationID = g.LocationID
    INNER JOIN dbo.DimProductGravity p ON stg.SKU = p.SKU
    WHERE stg.LoadedTimestamp > @LastLoadTimestamp
        AND NOT EXISTS (
            SELECT 1 FROM dbo.FactWarehouseOperations wo
            WHERE wo.BatchID = stg.BatchID
        );
    
    -- Log the load
    INSERT INTO dbo.ETL_LoadLog (TableName, LoadTimestamp, RowsLoaded, LastUpdated)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT, @LastLoadTimestamp);
END;
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

```dax
// Total Warehouse Dwell Time
Total Dwell Time = 
SUM(FactWarehouseOperations[DwellTimeMinutes])

// Average Fleet Idle Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + SUM(FactFleetTrips[LoadingTimeMinutes]) + SUM(FactFleetTrips[UnloadingTimeMinutes]),
    0
) * 100

// Cross-Fact Efficiency Index
Supply Chain Efficiency Index = 
VAR WarehouseThroughput = 
    DIVIDE(
        COUNTROWS(FactWarehouseOperations),
        [Total Dwell Time],
        0
    )
VAR FleetUtilization = 
    1 - ([Fleet Idle %] / 100)
RETURN
    (WarehouseThroughput * FleetUtilization) * 100

// Gravity Zone Compliance
Gravity Zone Compliance % = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimGeography[LocationType] = DimProductGravity[OptimalZoneType]
    ),
    COUNTROWS(FactWarehouseOperations),
    0
) * 100

// Predictive Bottleneck Score (time-based)
Bottleneck Risk Score = 
VAR CurrentHour = HOUR(NOW())
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], TODAY(), -90, DAY),
        DimTime[HourOfDay] = CurrentHour
    )
VAR CurrentAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], NOW(), -2, HOUR)
    )
VAR StdDev = 
    CALCULATE(
        STDEV.P(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], TODAY(), -90, DAY),
        DimTime[HourOfDay] = CurrentHour
    )
RETURN
    IF(
        StdDev > 0,
        DIVIDE(CurrentAvg - HistoricalAvg, StdDev),
        0
    )
```

### Power BI Report Connection

```python
# Python script for automated Power BI refresh via REST API
import requests
import os
from datetime import datetime

# Configuration from environment variables
POWERBI_WORKSPACE_ID = os.getenv('POWERBI_WORKSPACE_ID')
POWERBI_DATASET_ID = os.getenv('POWERBI_DATASET_ID')
POWERBI_CLIENT_ID = os.getenv('POWERBI_CLIENT_ID')
POWERBI_CLIENT_SECRET = os.getenv('POWERBI_CLIENT_SECRET')
POWERBI_TENANT_ID = os.getenv('POWERBI_TENANT_ID')

def get_access_token():
    """Obtain Azure AD access token for Power BI API"""
    auth_url = f"https://login.microsoftonline.com/{POWERBI_TENANT_ID}/oauth2/v2.0/token"
    
    payload = {
        'client_id': POWERBI_CLIENT_ID,
        'client_secret': POWERBI_CLIENT_SECRET,
        'scope': 'https://analysis.windows.net/powerbi/api/.default',
        'grant_type': 'client_credentials'
    }
    
    response = requests.post(auth_url, data=payload)
    response.raise_for_status()
    return response.json()['access_token']

def trigger_dataset_refresh():
    """Trigger Power BI dataset refresh"""
    token = get_access_token()
    
    refresh_url = f"https://api.powerbi.com/v1.0/myorg/groups/{POWERBI_WORKSPACE_ID}/datasets/{POWERBI_DATASET_ID}/refreshes"
    
    headers = {
        'Authorization': f'Bearer {token}',
        'Content-Type': 'application/json'
    }
    
    response = requests.post(refresh_url, headers=headers)
    
    if response.status_code == 202:
        print(f"[{datetime.now()}] Dataset refresh triggered successfully")
    else:
        print(f"[{datetime.now()}] Failed to trigger refresh: {response.status_code}")
        print(response.text)

if __name__ == "__main__":
    trigger_dataset_refresh()
```

## Alerting & Monitoring

### SQL Server Agent Job for Threshold Alerts

```sql
-- Create alert stored procedure
CREATE PROCEDURE dbo.sp_CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @AlertSubject NVARCHAR(255);
    
    -- Check fleet idle time threshold
    IF EXISTS (
        SELECT 1
        FROM dbo.FactFleetTrips ft
        INNER JOIN dbo.DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.DateKey = CAST(GETDATE() AS DATE)
        GROUP BY ft.VehicleID
        HAVING AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / 
               NULLIF(ft.IdleTimeMinutes + ft.LoadingTimeMinutes + ft.UnloadingTimeMinutes, 0)) > 0.15
    )
    BEGIN
        SET @AlertSubject = 'LogiFleet Pulse Alert: Fleet Idle Time Threshold Exceeded';
        SET @AlertMessage = 'One or more vehicles exceeded 15% idle time threshold today. Check the Fleet Triage dashboard.';
        
        -- Send email via Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'logistics-team@company.com',
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END
    
    -- Check warehouse bottleneck risk
    DECLARE @BottleneckRisk FLOAT;
    
    SELECT @BottleneckRisk = AVG(
        CASE 
            WHEN hp.StdDevDwell > 0 THEN 
                (cp.CurrentDwell - hp.AvgHistoricalDwell) / hp.StdDevDwell
            ELSE 0
        END
    )
    FROM (
        SELECT 
            t.HourOfDay,
            AVG(wo.DwellTimeMinutes) AS AvgHistoricalDwell,
            STDEV(wo.DwellTimeMinutes) AS StdDevDwell
        FROM dbo.FactWarehouseOperations wo
        INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateKey >= DATEADD(day, -90, GETDATE())
        GROUP BY t.HourOfDay
    ) hp
    INNER JOIN (
        SELECT 
            t.HourOfDay,
            AVG(wo.DwellTimeMinutes) AS CurrentDwell
        FROM dbo.FactWarehouseOperations wo
        INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(hour, -1, GETDATE())
        GROUP BY t.HourOfDay
    ) cp ON hp.HourOfDay = cp.HourOfDay;
    
    IF @BottleneckRisk > 2.0
    BEGIN
        SET @AlertSubject = 'LogiFleet Pulse Alert: High Bottleneck Risk Detected';
        SET @AlertMessage = 'Warehouse dwell time is 2+ standard deviations above historical average. Review Bottleneck Heatmap.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'warehouse-ops@company.com',
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END
END;
GO

-- Schedule via SQL Server Agent
-- Run every 15 minutes during business hours
```

## Troubleshooting

### Common Issues

**Issue: Power BI refresh fails with "relationship not found" error**
```sql
-- Verify all foreign key relationships exist
SELECT 
    fk.name AS ForeignKeyName,
    OBJECT_NAME(fk.parent_object_id) AS TableName,
    COL_NAME(fkc.parent_object_id, fkc.parent_column_id) AS ColumnName,
    OBJECT_NAME(fk.referenced_object_id) AS ReferencedTable,
    COL_NAME(fkc.referenced_object_id, fkc.referenced_column_id) AS ReferencedColumn
FROM sys.foreign_keys fk
INNER JOIN sys.foreign_key_columns fkc 
    ON fk.object_id = fkc.constraint_object_id
WHERE fk.is_disabled = 0
ORDER BY TableName;
```

**Issue: Query performance degradation on large fact tables**
```sql
-- Create columnstore indexes for fact tables (SQL Server 2016+)
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOperations
ON dbo.FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, OperationType, 
    Quantity, DwellTimeMinutes, ZoneDistance
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON dbo.FactFleetTrips (
    TimeKey, VehicleID, OriginGeographyKey, DestinationGeographyKey,
    DistanceKM, FuelConsumedLiters, IdleTimeMinutes
);

-- Partition fact tables by date for better performance
CREATE PARTITION FUNCTION PF_DateKey (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601,
    20260701, 20260801, 20260901, 20261001, 20261101, 20261201
);

CREATE PARTITION SCHEME PS_DateKey
AS PARTITION PF_DateKey ALL TO ([PRIMARY]);

-- Apply to existing table (requires rebuilding)
CREATE TABLE dbo.FactWarehouseOperations_New (
    -- same columns as original
) ON PS_DateKey(TimeKey);
```

**Issue: Time dimension not aligning with 15-minute intervals**
```sql
-- Regenerate time dimension with correct 15-minute slots
TRUNCATE TABLE dbo.DimTime;

DECLARE @StartDate DATETIME2 = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2027-12-31 23:45:00';
DECLARE @CurrentDate DATETIME2 = @StartDate;
DECLARE @TimeKey INT = 1;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO dbo.DimTime (
        TimeKey, FullDateTime, DateKey, TimeSlot15Min, 
        HourOfDay, DayOfWeek, FiscalPeriod, IsBusinessHour
    )
    VALUES (
        @TimeKey,
        @CurrentDate,
        CAST(FORMAT(@CurrentDate, 'yyyyMMdd') AS INT),
        (DATEPART(HOUR, @CurrentDate) * 4) + (DATEPART(MINUTE, @CurrentDate) / 15),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        CONCAT('FY', YEAR(@CurrentDate), 'Q', DATEPART(QUARTER, @CurrentDate)),
        CASE 
            WHEN DATEPART(HOUR, @CurrentDate) BETWEEN 8 AND 17 
                 AND DATEPART(WEEKDAY, @CurrentDate) NOT IN (1, 7)
            THEN 1 
            ELSE 0 
        END
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
    SET @TimeKey = @TimeKey + 1;
END;
```

**Issue: Gravity score not updating automatically**
```sql
-- Create scheduled job to recalculate gravity scores based on recent activity
CREATE PROCEDURE dbo.sp_RecalculateGravityScores
AS
BEGIN
    WITH RecentActivity AS (
        SELECT 
            wo.ProductKey,
            COUNT(*) AS PickCount,
            AVG(wo.ZoneDistance) AS AvgDistance,
            SUM(CASE WHEN wo.DwellTimeMinutes > 60 THEN 1 ELSE 0 END) AS HighDwellCount
        FROM dbo.FactWarehouseOperations wo
        INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
        WHERE 
            t.DateKey >= DATEADD(day, -30, GETDATE())
            AND wo.OperationType = 'Picking'
        GROUP BY wo.ProductKey
    )
    UPDATE p
    SET 
        p.GravityScore = 
            CASE 
                WHEN ra.PickCount > 100 THEN 90 + (RAND() * 10)
                WHEN ra.PickCount > 50 THEN 70 + (RAND() * 20)
                WHEN ra.PickCount > 20 THEN 50 + (RAND() * 20)
