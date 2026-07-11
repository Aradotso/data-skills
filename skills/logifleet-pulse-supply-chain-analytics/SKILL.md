---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI dashboards for real-time logistics, fleet, and warehouse analytics with multi-fact star schema
triggers:
  - set up logistics supply chain analytics dashboard
  - configure warehouse and fleet tracking system
  - implement multi-fact star schema for logistics
  - deploy logifleet pulse analytics platform
  - create power bi logistics dashboard
  - build sql server supply chain data warehouse
  - integrate warehouse management with fleet telemetry
  - design cross-modal supply chain analytics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It provides a multi-fact star schema data warehouse that harmonizes warehouse operations, fleet telemetry, inventory management, and external signals (weather, traffic) into a unified semantic layer for real-time supply chain analytics.

**Key capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time dashboards with 15-minute refresh cycles
- Predictive bottleneck detection and fleet triage
- Warehouse gravity zone optimization
- Role-based access with row-level security

## Project Structure

The repository typically contains:
- `schema/` - SQL DDL scripts for tables, views, stored procedures
- `data_loaders/` - ETL scripts and incremental load procedures
- `power_bi/` - `.pbit` Power BI template files
- `config_sample.json` - Configuration template for data source connections
- `docs/` - Data model documentation and implementation guides

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to source systems (WMS, TMS, telematics APIs)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script

USE master;
GO

-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy fact tables
-- FactWarehouseOperations
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    EmployeeKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity DECIMAL(18,2),
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    StorageZoneKey INT,
    BatchNumber VARCHAR(50),
    Timestamp DATETIME2 NOT NULL,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
GO

-- FactFleetTrips
CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    StartGeographyKey INT NOT NULL,
    EndGeographyKey INT NOT NULL,
    TripDistanceKm DECIMAL(10,2),
    TripDurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    DelayMinutes DECIMAL(10,2),
    DelayReasonKey INT,
    StartTimestamp DATETIME2 NOT NULL,
    EndTimestamp DATETIME2 NOT NULL,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);
GO

-- Deploy dimension tables
-- DimTime (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    MonthName VARCHAR(20),
    Week INT,
    DayOfYear INT,
    DayOfMonth INT,
    DayOfWeek INT,
    DayName VARCHAR(20),
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalYear INT,
    FiscalQuarter INT,
    UNIQUE (FullDateTime)
);
GO

-- DimProduct with Gravity Score
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductID VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    SKU VARCHAR(100),
    UnitOfMeasure VARCHAR(20),
    WeightKg DECIMAL(10,2),
    VolumeM3 DECIMAL(10,2),
    IsFragile BIT,
    IsPerishable BIT,
    LeadTimeDays INT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity + value + fragility
    CurrentDate DATE,
    EffectiveDate DATE,
    ExpirationDate DATE,
    IsCurrent BIT
);
GO

-- DimWarehouse
CREATE TABLE DimWarehouse (
    WarehouseKey INT IDENTITY(1,1) PRIMARY KEY,
    WarehouseID VARCHAR(50) NOT NULL UNIQUE,
    WarehouseName VARCHAR(200),
    GeographyKey INT,
    TotalCapacityM3 DECIMAL(18,2),
    CurrentUtilizationPercent DECIMAL(5,2),
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);
GO

-- Storage Zone with Gravity mapping
CREATE TABLE DimStorageZone (
    StorageZoneKey INT IDENTITY(1,1) PRIMARY KEY,
    ZoneID VARCHAR(50) NOT NULL,
    WarehouseKey INT NOT NULL,
    ZoneName VARCHAR(100),
    ZoneType VARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity', 'Cold-Storage'
    DistanceFromDockMeters DECIMAL(10,2),
    CapacityM3 DECIMAL(18,2),
    TemperatureControlled BIT,
    CONSTRAINT FK_Zone_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);
