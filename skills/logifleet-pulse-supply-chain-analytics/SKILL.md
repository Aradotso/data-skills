---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up logistics analytics dashboard with Power BI
  - create supply chain data warehouse schema
  - implement fleet management analytics
  - build warehouse operations KPI tracking
  - configure cross-modal logistics intelligence
  - deploy multi-fact star schema for supply chain
  - integrate fleet telemetry with warehouse data
  - analyze logistics bottlenecks and dwell time
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics platform for logistics operations built on MS SQL Server and Power BI. It provides a multi-fact star schema that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for real-time supply chain intelligence.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time Power BI dashboards (15-minute refresh cycles)
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Fleet triage and maintenance prioritization
- Role-based access with row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, fleet telemetry, ERP systems

### Step 1: Deploy SQL Schema

Clone the repository and locate the SQL schema files:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

Execute the schema creation script in SSMS:

```sql
-- Connect to your target database first
USE [LogiFleetPulse]
GO

-- Run the main schema creation script
-- This creates all fact tables, dimensions, and relationships
:r ./sql/01_create_schema.sql

-- Create indexes for performance optimization
:r ./sql/02_create_indexes.sql

-- Set up row-level security
:r ./sql/03_configure_security.sql

-- Deploy stored procedures for incremental loading
:r ./sql/04_etl_procedures.sql
```

### Step 2: Configure Data Sources

Update the configuration file with your connection strings:

```json
{
  "dataSources": {
    "wms": {
      "connectionString": "${WMS_CONNECTION_STRING}",
      "refreshInterval": 15
    },
    "fleetTelemetry": {
      "apiEndpoint": "${FLEET_API_ENDPOINT}",
      "apiKey": "${FLEET_API_KEY}",
      "refreshInterval": 5
    },
    "erp": {
      "connectionString": "${ERP_CONNECTION_STRING}",
      "refreshInterval": 30
    }
  },
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "integrated"
  }
}
```

### Step 3: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter SQL Server connection details when prompted
5. Verify data model relationships are intact

## Core Data Model Architecture

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity metrics:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    PackingTimeMinutes INT,
    LocationGravityScore DECIMAL(5,2),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

CREATE NONCLUSTERED INDEX IX_FactWH_Time_Warehouse 
ON FactWarehouseOperations(TimeKey, WarehouseKey) 
INCLUDE (DwellTimeMinutes, OperationType);
```

**FactFleetTrips** - Fleet and vehicle telemetry:

```sql
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    TripDistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AvgSpeedMPH DECIMAL(5,2),
    DelayMinutes INT,
    DelayReasonKey INT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);
```

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT, -- YYYYMMDD format
    TimeOfDay TIME,
    Hour INT,
    Minute INT,
    QuarterHour INT, -- 1-4 within the hour
    DayOfWeek VARCHAR(20),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    FiscalYear INT
);

-- Populate with 15-minute buckets
DECLARE @StartDate DATETIME = '2024-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2026-12-31 23:45:00';

WITH TimeBuckets AS (
    SELECT @StartDate AS TimeSlot
    UNION ALL
    SELECT DATEADD(MINUTE, 15, TimeSlot)
    FROM TimeBuckets
    WHERE TimeSlot < @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeOfDay, Hour, Minute, QuarterHour, DayOfWeek, IsWeekend)
SELECT 
    CONVERT(INT, FORMAT(TimeSlot, 'yyyyMMddHHmm')),
    TimeSlot,
    CONVERT(INT, FORMAT(TimeSlot, 'yyyyMMdd')),
    CAST(TimeSlot AS TIME),
    DATEPART(HOUR, TimeSlot),
    DATEPART(MINUTE, TimeSlot),
    CASE WHEN DATEPART(MINUTE, TimeSlot) BETWEEN 0 AND 14 THEN 1
         WHEN DATEPART(MINUTE, TimeSlot) BETWEEN 15 AND 29 THEN 2
         WHEN DATEPART(MINUTE, TimeSlot) BETWEEN 30 AND 44 THEN 3
         ELSE 4 END,
    DATENAME(WEEKDAY, TimeSlot),
    CASE WHEN DATEPART(WEEKDAY, TimeSlot) IN (1,7) THEN 1 ELSE 0 END
FROM TimeBuckets
OPTION (MAXRECURSION 0);
```

