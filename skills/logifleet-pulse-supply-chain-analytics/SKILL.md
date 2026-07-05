---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics, fleet management, and supply chain intelligence with multi-fact star schema modeling
triggers:
  - "set up supply chain analytics dashboard"
  - "create logistics data warehouse with Power BI"
  - "implement fleet tracking and warehouse KPI reporting"
  - "build multi-fact star schema for logistics"
  - "integrate warehouse and fleet telemetry data"
  - "deploy real-time supply chain intelligence"
  - "configure LogiFleet Pulse analytics platform"
  - "create cross-modal logistics dashboards"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What It Does

LogiFleet Pulse is an advanced data warehousing and analytics platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse, fleet, and cross-dock operations
- **Real-time dashboards** refreshed every 15 minutes
- **Predictive analytics** for bottleneck detection and fleet maintenance
- **Warehouse Gravity Zones** for optimal storage placement
- **Cross-fact KPI harmonization** (e.g., inventory turnover vs. fuel consumption)
- **Role-based access control** with row-level security

The platform ingests data from WMS, telematics/GPS, supplier portals, weather APIs, and order history to create unified logistics intelligence.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to source systems (WMS, TMS, telematics APIs)

### Deployment Steps

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance and create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema creation script
-- Run the provided schema.sql file from the repository
```

3. **Configure data sources:**
```json
-- Edit config.json with your connection details
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "auth": "integrated"
  },
  "wms_api": {
    "endpoint": "${WMS_API_ENDPOINT}",
    "api_key": "${WMS_API_KEY}"
  },
  "telematics_api": {
    "endpoint": "${TELEMATICS_API_ENDPOINT}",
    "api_key": "${TELEMATICS_API_KEY}"
  },
  "weather_api": {
    "endpoint": "${WEATHER_API_ENDPOINT}",
    "api_key": "${WEATHER_API_KEY}"
  }
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
- Enter SQL Server connection string when prompted
- Configure refresh schedule in Power BI Service

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse micro-operations
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'receiving', 'putaway', 'picking', 'packing', 'shipping'
    DwellTimeMinutes INT,
    PickingCycleSeconds INT,
    PackingCycleSeconds INT,
    OperatorID INT,
    CONSTRAINT FK_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Create indexed view for common aggregations
CREATE VIEW vw_WarehouseDwellSummary WITH SCHEMABINDING AS
SELECT 
    ProductKey,
    WarehouseKey,
    COUNT_BIG(*) AS OperationCount,
    AVG(DwellTimeMinutes) AS AvgDwellTime,
    MAX(DwellTimeMinutes) AS MaxDwellTime
FROM dbo.FactWarehouseOperations
WHERE OperationType = 'putaway'
GROUP BY ProductKey, WarehouseKey;

CREATE UNIQUE CLUSTERED INDEX IX_DwellSummary 
ON vw_WarehouseDwellSummary(ProductKey, WarehouseKey);
```

**FactFleetTrips** - Fleet telemetry and route data
```sql
CREATE TABLE FactFleetTrips (
    TripKey INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TotalDistanceKM DECIMAL(10,2),
    AvgSpeedKMH DECIMAL(5,2),
    WeatherCondition VARCHAR(50),
    CONSTRAINT FK_FleetTime FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

-- Partitioning for performance (monthly partitions)
CREATE PARTITION FUNCTION PF_TripsByMonth (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401);

CREATE PARTITION SCHEME PS_TripsByMonth
AS PARTITION PF_TripsByMonth ALL TO ([PRIMARY]);

-- Apply partitioning (rebuild table with partition scheme)
CREATE TABLE FactFleetTrips_Partitioned (
    TripKey INT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    -- ... other columns
    CONSTRAINT PK_FleetTrips_Part PRIMARY KEY (TripKey, TimeKey)
) ON PS_TripsByMonth(TimeKey);
```

**FactCrossDock** - Cross-dock transfer operations
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundVehicleKey INT,
    OutboundVehicleKey INT,
    TransferTimeMinutes INT,
    QuantityUnits INT,
    TemperatureCompliance BIT,
    CONSTRAINT FK_CDTime FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey)
);
```

### Dimension Tables

**DimTime** - Time-phased dimension (15-minute granularity)
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfWeek INT,
    HourOfDay INT,
    QuarterHour INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10)
);