GO

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey) INCLUDE (Quantity, DurationMinutes);
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey) INCLUDE (DwellTimeHours);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey) INCLUDE (TripDistanceKm, IdleTimeMinutes);
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(RouteKey) INCLUDE (DelayMinutes);
GO
```

### Step 2: Configure Data Sources

Create a configuration file from the template:

```json
// config.json (from config_sample.json)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "auth_type": "sql",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}",
      "poll_interval_minutes": 15
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}",
      "poll_interval_minutes": 15
    },
    "weather_api": {
      "endpoint": "https://api.openweathermap.org/data/2.5",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_schedule": {
    "dimension_refresh_hours": 24,
    "fact_refresh_minutes": 15
  }
}
```

### Step 3: Populate Dimension Tables

```sql
-- Stored procedure for time dimension population
CREATE PROCEDURE PopulateDimTime
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        -- Round to 15-minute bucket
        DECLARE @Rounded DATETIME2 = DATEADD(MINUTE, 
            (DATEDIFF(MINUTE, 0, @CurrentDateTime) / 15) * 15, 0);
        
        INSERT INTO DimTime (
            TimeKey, FullDateTime, Year, Quarter, Month, MonthName,
            Week, DayOfYear, DayOfMonth, DayOfWeek, DayName,
            Hour, Minute15Bucket, IsWeekend, FiscalYear, FiscalQuarter
        )
        SELECT 
            CONVERT(INT, FORMAT(@Rounded, 'yyyyMMddHHmm')),
            @Rounded,
            YEAR(@Rounded),
            DATEPART(QUARTER, @Rounded),
            MONTH(@Rounded),
            DATENAME(MONTH, @Rounded),
            DATEPART(WEEK, @Rounded),
            DATEPART(DAYOFYEAR, @Rounded),
            DAY(@Rounded),
            DATEPART(WEEKDAY, @Rounded),
            DATENAME(WEEKDAY, @Rounded),
            DATEPART(HOUR, @Rounded),
            DATEPART(MINUTE, @Rounded),
            CASE WHEN DATEPART(WEEKDAY, @Rounded) IN (1, 7) THEN 1 ELSE 0 END,
            CASE WHEN MONTH(@Rounded) >= 7 THEN YEAR(@Rounded) + 1 ELSE YEAR(@Rounded) END,
            CASE WHEN MONTH(@Rounded) >= 7 THEN DATEPART(QUARTER, @Rounded) - 2 ELSE DATEPART(QUARTER, @Rounded) + 2 END
        WHERE NOT EXISTS (SELECT 1 FROM DimTime WHERE FullDateTime = @Rounded);
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute to populate 2 years of time dimension
EXEC PopulateDimTime '2024-01-01', '2026-12-31';
GO
```

### Step 4: Create ETL Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE LoadFactWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, EmployeeKey, OperationType,
        Quantity, DurationMinutes, DwellTimeHours, StorageZoneKey,
        BatchNumber, Timestamp
    )
    SELECT 
        t.TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        e.EmployeeKey,
        src.OperationType,
        src.Quantity,
        src.DurationMinutes,
        DATEDIFF(HOUR, src.ArrivalTime, src.DepartureTime) AS DwellTimeHours,
        z.StorageZoneKey,
        src.BatchNumber,
        src.Timestamp
    FROM StagingWarehouseOperations src
    INNER JOIN DimTime t ON t.FullDateTime = DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, 0, src.Timestamp) / 15) * 15, 0)
    INNER JOIN DimWarehouse w ON w.WarehouseID = src.WarehouseID
    INNER JOIN DimProduct p ON p.ProductID = src.ProductID AND p.IsCurrent = 1
    LEFT JOIN DimEmployee e ON e.EmployeeID = src.EmployeeID
    LEFT JOIN DimStorageZone z ON z.ZoneID = src.StorageZoneID
    WHERE src.Timestamp > @LastLoadTimestamp
        AND src.IsProcessed = 0;
    
    -- Mark as processed
    UPDATE StagingWarehouseOperations
    SET IsProcessed = 1, ProcessedTimestamp = GETDATE()
    WHERE Timestamp > @LastLoadTimestamp AND IsProcessed = 0;
END;
GO

-- Calculate product gravity scores
CREATE PROCEDURE CalculateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on velocity, value, and fragility
    UPDATE DimProduct
    SET GravityScore = (
        -- Velocity component (30-day pick frequency)
        (SELECT COUNT(*) * 0.4
         FROM FactWarehouseOperations f
         WHERE f.ProductKey = DimProduct.ProductKey
           AND f.OperationType = 'Picking'
           AND f.Timestamp >= DATEADD(DAY, -30, GETDATE()))
        +
        -- Value component (normalized unit price)
        (SELECT ISNULL(AVG(UnitPrice) / 100, 0) * 0.3
         FROM StagingProductPricing
         WHERE ProductID = DimProduct.ProductID)
        +
        -- Fragility/urgency component
        (CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END +
         CASE WHEN IsPerishable = 1 THEN 30 ELSE 0 END) * 0.3
    )
    WHERE IsCurrent = 1;
END;
GO
```

### Step 5: Connect Power BI

1. Open Power BI Desktop
2. Load the `.pbit` template file from the repository
3. Configure SQL Server connection:
   - Server: Your SQL Server instance
   - Database: LogiFleetPulse
   - Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)

