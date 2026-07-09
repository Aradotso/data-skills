---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics intelligence with multi-fact star schema, fleet optimization, and warehouse analytics
triggers:
  - set up logistics data warehouse with power bi
  - configure supply chain analytics dashboard
  - implement warehouse and fleet tracking system
  - create multi-fact star schema for logistics
  - build real-time fleet optimization dashboard
  - design cross-modal supply chain data model
  - integrate warehouse operations with fleet telemetry
  - deploy logifleet pulse analytics platform
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform combining MS SQL Server data warehousing with Power BI visualization. It provides a unified semantic layer for warehouse operations, fleet telemetry, inventory management, and supply chain analytics through a multi-fact star schema architecture.

**Key Capabilities:**
- Multi-fact data warehouse with time-phased dimensions
- Unified warehouse and fleet operations analytics
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Real-time fleet triage and maintenance prioritization
- Cross-fact KPI harmonization

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) recommended
- .NET Framework 4.7.2+ for SQL CLR functions

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Execute the schema scripts in order:**
```sql
-- Connect to your SQL Server instance
-- Run scripts in this sequence:

-- 1. Create database
USE master;
GO
CREATE DATABASE LogiFleetPulse;
GO

-- 2. Create dimension tables
USE LogiFleetPulse;
GO
:r schema/01_DimensionTables.sql

-- 3. Create fact tables
:r schema/02_FactTables.sql

-- 4. Create bridge tables
:r schema/03_BridgeTables.sql

-- 5. Create views and stored procedures
:r schema/04_Views.sql
:r schema/05_StoredProcedures.sql

-- 6. Create indexes
:r schema/06_Indexes.sql
```

3. **Configure data sources in `config.json`:**
```json
{
  "dataConnections": {
    "wmsConnection": "Server=${WMS_SQL_SERVER};Database=${WMS_DATABASE};Integrated Security=true;",
    "telematicsApi": "${TELEMATICS_API_ENDPOINT}",
    "weatherApi": "${WEATHER_API_ENDPOINT}",
    "erpConnection": "Server=${ERP_SQL_SERVER};Database=${ERP_DATABASE};Integrated Security=true;"
  },
  "refreshSchedule": {
    "intervalMinutes": 15,
    "alertThresholds": {
      "fleetIdleTimePercent": 15,
      "warehouseDwellHours": 72,
      "temperatureDeviationMinutes": 20
    }
  }
}
```

### Power BI Template Setup

1. **Open the template:**
```bash
# Open Power BI Desktop and load the template
start LogiFleet_Pulse_Master.pbit
```

2. **Configure connection parameters:**
- SQL Server instance name
- Database name: `LogiFleetPulse`
- Authentication method (Windows or SQL Server)

3. **Apply row-level security (optional):**
```sql
-- Create security roles in Power BI
-- Define RLS using this DAX pattern:
[UserEmail] = USERPRINCIPALNAME()
```

## Core Data Model

### Dimension Tables

**DimTime** - Time intelligence with 15-minute granularity:
```sql
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod VARCHAR(10),
    QuarterYear VARCHAR(10)
);

-- Usage example: Join any fact table on TimeKey
SELECT 
    dt.FullDateTime,
    dt.HourOfDay,
    SUM(f.TripCount) as TotalTrips
FROM FactFleetTrips f
INNER JOIN DimTime dt ON f.TimeKey = dt.TimeKey
WHERE dt.DateKey = CONVERT(INT, FORMAT(GETDATE(), 'yyyyMMdd'))
GROUP BY dt.FullDateTime, dt.HourOfDay;
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE dbo.DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50), -- 'Warehouse', 'RouteNode', 'CrossDock'
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    RegionName VARCHAR(100),
    CountryName VARCHAR(100),
    ContinentName VARCHAR(50),
    IsActive BIT DEFAULT 1
);
```

**DimProductGravity** - Product classification with warehouse gravity score:
```sql
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueClass VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityScore DECIMAL(3,2),
    OptimalZone VARCHAR(50),
    LastRecalculated DATETIME
);

-- Calculate gravity score
UPDATE dbo.DimProductGravity
SET GravityScore = (
    (CASE VelocityClass 
        WHEN 'Fast' THEN 3 
        WHEN 'Medium' THEN 2 
        ELSE 1 END) *
    (CASE ValueClass 
        WHEN 'High' THEN 3 
        WHEN 'Medium' THEN 2 
        ELSE 1 END) *
    FragilityScore
);
```

### Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics:
```sql
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID VARCHAR(100),
    DwellTimeHours DECIMAL(10,2),
    ProcessingTimeMinutes DECIMAL(10,2),
    StorageZone VARCHAR(50),
    QuantityHandled INT,
    BatchNumber VARCHAR(50),
    OperatorID VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time 
ON dbo.FactWarehouseOperations(TimeKey) INCLUDE (DwellTimeHours, ProcessingTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product 
ON dbo.FactWarehouseOperations(ProductKey) INCLUDE (QuantityHandled, StorageZone);
```

**FactFleetTrips** - Fleet telemetry and trip data:
```sql
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripID VARCHAR(100),
    RouteID VARCHAR(100),
    TripDistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    AverageSpeedKmh DECIMAL(5,2),
    DriverID VARCHAR(50),
    MaintenanceFlag BIT DEFAULT 0,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time 
ON dbo.FactFleetTrips(TimeKey) INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
```

**FactCrossDock** - Cross-dock transfer operations:
```sql
CREATE TABLE dbo.FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferID VARCHAR(100),
    DwellTimeMinutes DECIMAL(10,2),
    QuantityTransferred INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey)
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @LoadDateFrom DATETIME,
    @LoadDateTo DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Stage data from WMS
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationID, DwellTimeHours, ProcessingTimeMinutes,
        StorageZone, QuantityHandled, BatchNumber, OperatorID
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        wms.OperationType,
        wms.OperationID,
        wms.DwellTimeHours,
        wms.ProcessingTimeMinutes,
        wms.StorageZone,
        wms.QuantityHandled,
        wms.BatchNumber,
        wms.OperatorID
    FROM [WMS].[dbo].[Operations] wms
    INNER JOIN dbo.DimTime dt ON DATEADD(MINUTE, (DATEPART(MINUTE, wms.OperationDateTime) / 15) * 15, 
                                         CAST(CAST(wms.OperationDateTime AS DATE) AS DATETIME) + 
                                         CAST(CAST(DATEPART(HOUR, wms.OperationDateTime) AS VARCHAR) + ':00' AS TIME)) = dt.FullDateTime
    INNER JOIN dbo.DimGeography dg ON wms.WarehouseID = dg.LocationID
    INNER JOIN dbo.DimProductGravity dp ON wms.SKU = dp.SKU
    WHERE wms.OperationDateTime >= @LoadDateFrom 
      AND wms.OperationDateTime < @LoadDateTo
      AND NOT EXISTS (
          SELECT 1 FROM dbo.FactWarehouseOperations f 
          WHERE f.OperationID = wms.OperationID
      );
      
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' warehouse operations';
END;
GO
```

### Cross-Fact Analysis