**DimProductGravity** - Product dimension with gravity scoring:

```sql
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    ProductSKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitValue DECIMAL(10,2),
    WeightKg DECIMAL(8,2),
    IsFragile BIT,
    VelocityScore DECIMAL(5,2), -- Calculated: avg daily turnover
    GravityScore DECIMAL(5,2), -- Composite: velocity * value * fragility
    StorageZoneRecommendation VARCHAR(50)
);

-- Calculate gravity score (higher = needs closer placement to shipping dock)
UPDATE DimProduct
SET GravityScore = (VelocityScore * 0.5) + 
                   (LOG(UnitValue + 1) * 0.3) + 
                   (CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END);
```

## Key Queries & Patterns

### Cross-Fact Analysis: Fleet Efficiency vs Warehouse Dwell Time

```sql
-- Find correlations between warehouse dwell time and fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        t.DateKey,
        w.WarehouseID,
        AVG(f.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS TotalOperations
    FROM FactWarehouseOperations f
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    JOIN DimWarehouse w ON f.WarehouseKey = w.WarehouseKey
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd'))
    GROUP BY t.DateKey, w.WarehouseID
),
FleetIdle AS (
    SELECT 
        t.DateKey,
        r.OriginWarehouseID,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        SUM(f.FuelConsumedGallons) AS TotalFuel
    FROM FactFleetTrips f
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    JOIN DimRoute r ON f.RouteKey = r.RouteKey
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd'))
    GROUP BY t.DateKey, r.OriginWarehouseID
)
SELECT 
    wd.DateKey,
    wd.WarehouseID,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.TotalFuel,
    -- Correlation indicator
    CASE 
        WHEN wd.AvgDwellTime > 120 AND fi.AvgIdleTime > 30 
        THEN 'High Congestion Risk'
        WHEN wd.AvgDwellTime < 60 AND fi.AvgIdleTime < 15 
        THEN 'Optimal Flow'
        ELSE 'Monitor'
    END AS FlowStatus
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi ON wd.DateKey = fi.DateKey AND wd.WarehouseID = fi.OriginWarehouseID
ORDER BY wd.DateKey DESC, wd.AvgDwellTime DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks using 3-day rolling averages
CREATE PROCEDURE sp_DetectBottlenecks
AS
BEGIN
    WITH RollingMetrics AS (
        SELECT 
            f.WarehouseKey,
            t.DateKey,
            AVG(f.DwellTimeMinutes) OVER (
                PARTITION BY f.WarehouseKey 
                ORDER BY t.DateKey 
                ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
            ) AS RollingAvgDwell,
            AVG(f.PickRateUnitsPerHour) OVER (
                PARTITION BY f.WarehouseKey 
                ORDER BY t.DateKey 
                ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
            ) AS RollingAvgPickRate
        FROM FactWarehouseOperations f
        JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd'))
    )
    SELECT DISTINCT
        w.WarehouseName,
        r.DateKey,
        r.RollingAvgDwell,
        r.RollingAvgPickRate,
        -- Alert if dwell time increasing AND pick rate decreasing
        CASE 
            WHEN r.RollingAvgDwell > LAG(r.RollingAvgDwell, 1) OVER (PARTITION BY r.WarehouseKey ORDER BY r.DateKey)
                AND r.RollingAvgPickRate < LAG(r.RollingAvgPickRate, 1) OVER (PARTITION BY r.WarehouseKey ORDER BY r.DateKey)
            THEN 'BOTTLENECK_FORMING'
            ELSE 'NORMAL'
        END AS AlertStatus
    FROM RollingMetrics r
    JOIN DimWarehouse w ON r.WarehouseKey = w.WarehouseKey
    WHERE r.DateKey = CONVERT(INT, FORMAT(GETDATE(), 'yyyyMMdd'))
    ORDER BY AlertStatus DESC, r.RollingAvgDwell DESC;
END;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend storage zone reassignments based on gravity scores
WITH CurrentPlacements AS (
    SELECT 
        p.ProductSKU,
        p.GravityScore,
        p.StorageZoneRecommendation AS CurrentZone,
        CASE 
            WHEN p.GravityScore >= 80 THEN 'Zone_A_HighGravity'
            WHEN p.GravityScore BETWEEN 50 AND 79 THEN 'Zone_B_MediumGravity'
            WHEN p.GravityScore BETWEEN 20 AND 49 THEN 'Zone_C_LowGravity'
            ELSE 'Zone_D_BulkStorage'
        END AS OptimalZone,
        AVG(wo.DwellTimeMinutes) AS AvgDwell
    FROM DimProduct p
    LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    GROUP BY p.ProductSKU, p.GravityScore, p.StorageZoneRecommendation
)
SELECT 
    ProductSKU,
    GravityScore,
    CurrentZone,
    OptimalZone,
    AvgDwell,
    CASE 
        WHEN CurrentZone <> OptimalZone THEN 'REASSIGN_RECOMMENDED'
        ELSE 'OPTIMAL_PLACEMENT'
    END AS Action
FROM CurrentPlacements
WHERE CurrentZone <> OptimalZone
ORDER BY GravityScore DESC;
```