```powerquery
// Power Query M code for connecting to SQL Server
let
    Source = Sql.Database(
        EnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse", 
        [Query="
            SELECT 
                t.FullDateTime,
                t.DayName,
                t.Hour,
                w.WarehouseName,
                p.ProductName,
                p.Category,
                f.OperationType,
                f.Quantity,
                f.DurationMinutes,
                f.DwellTimeHours,
                z.ZoneType
            FROM FactWarehouseOperations f
            INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
            INNER JOIN DimWarehouse w ON f.WarehouseKey = w.WarehouseKey
            INNER JOIN DimProduct p ON f.ProductKey = p.ProductKey
            LEFT JOIN DimStorageZone z ON f.StorageZoneKey = z.StorageZoneKey
            WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
        "]
    )
in
    Source
```

## Key DAX Measures for Power BI

```dax
// Average dwell time by product category
AvgDwellTime = 
AVERAGE(FactWarehouseOperations[DwellTimeHours])

// Fleet idle time percentage
IdleTimePercent = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// Cross-fact KPI: Dwell time vs fleet delay correlation
DwellDelayCorrelation = 
VAR DwellAvg = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR DelayAvg = AVERAGE(FactFleetTrips[DelayMinutes])
RETURN
    IF(DwellAvg > 48 && DelayAvg > 30, "High Risk", "Normal")

// Warehouse utilization by gravity zone
UtilizationByGravityZone = 
CALCULATE(
    DIVIDE(
        SUM(FactWarehouseOperations[Quantity]),
        RELATED(DimStorageZone[CapacityM3]),
        0
    )
)

// Predictive bottleneck index (simplified)
BottleneckIndex = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR HistoricalDwell = CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -30, DAY)
)
RETURN
    IF(CurrentDwell > HistoricalDwell * 1.5, "Bottleneck Likely", "Normal")

// Fleet fuel efficiency
FuelEfficiencyKmPerLiter = 
DIVIDE(
    SUM(FactFleetTrips[TripDistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)
```

## Common Usage Patterns

### Pattern 1: Real-Time Dashboard Refresh

```sql
-- Scheduled job to refresh facts every 15 minutes
CREATE PROCEDURE RefreshAllFacts
AS
BEGIN
    DECLARE @LastLoad DATETIME2;
    
    -- Get last load timestamp
    SELECT @LastLoad = MAX(LastRefreshTime) FROM ETLControl WHERE TableName = 'FactWarehouseOperations';
    
    -- Load warehouse operations
    EXEC LoadFactWarehouseOperations @LastLoad;
    
    -- Load fleet trips
    EXEC LoadFactFleetTrips @LastLoad;
    
    -- Update control table
    UPDATE ETLControl SET LastRefreshTime = GETDATE() WHERE TableName IN ('FactWarehouseOperations', 'FactFleetTrips');
    
    -- Recalculate gravity scores daily
    IF DATEPART(HOUR, GETDATE()) = 2 -- 2 AM
        EXEC CalculateProductGravityScores;
END;
GO

-- Schedule as SQL Agent job
USE msdb;
GO
EXEC sp_add_job @job_name = 'LogiFleetPulse_Refresh';
EXEC sp_add_jobstep @job_name = 'LogiFleetPulse_Refresh', @step_name = 'Refresh', @command = 'EXEC RefreshAllFacts';
EXEC sp_add_schedule @schedule_name = 'Every15Minutes', @freq_type = 4, @freq_interval = 1, @freq_subday_type = 4, @freq_subday_interval = 15;
EXEC sp_attach_schedule @job_name = 'LogiFleetPulse_Refresh', @schedule_name = 'Every15Minutes';
GO
```

### Pattern 2: Warehouse Gravity Zone Recommendations