```sql
CREATE PROCEDURE dbo.usp_AnalyzeProductFlowEfficiency
    @AnalysisDateFrom DATE,
    @AnalysisDateTo DATE
AS
BEGIN
    -- Cross-fact query: warehouse dwell time vs fleet idle time by product
    SELECT 
        dp.SKU,
        dp.ProductName,
        dp.GravityScore,
        AVG(fw.DwellTimeHours) as AvgWarehouseDwellHours,
        AVG(ff.IdleTimeMinutes) as AvgFleetIdleMinutes,
        COUNT(DISTINCT fw.OperationKey) as WarehouseOperations,
        COUNT(DISTINCT ff.TripKey) as FleetTrips,
        -- Efficiency score: lower is better
        (AVG(fw.DwellTimeHours) * 60 + AVG(ff.IdleTimeMinutes)) / NULLIF(dp.GravityScore, 0) as EfficiencyScore
    FROM dbo.DimProductGravity dp
    LEFT JOIN dbo.FactWarehouseOperations fw ON dp.ProductKey = fw.ProductKey
    LEFT JOIN dbo.DimTime dt_w ON fw.TimeKey = dt_w.TimeKey
    LEFT JOIN dbo.FactFleetTrips ff ON dp.ProductKey = ff.ProductKey -- Requires bridge table in production
    LEFT JOIN dbo.DimTime dt_f ON ff.TimeKey = dt_f.TimeKey
    WHERE (dt_w.DateKey >= CONVERT(INT, FORMAT(@AnalysisDateFrom, 'yyyyMMdd'))
       AND dt_w.DateKey <= CONVERT(INT, FORMAT(@AnalysisDateTo, 'yyyyMMdd')))
       OR (dt_f.DateKey >= CONVERT(INT, FORMAT(@AnalysisDateFrom, 'yyyyMMdd'))
       AND dt_f.DateKey <= CONVERT(INT, FORMAT(@AnalysisDateTo, 'yyyyMMdd')))
    GROUP BY dp.SKU, dp.ProductName, dp.GravityScore
    HAVING COUNT(DISTINCT fw.OperationKey) > 0
    ORDER BY EfficiencyScore DESC;
END;
GO
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE dbo.usp_IdentifyBottleneckRisks
AS
BEGIN
    -- Detect high-risk areas based on trending metrics
    WITH WarehouseMetrics AS (
        SELECT 
            dg.LocationName,
            dp.Category,
            AVG(DwellTimeHours) OVER (PARTITION BY fw.GeographyKey, fw.ProductKey 
                                      ORDER BY dt.FullDateTime 
                                      ROWS BETWEEN 95 PRECEDING AND CURRENT ROW) as MA96_DwellTime,
            fw.DwellTimeHours as CurrentDwellTime,
            dt.FullDateTime
        FROM dbo.FactWarehouseOperations fw
        INNER JOIN dbo.DimTime dt ON fw.TimeKey = dt.TimeKey
        INNER JOIN dbo.DimGeography dg ON fw.GeographyKey = dg.GeographyKey
        INNER JOIN dbo.DimProductGravity dp ON fw.ProductKey = dp.ProductKey
        WHERE dt.FullDateTime >= DATEADD(DAY, -7, GETDATE())
    )
    SELECT 
        LocationName,
        Category,
        AVG(CurrentDwellTime) as CurrentAvgDwell,
        AVG(MA96_DwellTime) as HistoricalAvgDwell,
        (AVG(CurrentDwellTime) - AVG(MA96_DwellTime)) / NULLIF(AVG(MA96_DwellTime), 0) * 100 as PercentIncrease,
        CASE 
            WHEN (AVG(CurrentDwellTime) - AVG(MA96_DwellTime)) / NULLIF(AVG(MA96_DwellTime), 0) > 0.25 THEN 'High Risk'
            WHEN (AVG(CurrentDwellTime) - AVG(MA96_DwellTime)) / NULLIF(AVG(MA96_DwellTime), 0) > 0.10 THEN 'Medium Risk'
            ELSE 'Low Risk'
        END as RiskLevel
    FROM WarehouseMetrics
    GROUP BY LocationName, Category
    HAVING (AVG(CurrentDwellTime) - AVG(MA96_DwellTime)) / NULLIF(AVG(MA96_DwellTime), 0) > 0.10
    ORDER BY PercentIncrease DESC;
END;
GO
```

## Power BI DAX Measures

### Cross-Fact KPIs

```dax
// Total Logistics Cost (warehouse + fleet)
Total Logistics Cost = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        [ProcessingTimeMinutes] * [WarehouseLaborCostPerMinute]
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        [FuelConsumedLiters] * [FuelCostPerLiter] + 
        [IdleTimeMinutes] * [DriverCostPerMinute]
    )
RETURN WarehouseCost + FleetCost

// Fleet Idle Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + 
    SUM(FactFleetTrips[TripDistanceKm]) / AVERAGE(FactFleetTrips[AverageSpeedKmh]) * 60,
    0
)

// Warehouse Gravity Score Alignment
Gravity Alignment Score = 
VAR ActualZone = SELECTEDVALUE(FactWarehouseOperations[StorageZone])
VAR OptimalZone = SELECTEDVALUE(DimProductGravity[OptimalZone])
RETURN IF(ActualZone = OptimalZone, 1, 0)

// Weighted Gravity Alignment %
Weighted Gravity Alignment = 
AVERAGEX(
    FactWarehouseOperations,
    IF([StorageZone] = RELATED(DimProductGravity[OptimalZone]), 1, 0)
)

// Predictive Bottleneck Index
Bottleneck Index = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR HistoricalDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -30, DAY)
    )
VAR DwellTrend = DIVIDE(CurrentDwell - HistoricalDwell, HistoricalDwell, 0)
VAR IdlePercent = [Fleet Idle %]
RETURN (DwellTrend * 0.6) + (IdlePercent * 0.4) // Weighted composite score
```

### Time Intelligence

```dax
// Rolling 96-Period (24 hour) Average Dwell Time
Rolling 24hr Dwell = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -24,
        HOUR
    )
)

// Week over Week Change
WoW Change % = 
VAR CurrentWeek = [Total Logistics Cost]
VAR PreviousWeek = 
    CALCULATE(
        [Total Logistics Cost],
        DATEADD(DimTime[FullDateTime], -7, DAY)
    )
RETURN DIVIDE(CurrentWeek - PreviousWeek, PreviousWeek, 0)
```

## Automation & Alerts

### Scheduled Data Refresh