## ETL & Data Loading Patterns

### Incremental Load Pattern

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last hour if no parameter provided
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -1, GETDATE());
    
    -- Stage data from source system
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        DwellTimeMinutes, PickRateUnitsPerHour, PackingTimeMinutes
    )
    SELECT 
        CONVERT(INT, FORMAT(src.OperationTimestamp, 'yyyyMMddHHmm')) AS TimeKey,
        dw.WarehouseKey,
        dp.ProductKey,
        src.OperationType,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        CASE 
            WHEN src.OperationType = 'Picking' 
            THEN src.UnitsProcessed / NULLIF(DATEDIFF(MINUTE, src.StartTime, src.EndTime), 0) * 60
            ELSE NULL 
        END AS PickRateUnitsPerHour,
        CASE 
            WHEN src.OperationType = 'Packing' 
            THEN DATEDIFF(MINUTE, src.StartTime, src.EndTime)
            ELSE NULL 
        END AS PackingTimeMinutes
    FROM ${WMS_DB}.dbo.WarehouseTransactions src
    JOIN DimWarehouse dw ON src.WarehouseCode = dw.WarehouseCode
    JOIN DimProduct dp ON src.ProductSKU = dp.ProductSKU
    WHERE src.OperationTimestamp > @LastLoadDateTime
        AND src.IsProcessed = 0;
    
    -- Update source system processed flag
    UPDATE ${WMS_DB}.dbo.WarehouseTransactions
    SET IsProcessed = 1
    WHERE OperationTimestamp > @LastLoadDateTime;
    
    -- Log ETL metadata
    INSERT INTO ETL_LoadLog (TableName, LoadDateTime, RowsInserted)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
END;
```

### Real-Time Fleet Telemetry Integration

```sql
-- External table for streaming fleet data (requires PolyBase)
CREATE EXTERNAL TABLE FleetTelemetryStream
WITH (
    DATA_SOURCE = FleetAPIDataSource,
    LOCATION = '/api/v1/telemetry/stream',
    FILE_FORMAT = JSONFileFormat
)
AS
SELECT 
    vehicle_id,
    timestamp,
    latitude,
    longitude,
    speed_mph,
    fuel_level,
    engine_temp,
    is_idle
FROM FleetTelemetryRawData;

-- Materialize into fact table every 5 minutes
CREATE PROCEDURE sp_LoadFleetTelemetry
AS
BEGIN
    INSERT INTO FactFleetTrips (TimeKey, VehicleKey, IdleTimeMinutes, AvgSpeedMPH)
    SELECT 
        CONVERT(INT, FORMAT(fts.timestamp, 'yyyyMMddHHmm')) AS TimeKey,
        dv.VehicleKey,
        SUM(CASE WHEN fts.is_idle = 1 THEN 5 ELSE 0 END) AS IdleTimeMinutes,
        AVG(fts.speed_mph) AS AvgSpeedMPH
    FROM FleetTelemetryStream fts
    JOIN DimVehicle dv ON fts.vehicle_id = dv.VehicleID
    WHERE fts.timestamp > DATEADD(MINUTE, -5, GETDATE())
    GROUP BY CONVERT(INT, FORMAT(fts.timestamp, 'yyyyMMddHHmm')), dv.VehicleKey;
