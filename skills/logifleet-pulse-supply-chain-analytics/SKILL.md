---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server + Power BI supply chain analytics engine with multi-fact star schema for warehouse, fleet, and logistics intelligence
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - create multi-fact star schema for supply chain
  - implement warehouse and fleet analytics dashboard
  - configure logifleet pulse sql database
  - build logistics intelligence data model
  - integrate warehouse operations with fleet telemetry
  - set up supply chain kpi harmonization
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI visualization platform for unified supply chain analytics. It implements a multi-fact star schema that harmonizes warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for comprehensive logistics intelligence.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time operational dashboards (15-minute refresh cycles)
- Predictive bottleneck detection and fleet triage
- Warehouse gravity zone optimization
- Role-based access control with row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- WMS/TMS data sources (REST API, SQL export, or flat files)

### Database Deployment

1. **Clone or download the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Run the main schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse
ON PRIMARY 
(
    NAME = N'LogiFleetPulse_Data',
    FILENAME = N'C:\SQLData\LogiFleetPulse.mdf',
    SIZE = 512MB,
    MAXSIZE = UNLIMITED,
    FILEGROWTH = 64MB
)
LOG ON
(
    NAME = N'LogiFleetPulse_Log',
    FILENAME = N'C:\SQLData\LogiFleetPulse_log.ldf',
    SIZE = 128MB,
    MAXSIZE = 2GB,
    FILEGROWTH = 64MB
);
GO

USE LogiFleetPulse;
GO

-- Execute schema scripts in order
-- (scripts should be provided in /sql directory)
```

3. **Configure connection strings:**
```json
{
  "data_sources": {
    "wms_connection": "Server=${WMS_SQL_SERVER};Database=${WMS_DATABASE};User Id=${WMS_USER};Password=${WMS_PASSWORD};",
    "telemetry_api": "${FLEET_TELEMETRY_API_ENDPOINT}",
    "telemetry_api_key": "${FLEET_API_KEY}",
    "erp_connection": "Server=${ERP_SQL_SERVER};Database=${ERP_DATABASE};Integrated Security=true;"
  },
  "refresh_interval_minutes": 15,
  "alert_email_smtp": "${SMTP_SERVER}",
  "alert_email_from": "${ALERT_FROM_EMAIL}"
}
```

## Core Data Model Structure

### Fact Tables

#### FactWarehouseOperations
```sql
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2 NOT NULL,
    OperationEndTime DATETIME2 NOT NULL,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime),
    DwellTimeHours DECIMAL(10,2),
    QuantityHandled INT NOT NULL,
    EmployeeKey INT,
    EquipmentKey INT,
    ZoneKey INT,
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Date FOREIGN KEY (DateKey) REFERENCES dbo.DimDate(DateKey),
    CONSTRAINT FK_WarehouseOps_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES dbo.DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES dbo.DimProduct(ProductKey)
);

-- Create columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOperations
ON dbo.FactWarehouseOperations (TimeKey, DateKey, WarehouseKey, ProductKey, OperationType, DurationMinutes, QuantityHandled);
GO
```

#### FactFleetTrips
```sql
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripStartTimeKey INT NOT NULL,
    TripEndTimeKey INT NOT NULL,
    TripDateKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripStartOdometer DECIMAL(12,2),
    TripEndOdometer DECIMAL(12,2),
    DistanceMiles AS (TripEndOdometer - TripStartOdometer),
    FuelConsumedGallons DECIMAL(10,3),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TotalStopCount INT,
    WeatherConditionKey INT,
    OnTimeDeliveryFlag BIT,
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES dbo.DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrips_Route FOREIGN KEY (RouteKey) REFERENCES dbo.DimRoute(RouteKey)
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON dbo.FactFleetTrips (TripDateKey, VehicleKey, RouteKey, DistanceMiles, FuelConsumedGallons, IdleTimeMinutes);
GO
```

### Dimension Tables

#### DimTime (15-minute granularity)
```sql
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    TimeValue TIME(0) NOT NULL,
    HourNumber INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    HourName VARCHAR(10),
    DayPartName VARCHAR(20), -- 'Morning', 'Afternoon', 'Evening', 'Night'
    IsBusinessHours BIT,
    ShiftName VARCHAR(20) -- 'First', 'Second', 'Third', 'Weekend'
);

-- Populate DimTime with 15-minute intervals
DECLARE @Time TIME = '00:00:00';
DECLARE @Counter INT = 0;