```sql
-- SQL Server Agent Job script
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_IncrementalRefresh',
    @enabled = 1,
    @description = N'15-minute incremental refresh for LogiFleet Pulse';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalRefresh',
    @step_name = N'Load Warehouse Data',
    @subsystem = N'TSQL',
    @database_name = N'LogiFleetPulse',
    @command = N'
        DECLARE @LoadFrom DATETIME = DATEADD(MINUTE, -20, GETDATE());
        DECLARE @LoadTo DATETIME = GETDATE();
        EXEC dbo.usp_LoadWarehouseOperations @LoadFrom, @LoadTo;
        EXEC dbo.usp_LoadFleetTrips @LoadFrom, @LoadTo;
    ',
    @retry_attempts = 3,
    @retry_interval = 5;

EXEC sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15,
    @active_start_time = 000000;

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalRefresh',
    @schedule_name = N'Every15Minutes';

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_IncrementalRefresh',
    @server_name = N'(LOCAL)';
GO
```

### Email Alerts for KPI Breaches

```sql
CREATE PROCEDURE dbo.usp_SendAlerts
AS
BEGIN
    DECLARE @AlertBody NVARCHAR(MAX);
    
    -- Check for high idle time
    IF EXISTS (
        SELECT 1 
        FROM dbo.FactFleetTrips ff
        INNER JOIN dbo.DimTime dt ON ff.TimeKey = dt.TimeKey
        WHERE dt.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
          AND (ff.IdleTimeMinutes / NULLIF(
              ff.TripDistanceKm / NULLIF(ff.AverageSpeedKmh, 0) * 60, 0)) > 0.15
    )
    BEGIN
        SET @AlertBody = 'Fleet idle time exceeded 15% threshold in the last hour. Check dashboard for details.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = '${DB_MAIL_PROFILE}',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Alert: High Fleet Idle Time',
            @body = @AlertBody;
    END;
    
    -- Check for extended dwell time
    IF EXISTS (
        SELECT 1 
        FROM dbo.FactWarehouseOperations fw
        INNER JOIN dbo.DimTime dt ON fw.TimeKey = dt.TimeKey
        WHERE dt.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
          AND fw.DwellTimeHours > 72
    )
    BEGIN
        SET @AlertBody = 'Items with dwell time exceeding 72 hours detected. Review warehouse operations.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = '${DB_MAIL_PROFILE}',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Alert: Extended Dwell Time',
            @body = @AlertBody;
    END;
END;
GO
```

## Common Patterns

### Pattern 1: Warehouse Gravity Zone Optimization

```sql
-- Identify products in suboptimal zones
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.OptimalZone,
    fw.StorageZone as CurrentZone,
    COUNT(*) as Operations,
    AVG(fw.DwellTimeHours) as AvgDwell,
    AVG(fw.ProcessingTimeMinutes) as AvgProcessing
FROM dbo.DimProductGravity dp
INNER JOIN dbo.FactWarehouseOperations fw ON dp.ProductKey = fw.ProductKey
INNER JOIN dbo.DimTime dt ON fw.TimeKey = dt.TimeKey
WHERE dt.DateKey >= CONVERT(INT, FORMAT(DATEADD(MONTH, -1, GETDATE()), 'yyyyMMdd'))
  AND fw.StorageZone <> dp.OptimalZone
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, dp.OptimalZone, fw.StorageZone
HAVING COUNT(*) > 10
ORDER BY dp.GravityScore DESC, AVG(fw.DwellTimeHours) DESC;
```

### Pattern 2: Fleet Maintenance Prioritization

```sql
-- Adaptive fleet triage based on trip data
WITH FleetMetrics AS (
    SELECT 
        VehicleKey,
        AVG(FuelConsumedLiters / NULLIF(TripDistanceKm, 0)) as AvgFuelEfficiency,
        AVG(IdleTimeMinutes) as AvgIdleTime,
        SUM(LoadWeightKg) as TotalLoadWeight,
        COUNT(*) as TripCount
    FROM dbo.FactFleetTrips
    WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -7, GETDATE()) ORDER BY TimeKey LIMIT 1)
    GROUP BY VehicleKey
)
SELECT 
    fm.VehicleKey,
    fm.AvgFuelEfficiency,
    fm.AvgIdleTime,
    fm.TripCount,
    -- Maintenance priority score (higher = more urgent)
    (fm.AvgFuelEfficiency * 100) + -- Poor efficiency
    (fm.AvgIdleTime / 10) + -- High idle time
    (fm.TripCount / 5) as MaintenancePriorityScore
FROM FleetMetrics fm
WHERE fm.AvgFuelEfficiency > (SELECT AVG(AvgFuelEfficiency) * 1.15 FROM FleetMetrics) -- 15% worse than average
   OR fm.AvgIdleTime > 30 -- More than 30 minutes average idle
ORDER BY MaintenancePriorityScore DESC;
```

