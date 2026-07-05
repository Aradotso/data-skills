---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing template for logistics intelligence, fleet optimization, and multi-modal supply chain analytics
triggers:
  - set up logistics analytics dashboard
  - implement supply chain data warehouse
  - create fleet optimization Power BI reports
  - build warehouse operations star schema
  - configure LogiFleet Pulse analytics
  - deploy multi-fact logistics data model
  - integrate fleet telemetry with warehouse data
  - implement cross-modal supply chain KPIs
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics operations. It provides a multi-fact star schema combining warehouse operations, fleet telemetry, inventory management, and external data sources into a unified Power BI reporting layer. The system uses MS SQL Server for the data warehouse and Power BI for visualization, enabling cross-functional KPI analysis across warehousing, transportation, and supply chain operations.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop
- Access to data sources (WMS, TMS, telemetry systems)

### Database Deployment

1. Clone the repository:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. Deploy the SQL schema to your SQL Server instance:
```sql
-- Connect to your SQL Server instance
-- Run the schema creation script
:r schema/create_warehouse_schema.sql

-- Verify table creation
SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'dbo'
ORDER BY TABLE_NAME;
```

3. Configure data source connections by updating the configuration file:
```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${TELEMETRY_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

### Power BI Template Setup

1. Open the Power BI template file:
```
LogiFleet_Pulse_Master.pbit
```

2. When prompted, enter your SQL Server connection details:
   - Server: Your SQL Server hostname
   - Database: LogiFleetPulse
   - Authentication: Windows or SQL Server authentication

3. The template will automatically load the data model and establish relationships between fact and dimension tables.

## Data Model Architecture

### Core Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes DECIMAL(10,2),
    PickRatePerHour DECIMAL(10,2),
    PackingTimeMinutes DECIMAL(10,2),
    QuantityHandled INT,
    ErrorCount INT,
    OperationCost DECIMAL(12,2)
);

-- Example query: Average dwell time by product gravity zone
SELECT 
    pg.GravityZone,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS TotalOperations
FROM FactWarehouseOperations wo
JOIN DimProductGravity pg ON wo.ProductKey = pg.ProductKey
WHERE wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE Date >= DATEADD(day, -30, GETDATE()))
GROUP BY pg.GravityZone
ORDER BY AvgDwellTime DESC;
```

**FactFleetTrips** - Fleet and route performance:
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimRoute(RouteKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    TripDurationMinutes DECIMAL(10,2),
    DelayMinutes DECIMAL(10,2),
    DelayReason VARCHAR(100),
    TripRevenue DECIMAL(12,2),
    TripCost DECIMAL(12,2)
);

-- Example query: Fleet idle time analysis by route
SELECT 
    r.RouteName,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    AVG(ft.TripDurationMinutes) AS AvgTripDuration,
    (AVG(ft.IdleTimeMinutes) / NULLIF(AVG(ft.TripDurationMinutes), 0)) * 100 AS IdlePercentage
FROM FactFleetTrips ft
JOIN DimRoute r ON ft.RouteKey = r.RouteKey
GROUP BY r.RouteName
HAVING AVG(ft.TripDurationMinutes) > 0
ORDER BY IdlePercentage DESC;
```

### Key Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME,
    Date DATE,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    Day INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    DayOfWeek INT,
    DayName VARCHAR(20),
    IsWeekend BIT,
    IsHoliday BIT,
    FiscalYear INT,
    FiscalQuarter INT,
    FiscalPeriod INT
);

-- Populate time dimension
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (
            TimeKey, DateTime, Date, Year, Quarter, Month, Week, Day,
            Hour, Minute15Bucket, DayOfWeek, DayName, IsWeekend
        )
        SELECT 
            CONVERT(INT, FORMAT(@CurrentDateTime, 'yyyyMMddHHmm')),
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DAY(@CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15,
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END;
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
```

**DimProductGravity** - Product classification with velocity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- 0-100, higher = faster moving
    GravityZone VARCHAR(20), -- 'High', 'Medium', 'Low'
    AveragePickFrequency DECIMAL(10,2),
    ValuePerUnit DECIMAL(12,2),
    FragilityScore DECIMAL(5,2), -- 0-100
    ReplenishmentLeadTimeDays INT,
    OptimalStorageLocation VARCHAR(50),
    LastGravityUpdate DATETIME
);