WHILE @Counter < 96 -- 24 hours * 4 (15-min buckets)
BEGIN
    INSERT INTO dbo.DimTime (TimeKey, TimeValue, HourNumber, MinuteBucket, IsBusinessHours)
    VALUES (
        @Counter,
        @Time,
        DATEPART(HOUR, @Time),
        DATEPART(MINUTE, @Time),
        CASE WHEN DATEPART(HOUR, @Time) BETWEEN 8 AND 17 THEN 1 ELSE 0 END
    );
    
    SET @Time = DATEADD(MINUTE, 15, @Time);
    SET @Counter = @Counter + 1;
END;
GO
```

#### DimProductGravity
```sql
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200),
    CategoryHierarchy NVARCHAR(500),
    GravityScore DECIMAL(5,2), -- Calculated metric
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueClass VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityScore INT, -- 1-10
    RecommendedZoneType VARCHAR(50),
    LastRecalculatedDate DATE
);

-- Calculate gravity score based on velocity, value, and fragility
CREATE PROCEDURE dbo.sp_RecalculateProductGravity
AS
BEGIN
    UPDATE p
    SET 
        p.GravityScore = (
            (v.PickFrequency * 0.4) + 
            (p.UnitValue / 100.0 * 0.3) + 
            (p.FragilityScore * 0.3)
        ),
        p.VelocityClass = CASE 
            WHEN v.PickFrequency > 100 THEN 'Fast'
            WHEN v.PickFrequency > 20 THEN 'Medium'
            ELSE 'Slow'
        END,
        p.LastRecalculatedDate = GETDATE()
    FROM dbo.DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) AS PickFrequency
        FROM dbo.FactWarehouseOperations
        WHERE 
            OperationType = 'Picking'
            AND DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd'))
        GROUP BY ProductKey
    ) v ON p.ProductKey = v.ProductKey;
END;
GO
```

## Cross-Fact Analysis Queries

### Warehouse Dwell Time vs Fleet Idle Time Correlation
```sql
-- Identify products with high warehouse dwell time that correlate with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        w.ProductKey,
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeHours) AS AvgDwellHours,
        COUNT(*) AS OperationCount
    FROM dbo.FactWarehouseOperations w
    INNER JOIN dbo.DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE 
        w.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd'))
        AND w.OperationType = 'Shipping'
    GROUP BY w.ProductKey, p.SKU, p.ProductName
),
FleetIdle AS (
    SELECT 
        sl.ProductKey,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        COUNT(DISTINCT ft.TripKey) AS TripCount
    FROM dbo.FactFleetTrips ft
    INNER JOIN dbo.BridgeShipmentLoad sl ON ft.TripKey = sl.TripKey
    WHERE ft.TripDateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd'))
    GROUP BY sl.ProductKey
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellHours,
    fi.AvgIdleMinutes,
    -- Correlation indicator
    CASE 
        WHEN wd.AvgDwellHours > 72 AND fi.AvgIdleMinutes > 30 THEN 'High Risk'
        WHEN wd.AvgDwellHours > 48 OR fi.AvgIdleMinutes > 20 THEN 'Moderate Risk'
        ELSE 'Low Risk'
    END AS RiskLevel,
    wd.OperationCount,
    fi.TripCount
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.ProductKey = fi.ProductKey
WHERE wd.AvgDwellHours > 24
ORDER BY wd.AvgDwellHours DESC, fi.AvgIdleMinutes DESC;
```

### Warehouse Gravity Zone Optimization
```sql
-- Recommend zone reassignments based on gravity scores
WITH CurrentZonePerformance AS (
    SELECT 
        z.ZoneKey,
        z.ZoneName,
        z.DistanceFromShippingDock,
        p.ProductKey,
        p.SKU,
        p.GravityScore,
        AVG(wo.DurationMinutes) AS AvgPickTimeMinutes,
        COUNT(*) AS PickCount
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimZone z ON wo.ZoneKey = z.ZoneKey
    INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE 
        wo.OperationType = 'Picking'
        AND wo.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd'))
    GROUP BY z.ZoneKey, z.ZoneName, z.DistanceFromShippingDock, p.ProductKey, p.SKU, p.GravityScore
),
OptimalZones AS (
    SELECT 
        ProductKey,
        SKU,
        GravityScore,
        CurrentZoneName = ZoneName,
        CurrentDistance = DistanceFromShippingDock,
        RecommendedDistance = CASE 
            WHEN GravityScore > 7 THEN 20  -- High gravity: close to dock
            WHEN GravityScore > 4 THEN 50  -- Medium gravity: mid-range
            ELSE 100                        -- Low gravity: farther zones
        END
    FROM CurrentZonePerformance
)
SELECT 
    oz.SKU,
    oz.GravityScore,
    oz.CurrentZoneName,
    oz.CurrentDistance,
    oz.RecommendedDistance,
    CASE 
        WHEN ABS(oz.CurrentDistance - oz.RecommendedDistance) > 30 THEN 'Immediate Reassignment'
        WHEN ABS(oz.CurrentDistance - oz.RecommendedDistance) > 15 THEN 'Consider Reassignment'
        ELSE 'Optimal Placement'
    END AS RecommendationLevel
FROM OptimalZones oz
WHERE ABS(oz.CurrentDistance - oz.RecommendedDistance) > 15
ORDER BY ABS(oz.CurrentDistance - oz.RecommendedDistance) DESC;
```

## Power BI Integration

### Connecting Power BI to SQL Server

1. **Open Power BI Desktop**
2. **Get Data > SQL Server**
3. **Enter connection details:**
```
Server: ${SQL_SERVER_NAME}
Database: LogiFleetPulse
Data Connectivity mode: DirectQuery (recommended for real-time) or Import
```

4. **Load fact and dimension tables**

### Key DAX Measures

#### Cross-Fact Fleet Efficiency Score
```dax
Fleet Efficiency Score = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceMiles])
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedGallons])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalTripTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR IdlePercentage = DIVIDE(TotalIdleTime, TotalTripTime, 0)
VAR MPG = DIVIDE(TotalDistance, TotalFuel, 0)

RETURN
    (MPG * 0.4) + ((1 - IdlePercentage) * 100 * 0.6)
```

#### Warehouse Throughput Rate
```dax
Warehouse Throughput (Units/Hour) = 
VAR TotalUnits = SUM(FactWarehouseOperations[QuantityHandled])
VAR TotalHours = SUM(FactWarehouseOperations[DurationMinutes]) / 60

RETURN
    DIVIDE(TotalUnits, TotalHours, 0)
```

#### Predictive Bottleneck Index
```dax
Bottleneck Risk Index = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR CapacityUtilization = DIVIDE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    MAX(DimWarehouse[MaxCapacity]),
    0
)

VAR DwellScore = MIN(AvgDwellTime / 48 * 100, 100)  -- Normalize to 100
VAR IdleScore = MIN(AvgIdleTime / 60 * 100, 100)
VAR CapacityScore = CapacityUtilization * 100

RETURN
    (DwellScore * 0.4) + (IdleScore * 0.3) + (CapacityScore * 0.3)
```

### Row-Level Security Configuration
```dax
-- Create role: Regional Managers
[DimGeography[Region]] = USERNAME()

-- Create role: Warehouse Supervisors
[DimWarehouse[WarehouseCode]] IN 
    LOOKUPVALUE(
        UserWarehouseAccess[WarehouseCode],
        UserWarehouseAccess[Username],
        USERNAME()
    )
```

## Automated Alerting

### SQL Server Agent Job for KPI Breach Alerts
```sql
CREATE PROCEDURE dbo.sp_MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @AlertSubject NVARCHAR(255);
    
    -- Check for excessive fleet idle time
    IF EXISTS (
        SELECT 1
        FROM dbo.FactFleetTrips
        WHERE 
            TripDateKey = CONVERT(INT, FORMAT(GETDATE(), 'yyyyMMdd'))
            AND IdleTimeMinutes > 45  -- Threshold: 45 minutes
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Excessive Fleet Idle Time Detected';
        SET @AlertMessage = 'One or more fleet trips today have exceeded 45 minutes of idle time. Review immediately.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = @AlertSubject,
            @body = @AlertMessage,
            @importance = 'High';
    END;
    
    -- Check for warehouse capacity approaching limits
    DECLARE @CurrentCapacity DECIMAL(5,2);
    SELECT @CurrentCapacity = 
        SUM(CurrentStock) * 100.0 / MAX(w.MaxCapacity)
    FROM dbo.FactInventory i
    INNER JOIN dbo.DimWarehouse w ON i.WarehouseKey = w.WarehouseKey
    WHERE i.SnapshotDate = CAST(GETDATE() AS DATE);
    
    IF @CurrentCapacity > 90
    BEGIN
        SET @AlertSubject = 'ALERT: Warehouse Capacity Critical';
        SET @AlertMessage = 'Warehouse capacity is at ' + CAST(@CurrentCapacity AS VARCHAR(10)) + '%. Immediate action required.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = @AlertSubject,
            @body = @AlertMessage,
            @importance = 'High';
    END;
END;
GO

-- Schedule the job to run every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleet_KPI_Monitor',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet_KPI_Monitor',
    @step_name = N'Check_Thresholds',
    @subsystem = N'TSQL',
    @command = N'EXEC dbo.sp_MonitorKPIThresholds;',
    @database_name = N'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'LogiFleet_KPI_Monitor',
    @schedule_name = N'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = N'LogiFleet_KPI_Monitor';
```

## Incremental Data Loading

### ETL Pattern for Warehouse Operations
```sql
CREATE PROCEDURE dbo.sp_LoadWarehouseOperations_Incremental
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Staging table for new records
    IF OBJECT_ID('tempdb..#StagingWarehouseOps') IS NOT NULL
        DROP TABLE #StagingWarehouseOps;
    
    SELECT 
        t.TimeKey,
        d.DateKey,
        w.WarehouseKey,
        p.ProductKey,
        src.OperationType,
        src.OperationStartTime,
        src.OperationEndTime,
        src.DwellTimeHours,
        src.QuantityHandled,
        e.EmployeeKey,
        eq.EquipmentKey,
        z.ZoneKey
    INTO #StagingWarehouseOps
    FROM ExternalWMS.dbo.OperationsLog src
    INNER JOIN dbo.DimTime t ON 
        CAST(src.OperationStartTime AS TIME) BETWEEN t.TimeValue AND DATEADD(MINUTE, 14, t.TimeValue)
    INNER JOIN dbo.DimDate d ON 
        CAST(src.OperationStartTime AS DATE) = d.FullDate
    INNER JOIN dbo.DimWarehouse w ON 
        src.WarehouseCode = w.WarehouseCode
    INNER JOIN dbo.DimProduct p ON 
        src.SKU = p.SKU
    LEFT JOIN dbo.DimEmployee e ON 
        src.EmployeeID = e.EmployeeID
    LEFT JOIN dbo.DimEquipment eq ON 
        src.EquipmentID = eq.EquipmentID
    LEFT JOIN dbo.DimZone z ON 
        src.ZoneCode = z.ZoneCode AND w.WarehouseKey = z.WarehouseKey
    WHERE 
        src.OperationStartTime > @LastLoadDateTime
        AND src.OperationStartTime <= GETDATE();
    
    -- Insert new records
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, DateKey, WarehouseKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, DwellTimeHours,
        QuantityHandled, EmployeeKey, EquipmentKey, ZoneKey
    )
    SELECT 
        TimeKey, DateKey, WarehouseKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, DwellTimeHours,
        QuantityHandled, EmployeeKey, EquipmentKey, ZoneKey
    FROM #StagingWarehouseOps;
    
    -- Log the load metadata
    INSERT INTO dbo.ETL_LoadLog (TableName, LoadDateTime, RowsInserted)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
    
    -- Trigger gravity score recalculation if significant volume
    IF @@ROWCOUNT > 1000
    BEGIN
        EXEC dbo.sp_RecalculateProductGravity;
    END;
END;
GO
```

## Configuration Files

### External Data Source Configuration
```sql
-- Create external data source for telemetry API
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${TELEMETRY_BLOB_STORAGE_URL}',
    CREDENTIAL = FleetTelemetryCredential
);

-- Create external table for streaming telemetry
CREATE EXTERNAL TABLE dbo.ExtFleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineStatus VARCHAR(20)
)
WITH (
    LOCATION = '/telemetry',
    DATA_SOURCE = FleetTelemetryAPI,
    FILE_FORMAT = JSONFormat
);
```

## Troubleshooting

### Issue: Power BI Dashboard Refresh Fails

**Symptom:** Scheduled refresh in Power BI Service returns timeout error.

**Solution:**
```sql
-- Check long-running queries
SELECT 
    r.session_id,
    r.start_time,
    r.status,
    r.command,
    DB_NAME(r.database_id) AS DatabaseName,
    t.text AS QueryText,
    r.cpu_time,
    r.total_elapsed_time / 1000 AS ElapsedSeconds
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.database_id = DB_ID('LogiFleetPulse')
ORDER BY r.total_elapsed_time DESC;

-- Optimize with indexed views for common aggregations
CREATE VIEW dbo.vw_DailyWarehouseSummary
WITH SCHEMABINDING
AS
SELECT 
    DateKey,
    WarehouseKey,
    OperationType,
    COUNT_BIG(*) AS OperationCount,
    SUM(QuantityHandled) AS TotalQuantity,
    AVG(DurationMinutes) AS AvgDurationMinutes