### Pattern 3: Cross-Dock Efficiency Analysis

```sql
-- Analyze cross-dock transfer efficiency
SELECT 
    dg.LocationName as CrossDockLocation,
    dp.Category,
    COUNT(DISTINCT fcd.CrossDockKey) as TotalTransfers,
    AVG(fcd.DwellTimeMinutes) as AvgDwellMinutes,
    AVG(ffi.IdleTimeMinutes) as AvgInboundIdle,
    AVG(ffo.IdleTimeMinutes) as AvgOutboundIdle,
    -- Total cross-dock time
    AVG(fcd.DwellTimeMinutes + ffi.IdleTimeMinutes + ffo.IdleTimeMinutes) as TotalCrossDockTime
FROM dbo.FactCrossDock fcd
INNER JOIN dbo.DimGeography dg ON fcd.GeographyKey = dg.GeographyKey
INNER JOIN dbo.DimProductGravity dp ON fcd.ProductKey = dp.ProductKey
LEFT JOIN dbo.FactFleetTrips ffi ON fcd.InboundTripKey = ffi.TripKey
LEFT JOIN dbo.FactFleetTrips ffo ON fcd.OutboundTripKey = ffo.TripKey
INNER JOIN dbo.DimTime dt ON fcd.TimeKey = dt.TimeKey
WHERE dt.DateKey >= CONVERT(INT, FORMAT(DATEADD(MONTH, -1, GETDATE()), 'yyyyMMdd'))
GROUP BY dg.LocationName, dp.Category
ORDER BY TotalCrossDockTime DESC;
```

## Troubleshooting

### Issue: Slow Query Performance

**Symptom:** Power BI dashboards take >30 seconds to load

**Solutions:**
1. Verify indexes are present:
```sql
-- Check for missing indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.avg_fragmentation_in_percent > 30
ORDER BY s.avg_fragmentation_in_percent DESC;
```

2. Rebuild fragmented indexes:
```sql
ALTER INDEX ALL ON dbo.FactWarehouseOperations REBUILD;
ALTER INDEX ALL ON dbo.FactFleetTrips REBUILD;
```

3. Update statistics:
```sql
UPDATE STATISTICS dbo.FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS dbo.FactFleetTrips WITH FULLSCAN;
```

### Issue: Dimension Mismatch Errors

**Symptom:** "Could not find relationship" errors in Power BI

**Solution:** Verify referential integrity:
```sql
-- Check for orphaned fact records
SELECT 'FactWarehouseOperations' as TableName, COUNT(*) as OrphanedRecords
FROM dbo.FactWarehouseOperations fw
WHERE NOT EXISTS (SELECT 1 FROM DimTime WHERE TimeKey = fw.TimeKey)
   OR NOT EXISTS (SELECT 1 FROM DimGeography WHERE GeographyKey = fw.GeographyKey)
   OR NOT EXISTS (SELECT 1 FROM DimProductGravity WHERE ProductKey = fw.ProductKey)

UNION ALL

SELECT 'FactFleetTrips', COUNT(*)
FROM dbo.FactFleetTrips ff
WHERE NOT EXISTS (SELECT 1 FROM DimTime WHERE TimeKey = ff.TimeKey)
   OR NOT EXISTS (SELECT 1 FROM DimGeography WHERE GeographyKey = ff.OriginGeographyKey)
   OR NOT EXISTS (SELECT 1 FROM DimGeography WHERE GeographyKey = ff.DestinationGeographyKey);
```

### Issue: Duplicate Time Keys

**Symptom:** Inflated metrics in dashboards

**Solution:** Ensure DimTime has proper granularity:
```sql
-- Check for duplicate time keys
SELECT TimeKey, COUNT(*) as DuplicateCount
FROM dbo.DimTime
GROUP BY TimeKey
HAVING COUNT(*) > 1;

-- If duplicates exist, rebuild DimTime with proper 15-minute rounding
TRUNCATE TABLE dbo.DimTime;

WITH TimeSequence AS (
    SELECT 
        DATEADD(MINUTE, 15 * ROW_NUMBER() OVER (ORDER BY (SELECT NULL)), '2020-01-01') as FullDateTime
    FROM sys.all_objects a
    CROSS JOIN sys.all_objects b
    WHERE DATE