```sql
-- Query to recommend zone reassignments
CREATE VIEW vw_ZoneReassignmentRecommendations AS
SELECT 
    p.ProductName,
    p.GravityScore,
    z.ZoneName AS CurrentZone,
    z.ZoneType AS CurrentZoneType,
    CASE 
        WHEN p.GravityScore >= 70 THEN 'High-Gravity'
        WHEN p.GravityScore >= 40 THEN 'Medium-Gravity'
        ELSE 'Low-Gravity'
    END AS RecommendedZoneType,
    CASE 
        WHEN (p.GravityScore >= 70 AND z.ZoneType != 'High-Gravity') THEN 'Relocate to High-Gravity'
        WHEN (p.GravityScore < 70 AND p.GravityScore >= 40 AND z.ZoneType = 'Low-Gravity') THEN 'Relocate to Medium-Gravity'
        WHEN (p.GravityScore < 40 AND z.ZoneType = 'High-Gravity') THEN 'Relocate to Low-Gravity'
        ELSE 'Optimal'
    END AS Recommendation,
    AVG(f.DwellTimeHours) AS AvgDwellTime,
    COUNT(*) AS PickCount
FROM FactWarehouseOperations f
INNER JOIN DimProduct p ON f.ProductKey = p.ProductKey
INNER JOIN DimStorageZone z ON f.StorageZoneKey = z.StorageZoneKey
WHERE f.Timestamp >= DATEADD(DAY, -30, GETDATE())
  AND p.IsCurrent = 1
GROUP BY p.ProductName, p.GravityScore, z.ZoneName, z.ZoneType
HAVING COUNT(*) > 10; -- Only products with sufficient activity
GO
```

### Pattern 3: Fleet Maintenance Triage

```sql
-- Create proactive maintenance queue
CREATE VIEW vw_MaintenancePriorityQueue AS
WITH FleetHealth AS (
    SELECT 
        v.VehicleID,
        v.VehicleName,
        v.MaintenanceScore, -- From DimVehicle
        AVG(f.IdleTimeMinutes / NULLIF(f.TripDurationMinutes, 0)) AS IdleRatio,
        AVG(f.FuelConsumedLiters / NULLIF(f.TripDistanceKm, 0)) AS FuelConsumptionRate,
        SUM(f.LoadWeightKg) AS TotalLoadCarried,
        COUNT(*) AS TripCount
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    WHERE f.StartTimestamp >= DATEADD(DAY, -30, GETDATE())
    GROUP BY v.VehicleID, v.VehicleName, v.MaintenanceScore
),
LoadValue AS (
    SELECT 
        f.VehicleKey,
        SUM(p.GravityScore * f.LoadWeightKg) AS WeightedCargoValue
    FROM FactFleetTrips f
    INNER JOIN BridgeRouteProduct brp ON f.TripID = brp.TripID
    INNER JOIN DimProduct p ON brp.ProductKey = p.ProductKey
    WHERE f.StartTimestamp >= DATEADD(DAY, -7, GETDATE())
    GROUP BY f.VehicleKey
)
SELECT 
    fh.VehicleID,
    fh.VehicleName,
    fh.MaintenanceScore,
    fh.IdleRatio,
    fh.FuelConsumptionRate,
    ISNULL(lv.WeightedCargoValue, 0) AS RecentCargoValue,
    -- Priority score: lower maintenance + high cargo value = higher priority
    (100 - fh.MaintenanceScore) * 0.6 + 
    (ISNULL(lv.WeightedCargoValue, 0) / 1000) * 0.4 AS PriorityScore
FROM FleetHealth fh
LEFT JOIN LoadValue lv ON fh.VehicleID = lv.VehicleKey
WHERE fh.MaintenanceScore < 80 -- Below acceptable threshold
ORDER BY PriorityScore DESC;
GO
```

### Pattern 4: Cross-Fact Analysis

```sql
-- Analyze correlation between warehouse dwell time and fleet delays
SELECT 
    p.Category,
    z.ZoneType,
    AVG(wh.DwellTimeHours) AS AvgWarehouseDwell,
    AVG(fl.DelayMinutes) AS AvgFleetDelay,
    COUNT(DISTINCT wh.OperationID) AS WarehouseOps,
    COUNT(DISTINCT fl.TripID) AS FleetTrips
FROM FactWarehouseOperations wh
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
INNER JOIN DimStorageZone z ON wh.StorageZoneKey = z.StorageZoneKey
INNER JOIN BridgeWarehouseFleet bwf ON wh.WarehouseKey = bwf.WarehouseKey
INNER JOIN FactFleetTrips fl ON bwf.TripID = fl.TripID AND ABS(DATEDIFF(HOUR, wh.Timestamp, fl.StartTimestamp)) <= 24
WHERE wh.Timestamp >= DATEADD(DAY, -60, GETDATE())
GROUP BY p.Category, z.ZoneType
HAVING AVG(wh.DwellTimeHours) > 48
ORDER BY AvgWarehouseDwell DESC, AvgFleetDelay DESC;
```

## Configuration & Best Practices

### Row-Level Security Setup