-- Populate time dimension
DECLARE @StartDate DATETIME = '2026-01-01';
DECLARE @EndDate DATETIME = '2027-12-31';

WITH TimeSeries AS (
    SELECT @StartDate AS DateTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, DateTime)
    FROM TimeSeries
    WHERE DateTime < @EndDate
)
INSERT INTO DimTime (TimeKey, DateTime, Year, Quarter, Month, Week, DayOfWeek, HourOfDay, QuarterHour, IsWeekend)
SELECT 
    CAST(FORMAT(DateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    DateTime,
    YEAR(DateTime),
    DATEPART(QUARTER, DateTime),
    MONTH(DateTime),
    DATEPART(WEEK, DateTime),
    DATEPART(WEEKDAY, DateTime),
    DATEPART(HOUR, DateTime),
    (DATEPART(MINUTE, DateTime) / 15) * 15,
    CASE WHEN DATEPART(WEEKDAY, DateTime) IN (1, 7) THEN 1 ELSE 0 END
FROM TimeSeries
OPTION (MAXRECURSION 0);
```

**DimProductGravity** - Product classification with gravity scoring
```sql
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitValue DECIMAL(10,2),
    FragilityScore INT, -- 1-10
    PickFrequencyScore INT, -- Calculated from historical data
    GravityZone VARCHAR(20), -- 'High', 'Medium', 'Low'
    GravityScore AS (PickFrequencyScore * UnitValue / NULLIF(FragilityScore, 0)) PERSISTED
);

-- Update gravity zones based on computed score
UPDATE DimProduct
SET GravityZone = CASE 
    WHEN GravityScore > 1000 THEN 'High'
    WHEN GravityScore > 500 THEN 'Medium'
    ELSE 'Low'
END;
```

**DimGeography** - Hierarchical geography
```sql
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50),
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Hub', 'RouteNode'
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- Spatial indexing for distance calculations
ALTER TABLE DimGeography ADD GeographyPoint GEOGRAPHY;

UPDATE DimGeography
SET GeographyPoint = GEOGRAPHY::Point(Latitude, Longitude, 4326);

CREATE SPATIAL INDEX SI_Geography 
ON DimGeography(GeographyPoint);
```

## Key Stored Procedures

### Incremental Data Loading

**Load Warehouse Operations:**
```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last successful load if not provided
    IF @LastLoadDateTime IS NULL
        SELECT @LastLoadDateTime = MAX(LoadDateTime) 
        FROM ETL_LoadLog 
        WHERE TableName = 'FactWarehouseOperations' AND Status = 'Success';
    
    -- Stage data from WMS
    TRUNCATE TABLE Stage_WarehouseOps;
    
    INSERT INTO Stage_WarehouseOps (
        OperationTimestamp, SKU, WarehouseID, OperationType, 
        DwellTimeMinutes, PickingCycleSeconds, OperatorID
    )
    SELECT 
        op.OperationTimestamp,
        op.SKU,
        op.WarehouseID,
        op.OperationType,
        DATEDIFF(MINUTE, op.StartTime, op.EndTime) AS DwellTimeMinutes,
        op.CycleTimeSeconds,
        op.OperatorID
    FROM OPENQUERY(WMS_LinkedServer, 
        'SELECT * FROM warehouse_operations 
         WHERE operation_timestamp > ''' + CONVERT(VARCHAR, @LastLoadDateTime, 120) + '''') op;
    
    -- Insert into fact table with dimension lookups
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        DwellTimeMinutes, PickingCycleSeconds, OperatorID
    )
    SELECT 
        CAST(FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        s.OperationType,
        s.DwellTimeMinutes,
        s.PickingCycleSeconds,
        s.OperatorID
    FROM Stage_WarehouseOps s
    INNER JOIN DimProduct p ON s.SKU = p.SKU
    INNER JOIN DimWarehouse w ON s.WarehouseID = w.WarehouseID
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.TimeKey = CAST(FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm') AS INT)
          AND f.ProductKey = p.ProductKey
          AND f.OperationType = s.OperationType
    );
    
    -- Log successful load
    INSERT INTO ETL_LoadLog (TableName, LoadDateTime, RowsLoaded, Status)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT, 'Success');
END;
```

**Load Fleet Telemetry:**
```sql
CREATE PROCEDURE usp_LoadFleetTrips
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from telematics API (using external table or OPENROWSET)
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, RouteKey, DriverKey,
        FuelConsumedLiters, IdleTimeMinutes, TotalDistanceKM, AvgSpeedKMH
    )
    SELECT 
        CAST(FORMAT(t.TripEndTime, 'yyyyMMddHHmm') AS INT),
        v.VehicleKey,
        r.RouteKey,
        d.DriverKey,
        t.FuelConsumed,
        t.IdleTime,
        t.TotalDistance,
        t.AvgSpeed
    FROM OPENROWSET(
        BULK 'https://${STORAGE_ACCOUNT}.blob.core.windows.net/telemetry/trips/*.json',
        FORMAT = 'CSV',
        FIELDTERMINATOR = '0x0b',
        FIELDQUOTE = '0x0b',
        ROWTERMINATOR = '0x0a'
    ) WITH (JsonDoc NVARCHAR(MAX)) AS DataSource
    CROSS APPLY OPENJSON(JsonDoc)
    WITH (
        TripEndTime DATETIME,
        VehicleID VARCHAR(50),
        RouteID VARCHAR(50),
        DriverID INT,
        FuelConsumed DECIMAL(10,2),
        IdleTime INT,
        TotalDistance DECIMAL(10,2),
        AvgSpeed DECIMAL(5,2)
    ) t
    INNER JOIN DimVehicle v ON t.VehicleID = v.VehicleID
    INNER JOIN DimRoute r ON t.RouteID = r.RouteID
    INNER JOIN DimDriver d ON t.DriverID = d.DriverID
    WHERE t.TripEndTime > ISNULL(@LastLoadDateTime, '1900-01-01');
END;
```

### Cross-Fact Analytics

**Calculate Fleet Efficiency by Gravity Zone:**
```sql
CREATE PROCEDURE usp_FleetEfficiencyByGravity
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SELECT 
        p.GravityZone,
        COUNT(DISTINCT ft.TripKey) AS TotalTrips,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.TotalDistanceKM, 0)) AS AvgFuelPerKM,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwell,
        SUM(ft.TotalDistanceKM) AS TotalKM,
        -- Composite efficiency score
        (1 / NULLIF(AVG(ft.FuelConsumedLiters / NULLIF(ft.TotalDistanceKM, 0)), 0)) * 
        (1 / NULLIF(AVG(ft.IdleTimeMinutes), 0)) * 100 AS EfficiencyScore
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    INNER JOIN FactWarehouseOperations wo ON 
        ft.TimeKey = wo.TimeKey AND 
        wo.OperationType IN ('loading', 'shipping')
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    WHERE t.DateTime BETWEEN @StartDate AND @EndDate
    GROUP BY p.GravityZone
    ORDER BY EfficiencyScore DESC;
END;
```

**Predictive Bottleneck Detection:**
```sql
CREATE PROCEDURE usp_PredictBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    -- Calculate historical patterns
    WITH HistoricalPatterns AS (
        SELECT 
            t.HourOfDay,
            t.DayOfWeek,
            AVG(wo.DwellTimeMinutes) AS AvgDwell,
            STDEV(wo.DwellTimeMinutes) AS StdDevDwell,
            AVG(ft.IdleTimeMinutes) AS AvgIdle
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        LEFT JOIN FactFleetTrips ft ON wo.TimeKey = ft.TimeKey
        WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY t.HourOfDay, t.DayOfWeek
    ),
    -- Current conditions
    CurrentState AS (
        SELECT 
            p.GravityZone,
            w.WarehouseID,
            COUNT(*) AS ActiveOperations,
            AVG(wo.DwellTimeMinutes) AS CurrentDwell
        FROM FactWarehouseOperations wo
        INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
        INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateTime >= DATEADD(HOUR, -2, GETDATE())
        GROUP BY p.GravityZone, w.WarehouseID
    )
    -- Predict bottlenecks
    SELECT 
        cs.WarehouseID,
        cs.GravityZone,
        cs.CurrentDwell,
        hp.AvgDwell + (2 * hp.StdDevDwell) AS ThresholdDwell,
        CASE 
            WHEN cs.CurrentDwell > (hp.AvgDwell + 2 * hp.StdDevDwell) 
            THEN 'CRITICAL'
            WHEN cs.CurrentDwell > (hp.AvgDwell + hp.StdDevDwell)
            THEN 'WARNING'
            ELSE 'NORMAL'
        END AS BottleneckRisk,
        cs.ActiveOperations
    FROM CurrentState cs
    CROSS APPLY (
        SELECT TOP 1 * 
        FROM HistoricalPatterns hp
        WHERE hp.HourOfDay = DATEPART(HOUR, GETDATE())
          AND hp.DayOfWeek = DATEPART(WEEKDAY, GETDATE())
    ) hp
    WHERE cs.CurrentDwell > hp.AvgDwell
    ORDER BY BottleneckRisk DESC, cs.CurrentDwell DESC;
END;
```

## Power BI DAX Measures

### Key Performance Indicators

**Warehouse Velocity:**
```dax
WarehouseVelocity = 
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
VAR TotalHours = 
    DATEDIFF(
        MIN(DimTime[DateTime]),
        MAX(DimTime[DateTime]),
        HOUR
    )
RETURN
DIVIDE(TotalOperations, TotalHours, 0)
```

**Fleet Utilization Rate:**
```dax
FleetUtilization = 
VAR TotalTripTime = SUM(FactFleetTrips[TotalDistanceKM]) / AVERAGE(FactFleetTrips[AvgSpeedKMH])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes]) / 60
VAR TotalAvailableTime = COUNTROWS(DimVehicle) * 24 * DISTINCTCOUNT(DimTime[Date])
RETURN
DIVIDE(TotalTripTime, TotalAvailableTime - TotalIdleTime, 0)
```

**Cross-Fact Efficiency Score:**
```dax
LogisticsEfficiencyScore = 
VAR WarehouseScore = 
    1 / AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR FleetScore = 
    AVERAGE(FactFleetTrips[TotalDistanceKM]) / 
    AVERAGE(FactFleetTrips[FuelConsumedLiters])
RETURN
(WarehouseScore * 100 + FleetScore * 10) / 2
```

**Gravity Zone Compliance:**
```dax
GravityZoneCompliance = 
VAR HighGravityInCorrectZone = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProduct[GravityZone] = "High",
        DimWarehouse[ZoneType] = "FastPick"
    )
VAR TotalHighGravity = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProduct[GravityZone] = "High"
    )
RETURN
DIVIDE(HighGravityInCorrectZone, TotalHighGravity, 0)
```

**Time Intelligence - Moving Average:**
```dax
DwellTime_MA7 = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(
        DimTime[Date],
        LASTDATE(DimTime[Date]),
        -7,
        DAY
    )
)
```

## Automated Alerting

### SQL Server Agent Job for Critical Alerts

```sql
CREATE PROCEDURE usp_SendCriticalAlerts
AS
BEGIN
    -- Check for high dwell times
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
          AND wo.DwellTimeMinutes > 120
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Critical dwell time detected in warehouse operations. ' +
                           'Products exceeding 2 hours in staging area.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleet_Alerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet CRITICAL: High Dwell Time',
            @body = @AlertMessage,
            @importance = 'High';
    END
    
    -- Check for excessive fleet idle time
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.DateTime >= DATEADD(HOUR, -4, GETDATE())
          AND (ft.IdleTimeMinutes * 1.0 / NULLIF(DATEDIFF(MINUTE, t.DateTime, GETDATE()), 0)) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeds 15% threshold. ' +
                           'Review route optimization and driver behavior.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleet_Alerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet WARNING: High Fleet Idle Time',
            @body = @AlertMessage;
    END
END;

-- Schedule job to run every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_CriticalAlerts';

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_CriticalAlerts',
    @step_name = 'Check_And_Alert',
    @subsystem = 'TSQL',
    @command = 'EXEC usp_SendCriticalAlerts',
    @database_name = 'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_CriticalAlerts',
    @schedule_name = 'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = 'LogiFleet_CriticalAlerts';
```

## Common Patterns

### Pattern 1: Daily Operational Dashboard Refresh

```sql
-- Create master refresh procedure
CREATE PROCEDURE usp_DailyRefresh
AS
BEGIN
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Load warehouse operations
        EXEC usp_LoadWarehouseOperations;
        
        -- Load fleet trips
        EXEC usp_LoadFleetTrips;
        
        -- Update gravity scores
        EXEC usp_RecalculateGravityScores;
        
        -- Refresh materialized views
        EXEC sp_refreshview 'vw_WarehouseDwellSummary';
        
        -- Update statistics
        UPDATE STATISTICS FactWarehouseOperations;
        UPDATE STATISTICS FactFleetTrips;
        
        COMMIT TRANSACTION;
        
        -- Trigger Power BI refresh via REST API
        EXEC usp_TriggerPowerBIRefresh;
        
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleet_Alerts',
            @recipients = '${ADMIN_EMAIL}',
            @subject = 'LogiFleet ETL FAILURE',
            @body = @ErrorMessage;
            
        THROW;
    END CATCH
END;
```

### Pattern 2: Warehouse Optimization Analysis

```sql
-- Identify products in wrong gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityZone AS CurrentZone,
    CASE 
        WHEN p.GravityScore > 1000 THEN 'High'
        WHEN p.GravityScore > 500 THEN 'Medium'
        ELSE 'Low'
    END AS RecommendedZone,
    w.WarehouseName,
    w.ZoneType AS CurrentPhysicalZone,
    AVG(wo.DwellTimeMinutes) AS AvgDwell,
    COUNT(*) AS PickFrequency
FROM FactWarehouseOperations wo
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
  AND wo.OperationType = 'picking'
GROUP BY p.SKU, p.ProductName, p.GravityZone, p.GravityScore, 
         w.WarehouseName, w.ZoneType
HAVING CASE 
    WHEN p.GravityScore > 1000 THEN 'High'
    WHEN p.GravityScore > 500 THEN 'Medium'
    ELSE 'Low'
END <> p.GravityZone
ORDER BY PickFrequency DESC;
```

### Pattern 3: Route Optimization Query

```sql
-- Find routes with high fuel consumption relative to distance
WITH RouteMetrics AS (
    SELECT 
        r.RouteID,
        r.RouteName,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.TotalDistanceKM, 0)) AS AvgFuelPerKM,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        COUNT(*) AS TripCount,
        AVG(ft.TotalDistanceKM) AS AvgDistance
    FROM FactFleetTrips ft
    INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY r.RouteID, r.RouteName
),
Benchmark AS (
    SELECT AVG(AvgFuelPerKM) AS BenchmarkFuelPerKM
    FROM RouteMetrics
)
SELECT 
    rm.RouteID,
    rm.RouteName,
    rm.AvgFuelPerKM,
    b.BenchmarkFuelPerKM,
    ((rm.AvgFuelPerKM - b.BenchmarkFuelPerKM) / b.BenchmarkFuelPerKM) * 100 AS PercentAboveBenchmark,
    rm.AvgIdleTime,
    rm.TripCount
FROM RouteMetrics rm
CROSS JOIN Benchmark b
WHERE rm.AvgFuelPerKM > b.BenchmarkFuelPerKM * 1.1 -- 10% above benchmark
ORDER BY PercentAboveBenchmark DESC;
```

## Configuration

### Row-Level Security Setup

```sql
-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@WarehouseRegion VARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS AccessAllowed
    FROM dbo.UserRegionAccess ura
    WHERE ura.UserName = USER_NAME()
      AND (ura.Region = @WarehouseRegion OR ura.Region = 'ALL');

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseRegion)
ON dbo.DimWarehouse,
ADD BLOCK PREDICATE dbo.fn_SecurityPredicate(WarehouseRegion)