END;
```

## Power BI Configuration

### DAX Measures for Cross-Fact KPIs

**Composite Supply Chain Efficiency Score:**

```dax
Supply Chain Efficiency = 
VAR WarehouseEfficiency = 
    DIVIDE(
        AVERAGE(FactWarehouseOperations[PickRateUnitsPerHour]),
        60, -- Target: 60 units/hour
        0
    )
VAR FleetEfficiency = 
    DIVIDE(
        AVERAGE(FactFleetTrips[AvgSpeedMPH]),
        45, -- Target: 45 MPH average
        0
    )
VAR IdlePenalty = 
    1 - DIVIDE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        120, -- Max tolerable: 120 minutes
        0
    )
RETURN
    (WarehouseEfficiency * 0.4) + (FleetEfficiency * 0.3) + (IdlePenalty * 0.3)
```

**Dynamic Gravity Zone Heatmap:**

```dax
Gravity Zone Alert = 
VAR CurrentGravityZone = SELECTEDVALUE(DimProduct[StorageZoneRecommendation])
VAR OptimalZone = 
    SWITCH(
        TRUE(),
        SELECTEDVALUE(DimProduct[GravityScore]) >= 80, "Zone_A_HighGravity",
        SELECTEDVALUE(DimProduct[GravityScore]) >= 50, "Zone_B_MediumGravity",
        SELECTEDVALUE(DimProduct[GravityScore]) >= 20, "Zone_C_LowGravity",
        "Zone_D_BulkStorage"
    )
RETURN
    IF(CurrentGravityZone <> OptimalZone, "⚠️ Reassign", "✓ Optimal")
```

### Row-Level Security Configuration

```dax
-- RLS filter for warehouse managers (only see their assigned warehouses)
[WarehouseManagerEmail] = USERPRINCIPALNAME()

-- RLS filter for regional directors (see all warehouses in their region)
PATHCONTAINS(
    DimGeography[GeographyPath], 
    LOOKUPVALUE(
        DimUser[AssignedRegion],
        DimUser[Email],
        USERPRINCIPALNAME()
    )
)
```

## Troubleshooting & Common Issues

### Issue: Power BI Refresh Timeouts

**Symptom:** Dashboard refresh fails after 2+ hours with timeout error.

**Solution:** Implement incremental refresh on large fact tables:

```powerquery
// In Power Query, add parameters
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    FilteredRows = Table.SelectRows(Source, each [OperationTimestamp] >= RangeStart and [OperationTimestamp] < RangeEnd)
in
    FilteredRows
```

Configure incremental refresh in Power BI Desktop:
- Store rows in last 3 years
- Refresh data in last 7 days
- Detect data changes: No (performance optimization)

### Issue: Slow Cross-Fact Queries

**Symptom:** Queries joining FactWarehouseOperations and FactFleetTrips take 30+ seconds.

**Solution:** Create indexed views for common joins:

```sql
CREATE VIEW vw_WarehouseFleetCrossFactMetrics
WITH SCHEMABINDING
AS
SELECT 
    t.DateKey,
    w.WarehouseID,
    AVG(wo.DwellTimeMinutes) AS AvgDwell,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdle,
    COUNT_BIG(*) AS RowCount
FROM dbo.FactWarehouseOperations wo
JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
JOIN dbo.DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
LEFT JOIN dbo.FactFleetTrips ft ON t.TimeKey = ft.TimeKey
GROUP BY t.DateKey, w.WarehouseID;

CREATE UNIQUE CLUSTERED INDEX IX_CrossFact_Date_Warehouse
ON vw_WarehouseFleetCrossFactMetrics(DateKey, WarehouseID);
```

### Issue: Gravity Score Not Updating

**Symptom:** Products still show old gravity scores despite velocity changes.

**Solution:** Schedule nightly gravity recalculation:

```sql
CREATE PROCEDURE sp_RecalculateGravityScores
AS
BEGIN
    -- Update velocity scores based on last 30 days
    WITH RecentVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) / 30.0 AS DailyTurnover
        FROM FactWarehouseOperations
        WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd0000'))
        GROUP BY ProductKey
    )
    UPDATE p
    SET VelocityScore = rv.DailyTurnover,
        GravityScore = (rv.DailyTurnover * 0.5) + 
                       (LOG(p.UnitValue + 1) * 0.3) + 
                       (CASE WHEN p.IsFragile = 1 THEN 20 ELSE 0 END)
    FROM DimProduct p
    JOIN RecentVelocity rv ON p.ProductKey = rv.ProductKey;