-- Recalculate gravity scores based on recent activity
CREATE PROCEDURE RecalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET 
        GravityScore = (
            (PickFrequency.Frequency * 0.5) +
            (ValueScore.Score * 0.3) +
            ((100 - FragilityScore) * 0.2)
        ),
        LastGravityUpdate = GETDATE()
    FROM DimProductGravity pg
    CROSS APPLY (
        SELECT AVG(wo.QuantityHandled) AS Frequency
        FROM FactWarehouseOperations wo
        WHERE wo.ProductKey = pg.ProductKey
          AND wo.OperationType = 'Picking'
          AND wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE Date >= DATEADD(day, -30, GETDATE()))
    ) PickFrequency
    CROSS APPLY (
        SELECT MIN(100, (pg.ValuePerUnit / 100) * 100) AS Score
    ) ValueScore;
    
    UPDATE DimProductGravity
    SET GravityZone = 
        CASE 
            WHEN GravityScore >= 70 THEN 'High'
            WHEN GravityScore >= 40 THEN 'Medium'
            ELSE 'Low'
        END;
END;
```

## Cross-Fact KPI Analysis

### Dwell Time vs Fleet Utilization
```sql
-- Correlate warehouse dwell time with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        t.Date,
        pg.GravityZone,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations wo
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    JOIN DimProductGravity pg ON wo.ProductKey = pg.ProductKey
    WHERE t.Date >= DATEADD(day, -90, GETDATE())
    GROUP BY t.Date, pg.GravityZone
),
FleetIdle AS (
    SELECT 
        t.Date,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(ft.TripDurationMinutes) AS AvgTripDuration
    FROM FactFleetTrips ft
    JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(day, -90, GETDATE())
    GROUP BY t.Date
)
SELECT 
    wd.Date,
    wd.GravityZone,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.AvgTripDuration,
    (fi.AvgIdleTime / NULLIF(fi.AvgTripDuration, 0)) * 100 AS IdlePercentage
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.Date = fi.Date
ORDER BY wd.Date DESC, wd.GravityZone;
```

### Bottleneck Prediction Index
```sql
-- Identify potential bottlenecks using weighted scoring
CREATE VIEW vw_BottleneckIndex AS
SELECT 
    t.Date,
    t.Hour,
    g.WarehouseName,
    -- Warehouse pressure (normalized to 0-100)
    (COUNT(wo.OperationID) / 
     (SELECT MAX(OpCount) FROM (
         SELECT COUNT(*) AS OpCount 
         FROM FactWarehouseOperations 
         GROUP BY TimeKey
     ) x)) * 100 AS WarehousePressure,
    -- Fleet pressure
    (SUM(ft.DelayMinutes) / NULLIF(SUM(ft.TripDurationMinutes), 0)) * 100 AS FleetDelayRate,
    -- Combined bottleneck index
    (
        ((COUNT(wo.OperationID) / 
          (SELECT MAX(OpCount) FROM (
              SELECT COUNT(*) AS OpCount 
              FROM FactWarehouseOperations 
              GROUP BY TimeKey
          ) x)) * 100 * 0.6) +
        ((SUM(ft.DelayMinutes) / NULLIF(SUM(ft.TripDurationMinutes), 0)) * 100 * 0.4)
    ) AS BottleneckIndex
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
LEFT JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
WHERE t.Date >= DATEADD(day, -7, GETDATE())
GROUP BY t.Date, t.Hour, g.WarehouseName;
```

## Data Loading Patterns

### Incremental Loading Stored Procedure
```sql
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTimestamp DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last successful load or 24 hours ago
    IF @LastLoadTimestamp IS NULL
    BEGIN
        SELECT @LastLoadTimestamp = ISNULL(MAX(LastModified), DATEADD(day, -1, GETDATE()))
        FROM FactWarehouseOperations;
    END
    
    -- Load from staging table (populated via external data source)
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        DwellTimeMinutes, PickRatePerHour, PackingTimeMinutes,
        QuantityHandled, ErrorCount, OperationCost
    )
    SELECT 
        dt.TimeKey,
        pg.ProductKey,
        g.GeographyKey,
        st.OperationType,
        st.DwellTimeMinutes,
        st.PickRatePerHour,
        st.PackingTimeMinutes,
        st.QuantityHandled,
        st.ErrorCount,
        st.OperationCost
    FROM StagingWarehouseOperations st
    JOIN DimTime dt ON dt.DateTime = DATEADD(MINUTE, 
        -(DATEPART(MINUTE, st.OperationTimestamp) % 15), 
        st.OperationTimestamp)
    JOIN DimProductGravity pg ON pg.SKU = st.SKU
    JOIN DimGeography g ON g.WarehouseCode = st.WarehouseCode
    WHERE st.LastModified > @LastLoadTimestamp
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations 
          WHERE OperationID = st.SourceOperationID
      );
    
    -- Clean up staging
    DELETE FROM StagingWarehouseOperations
    WHERE LastModified <= @LastLoadTimestamp;
END;
```

### Real-Time Fleet Telemetry Integration
```sql
-- External table for streaming fleet data (SQL Server 2019+ PolyBase)
CREATE EXTERNAL TABLE ext_FleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME,
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2),
    TirePressureFL DECIMAL(4,1),
    TirePressureFR DECIMAL(4,1),
    TirePressureRL DECIMAL(4,1),
    TirePressureRR DECIMAL(4,1),
    IsIdling BIT
)
WITH (
    LOCATION = '${TELEMETRY_API_ENDPOINT}',
    DATA_SOURCE = TelemetryDataSource,
    FILE_FORMAT = JSONFormat
);

-- Aggregate telemetry into trip facts
CREATE PROCEDURE AggregateFleetTelemetry
AS
BEGIN
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, RouteKey, OriginGeographyKey, DestinationGeographyKey,
        DistanceMiles, FuelConsumedGallons, IdleTimeMinutes, TripDurationMinutes
    )
    SELECT 
        dt.TimeKey,
        v.VehicleKey,
        r.RouteKey,
        origin.GeographyKey,
        dest.GeographyKey,
        SUM(telem.DistanceSegment) AS DistanceMiles,
        (MIN(telem.FuelLevel) - MAX(telem.FuelLevel)) AS FuelConsumedGallons,
        SUM(CASE WHEN telem.IsIdling = 1 THEN 1 ELSE 0 END) AS IdleTimeMinutes,
        DATEDIFF(MINUTE, MIN(telem.Timestamp), MAX(telem.Timestamp)) AS TripDurationMinutes
    FROM ext_FleetTelemetry telem
    JOIN DimVehicle v ON v.VehicleID = telem.VehicleID
    JOIN DimTime dt ON dt.DateTime = DATEADD(MINUTE, 
        -(DATEPART(MINUTE, telem.Timestamp) % 15), telem.Timestamp)
    -- Additional joins for route, origin, destination
    WHERE telem.Timestamp >= DATEADD(MINUTE, -30, GETDATE())
    GROUP BY dt.TimeKey, v.VehicleKey, r.RouteKey, origin.GeographyKey, dest.GeographyKey;
END;
```

## Power BI DAX Measures

### Key Performance Indicators
```dax
// Warehouse Efficiency Score (0-100)
Warehouse Efficiency = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR TargetDwellTime = 120 // 2 hours target
VAR DwellScore = MAX(0, MIN(100, (1 - (AvgDwellTime / TargetDwellTime)) * 100))

VAR AvgPickRate = AVERAGE(FactWarehouseOperations[PickRatePerHour])
VAR TargetPickRate = 50
VAR PickScore = MAX(0, MIN(100, (AvgPickRate / TargetPickRate) * 100))

VAR ErrorRate = DIVIDE(SUM(FactWarehouseOperations[ErrorCount]), COUNTROWS(FactWarehouseOperations))
VAR ErrorScore = MAX(0, 100 - (ErrorRate * 10))

RETURN (DwellScore * 0.4) + (PickScore * 0.4) + (ErrorScore * 0.2)

// Fleet Utilization Rate
Fleet Utilization = 
DIVIDE(
    SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes])
) * 100

// Cross-Fact: Cost per Delivered Unit
Cost per Unit = 
VAR TotalWarehouseCost = SUM(FactWarehouseOperations[OperationCost])
VAR TotalFleetCost = SUM(FactFleetTrips[TripCost])
VAR TotalUnitsDelivered = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
RETURN DIVIDE(TotalWarehouseCost + TotalFleetCost, TotalUnitsDelivered)

// Time Intelligence: Rolling 30-Day Average Dwell Time
Dwell Time 30D Avg = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -30, DAY)
)
```

### Predictive Measures
```dax
// Predictive Bottleneck Score
Bottleneck Risk Score = 
VAR CurrentWarehouseLoad = 
    DIVIDE(
        COUNTROWS(FactWarehouseOperations),
        CALCULATE(
            COUNTROWS(FactWarehouseOperations),
            DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -7, DAY)
        ) / 7
    )

VAR CurrentFleetDelay = 
    DIVIDE(
        SUM(FactFleetTrips[DelayMinutes]),
        SUM(FactFleetTrips[TripDurationMinutes])
    )

VAR HistoricalAvgDelay = 
    CALCULATE(
        DIVIDE(
            SUM(FactFleetTrips[DelayMinutes]),
            SUM(FactFleetTrips[TripDurationMinutes])
        ),
        DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -30, DAY)
    )

RETURN 
    (CurrentWarehouseLoad * 60) + 
    (DIVIDE(CurrentFleetDelay, HistoricalAvgDelay) * 40)
```

## Automated Alerting

### SQL Server Agent Job for Threshold Monitoring
```sql
CREATE PROCEDURE CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertEmail VARCHAR(500) = '${ALERT_EMAIL}';
    DECLARE @AlertSubject VARCHAR(200);
    DECLARE @AlertBody VARCHAR(MAX);
    
    -- Check warehouse dwell time threshold
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations wo
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
          AND wo.DwellTimeMinutes > 240 -- 4 hours
        GROUP BY t.Hour
        HAVING COUNT(*) > 10
    )
    BEGIN
        SET @AlertSubject = 'LogiFleet Alert: High Dwell Time Detected';
        SET @AlertBody = 'Multiple warehouse operations are exceeding 4-hour dwell time threshold.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetProfile',
            @recipients = @AlertEmail,
            @subject = @AlertSubject,
            @body = @AlertBody;
    END
    
    -- Check fleet idle time threshold
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips ft
        JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
          AND (ft.IdleTimeMinutes / NULLIF(ft.TripDurationMinutes, 0)) > 0.20 -- 20%
        GROUP BY ft.VehicleKey
        HAVING COUNT(*) > 3
    )
    BEGIN
        SET @AlertSubject = 'LogiFleet Alert: Excessive Fleet Idle Time';
        SET @AlertBody = 'One or more vehicles showing idle time >20% across multiple trips today.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetProfile',
            @recipients = @AlertEmail,
            @subject = @AlertSubject,
            @body = @AlertBody;
    END
END;

-- Schedule to run every hour
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet KPI Monitoring';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet KPI Monitoring',
    @step_name = 'Check Thresholds',
    @command = 'EXEC CheckKPIThresholds';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Hourly',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 8, -- Hours
    @freq_subday_interval = 1;
EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'LogiFleet KPI Monitoring',
    @schedule_name = 'Hourly';
```

## Row-Level Security

### Implement role-based data access
```sql
-- Create security table
CREATE TABLE UserSecurity (
    UserID INT PRIMARY KEY,
    Username VARCHAR(100),
    Role VARCHAR(50), -- 'Warehouse', 'Fleet', 'Executive', 'Admin'
    WarehouseFilter VARCHAR(MAX), -- JSON array of allowed warehouse keys
    RegionFilter VARCHAR(MAX) -- JSON array of allowed region keys
);

-- Create security function
CREATE FUNCTION fn_SecurityFilter(@Username VARCHAR(100))
RETURNS TABLE
AS
RETURN
    SELECT WarehouseFilter, RegionFilter
    FROM UserSecurity
    WHERE Username = @Username;

-- Apply to warehouse operations
CREATE FUNCTION fn_SecureWarehouseOperations(@Username VARCHAR(100))
RETURNS TABLE
AS
RETURN
    SELECT wo.*
    FROM FactWarehouseOperations wo
    JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
    CROSS APPLY fn_SecurityFilter(@Username) sec
    WHERE 
        (sec.WarehouseFilter IS NULL OR wo.WarehouseKey IN (SELECT value FROM OPENJSON(sec.WarehouseFilter)))
        AND (sec.RegionFilter IS NULL OR g.RegionKey IN (SELECT value FROM OPENJSON(sec.RegionFilter)));

-- Power BI uses this in connection string:
-- Data Source=server;Initial Catalog=LogiFleetPulse;
-- CustomData="USERNAME()"
```

## Common Troubleshooting

### Performance Optimization

**Slow dashboard refresh:**
```sql
-- Add columnstore indexes to fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOperations_CS
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, DwellTimeMinutes, QuantityHandled);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips_CS
ON FactFleetTrips (TimeKey, VehicleKey, RouteKey, IdleTimeMinutes, TripDurationMinutes);

