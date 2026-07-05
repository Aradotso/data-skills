---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for warehouse operations, fleet tracking, and cross-modal supply chain analytics
triggers:
  - "set up logifleet pulse logistics dashboard"
  - "configure supply chain analytics with power bi"
  - "implement warehouse gravity zone modeling"
  - "create fleet operations data warehouse"
  - "build cross-fact logistics KPI reports"
  - "deploy logicore analytics sql schema"
  - "integrate warehouse and fleet telemetry data"
  - "design multi-modal supply chain star schema"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified MS SQL Server data warehouse with Power BI visualization. It uses a custom multi-fact star schema to enable cross-functional KPI analysis like correlating warehouse dwell time with fleet fuel costs.

**Core capabilities:**
- Multi-fact star schema (warehouse operations, fleet trips, cross-dock transfers)
- Time-phased dimensions with 15-minute granularity
- Warehouse Gravity Zones™ for spatial optimization
- Predictive bottleneck detection
- Real-time Power BI dashboards with role-based security
- Cross-fact KPI harmonization

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telemetry APIs, ERP systems

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema creation scripts in order:
-- 1. Create dimension tables
-- 2. Create fact tables
-- 3. Create views and stored procedures
-- 4. Set up indexes and partitions
```

### Step 3: Configure Data Sources

Edit the configuration file with your connection details:

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}",
    "port": 1433
  },
  "dataSources": {
    "wms": {
      "apiEndpoint": "${WMS_API_URL}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telemetry": {
      "apiEndpoint": "${FLEET_TELEMETRY_URL}",
      "apiKey": "${TELEMETRY_API_KEY}"
    },
    "erp": {
      "connectionString": "${ERP_CONNECTION_STRING}"
    }
  },
  "refreshInterval": 15
}
```

### Step 4: Open Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection details when prompted
3. Verify data model relationships auto-detect correctly
4. Publish to Power BI Service for team access

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse micro-operations
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    PickingTimeSeconds INT,
    QuantityHandled INT,
    GravityZone VARCHAR(10),
    EmployeeKey INT,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Columnstore index for fast aggregations
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse 
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, OperationType, DwellTimeMinutes);
```

**FactFleetTrips** - Vehicle telemetry and route segments
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginWarehouseKey INT,
    DestinationKey INT,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReasonKey INT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleet 
ON FactFleetTrips (TimeKey, VehicleKey, RouteKey, FuelConsumedLiters, IdleTimeMinutes);
```

**FactCrossDock** - Direct transfers without storage
```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripID BIGINT,
    OutboundTripID BIGINT,
    TransferTimeMinutes INT,
    QuantityTransferred INT,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfWeek INT NOT NULL,
    Hour INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL,
    FiscalPeriod VARCHAR(10)
);

-- Populate with 15-minute intervals
DECLARE @StartDate DATETIME = '2024-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2028-12-31 23:45:00';

WITH TimeIntervals AS (
    SELECT @StartDate AS DateTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, DateTime)
    FROM TimeIntervals
    WHERE DateTime < @EndDate
)
INSERT INTO DimTime (TimeKey, DateTime, Date, Year, Quarter, Month, Week, DayOfWeek, Hour, MinuteBucket, IsWeekend, IsHoliday, FiscalPeriod)
SELECT 
    CONVERT(INT, FORMAT(DateTime, 'yyyyMMddHHmm')),
    DateTime,
    CAST(DateTime AS DATE),
    YEAR(DateTime),
    DATEPART(QUARTER, DateTime),
    MONTH(DateTime),
    DATEPART(WEEK, DateTime),
    DATEPART(WEEKDAY, DateTime),
    DATEPART(HOUR, DateTime),
    DATEPART(MINUTE, DateTime),
    CASE WHEN DATEPART(WEEKDAY, DateTime) IN (1, 7) THEN 1 ELSE 0 END,
    0, -- Update with holiday logic
    CONCAT('FY', YEAR(DateTime), '-P', DATEPART(QUARTER, DateTime))
FROM TimeIntervals
OPTION (MAXRECURSION 0);
```

**DimProductGravity** - Product classification with gravity scoring
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Picks per day
    ValueScore DECIMAL(10,2), -- Unit value
    FragilityScore INT, -- 1-10 scale
    GravityScore AS (VelocityScore * ValueScore * (11 - FragilityScore)) PERSISTED,
    GravityZone AS (
        CASE 
            WHEN (VelocityScore * ValueScore * (11 - FragilityScore)) > 5000 THEN 'A-HIGH'
            WHEN (VelocityScore * ValueScore * (11 - FragilityScore)) > 1000 THEN 'B-MED'
            ELSE 'C-LOW'
        END
    ) PERSISTED
);