END;

-- Schedule via SQL Agent daily at 2 AM
```

## Performance Optimization Best Practices

1. **Partition large fact tables by date:**

```sql
ALTER DATABASE LogiFleetPulse ADD FILEGROUP FG_2025;
ALTER DATABASE LogiFleetPulse ADD FILEGROUP FG_2026;

CREATE PARTITION FUNCTION PF_DateKey (INT)
AS RANGE RIGHT FOR VALUES (20250101, 20260101);

CREATE PARTITION SCHEME PS_DateKey
AS PARTITION PF_DateKey
TO (FG_2024, FG_2025, FG_2026);

CREATE TABLE FactWarehouseOperations (...) ON PS_DateKey(TimeKey);
```

2. **Use columnstore indexes for analytics workloads:**

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps
ON FactWarehouseOperations (TimeKey, WarehouseKey, ProductKey, DwellTimeMinutes, PickRateUnitsPerHour);
```

3. **Enable query store for performance tracking:**

```sql
ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;
ALTER DATABASE LogiFleetPulse SET QUERY_STORE (OPERATION_MODE = READ_WRITE);
```

## Alerting & Automation

### SQL Agent Job for Daily KPI Alerts

```sql
CREATE PROCEDURE sp_SendDailyAlerts
AS
BEGIN
    DECLARE @AlertHTML NVARCHAR(MAX);
    
    -- Build HTML table of bottleneck alerts
    SET @AlertHTML = 
        N'<html><body><h2>Daily Logistics Alerts</h2>' +
        CAST((
            SELECT 
                td = w.WarehouseName, '',
                td = CAST(AVG(f.DwellTimeMinutes) AS VARCHAR(10)) + ' min', '',
                td = CASE WHEN AVG(f.DwellTimeMinutes) > 120 THEN 'CRITICAL' ELSE 'OK' END
            FROM FactWarehouseOperations f
            JOIN DimWarehouse w ON f.WarehouseKey = w.WarehouseKey
            JOIN DimTime t ON f.TimeKey = t.TimeKey
            WHERE t.DateKey = CONVERT(INT, FORMAT(GETDATE(), 'yyyyMMdd'))
            GROUP BY w.WarehouseName
            HAVING AVG(f.DwellTimeMinutes) > 120
            FOR XML PATH('tr'), TYPE
        ) AS NVARCHAR(MAX)) +
        N'</body></html>';
    
    -- Send email via Database Mail
    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = 'LogiFleetAlerts',
        @recipients = '${ALERT_EMAIL_RECIPIENTS}',
        @subject = 'LogiFleet Pulse - Daily Alert Summary',
        @body = @AlertHTML,
        @body_format = 'HTML';
END;
```

## Integration Examples

### REST API Integration for External Weather Data

```sql
-- Use OPENROWSET for one-time weather API calls
DECLARE @WeatherJSON NVARCHAR(MAX);

SELECT @WeatherJSON = BulkColumn
FROM OPENROWSET(
    BULK 'https://api.weatherapi.com/v1/forecast.json?key=${WEATHER_API_KEY}&q=40.7128,-74.0060&days=3',
    SINGLE_CLOB
) AS WeatherData;

-- Parse JSON and insert into delay reasons
INSERT INTO DimDelayReason (ReasonType, Description, SeverityScore)
SELECT 
    'Weather' AS ReasonType,
    JSON_VALUE(weather, '$.forecast.forecastday[0].day.condition.text') AS Description,
    CASE 
        WHEN JSON_VALUE(weather, '$.forecast.forecastday[0].day.condition.text') LIKE '%snow%' THEN 8
        WHEN JSON_VALUE(weather, '$.forecast.forecastday[0].day.condition.text') LIKE '%rain%' THEN 5
        ELSE 2
    END AS SeverityScore
FROM OPENJSON(@WeatherJSON, '$.forecast.forecastday') weather;
```

This skill covers the essential patterns for implementing, configuring, and optimizing the LogiFleet Pulse supply chain analytics platform using MS SQL Server and Power BI.
