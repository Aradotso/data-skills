---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing for logistics and supply chain analytics with multi-fact star schema
triggers:
  - set up logifleet pulse supply chain analytics
  - configure power bi logistics dashboard
  - create warehouse fleet data model
  - build supply chain star schema
  - implement logistics kpi dashboard
  - deploy logifleet pulse sql database
  - integrate warehouse and fleet telemetry data
  - optimize supply chain analytics warehouse
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers implement LogiFleet Pulse, an advanced supply chain analytics platform that unifies warehouse operations, fleet telemetry, and logistics KPIs through a multi-fact star schema in MS SQL Server with Power BI visualization.

## What LogiFleet Pulse Does

LogiFleet Pulse is a data warehousing and analytics solution that:

- **Unifies multi-modal logistics data** from warehouse management systems, fleet telemetry, supplier portals, and external APIs
- **Implements a multi-fact star schema** with time-phased dimensions for cross-fact KPI analysis
- **Provides real-time dashboards** in Power BI refreshed at 15-minute intervals
- **Enables predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Supports role-based access** with row-level security for compliance

The platform connects warehouse velocity metrics (putaway cycles, pick rates, dwell time) with fleet performance (fuel consumption, idle time, route efficiency) through shared dimensions.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (with PolyBase for external tables optional)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to data sources: WMS, fleet telemetry API, ERP system

### Step 1: Deploy SQL Schema

Clone the repository and locate the SQL schema files:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

Execute the schema deployment script in SSMS:

```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema script (run the provided schema.sql)
-- This creates fact tables, dimension tables, and stored procedures
```

### Step 2: Configure Data Sources

Create a configuration for data connections (store sensitive values in environment variables):

```json
{
  "connections": {
    "wms_database": {
      "server": "${WMS_SQL_SERVER}",
      "database": "${WMS_DATABASE}",
      "auth": "integrated"
    },
    "fleet_api": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    },
    "erp_database": {
      "server": "${ERP_SQL_SERVER}",
      "database": "${ERP_DATABASE}",
      "username": "${ERP_USERNAME}",
      "password": "${ERP_PASSWORD}"
    }
  },
  "refresh_interval_minutes": 15
}
```

### Step 3: Import Power BI Template

Open the `.pbit` template file in Power BI Desktop and configure the SQL Server connection:

1. Open `LogiFleet_Pulse_Master.pbit`
2. Enter your SQL Server instance name
3. Select DirectQuery or Import mode (DirectQuery recommended for real-time)
4. Configure refresh schedule in Power BI Service

## Core Data Model Structure

### Fact Tables

The schema includes multiple fact tables for different operational areas:

```sql
-- FactWarehouseOperations: Warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseZoneKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    QuantityHandled INT,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    WorkerID INT,
    CONSTRAINT FK_Warehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Warehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_Warehouse_Zone FOREIGN KEY (WarehouseZoneKey) REFERENCES DimWarehouseZone(ZoneKey)
);

-- Create clustered columnstore index for analytics performance
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations 
ON FactWarehouseOperations;
```

```sql
-- FactFleetTrips: Fleet performance and route data
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    DelayMinutes DECIMAL(10,2),
    DelayReasonKey INT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON FactFleetTrips;
```

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT,
    TimeOfDay TIME,
    HourOfDay INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    DayOfWeek VARCHAR(20),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(20),
    FiscalQuarter VARCHAR(20),
    FiscalYear INT
);

-- Populate time dimension
DECLARE @StartDate DATETIME = '2024-01-01';
DECLARE @EndDate DATETIME = '2027-12-31';

WITH TimeCTE AS (
    SELECT @StartDate AS FullDateTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, FullDateTime)
    FROM TimeCTE
    WHERE DATEADD(MINUTE, 15, FullDateTime) < @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeOfDay, HourOfDay, MinuteBucket, DayOfWeek, IsWeekend)
SELECT 
    CAST(FORMAT(FullDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    FullDateTime,
    CAST(FORMAT(FullDateTime, 'yyyyMMdd') AS INT) AS DateKey,
    CAST(FullDateTime AS TIME) AS TimeOfDay,
    DATEPART(HOUR, FullDateTime) AS HourOfDay,
    (DATEPART(MINUTE, FullDateTime) / 15) * 15 AS MinuteBucket,
    DATENAME(WEEKDAY, FullDateTime) AS DayOfWeek,
    CASE WHEN DATEPART(WEEKDAY, FullDateTime) IN (1, 7) THEN 1 ELSE 0 END AS IsWeekend
FROM TimeCTE
OPTION (MAXRECURSION 0);
```

```sql
-- DimProductGravity: Products with warehouse gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    ProductSKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitValue DECIMAL(10,2),
    IsFragile BIT,
    RequiresColdStorage BIT,
    VelocityScore DECIMAL(5,2), -- Calculated from historical pick frequency
    GravityScore DECIMAL(5,2), -- Composite: velocity * value * fragility
    OptimalZoneType VARCHAR(50) -- 'HIGH_GRAVITY', 'MEDIUM_GRAVITY', 'LOW_GRAVITY'
);

-- Calculate gravity score trigger
CREATE TRIGGER trg_CalculateGravity
ON DimProductGravity
AFTER INSERT, UPDATE
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = 
        (VelocityScore * 0.5) + 
        (UnitValue / 100 * 0.3) + 
        (CASE WHEN IsFragile = 1 THEN 0.2 ELSE 0 END * 100),
        OptimalZoneType = 
            CASE 
                WHEN GravityScore > 75 THEN 'HIGH_GRAVITY'
                WHEN GravityScore > 40 THEN 'MEDIUM_GRAVITY'
                ELSE 'LOW_GRAVITY'
            END
    WHERE ProductKey IN (SELECT ProductKey FROM inserted);
END;
```

```sql
-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(50) UNIQUE,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'WAREHOUSE', 'DISTRIBUTION_CENTER', 'DELIVERY_ZONE'
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    ParentGeographyKey INT,
    CONSTRAINT FK_Geography_Parent FOREIGN KEY (ParentGeographyKey) 
        REFERENCES DimGeography(GeographyKey)
);
```

## Data Loading Procedures

### Incremental Load from WMS

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load new operations from WMS staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseZoneKey, OperationType,
        QuantityHandled, DurationMinutes, DwellTimeHours, WorkerID
    )
    SELECT 
        CAST(FORMAT(wms.OperationTimestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        dp.ProductKey,
        dz.ZoneKey,
        wms.OperationType,
        wms.Quantity,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) AS DurationMinutes,
        DATEDIFF(MINUTE, wms.ItemReceivedTime, wms.OperationTimestamp) / 60.0 AS DwellTimeHours,
        wms.WorkerID
    FROM WMS_StagingOperations wms
    INNER JOIN DimProductGravity dp ON wms.SKU = dp.ProductSKU
    INNER JOIN DimWarehouseZone dz ON wms.ZoneCode = dz.ZoneCode
    WHERE wms.OperationTimestamp > @LastLoadDateTime
        AND wms.IsProcessed = 0;
    
    -- Mark staged records as processed
    UPDATE WMS_StagingOperations
    SET IsProcessed = 1, ProcessedDateTime = GETDATE()
    WHERE OperationTimestamp > @LastLoadDateTime
        AND IsProcessed = 0;
    
    -- Update last load timestamp
    UPDATE ETL_LoadLog
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Fleet Telemetry Integration

```sql
-- Load fleet trip data from API staging table
CREATE PROCEDURE sp_LoadFleetTrips
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, RouteKey, DriverKey,
        OriginGeographyKey, DestinationGeographyKey,
        DistanceKM, FuelLiters, IdleTimeMinutes,
        LoadingTimeMinutes, UnloadingTimeMinutes,
        DelayMinutes, DelayReasonKey
    )
    SELECT 
        CAST(FORMAT(ft.TripStartTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        dv.VehicleKey,
        dr.RouteKey,
        dd.DriverKey,
        geo_origin.GeographyKey AS OriginGeographyKey,
        geo_dest.GeographyKey AS DestinationGeographyKey,
        ft.TotalDistanceKM,
        ft.FuelConsumedLiters,
        ft.IdleTimeMinutes,
        ft.LoadingMinutes,
        ft.UnloadingMinutes,
        CASE WHEN ft.ActualArrival > ft.PlannedArrival 
            THEN DATEDIFF(MINUTE, ft.PlannedArrival, ft.ActualArrival)
            ELSE 0 END AS DelayMinutes,
        ISNULL(ddr.DelayReasonKey, -1) AS DelayReasonKey
    FROM FleetTelemetry_Staging ft
    INNER JOIN DimVehicle dv ON ft.VehicleID = dv.VehicleID
    INNER JOIN DimRoute dr ON ft.RouteCode = dr.RouteCode
    INNER JOIN DimDriver dd ON ft.DriverID = dd.DriverID
    INNER JOIN DimGeography geo_origin ON ft.OriginLocationCode = geo_origin.LocationCode
    INNER JOIN DimGeography geo_dest ON ft.DestinationLocationCode = geo_dest.LocationCode
    LEFT JOIN DimDelayReason ddr ON ft.DelayReason = ddr.DelayReasonDescription
    WHERE ft.TripStartTime > @LastLoadDateTime
        AND ft.IsProcessed = 0;
    
    UPDATE FleetTelemetry_Staging
    SET IsProcessed = 1, ProcessedDateTime = GETDATE()
    WHERE TripStartTime > @LastLoadDateTime
        AND IsProcessed = 0;
END;
```

## Cross-Fact KPI Queries

### Dwell Time vs Fleet Idle Correlation

```sql
-- Analyze correlation between warehouse dwell time and fleet idling
WITH WarehouseDwell AS (
    SELECT 
        dt.DateKey,
        dp.Category,
        AVG(fwo.DwellTimeHours) AS AvgDwellHours,
        SUM(fwo.QuantityHandled) AS TotalUnitsHandled
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE dt.DateKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY dt.DateKey, dp.Category
),
FleetIdle AS (
    SELECT 
        dt.DateKey,
        AVG(fft.IdleTimeMinutes) AS AvgIdleMinutes,
        COUNT(DISTINCT fft.VehicleKey) AS VehiclesActive
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
    WHERE dt.DateKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY dt.DateKey
)
SELECT 
    wd.DateKey,
    wd.Category,
    wd.AvgDwellHours,
    fi.AvgIdleMinutes,
    CAST(wd.TotalUnitsHandled AS FLOAT) / fi.VehiclesActive AS UnitsPerVehicle,
    -- Calculate correlation coefficient would require statistical functions
    CASE 
        WHEN wd.AvgDwellHours > 72 AND fi.AvgIdleMinutes > 60 
        THEN 'HIGH_RISK'
        WHEN wd.AvgDwellHours > 48 OR fi.AvgIdleMinutes > 45
        THEN 'MEDIUM_RISK'
        ELSE 'NORMAL'
    END AS RiskLevel
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.DateKey = fi.DateKey
ORDER BY wd.DateKey DESC, wd.Category;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks based on trend analysis
CREATE VIEW vw_BottleneckPrediction AS
WITH OperationTrends AS (
    SELECT 
        fwo.WarehouseZoneKey,
        dwz.ZoneName,
        dt.DateKey,
        dt.HourOfDay,
        COUNT(*) AS OperationCount,
        AVG(fwo.DurationMinutes) AS AvgDurationMinutes,
        AVG(fwo.DwellTimeHours) AS AvgDwellHours,
        -- Calculate moving average over last 7 days
        AVG(COUNT(*)) OVER (
            PARTITION BY fwo.WarehouseZoneKey, dt.HourOfDay
            ORDER BY dt.DateKey
            ROWS BETWEEN 7 PRECEDING AND CURRENT ROW
        ) AS MovingAvgOperations
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimWarehouseZone dwz ON fwo.WarehouseZoneKey = dwz.ZoneKey
    WHERE dt.DateKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY fwo.WarehouseZoneKey, dwz.ZoneName, dt.DateKey, dt.HourOfDay
)
SELECT 
    ZoneName,
    DateKey,
    HourOfDay,
    OperationCount,
    MovingAvgOperations,
    AvgDurationMinutes,
    AvgDwellHours,
    -- Bottleneck score: operations above trend + high dwell time
    (OperationCount - MovingAvgOperations) / NULLIF(MovingAvgOperations, 0) * 100 
        + (AvgDwellHours / 24 * 10) AS BottleneckScore,
    CASE 
        WHEN (OperationCount > MovingAvgOperations * 1.3 AND AvgDwellHours > 48)
        THEN 'CRITICAL'
        WHEN (OperationCount > MovingAvgOperations * 1.15 OR AvgDwellHours > 36)
        THEN 'WARNING'
        ELSE 'NORMAL'
    END AS BottleneckStatus
FROM OperationTrends
WHERE DateKey = CAST(FORMAT(GETDATE(), 'yyyyMMdd') AS INT);
```

## Power BI DAX Measures

### Composite KPI: Fleet Efficiency Index

```dax
Fleet Efficiency Index = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKM])
VAR TotalFuel = SUM(FactFleetTrips[FuelLiters])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalTripTime = 
    SUMX(
        FactFleetTrips,
        DATEDIFF(
            RELATED(DimTime[FullDateTime]),
            RELATED(DimTime[FullDateTime]) + FactFleetTrips[IdleTimeMinutes]/1440,
            MINUTE
        )
    )
VAR FuelEfficiency = DIVIDE(TotalDistance, TotalFuel, 0) * 10 -- Weight: 40%
VAR TimeUtilization = (1 - DIVIDE(TotalIdleTime, TotalTripTime, 0)) * 100 -- Weight: 60%
RETURN
    (FuelEfficiency * 0.4) + (TimeUtilization * 0.6)
```

### Warehouse Gravity Score Alignment

```dax
Gravity Alignment % = 
VAR IdealPlacements = 
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            RELATED(DimProductGravity[OptimalZoneType]) = 
                RELATED(DimWarehouseZone[ZoneType])
        )
    )
VAR TotalPlacements = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(IdealPlacements, TotalPlacements, 0) * 100
```

### Cross-Fact Delay Impact Revenue

```dax
Delay Revenue Impact = 
VAR DelayedTrips = 
    FILTER(
        FactFleetTrips,
        FactFleetTrips[DelayMinutes] > 0
    )
VAR AffectedOrders = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        USERELATIONSHIP(FactWarehouseOperations[TimeKey], DimTime[TimeKey]),
        FILTER(
            DimTime,
            DimTime[TimeKey] IN SELECTCOLUMNS(DelayedTrips, "TK", [TimeKey])
        )
    )
VAR AvgOrderValue = 
    CALCULATE(
        AVERAGE(DimProductGravity[UnitValue]),
        FILTER(
            DimProductGravity,
            DimProductGravity[ProductKey] IN 
                SELECTCOLUMNS(FactWarehouseOperations, "PK", [ProductKey])
        )
    )
RETURN
    AffectedOrders * AvgOrderValue * 0.05 -- Assume 5% revenue loss per delayed order
```

## Automated Alerts Configuration

### SQL Server Agent Job for Threshold Monitoring

```sql
-- Create alert monitoring procedure
CREATE PROCEDURE sp_MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @AlertSubject NVARCHAR(200);
    
    -- Check for critical warehouse dwell time
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations fwo
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.DateKey = CAST(FORMAT(GETDATE(), 'yyyyMMdd') AS INT)
            AND fwo.DwellTimeHours > 72
        GROUP BY fwo.WarehouseZoneKey
        HAVING COUNT(*) > 10
    )
    BEGIN
        SET @AlertSubject = 'CRITICAL: High Dwell Time Detected';
        SET @AlertMessage = 'Multiple items have exceeded 72-hour dwell time threshold. Review warehouse gravity zones.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = @AlertSubject,
            @body = @AlertMessage,
            @importance = 'High';
    END
    
    -- Check for excessive fleet idle time
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips fft
        INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
        WHERE dt.DateKey = CAST(FORMAT(GETDATE(), 'yyyyMMdd') AS INT)
        GROUP BY dt.HourOfDay
        HAVING AVG(fft.IdleTimeMinutes) > DATEPART(MINUTE, GETDATE()) * 0.20
    )
    BEGIN
        SET @AlertSubject = 'WARNING: Fleet Idle Time Above 20%';
        SET @AlertMessage = 'Current fleet idle time exceeds acceptable threshold. Review route optimization.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = @AlertSubject,
            @body = @AlertMessage,
            @importance = 'Normal';
    END
END;

-- Schedule job to run every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_KPI_Monitor';

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_KPI_Monitor',
    @step_name = 'Check Thresholds',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_MonitorKPIThresholds',
    @database_name = 'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_KPI_Monitor',
    @schedule_name = 'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = 'LogiFleet_KPI_Monitor';
```

## Row-Level Security Implementation

```sql
-- Create security table for user access control
CREATE TABLE SecurityUserAccess (
    UserID INT PRIMARY KEY IDENTITY(1,1),
    Username NVARCHAR(100) UNIQUE NOT NULL,
    Role VARCHAR(50), -- 'EXECUTIVE', 'SUPERVISOR', 'OPERATOR'
    AllowedGeographyKeys NVARCHAR(MAX), -- Comma-separated list or JSON
    AllowedWarehouseKeys NVARCHAR(MAX),
    IsActive BIT DEFAULT 1
);

-- Create security predicate function
CREATE FUNCTION fn_SecurityPredicate(@GeographyKey INT, @WarehouseZoneKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS AccessGranted
    FROM dbo.SecurityUserAccess
    WHERE Username = USER_NAME()
        AND IsActive = 1
        AND (
            Role = 'EXECUTIVE' -- Full access
            OR @GeographyKey IN (
                SELECT CAST(value AS INT)
                FROM STRING_SPLIT(AllowedGeographyKeys, ',')
            )
            OR @WarehouseZoneKey IN (
                SELECT CAST(value AS INT)
                FROM STRING_SPLIT(AllowedWarehouseKeys, ',')
            )
        );

-- Apply security policy to fleet trips
CREATE SECURITY POLICY FleetTripsSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(OriginGeographyKey, NULL)
ON dbo.FactFleetTrips
WITH (STATE = ON);

-- Apply security policy to warehouse operations
CREATE SECURITY POLICY WarehouseOpsSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(NULL, WarehouseZoneKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
```

## Performance Optimization

### Partitioning Strategy

```sql
-- Create partition function for time-based partitioning
CREATE PARTITION FUNCTION pf_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (
    20240101, 20240201, 20240301, 20240401, 20240501, 20240601,
    20240701, 20240801, 20240901, 20241001, 20241101, 20241201,
    20250101, 20250201, 20250301, 20250401, 20250501, 20250601,
    20250701, 20250801, 20250901, 20251001, 20251101, 20251201
);

-- Create partition scheme
CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition
ALL TO ([PRIMARY]);

-- Apply partitioning to fact table (requires table recreation)
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationID INT IDENTITY(1,1),
    DateKey INT NOT NULL,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseZoneKey INT NOT NULL,
    OperationType VARCHAR(50),
    QuantityHandled INT,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    WorkerID INT
) ON ps_MonthlyPartition(DateKey);

-- Create partition-aligned index
CREATE CLUSTERED INDEX CI_WarehouseOps_DateKey
ON FactWarehouseOperations_Partitioned(DateKey, OperationID)
ON ps_MonthlyPartition(DateKey);
```

### Query Performance Views

```sql
-- Create indexed view for frequent aggregations
CREATE VIEW vw_DailyWarehouseMetrics
WITH SCHEMABINDING
AS
SELECT 
    dt.DateKey,
    fwo.WarehouseZoneKey,
    COUNT_BIG(*) AS TotalOperations,
    SUM(ISNULL(fwo.QuantityHandled, 0)) AS TotalUnitsHandled,
    AVG(ISNULL(fwo.DurationMinutes, 0)) AS AvgDurationMinutes,
    AVG(ISNULL(fwo.DwellTimeHours, 0)) AS AvgDwellHours
FROM dbo.FactWarehouseOperations fwo
INNER JOIN dbo.DimTime dt ON fwo.TimeKey = dt.TimeKey
GROUP BY dt.DateKey, fwo.WarehouseZoneKey;

CREATE UNIQUE CLUSTERED INDEX UCI_DailyWarehouseMetrics
ON vw_DailyWarehouseMetrics(DateKey, WarehouseZoneKey);
```

## Troubleshooting

### Common Issues

**Issue: Power BI refresh fails with timeout**
```sql
-- Check for blocking queries
SELECT 
    session_id,
    blocking_session_id,
    wait_type,
    wait_time,
    last_wait_type,
    text
FROM sys.dm_exec_requests
CROSS APPLY sys.dm_exec_sql_text(sql_handle)
WHERE database_id = DB_ID('LogiFleetPulse')
    AND blocking_session_id > 0;

-- Solution: Implement DirectQuery mode or optimize long-running queries
-- Add covering indexes for frequently accessed columns
```

**Issue: Dimension table not updating**
```sql
-- Verify ETL load logs
SELECT TOP 10 
    TableName,
    LastLoadDateTime,
    RowsProcessed,
    LoadStatus,
    ErrorMessage
FROM ETL_LoadLog
ORDER BY LastLoadDateTime DESC;

-- Force dimension refresh
EXEC sp_RefreshDimensions;
```

**Issue: Cross-fact query performance degradation**
```sql
-- Analyze query execution plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Run problematic query
SELECT * FROM vw