FROM dbo.FactWarehouseOperations
GROUP BY DateKey, WarehouseKey, OperationType;
GO

CREATE UNIQUE CLUSTERED INDEX IX_DailyWarehouseSummary
ON dbo.vw_DailyWarehouseSummary (DateKey, WarehouseKey, OperationType);
```

### Issue: High Gravity Score Products Not Updating

**Symptom:** Gravity scores remain static despite changing pick patterns.

**Solution:**
```sql
-- Verify last recalculation date
SELECT 
    ProductKey,
    SKU,
    GravityScore,
    LastRecalculatedDate,
    DATEDIFF(DAY, LastRecalculatedDate, GETDATE()) AS DaysSinceUpdate
FROM dbo.DimProductGravity
WHERE LastRecalculatedDate < DATEADD(DAY, -7, GETDATE());

-- Force immediate recalculation
EXEC dbo.sp_RecalculateProductGravity;

-- Verify SQL Agent job is running
SELECT 
    j.name AS JobName,
    ja.run_requested_date AS LastRunDate,
    CASE ja.run_status
        WHEN 0 THEN 'Failed'
        WHEN 1 THEN 'Succeeded'
        WHEN 2 THEN 'Retry'
        WHEN 3 THEN 'Canceled'
        WHEN 4 THEN 'In Progress'
    END AS LastRunStatus
FROM msdb.dbo.sysjobs j
INNER JOIN msdb.dbo.sysjobactivity ja ON j.job_id = ja.job_id
WHERE j.name LIKE '%Gravity%'
ORDER BY ja.run_requested_date DESC;
```

### Issue: Cross-Fact Queries Return Unexpected Results

**Symptom:** Fleet and warehouse metrics don't align temporally.

**Solution:**
```sql
-- Verify time dimension alignment
SELECT 
    'Warehouse' AS Source,
    MIN(OperationStartTime) AS EarliestRecord,
    MAX(OperationStartTime) AS LatestRecord
FROM dbo.FactWarehouseOperations
UNION ALL
SELECT 
    'Fleet' AS Source,
    MIN(TripStartTime) AS EarliestRecord,
    MAX(TripEndTime) AS LatestRecord
FROM dbo.FactFleetTrips;

-- Check for orphaned dimension keys
SELECT 
    'FactWarehouseOperations' AS TableName,
    COUNT(*) AS OrphanedRecords
FROM dbo.FactWarehouseOperations wo
WHERE NOT EXISTS (
    SELECT 1 FROM dbo.DimTime t WHERE wo.TimeKey = t.TimeKey
);
```

## Performance Optimization

### Partitioning Strategy for Large Fact Tables
```sql
-- Create partition function by month
CREATE PARTITION FUNCTION PF_MonthlyDate (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601,
    20260701, 20260801, 20260901, 20261001, 20261101, 20261201
);

-- Create partition scheme
CREATE PARTITION SCHEME PS_MonthlyDate
AS PARTITION PF_MonthlyDate
ALL TO ([PRIMARY]);

-- Recreate FactWarehouseOperations with partitioning
-- (backup and drop existing table first)
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1),
    DateKey INT NOT NULL,
    -- ... other columns ...
    CONSTRAINT PK_FactWarehouseOperations PRIMARY KEY (OperationKey, DateKey)
) ON PS_MonthlyDate(DateKey);
```

## Common Usage Patterns

### Daily Operations Review
```sql
-- Executive dashboard query: Today's key metrics
SELECT 
    'Warehouse Operations' AS Metric,
    COUNT(*) AS Count,
    AVG(DurationMinutes) AS AvgDuration,
    SUM(QuantityHandled) AS TotalUnits
FROM dbo.FactWarehouseOperations
WHERE DateKey = CONVERT(INT, FORMAT(GETDATE(), 'yyyyMMdd'))
UNION ALL
SELECT 
    'Fleet Trips' AS Metric,
    COUNT(*) AS Count,
    AVG(IdleTimeMinutes) AS AvgIdleTime,
    SUM(DistanceMiles) AS TotalMiles
FROM dbo.FactFleetTrips
WHERE TripDateKey = CONVERT(INT, FORMAT(GETDATE(), 'yyyyMMdd'));
```

### Weekly Performance Analysis
```sql
-- Week-over-week comparison
WITH CurrentWeek AS (
    SELECT 
        SUM(QuantityHandled) AS TotalUnits