```sql
-- Create security roles
CREATE ROLE FleetManager;
CREATE ROLE WarehouseManager;
CREATE ROLE Executive;

-- Create security filter function
CREATE FUNCTION fn_SecurityFilter(@UserRole VARCHAR(50))
RETURNS TABLE
AS
RETURN
(
    SELECT WarehouseKey, GeographyKey
    FROM DimWarehouseAccess
    WHERE UserRole = @UserRole
);
GO

-- Apply RLS to fact table
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityFilter(USER_NAME()) ON FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityFilter(USER_NAME()) ON FactFleetTrips
WITH (STATE = ON);
GO
```

### Performance Optimization

```sql
-- Partition fact tables by date for performance
CREATE PARTITION FUNCTION PF_LogiFleet_Date (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2024-01-01', '2024-04-01', '2024-07-01', '2024-10-01',
    '2025-01-01', '2025-04-01', '2025-07-01', '2025-10-01'
);
GO

CREATE PARTITION SCHEME PS_LogiFleet_Date
AS PARTITION PF_LogiFleet_Date
ALL TO ([PRIMARY]);
GO

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- Same columns as before
) ON PS_LogiFleet_Date(Timestamp);
GO

-- Create columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWH_Analytics
ON FactWarehouseOperations (
    TimeKey, ProductKey, WarehouseKey, Quantity, DurationMinutes, DwellTimeHours
);
GO
```

### Alerting Setup

```sql
-- Create alert thresholds table
CREATE TABLE AlertThresholds (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName VARCHAR(100),
    MetricName VARCHAR(100),
    ThresholdValue DECIMAL(18,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '=', '>=', '<='
    NotificationEmail VARCHAR(200),
    IsActive BIT DEFAULT 1
);
GO

-- Insert sample thresholds
INSERT INTO AlertThresholds (AlertName, MetricName, ThresholdValue, ComparisonOperator, NotificationEmail)
VALUES 
    ('High Dwell Time', 'DwellTimeHours', 72, '>', '${ALERT_EMAIL}'),
    ('Excessive Fleet Idle', 'IdleTimePercent', 15, '>', '${ALERT_EMAIL}'),
    ('Low Warehouse Utilization', 'UtilizationPercent', 60, '<', '${ALERT_EMAIL}');
GO

-- Alert checking procedure
CREATE PROCEDURE CheckAlertThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check dwell time threshold
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations 
        WHERE DwellTimeHours > (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'DwellTimeHours')
          AND Timestamp >= DATEADD(HOUR, -1, GETDATE())
    )
    BEGIN
        SET @AlertMessage = 'Alert: High dwell time detected in last hour';
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = (SELECT NotificationEmail FROM AlertThresholds WHERE MetricName = 'DwellTimeHours'),
            @subject = 'LogiFleet Alert: High Dwell Time',
            @body = @AlertMessage;
    END;
    
    -- Add similar checks for other metrics
END;
GO
```

## Troubleshooting

### Issue: Dashboard not refreshing in real-time

**Solution:** Check DirectQuery connection and SQL Server query timeout:

```sql
-- Verify last refresh times
SELECT * FROM ETLControl ORDER BY LastRefreshTime DESC;

-- Check for blocking queries
SELECT 
    session_id,
    blocking_session_id,
    wait_type,
    wait_time,
    last_wait_type,
    command,
    database_id
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
```

In Power BI Desktop:
- File → Options → Current File → Data Load
- Set Query timeout to 0 (unlimited)
- Refresh data source credentials

### Issue: Slow cross-fact queries

**Solution:** Verify indexes and consider materialized views:

```sql
-- Create aggregated view for common cross-fact queries
CREATE VIEW vw_WarehouseFleetSummary
WITH SCHEMABINDING
AS
SELECT 
    t.Year,
    t.Month,
    w.WarehouseID,
    p.Category,
    COUNT_BIG(*) AS OperationCount,
    SUM(wh.DwellTimeHours) AS TotalDwellTime,
    SUM(fl.DelayMinutes) AS TotalDelay
FROM dbo.FactWarehouseOperations wh
INNER JOIN dbo.DimTime t ON wh.TimeKey = t.TimeKey
INNER JOIN dbo.DimWarehouse w ON wh.WarehouseKey = w.WarehouseKey
INNER JOIN dbo.DimProduct p ON wh.ProductKey = p.ProductKey
LEFT JOIN dbo.FactFleetTrips fl ON ABS(DATEDIFF(HOUR, wh.Timestamp, fl.StartTimestamp)) <= 24
GROUP BY t.Year, t.Month, w.WarehouseID, p.Category;
GO

-- Create clustered index to materialize the view
CREATE UNIQUE CLUSTERED INDEX UCI_WHFleetSummary
ON vw_WarehouseFleetSummary (Year, Month, WarehouseID, Category);
GO
```

### Issue:
