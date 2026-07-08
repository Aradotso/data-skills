---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine with MS SQL Server star schema and Power BI dashboards for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse analytics platform
  - create logistics data warehouse with power bi
  - configure multi-fact star schema for supply chain
  - build warehouse and fleet analytics dashboard
  - implement cross-modal logistics intelligence
  - deploy sql server logistics data model
  - integrate warehouse gravity zones analytics
  - set up predictive supply chain bottleneck detection
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a single semantic layer. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema for cross-modal analytics including:

- **Warehouse operations** (putaway, picking, packing, dwell time)
- **Fleet management** (routes, fuel consumption, idle time, maintenance)
- **Cross-dock operations** (direct transfers without storage)
- **Predictive analytics** (bottleneck detection, maintenance triage)
- **Spatial optimization** (warehouse gravity zones based on velocity/value)

The platform uses time-phased dimensions (15-minute granularity) and bridge tables for many-to-many relationships to enable queries like: *"Show shipments delayed by weather from cold-storage zones with 72+ hour dwell time."*

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- WMS/TMS data sources (SQL, REST API, or flat files)

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance in SSMS
-- Execute the schema creation script
:r deploy_schema.sql

-- Verify deployment
SELECT TABLE_NAME 
FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_SCHEMA = 'dbo' 
  AND TABLE_NAME LIKE 'Fact%' OR TABLE_NAME LIKE 'Dim%';
```

3. **Configure data source connections:**
```json
// config_sample.json - copy to config.json and update
{
  "sql_server": {
    "host": "localhost",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "wms_api_key": "${WMS_API_KEY}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "fleet_api_key": "${FLEET_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Initialize dimension tables:**
```sql
-- Load time dimension (15-minute buckets for 3 years)
EXEC sp_LoadTimeDimension @StartDate = '2024-01-01', @EndDate = '2027-12-31';

-- Load geography hierarchy
EXEC sp_LoadGeographyDimension;

-- Initialize product gravity scoring
EXEC sp_CalculateProductGravity;
```

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2),
    PackingTimeSeconds INT,
    ZoneID VARCHAR(20),
    GravityScore DECIMAL(5,2),
    CreatedAt DATETIME2 DEFAULT GETDATE()
);

-- Optimized indexing strategy
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Time 
    ON FactWarehouseOperations(TimeKey) 
    INCLUDE (OperationType, DwellTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Product 
    ON FactWarehouseOperations(ProductKey, ZoneID) 
    INCLUDE (GravityScore);
```

**FactFleetTrips** - Vehicle and route tracking
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimRoute(RouteKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DistanceKM DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    CreatedAt DATETIME2 DEFAULT GETDATE()
);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Route 
    ON FactFleetTrips(RouteKey, TimeKey) 
    INCLUDE (IdleTimeMinutes, DelayMinutes);
```

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2,
    Date DATE,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfWeek INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalPeriod VARCHAR(20)
);

-- Generate time dimension
INSERT INTO DimTime (TimeKey, DateTime, Date, Year, Quarter, Month, Week, DayOfWeek, Hour, Minute15Bucket, IsWeekend)
SELECT 
    CAST(FORMAT(dt, 'yyyyMMddHHmm') AS INT) as TimeKey,
    dt as DateTime,
    CAST(dt AS DATE) as Date,
    YEAR(dt) as Year,
    DATEPART(QUARTER, dt) as Quarter,
    MONTH(dt) as Month,
    DATEPART(WEEK, dt) as Week,
    DATEPART(WEEKDAY, dt) as DayOfWeek,
    DATEPART(HOUR, dt) as Hour,
    (DATEPART(MINUTE, dt) / 15) * 15 as Minute15Bucket,
    CASE WHEN DATEPART(WEEKDAY, dt) IN (1, 7) THEN 1 ELSE 0 END as IsWeekend
FROM (
    SELECT DATEADD(MINUTE, 15 * n, '2024-01-01') as dt
    FROM (SELECT TOP 105120 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 as n 
          FROM sys.all_objects a CROSS JOIN sys.all_objects b) nums
) dates;
```

**DimProductGravity** - Warehouse gravity zone optimization
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    CategoryHierarchy VARCHAR(500), -- 'Electronics > Computers > Laptops'
    PickFrequencyScore DECIMAL(5,2), -- 0-100 based on historical picks
    ValueScore DECIMAL(5,2), -- 0-100 based on unit price
    FragilityScore DECIMAL(5,2), -- 0-100 (higher = more fragile)
    GravityScore AS (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(20), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    LastUpdated DATETIME2 DEFAULT GETDATE()
);

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE sp_CalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET 
        PickFrequencyScore = (
            SELECT TOP 1 
                (COUNT(*) * 100.0 / (SELECT COUNT(*) FROM FactWarehouseOperations WHERE OperationType = 'Pick'))
            FROM FactWarehouseOperations wo
            WHERE wo.ProductKey = DimProductGravity.ProductKey
                AND wo.OperationType = 'Pick'
                AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm') AS INT)
        ),
        RecommendedZone = CASE 
            WHEN GravityScore >= 70 THEN 'High-Gravity'
            WHEN GravityScore >= 40 THEN 'Medium-Gravity'
            ELSE 'Low-Gravity'
        END,
        LastUpdated = GETDATE();
END;
```

## Key Queries and Analytics

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time
```sql
-- Correlate warehouse dwell time with fleet idle time by product category
WITH WarehouseDwell AS (
    SELECT 
        p.CategoryHierarchy,
        AVG(wo.DwellTimeMinutes) as AvgDwellTime,
        dt.Date
    FROM FactWarehouseOperations wo
    JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.CategoryHierarchy, dt.Date
),
FleetIdle AS (
    SELECT 
        p.CategoryHierarchy,
        AVG(ft.IdleTimeMinutes) as AvgIdleTime,
        dt.Date
    FROM FactFleetTrips ft
    JOIN FactWarehouseOperations wo ON ft.OriginGeographyKey = wo.WarehouseKey
    JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    JOIN DimTime dt ON ft.TimeKey = dt.TimeKey
    WHERE dt.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.CategoryHierarchy, dt.Date
)
SELECT 
    wd.CategoryHierarchy,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    CORR(wd.AvgDwellTime, fi.AvgIdleTime) OVER (PARTITION BY wd.CategoryHierarchy) as Correlation
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.CategoryHierarchy = fi.CategoryHierarchy AND wd.Date = fi.Date
ORDER BY Correlation DESC;
```

### Predictive Bottleneck Detection
```sql
-- Identify zones with trending dwell time increases (potential bottlenecks)
WITH ZoneTrends AS (
    SELECT 
        wo.ZoneID,
        dt.Week,
        AVG(wo.DwellTimeMinutes) as AvgDwellTime,
        COUNT(*) as OperationCount,
        LAG(AVG(wo.DwellTimeMinutes), 1) OVER (PARTITION BY wo.ZoneID ORDER BY dt.Week) as PrevWeekDwell
    FROM FactWarehouseOperations wo
    JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.Date >= DATEADD(DAY, -60, GETDATE())
    GROUP BY wo.ZoneID, dt.Week
)
SELECT 
    ZoneID,
    AvgDwellTime,
    PrevWeekDwell,
    ((AvgDwellTime - PrevWeekDwell) / PrevWeekDwell * 100) as PercentIncrease,
    CASE 
        WHEN ((AvgDwellTime - PrevWeekDwell) / PrevWeekDwell * 100) > 20 THEN 'HIGH RISK'
        WHEN ((AvgDwellTime - PrevWeekDwell) / PrevWeekDwell * 100) > 10 THEN 'MODERATE RISK'
        ELSE 'LOW RISK'
    END as BottleneckRisk
FROM ZoneTrends
WHERE PrevWeekDwell IS NOT NULL
    AND ((AvgDwellTime - PrevWeekDwell) / PrevWeekDwell * 100) > 10
ORDER BY PercentIncrease DESC;
```

### Adaptive Fleet Triage Scoring
```sql
-- Proactive maintenance queue weighted by revenue impact
CREATE VIEW vw_FleetTriageQueue AS
SELECT 
    v.VehicleID,
    v.VehiclePlate,
    v.MaintenanceStatus,
    v.TirePressurePSI,
    v.EngineHealthScore,
    v.LastMaintenanceDate,
    DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) as DaysSinceMaintenance,
    SUM(ft.DistanceKM) OVER (PARTITION BY v.VehicleKey) as TotalDistanceKM,
    AVG(p.ValueScore) OVER (PARTITION BY v.VehicleKey) as AvgCargoValue,
    -- Triage score: higher = more urgent
    (
        CASE 
            WHEN v.TirePressurePSI < 28 THEN 30 -- Critical tire pressure
            WHEN v.TirePressurePSI < 32 THEN 15
            ELSE 0 
        END +
        CASE 
            WHEN v.EngineHealthScore < 60 THEN 40 -- Critical engine health
            WHEN v.EngineHealthScore < 75 THEN 20
            ELSE 0 
        END +
        CASE 
            WHEN DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) > 90 THEN 25 -- Overdue maintenance
            WHEN DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) > 60 THEN 10
            ELSE 0 
        END
    ) * (AVG(p.ValueScore) / 100) as TriageScore -- Weighted by cargo value
FROM DimVehicle v
JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
JOIN FactWarehouseOperations wo ON ft.OriginGeographyKey = wo.WarehouseKey
JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE ft.TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY v.VehicleID, v.VehiclePlate, v.MaintenanceStatus, v.TirePressurePSI, 
         v.EngineHealthScore, v.LastMaintenanceDate, v.VehicleKey;

-- Query the triage queue
SELECT TOP 20 * 
FROM vw_FleetTriageQueue 
WHERE TriageScore > 10
ORDER BY TriageScore DESC;
```

## Power BI Integration

### Import the Template

1. **Open Power BI Desktop**
2. **File > Import > Power BI template** (`LogiFleet_Pulse_Master.pbit`)
3. **Enter parameters:**
   - SQL Server: `your-server.database.windows.net`
   - Database: `LogiFleetPulse`
   - Authentication: SQL Server or Windows

### DAX Measures for Cross-Fact Analysis

```dax
// Total Dwell Cost (combines warehouse time with hourly cost)
Total Dwell Cost = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60 * 
    RELATED(DimWarehouse[HourlyCost])
)

// Fleet Efficiency Ratio (distance / fuel + idle penalty)
Fleet Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]) + 
    (SUM(FactFleetTrips[IdleTimeMinutes]) * 0.1)
)

// Cross-Fact: Shipment Delay Impact
Delayed Shipment Revenue Impact = 
VAR DelayedShipments = 
    FILTER(
        FactFleetTrips,
        FactFleetTrips[DelayMinutes] > 30
    )
RETURN
SUMX(
    DelayedShipments,
    RELATED(DimProductGravity[ValueScore]) * 
    FactFleetTrips[DelayMinutes] * 0.5 // $0.50 per minute per value point
)

// Warehouse Gravity Zone Compliance
Gravity Zone Compliance % = 
VAR CorrectZone = 
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[ZoneID] = RELATED(DimProductGravity[RecommendedZone])
        )
    )
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
RETURN
DIVIDE(CorrectZone, TotalOps, 0)
```

### Row-Level Security Setup

```dax
// Create role: Regional Manager
[DimGeography[Region]] = USERNAME()

// Create role: Warehouse Supervisor
[DimGeography[WarehouseKey]] IN (
    LOOKUPVALUE(
        UserWarehouseAccess[WarehouseKey],
        UserWarehouseAccess[UserEmail],
        USERNAME()
    )
)
```

## Automated Alerting

### Configure SQL Server Agent Job

```sql
-- Create alert stored procedure
CREATE PROCEDURE sp_LogiFleetAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Alert 1: High fleet idle time (>15%)
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips ft
        JOIN DimTime dt ON ft.TimeKey = dt.TimeKey
        WHERE dt.Date = CAST(GETDATE() AS DATE)
        HAVING AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / (ft.IdleTimeMinutes + (ft.DistanceKM / 60))) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 15% threshold today';
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet Pulse Alert: High Fleet Idle Time',
            @body = @AlertMessage;
    END;
    
    -- Alert 2: Warehouse zone misalignment (gravity score)
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations wo
        JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
        WHERE dt.Date >= DATEADD(DAY, -7, GETDATE())
            AND wo.ZoneID <> p.RecommendedZone
        GROUP BY wo.ZoneID
        HAVING COUNT(*) > 50
    )
    BEGIN
        SET @AlertMessage = 'ALERT: More than 50 operations in incorrect gravity zone this week';
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet Pulse Alert: Zone Misalignment',
            @body = @AlertMessage;
    END;
    
    -- Alert 3: Predictive maintenance triage
    DECLARE @CriticalVehicles INT;
    SELECT @CriticalVehicles = COUNT(*) 
    FROM vw_FleetTriageQueue 
    WHERE TriageScore > 50;
    
    IF @CriticalVehicles > 0
    BEGIN
        SET @AlertMessage = 'ALERT: ' + CAST(@CriticalVehicles AS VARCHAR) + ' vehicles require immediate maintenance';
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet Pulse Alert: Critical Maintenance Required',
            @body = @AlertMessage;
    END;
END;

-- Schedule job to run every 15 minutes
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleetPulse_Alerts';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleetPulse_Alerts',
    @step_name = 'Run Alert Checks',
    @command = 'EXEC sp_LogiFleetAlerts';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;
EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'LogiFleetPulse_Alerts',
    @schedule_name = 'Every15Minutes';
EXEC msdb.dbo.sp_add_jobserver 
    @job_name = 'LogiFleetPulse_Alerts';
```

## Data Ingestion Patterns

### Incremental Load from WMS API

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_IncrementalLoad_WarehouseOps
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    -- Create staging table
    CREATE TABLE #StagingWarehouseOps (
        ExternalID VARCHAR(50),
        OperationTime DATETIME2,
        WarehouseCode VARCHAR(20),
        SKU VARCHAR(50),
        OperationType VARCHAR(50),
        DwellTimeMinutes INT,
        PickRate DECIMAL(10,2),
        PackingTimeSeconds INT,
        ZoneID VARCHAR(20)
    );
    
    -- Load from external source (example using OPENROWSET)
    INSERT INTO #StagingWarehouseOps
    SELECT *
    FROM OPENROWSET(
        'MSDASQL',
        'Driver={WMS ODBC Driver};Server=${WMS_SERVER};UID=${WMS_USER};PWD=${WMS_PASSWORD}',
        'SELECT * FROM operations WHERE updated_at > ''' + CONVERT(VARCHAR, @LastLoadTimestamp, 120) + ''''
    );
    
    -- Transform and load into fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        DwellTimeMinutes, PickRate, PackingTimeSeconds, ZoneID, GravityScore
    )
    SELECT 
        CAST(FORMAT(s.OperationTime, 'yyyyMMddHHmm') AS INT) as TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.DwellTimeMinutes,
        s.PickRate,
        s.PackingTimeSeconds,
        s.ZoneID,
        p.GravityScore
    FROM #StagingWarehouseOps s
    JOIN DimGeography g ON s.WarehouseCode = g.WarehouseCode
    JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations wo 
        WHERE wo.TimeKey = CAST(FORMAT(s.OperationTime, 'yyyyMMddHHmm') AS INT)
            AND wo.ProductKey = p.ProductKey
    );
    
    DROP TABLE #StagingWarehouseOps;
    
    -- Update last load timestamp
    UPDATE SystemConfig SET LastWarehouseLoad = GETDATE();
END;
```

### Real-Time Fleet Telemetry Streaming (via External Table)

```sql
-- Configure external data source for fleet telemetry
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${FLEET_BLOB_STORAGE_URL}',
    CREDENTIAL = FleetStorageCredential
);

-- Create external table for streaming telemetry
CREATE EXTERNAL TABLE ExtFleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    FuelLevel DECIMAL(5,2),
    EngineTemp INT,
    TirePressurePSI DECIMAL(4,1),
    IdleStatus BIT
)
WITH (
    LOCATION = '/telemetry/',
    DATA_SOURCE = FleetTelemetryAPI,
    FILE_FORMAT = JSONFormat
);

-- Materialize into fact table every 15 minutes
CREATE PROCEDURE sp_MaterializeFleetTelemetry
AS
BEGIN
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, RouteKey, 
        FuelConsumedLiters, IdleTimeMinutes, DistanceKM
    )
    SELECT 
        CAST(FORMAT(t.Timestamp, 'yyyyMMddHHmm') AS INT),
        v.VehicleKey,
        r.RouteKey,
        LAG(t.FuelLevel) OVER (PARTITION BY t.VehicleID ORDER BY t.Timestamp) - t.FuelLevel as FuelConsumed,
        SUM(CASE WHEN t.IdleStatus = 1 THEN 1 ELSE 0 END) as IdleMinutes,
        -- Calculate distance using Haversine formula
        SUM(
            6371 * ACOS(
                COS(RADIANS(LAG(t.Latitude) OVER (PARTITION BY t.VehicleID ORDER BY t.Timestamp))) *
                COS(RADIANS(t.Latitude)) *
                COS(RADIANS(t.Longitude) - RADIANS(LAG(t.Longitude) OVER (PARTITION BY t.VehicleID ORDER BY t.Timestamp))) +
                SIN(RADIANS(LAG(t.Latitude) OVER (PARTITION BY t.VehicleID ORDER BY t.Timestamp))) *
                SIN(RADIANS(t.Latitude))
            )
        ) as DistanceKM
    FROM ExtFleetTelemetry t
    JOIN DimVehicle v ON t.VehicleID = v.VehicleID
    JOIN DimRoute r ON r.IsActive = 1 -- Match to active route based on geo
    WHERE t.Timestamp >= DATEADD(MINUTE, -15, GETDATE())
    GROUP BY 
        CAST(FORMAT(t.Timestamp, 'yyyyMMddHHmm') AS INT),
        v.VehicleKey, r.RouteKey;
END;
```

## Configuration Best Practices

### Partitioning Strategy for Large Fact Tables

```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION FUNCTION pf_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (
    202401, 202402, 202403, 202404, 202405, 202406,
    202407, 202408, 202409, 202410, 202411, 202412,
    202501 -- Extend as needed
);

CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition
ALL TO ([PRIMARY]);

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationID BIGINT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT,
    ProductKey INT,
    OperationType VARCHAR(50),
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2),
    PackingTimeSeconds INT,
    ZoneID VARCHAR(20),
    GravityScore DECIMAL(5,2),
    CreatedAt DATETIME2 DEFAULT GETDATE(),
    CONSTRAINT PK_FactWarehouseOps_Partitioned 
        PRIMARY KEY CLUSTERED (TimeKey, OperationID)
) ON ps_MonthlyPartition(TimeKey / 100); -- Partition by YYYYMM
```

### Columnstore Indexes for Analytics Performance

```sql
-- Create columnstore index on fact tables for faster aggregations
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOps_Columnstore
ON FactWarehouseOperations (
    TimeKey, WarehouseKey, ProductKey, OperationType,
    DwellTimeMinutes, PickRate, PackingTimeSeconds, GravityScore
);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips_Columnstore
ON FactFleetTrips (
    TimeKey, VehicleKey, RouteKey, 
    FuelConsumedLiters, IdleTimeMinutes, DistanceKM, DelayMinutes
);
```

## Troubleshooting

### Performance Issues with Cross-Fact Queries

**Problem:** Queries joining multiple fact tables are slow.

**Solution:** Use materialized views or summary tables for common cross-fact KPIs:

```sql
-- Create indexed view for common cross-fact analysis
CREATE VIEW vw_WarehouseFleetCorrelation
WITH SCHEMABINDING
AS
SELECT 
    dt.Date,
    p.CategoryHierarchy,
    AVG(wo.DwellTimeMinutes) as AvgDwellTime,
    AVG(ft.IdleTimeMinutes) as AvgIdleTime,
    COUNT_BIG(*) as RecordCount
FROM dbo.FactWarehouseOperations wo
JOIN dbo.FactFleetTrips ft ON wo.TimeKey = ft.TimeKey
JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
JOIN dbo.DimTime dt ON wo.TimeKey = dt.TimeKey
GROUP BY dt.Date, p.CategoryHierarchy;

CREATE UNIQUE CLUSTERED INDEX IX_WarehouseFleetCorr 
ON vw_WarehouseFleetCorrelation(Date, CategoryHierarchy);
```

### Power BI Refresh Timeout

**Problem:** Power BI dataset refresh fails with timeout error.

**Solution:** Implement incremental refresh policy:

```powerquery