-- Example insert
INSERT INTO DimProductGravity (SKU, ProductName, Category, VelocityScore, ValueScore, FragilityScore)
VALUES 
    ('SKU-001', 'Premium Coffee Beans 1kg', 'Food', 120.5, 25.00, 3),
    ('SKU-002', 'Ceramic Vase', 'Decor', 5.2, 150.00, 9),
    ('SKU-003', 'Bulk Paper Towels', 'Supplies', 200.0, 5.00, 1);
```

## Key SQL Queries

### Cross-Fact Analysis: Dwell Time vs Fleet Idle Cost

```sql
-- Correlate warehouse dwell time with fleet idle costs by product category
WITH WarehouseDwell AS (
    SELECT 
        p.Category,
        AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(w.QuantityHandled) AS TotalUnits
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.Category
),
FleetIdle AS (
    SELECT 
        p.Category,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        AVG(f.FuelConsumedLiters) AS AvgFuelLiters
    FROM FactFleetTrips f
    JOIN FactCrossDock cd ON f.TripID = cd.OutboundTripID
    JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.Category
)
SELECT 
    wd.Category,
    wd.AvgDwellMinutes,
    fi.AvgIdleMinutes,
    fi.AvgFuelLiters,
    (wd.AvgDwellMinutes / NULLIF(fi.AvgIdleMinutes, 0)) AS DwellToIdleRatio,
    CASE 
        WHEN (wd.AvgDwellMinutes / NULLIF(fi.AvgIdleMinutes, 0)) > 2 THEN 'High Warehouse Friction'
        WHEN (wd.AvgDwellMinutes / NULLIF(fi.AvgIdleMinutes, 0)) < 0.5 THEN 'High Fleet Friction'
        ELSE 'Balanced'
    END AS BottleneckType
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi ON wd.Category = fi.Category
ORDER BY DwellToIdleRatio DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend product relocations based on gravity score drift
WITH CurrentZones AS (
    SELECT 
        w.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.GravityZone AS CurrentZone,
        w.GravityZone AS ActualZone,
        COUNT(*) AS OperationCount,
        AVG(w.PickingTimeSeconds) AS AvgPickTime
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
        AND w.OperationType = 'PICK'
    GROUP BY w.ProductKey, p.SKU, p.ProductName, p.GravityScore, p.GravityZone, w.GravityZone
)
SELECT 
    SKU,
    ProductName,
    GravityScore,
    CurrentZone AS RecommendedZone,
    ActualZone,
    AvgPickTime,
    CASE 
        WHEN CurrentZone <> ActualZone THEN 'RELOCATE'
        ELSE 'OK'
    END AS Action,
    CASE 
        WHEN CurrentZone = 'A-HIGH' AND ActualZone <> 'A-HIGH' 
            THEN (AvgPickTime - 30) * OperationCount * 0.5 -- Estimated savings in seconds
        ELSE 0
    END AS EstimatedTimeSavingsSeconds
FROM CurrentZones
WHERE CurrentZone <> ActualZone
ORDER BY EstimatedTimeSavingsSeconds DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify probable congestion points using historical patterns
CREATE PROCEDURE sp_PredictBottlenecks
    @ForecastHorizonMinutes INT = 120
AS
BEGIN
    WITH HistoricalPatterns AS (
        SELECT 
            t.Hour,
            t.MinuteBucket,
            t.DayOfWeek,
            w.WarehouseKey,
            AVG(w.DwellTimeMinutes) AS AvgDwell,
            STDEV(w.DwellTimeMinutes) AS StdDevDwell,
            COUNT(*) AS SampleSize
        FROM FactWarehouseOperations w
        JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
        GROUP BY t.Hour, t.MinuteBucket, t.DayOfWeek, w.WarehouseKey
        HAVING COUNT(*) > 10
    ),
    ForecastPeriods AS (
        SELECT 
            TimeKey,
            Hour,
            MinuteBucket,
            DayOfWeek,
            DateTime
        FROM DimTime
        WHERE DateTime BETWEEN GETDATE() AND DATEADD(MINUTE, @ForecastHorizonMinutes, GETDATE())
    )
    SELECT 
        fp.DateTime AS ForecastDateTime,
        wh.WarehouseName,
        hp.AvgDwell AS ExpectedDwellMinutes,
        hp.AvgDwell + (2 * hp.StdDevDwell) AS UpperBoundDwell,
        CASE 
            WHEN hp.AvgDwell + (2 * hp.StdDevDwell) > 120 THEN 'HIGH RISK'
            WHEN hp.AvgDwell + (2 * hp.StdDevDwell) > 60 THEN 'MODERATE RISK'
            ELSE 'LOW RISK'
        END AS BottleneckRisk
    FROM ForecastPeriods fp
    JOIN HistoricalPatterns hp 
        ON fp.Hour = hp.Hour 
        AND fp.MinuteBucket = hp.MinuteBucket
        AND fp.DayOfWeek = hp.DayOfWeek
    JOIN DimWarehouse wh ON hp.WarehouseKey = wh.WarehouseKey
    WHERE hp.AvgDwell + (2 * hp.StdDevDwell) > 60
    ORDER BY BottleneckRisk DESC, fp.DateTime;
END;
GO

-- Execute prediction
EXEC sp_PredictBottlenecks @ForecastHorizonMinutes = 180;
```

## Power BI Configuration

### DAX Measures for Cross-Fact KPIs

**Composite Fleet Efficiency Score**
```dax
Fleet Efficiency Score = 
VAR AvgFuelPerKM = AVERAGE(FactFleetTrips[FuelConsumedLiters]) / AVERAGE(FactFleetTrips[DistanceKM])
VAR IdleRatio = DIVIDE(SUM(FactFleetTrips[IdleTimeMinutes]), SUM(FactFleetTrips[IdleTimeMinutes]) + SUM(FactFleetTrips[LoadingTimeMinutes]) + SUM(FactFleetTrips[UnloadingTimeMinutes]))
VAR DelayFactor = 1 - DIVIDE(SUM(FactFleetTrips[DelayMinutes]), SUM(FactFleetTrips[DistanceKM]) * 60)
RETURN
    (1 / AvgFuelPerKM) * (1 - IdleRatio) * DelayFactor * 100
```

**Warehouse Throughput Velocity**
```dax
Warehouse Velocity = 
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
VAR TotalMinutes = DISTINCTCOUNT(FactWarehouseOperations[TimeKey]) * 15
VAR AvgPickTime = AVERAGE(FactWarehouseOperations[PickingTimeSeconds])
RETURN
    DIVIDE(TotalOperations, TotalMinutes, 0) * (60 / AvgPickTime)
```

**Cross-Modal Harmony Index**
```dax
Harmony Index = 
VAR WarehouseScore = [Warehouse Velocity]
VAR FleetScore = [Fleet Efficiency Score]
VAR WarehouseNorm = DIVIDE(WarehouseScore, MAXX(ALL(FactWarehouseOperations), [Warehouse Velocity]))
VAR FleetNorm = DIVIDE(FleetScore, MAXX(ALL(FactFleetTrips), [Fleet Efficiency Score]))
RETURN
    (WarehouseNorm + FleetNorm) / 2 * 100
```

### Row-Level Security Implementation

```dax
-- Create role "Regional Manager"
-- Apply filter to DimWarehouse table:
[Region] = USERPRINCIPALNAME()

-- Or for specific users:
[Region] IN {"North", "South"} && USERNAME() = "manager@company.com"
```

## Data Loading & ETL

### Incremental Load Stored Procedure

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouse
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load new warehouse operations since last run
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType, 
        DwellTimeMinutes, PickingTimeSeconds, QuantityHandled, 
        GravityZone, EmployeeKey
    )
    SELECT 
        CONVERT(INT, FORMAT(wo.OperationDateTime, 'yyyyMMddHHmm')) AS TimeKey,
        p.ProductKey,
        wh.WarehouseKey,
        wo.OperationType,
        DATEDIFF(MINUTE, wo.StartTime, wo.EndTime) AS DwellTimeMinutes,
        wo.PickingTimeSeconds,
        wo.Quantity,
        p.GravityZone,
        e.EmployeeKey
    FROM [ExternalWMS].[dbo].[WarehouseOperations] wo
    JOIN DimProductGravity p ON wo.SKU = p.SKU
    JOIN DimWarehouse wh ON wo.WarehouseCode = wh.WarehouseCode
    JOIN DimEmployee e ON wo.EmployeeID = e.EmployeeID
    WHERE wo.OperationDateTime > @LastLoadDateTime
        AND wo.OperationDateTime <= GETDATE();
    
    -- Update load timestamp
    UPDATE ETLControl
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Automated Refresh Schedule

```sql
-- Create SQL Server Agent job for 15-minute refresh
USE msdb;
GO