-- Partition large fact tables by date
CREATE PARTITION FUNCTION pf_Date (DATE)
AS RANGE RIGHT FOR VALUES 
('2025-01-01', '2025-04-01', '2025-07-01', '2025-10-01', '2026-01-01');

CREATE PARTITION SCHEME ps_Date
AS PARTITION pf_Date ALL TO ([PRIMARY]);

-- Rebuild fact table with partitioning
CREATE TABLE FactWarehouseOperations_New (
    -- Same schema as before
) ON ps_Date(TimeKey);
```

**Memory errors in Power BI:**
```dax
// Use SUMMARIZECOLUMNS instead of nested CALCULATETABLE
Optimized Measure = 
SUMX(
    SUMMARIZECOLUMNS(
        DimTime[Date],
        DimProductGravity[GravityZone],
        "TotalDwell", SUM(FactWarehouseOperations[DwellTimeMinutes])
    ),
    [TotalDwell]
)

// Instead of:
// CALCULATE(SUM(...), ALL(...), FILTER(...))
```

### Data Quality Issues

**Missing time keys:**
```sql
-- Find and populate missing time entries
INSERT INTO DimTime (TimeKey, DateTime, Date, Year, Month, Day, Hour, Minute15Bucket)
SELECT DISTINCT
    CONVERT(INT, FORMAT(wo.OperationTimestamp, 'yyyyMMddHHmm')) AS TimeKey,
    DATEADD(MINUTE, -(DATEPART(MINUTE, wo.OperationTimestamp) % 15), wo.OperationTimestamp),
    CAST(wo.OperationTimestamp AS DATE),
    YEAR(wo.OperationTimestamp),
    MONTH(wo.OperationTimestamp),
    DAY(wo.OperationTimestamp),
    DATEPART(HOUR, wo.OperationTimestamp),
    (DATEPART(MINUTE, wo.OperationTimestamp) / 15) * 15
FROM StagingWarehouseOperations wo
WHERE NOT EXISTS (
    SELECT 1 FROM DimTime dt 
    WHERE dt.TimeKey = CONVERT(INT, FORMAT(wo.OperationTimestamp, 'yyyyMMddHHmm'))
);
```

**Orphaned fact records:**
```sql
-- Identify fact records with missing dimension keys
SELECT 'FactWarehouseOperations' AS TableName, COUNT(*) AS OrphanedRecords
FROM FactWarehouseOperations wo
WHERE NOT EXISTS (SELECT 1 FROM DimProductGravity WHERE ProductKey = wo.ProductKey)
   OR NOT EXISTS (SELECT 1 FROM DimGeography WHERE GeographyKey = wo.WarehouseKey)
   OR NOT EXISTS (SELECT 1 FROM DimTime WHERE TimeKey = wo.TimeKey)

UNION ALL

SELECT 'FactFleetTrips', COUNT(*)
FROM FactFleetTrips ft
WHERE NOT EXISTS (SELECT 1 FROM DimVehicle WHERE VehicleKey = ft.VehicleKey)
   OR NOT EXISTS (SELECT 1 FROM DimRoute WHERE RouteKey = ft.RouteKey);
```

## Advanced Patterns

### Temporal Elasticity Simulation
```sql
-- Simulate impact of warehouse capacity changes
CREATE PROCEDURE SimulateCapacityChange
    @CapacityIncreasePct DECIMAL(5,2),
    @SimulationDays INT = 30
AS
BEGIN
    WITH HistoricalBaseline AS (
        SELECT 
            t.Date,
            COUNT(*) AS OperationCount,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
            SUM(wo.QuantityHandled) AS TotalVolume
        FROM FactWarehouseOperations wo
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.Date >= DATEADD(day, -@SimulationDays, GETDATE())
        GROUP BY t.Date
    ),
    ProjectedScenario AS (
        SELECT 
            Date,
            OperationCount * (1 + @CapacityIncreasePct / 100) AS ProjectedOperations,
            AvgDwellTime * (1 / (1 + (@CapacityIncreasePct / 200))) AS ProjectedDwellTime,
            TotalVolume * (1 + @CapacityIncreasePct / 100) AS ProjectedVolume
        FROM HistoricalBaseline
    )
    SELECT 
        ps.Date,
        hb.OperationCount