EXEC sp_add_job 
    @job_name = N'LogiFleet_IncrementalRefresh',
    @enabled = 1,
    @description = N'Refresh LogiFleet Pulse data every 15 minutes';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalRefresh',
    @step_name = N'Load Warehouse Data',
    @subsystem = N'TSQL',
    @command = N'
        DECLARE @LastLoad DATETIME;
        SELECT @LastLoad = LastLoadDateTime FROM ETLControl WHERE TableName = ''FactWarehouseOperations'';
        EXEC sp_IncrementalLoadWarehouse @LastLoad;
    ',
    @database_name = N'LogiFleetPulse';

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

## Alert Configuration

### KPI Breach Notification

```sql
CREATE PROCEDURE sp_CheckKPIBreaches
AS
BEGIN
    DECLARE @AlertThresholds TABLE (
        KPIName VARCHAR(100),
        ThresholdValue DECIMAL(10,2),
        AlertEmail VARCHAR(255)
    );
    
    INSERT INTO @AlertThresholds VALUES
        ('Fleet Idle Time %', 15.0, '${FLEET_MANAGER_EMAIL}'),
        ('Warehouse Dwell Time Hours', 72.0, '${WAREHOUSE_MANAGER_EMAIL}'),
        ('Fuel Efficiency L/100km', 25.0, '${LOGISTICS_DIRECTOR_EMAIL}');
    
    -- Check Fleet Idle Time
    DECLARE @CurrentIdlePct DECIMAL(5,2);
    SELECT @CurrentIdlePct = 
        100.0 * SUM(IdleTimeMinutes) / NULLIF(SUM(IdleTimeMinutes + DATEDIFF(MINUTE, 0, CAST(DistanceKM / 60.0 AS TIME))), 0)
    FROM FactFleetTrips f
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.Date = CAST(GETDATE() AS DATE);
    
    IF @CurrentIdlePct > (SELECT ThresholdValue FROM @AlertThresholds WHERE KPIName = 'Fleet Idle Time %')
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = (SELECT AlertEmail FROM @AlertThresholds WHERE KPIName = 'Fleet Idle Time %'),
            @subject = 'ALERT: Fleet Idle Time Exceeded Threshold',
            @body = CONCAT('Current idle time: ', CAST(@CurrentIdlePct AS VARCHAR), '%. Threshold: 15%'),
            @profile_name = 'LogiFleet Alerts';
    END;
    
    -- Add similar checks for other KPIs
END;
GO
```

## Common Patterns

### Pattern 1: Time Intelligence Queries

```sql
-- Compare current week vs previous week
SELECT 
    'Current Week' AS Period,
    COUNT(*) AS TotalOperations,
    AVG(DwellTimeMinutes) AS AvgDwell
FROM FactWarehouseOperations w
JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.Week = DATEPART(WEEK, GETDATE())
    AND t.Year = YEAR(GETDATE())

UNION ALL

SELECT 
    'Previous Week' AS Period,
    COUNT(*) AS TotalOperations,
    AVG(DwellTimeMinutes) AS AvgDwell
FROM FactWarehouseOperations w
JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.Week = DATEPART(WEEK, DATEADD(WEEK, -1, GETDATE()))
    AND t.Year = YEAR(DATEADD(WEEK, -1, GETDATE()));
```

### Pattern 2: Many-to-Many Bridge Table

```sql
-- When routes serve multiple warehouses, use bridge table
CREATE TABLE BridgeRouteWarehouse (
    RouteKey INT,
    WarehouseKey INT,
    Sequence INT,
    PRIMARY KEY (RouteKey, WarehouseKey)
);

-- Query across bridge
SELECT 
    r.RouteName,
    wh.WarehouseName,
    SUM(f.DistanceKM) AS TotalDistance,
    AVG(f.FuelConsumedLiters) AS AvgFuel
FROM FactFleetTrips f
JOIN BridgeRouteWarehouse brw ON f.RouteKey = brw.RouteKey
JOIN DimWarehouse wh ON brw.WarehouseKey = wh.WarehouseKey
JOIN DimRoute r ON f.RouteKey = r.RouteKey
GROUP BY r.RouteName, wh.WarehouseName;
```

### Pattern 3: External Data Enrichment

```sql
-- Link weather delays to fleet performance
CREATE EXTERNAL TABLE WeatherData (
    LocationKey INT,
    DateTime DATETIME,
    Condition VARCHAR(50),
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2)
)
WITH (
    LOCATION = '${WEATHER_API_URL}',
    DATA_SOURCE = WeatherAPISource,
    FILE_FORMAT = JSONFormat
);

-- Analyze weather impact
SELECT 
    wd.Condition,
    COUNT(f.TripID) AS AffectedTrips,
    AVG(f.DelayMinutes) AS AvgDelay,
    AVG(f.FuelConsumedLiters) AS AvgFuel
FROM FactFleetTrips f
JOIN DimTime t ON f.TimeKey = t.TimeKey
JOIN DimGeography g ON f.DestinationKey = g.GeographyKey
LEFT JOIN WeatherData wd ON g.LocationKey = wd.LocationKey 
    AND ABS(DATEDIFF(MINUTE, t.DateTime, wd.DateTime)) < 30
GROUP BY wd.Condition
ORDER BY AvgDelay DESC;
```

## Troubleshooting

### Issue: Slow Dashboard Load Times

**Symptoms:** Power BI reports take >30 seconds to render

**Solutions:**
1. Check columnstore index fragmentation:
```sql
SELECT 
    OBJECT_NAME(object_id) AS TableName,
    index_id,
    partition_number,
    delta_store_hobt_id
FROM sys.column_store_row_groups
WHERE state_desc = 'OPEN' OR deleted_rows > total_rows * 0.1;

-- Rebuild if needed
ALTER INDEX NCCI_FactWarehouse ON FactWarehouseOperations REBUILD;
```

2. Implement aggregation tables:
```sql
CREATE TABLE AggWarehouseDaily (
    Date DATE,
    WarehouseKey INT,
    ProductKey INT,
    TotalOperations INT,
    AvgDwellTime DECIMAL(10,2),
    TotalQuantity INT
);

-- Populate nightly
INSERT INTO AggWarehouseDaily
SELECT 
    t.Date,
    w.WarehouseKey,
    w.ProductKey,
    COUNT(*) AS TotalOperations,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    SUM(w.QuantityHandled) AS TotalQuantity
FROM FactWarehouseOperations w
JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.Date = CAST(GETDATE() - 1 AS DATE)
GROUP BY t.Date, w.WarehouseKey, w.ProductKey;
```

3. Enable Power BI aggregations feature in model

### Issue: Data Latency Exceeds 15 Minutes

**Symptoms:** Real-time dashboards show stale data

**Solutions:**
1. Check SQL Agent job history:
```sql
SELECT 
    j.name AS JobName,
    h.run_date,
    h.run_time,
    h.run_duration,
    h.message
FROM msdb.dbo.sysjobhistory h
JOIN msdb.dbo.sysjobs j ON h.job_id = j.job_id
WHERE j.name = 'LogiFleet_IncrementalRefresh'
ORDER BY h.run_date DESC, h.run_time DESC;
```

2. Verify external table connectivity:
```sql
SELECT * FROM sys.external_data_sources;
-- Test connection
SELECT TOP 1 * FROM [ExternalWMS].[dbo].[WarehouseOperations];
```

3. Enable query store for performance analysis:
```sql
ALTER DATABASE LogiFleetPulse 
SET QUERY_STORE = ON (
    OPERATION_MODE = READ_WRITE,
    MAX_STORAGE_SIZE_MB = 1000
);
```

### Issue: Cross-Fact Queries Return No Results

**Symptoms:** Joins between FactWarehouseOperations and FactFleetTrips return empty sets

**Solutions:**
1. Verify bridge table population:
```sql
-- Check if cross-dock records exist
SELECT COUNT(*) FROM FactCrossDock 
WHERE InboundTripID IS NOT NULL AND OutboundTripID IS NOT NULL;

-- Verify time alignment
SELECT 
    MIN(t.DateTime) AS EarliestWarehouse,
    MAX(t.DateTime) AS LatestWarehouse
FROM FactWarehouseOperations w
JOIN DimTime t ON w.TimeKey = t.TimeKey;

SELECT 
    MIN(t.DateTime) AS EarliestFleet,
    MAX(t.DateTime) AS LatestFleet
FROM FactFleetTrips f
JOIN DimTime t ON f.TimeKey = t.TimeKey;
```

2. Use conformed dimensions properly:
```sql
-- Ensure both facts use same TimeKey format
SELECT DISTINCT TimeKey FROM FactWarehouseOperations
INTERSECT
SELECT DISTINCT TimeKey FROM FactFleetTrips;
```

### Issue: Gravity Zone Recommendations Not Updating

**Symptoms:** sp_PredictBottlenecks shows outdated patterns

**Solutions:**
1. Refresh computed columns:
```sql
-- Force recalculation
UPDATE D
