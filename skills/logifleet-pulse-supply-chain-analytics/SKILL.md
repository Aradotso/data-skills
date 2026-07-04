---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehousing and fleet logistics analytics engine with multi-fact star schema
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - configure Power BI logistics dashboard
  - implement warehouse and fleet data warehouse
  - build multi-fact star schema for logistics
  - create supply chain KPI tracking system
  - deploy LogiFleet Pulse analytics platform
  - integrate warehouse and fleet telemetry data
  - configure cross-modal logistics intelligence
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to help developers implement LogiFleet Pulse, a real-time logistics intelligence platform that combines warehouse operations, fleet telemetry, and supply chain data into a unified Power BI analytics solution backed by MS SQL Server.

## What LogiFleet Pulse Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that:

- **Unifies disparate data sources**: Warehouse management systems, fleet GPS/telematics, supplier portals, weather/traffic APIs, and customer orders
- **Implements multi-fact star schema**: Custom data warehouse architecture with time-phased dimensions and composite aggregation bridges
- **Provides cross-fact KPI harmonization**: Links warehouse metrics (dwell time, pick rates) with fleet metrics (fuel consumption, idle time)
- **Enables predictive analytics**: Bottleneck detection, maintenance triage, warehouse gravity zone optimization
- **Powers real-time dashboards**: Power BI dashboards with 15-minute refresh cycles and natural language querying

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, ERP systems
- Azure Synapse Analytics (optional, for big data enrichment)

### Database Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema**:
```sql
-- Execute the main schema creation script
-- This creates all tables, relationships, views, and stored procedures
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema deployment script
-- (Assuming the repository contains schema.sql)
:r schema.sql
GO
```

3. **Configure data source connections**:
```json
// config.json (create from config_sample.json)
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SqlServer",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "type": "REST",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telemetry": {
      "type": "MQTT",
      "broker": "${TELEMETRY_BROKER_URL}",
      "topic": "fleet/+/telemetry"
    },
    "weather": {
      "type": "REST",
      "endpoint": "https://api.weather.com/v3",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "refreshInterval": 900
}
```

## Core Data Model

### Fact Tables

The multi-fact star schema includes:

**FactWarehouseOperations**: Tracks all warehouse activities
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING'
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    GravityZoneID INT,
    BatchNumber VARCHAR(50),
    EmployeeKey INT,
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WO_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey)
);

-- Recommended indexes
CREATE NONCLUSTERED INDEX IX_WO_Time ON FactWarehouseOperations(TimeKey) INCLUDE (OperationType, Quantity);
CREATE NONCLUSTERED INDEX IX_WO_Product_Time ON FactWarehouseOperations(ProductKey, TimeKey);
CREATE COLUMNSTORE INDEX CCI_WarehouseOps ON FactWarehouseOperations;
```

**FactFleetTrips**: Captures fleet telemetry and route data
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    AvgSpeedKPH DECIMAL(5,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100), -- 'WEATHER', 'TRAFFIC', 'MECHANICAL', 'LOADING'
    CONSTRAINT FK_FT_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FT_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FT_Time_Vehicle ON FactFleetTrips(TimeKey, VehicleKey);
CREATE NONCLUSTERED INDEX IX_FT_Route ON FactFleetTrips(RouteKey) INCLUDE (DelayMinutes, IdleTimeMinutes);
```

**FactCrossDock**: Direct transfers without long-term storage
```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripID BIGINT,
    OutboundTripID BIGINT,
    DockDurationMinutes INT,
    TransferredQuantity DECIMAL(18,2),
    CONSTRAINT FK_CD_Inbound FOREIGN KEY (InboundTripID) REFERENCES FactFleetTrips(TripID),
    CONSTRAINT FK_CD_Outbound FOREIGN KEY (OutboundTripID) REFERENCES FactFleetTrips(TripID)
);
```

### Dimension Tables

**DimTime**: Time-phased dimension with 15-minute granularity
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfYear INT,
    DayOfMonth INT,
    DayOfWeek INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    ShiftID INT -- 1=Morning, 2=Afternoon, 3=Night
);

-- Populate time dimension
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, Week, DayOfYear, 
                             DayOfMonth, DayOfWeek, Hour, Minute15Bucket, IsWeekend)
        VALUES (
            CONVERT(INT, FORMAT(@CurrentDateTime, 'yyyyMMddHHmm')),
            @CurrentDateTime,
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(DAYOFYEAR, @CurrentDateTime),
            DAY(@CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
```

**DimProductGravity**: Products with calculated gravity scores
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Based on pick frequency
    ValueScore DECIMAL(5,2), -- Based on unit price
    FragilityScore DECIMAL(5,2), -- 0-10 scale
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(20), -- 'HIGH', 'MEDIUM', 'LOW'
    LastVelocityUpdate DATETIME2
);

-- Update gravity scores based on recent operations
CREATE PROCEDURE UpdateProductGravity
AS
BEGIN
    UPDATE pg
    SET VelocityScore = CASE 
        WHEN pick_freq.PicksPerDay >= 50 THEN 10.0
        WHEN pick_freq.PicksPerDay >= 20 THEN 7.5
        WHEN pick_freq.PicksPerDay >= 5 THEN 5.0
        ELSE 2.5
    END,
    LastVelocityUpdate = GETDATE()
    FROM DimProductGravity pg
    INNER JOIN (
        SELECT ProductKey, COUNT(*) / 30.0 AS PicksPerDay
        FROM FactWarehouseOperations
        WHERE OperationType = 'PICKING'
          AND TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm'))
        GROUP BY ProductKey
    ) pick_freq ON pg.ProductKey = pick_freq.ProductKey;
END;
```

**DimGeography**: Hierarchical location dimension
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(50), -- 'WAREHOUSE', 'DISTRIBUTION_CENTER', 'ROUTE_NODE'
    Address NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    ParentGeographyKey INT,
    CONSTRAINT FK_Geography_Parent FOREIGN KEY (ParentGeographyKey) REFERENCES DimGeography(GeographyKey)
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Load warehouse operations incrementally
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- If no timestamp provided, get the last successful load
    IF @LastLoadTimestamp IS NULL
        SELECT @LastLoadTimestamp = ISNULL(MAX(LastLoadTime), '1900-01-01')
        FROM ETLLoadLog WHERE TableName = 'FactWarehouseOperations' AND Status = 'SUCCESS';
    
    -- Stage data from external source (using OPENROWSET or linked server)
    INSERT INTO FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, OperationType, 
                                          Quantity, DwellTimeMinutes, PickRateUnitsPerHour, GravityZoneID)
    SELECT 
        CONVERT(INT, FORMAT(wo.OperationDateTime, 'yyyyMMddHHmm')) AS TimeKey,
        p.ProductKey,
        w.GeographyKey AS WarehouseKey,
        wo.OperationType,
        wo.Quantity,
        DATEDIFF(MINUTE, wo.StartTime, wo.EndTime) AS DwellTimeMinutes,
        CASE WHEN wo.OperationType = 'PICKING' 
             THEN wo.Quantity / NULLIF(DATEDIFF(MINUTE, wo.StartTime, wo.EndTime), 0) * 60.0
             ELSE NULL 
        END AS PickRateUnitsPerHour,
        pg.RecommendedZone AS GravityZoneID
    FROM OPENROWSET('SQLNCLI', '${WMS_CONNECTION_STRING}',
        'SELECT * FROM WarehouseOperations WHERE LastModified > ?', @LastLoadTimestamp) AS wo
    INNER JOIN DimProductGravity pg ON wo.SKU = pg.SKU
    INNER JOIN DimGeography w ON wo.WarehouseID = w.LocationID
    WHERE wo.OperationDateTime > @LastLoadTimestamp;
    
    -- Log the load
    INSERT INTO ETLLoadLog (TableName, LastLoadTime, RowsLoaded, Status)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT, 'SUCCESS');
END;
```

### Cross-Fact KPI Queries

```sql
-- Calculate composite KPI: Dwell time vs. fleet idle cost per route
CREATE PROCEDURE GetDwellTimeFleetIdleCost
    @StartDate DATE,
    @EndDate DATE,
    @MinGravityScore DECIMAL(5,2) = 7.0
AS
BEGIN
    WITH WarehouseDwell AS (
        SELECT 
            wo.ProductKey,
            pg.SKU,
            pg.GravityScore,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
            SUM(wo.Quantity) AS TotalQuantity
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        INNER JOIN DimProductGravity pg ON wo.ProductKey = pg.ProductKey
        WHERE t.FullDateTime BETWEEN @StartDate AND @EndDate
          AND wo.OperationType IN ('PUTAWAY', 'PICKING')
          AND pg.GravityScore >= @MinGravityScore
        GROUP BY wo.ProductKey, pg.SKU, pg.GravityScore
    ),
    FleetIdle AS (
        SELECT 
            ft.RouteKey,
            r.RouteName,
            AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
            SUM(ft.FuelConsumedLiters * 1.5) AS TotalIdleCostUSD -- Assuming $1.5/liter
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
        WHERE t.FullDateTime BETWEEN @StartDate AND @EndDate
        GROUP BY ft.RouteKey, r.RouteName
    )
    SELECT 
        wd.SKU,
        wd.GravityScore,
        wd.AvgDwellTimeMinutes,
        wd.TotalQuantity,
        fi.RouteName,
        fi.AvgIdleTimeMinutes,
        fi.TotalIdleCostUSD,
        -- Composite inefficiency score
        (wd.AvgDwellTimeMinutes * wd.GravityScore) + (fi.AvgIdleTimeMinutes * fi.TotalIdleCostUSD / 100.0) AS InefficencyScore
    FROM WarehouseDwell wd
    CROSS JOIN FleetIdle fi -- For demonstration; use actual route-product mapping
    ORDER BY InefficencyScore DESC;
END;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks using historical patterns
CREATE PROCEDURE PredictBottlenecks
    @ForecastHorizonHours INT = 24
AS
BEGIN
    -- Analyze historical patterns for similar time periods
    WITH HistoricalPatterns AS (
        SELECT 
            t.DayOfWeek,
            t.Hour,
            wo.WarehouseKey,
            AVG(wo.DwellTimeMinutes) AS AvgDwell,
            STDEV(wo.DwellTimeMinutes) AS StdDevDwell,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
        GROUP BY t.DayOfWeek, t.Hour, wo.WarehouseKey
        HAVING COUNT(*) > 10
    ),
    ForecastPeriods AS (
        SELECT DISTINCT
            DayOfWeek,
            Hour,
            TimeKey
        FROM DimTime
        WHERE FullDateTime BETWEEN GETDATE() AND DATEADD(HOUR, @ForecastHorizonHours, GETDATE())
    )
    SELECT 
        fp.TimeKey,
        dt.FullDateTime AS ForecastDateTime,
        w.LocationName AS WarehouseName,
        hp.AvgDwell,
        hp.StdDevDwell,
        hp.OperationCount AS HistoricalVolume,
        CASE 
            WHEN hp.AvgDwell > (SELECT AVG(AvgDwell) * 1.5 FROM HistoricalPatterns) THEN 'HIGH_RISK'
            WHEN hp.AvgDwell > (SELECT AVG(AvgDwell) * 1.2 FROM HistoricalPatterns) THEN 'MEDIUM_RISK'
            ELSE 'LOW_RISK'
        END AS BottleneckRisk
    FROM ForecastPeriods fp
    INNER JOIN DimTime dt ON fp.TimeKey = dt.TimeKey
    INNER JOIN HistoricalPatterns hp ON fp.DayOfWeek = hp.DayOfWeek AND fp.Hour = hp.Hour
    INNER JOIN DimGeography w ON hp.WarehouseKey = w.GeographyKey
    WHERE hp.AvgDwell > (SELECT AVG(AvgDwell) FROM HistoricalPatterns)
    ORDER BY BottleneckRisk DESC, fp.TimeKey;
END;
```

## Power BI Integration

### Connecting to SQL Server

1. **Open Power BI Desktop**
2. **Get Data** → **SQL Server**
3. **Connection Settings**:
```
Server: ${SQL_SERVER_HOST}
Database: LogiFleetPulse
Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)
Advanced options:
  - Command timeout: 600 seconds
  - Enable query folding: Yes
```

### Key DAX Measures

**Total Dwell Time by Gravity Zone**:
```dax
TotalDwellTime = 
CALCULATE(
    SUM(FactWarehouseOperations[DwellTimeMinutes]),
    ALLEXCEPT(DimProductGravity, DimProductGravity[RecommendedZone])
)
```

**Fleet Efficiency Ratio**:
```dax
FleetEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]) * 
    (1 + SUM(FactFleetTrips[IdleTimeMinutes]) / 
     CALCULATE(SUM(FactFleetTrips[IdleTimeMinutes]) + 
               DATEDIFF(MIN(FactFleetTrips[DepartureTime]), 
                        MAX(FactFleetTrips[ArrivalTime]), MINUTE)))
)
```

**Cross-Fact KPI: Warehouse-to-Fleet Impact**:
```dax
WarehouseFleetImpact = 
VAR HighGravityDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DimProductGravity[GravityScore] >= 7
    )
VAR RouteDelayCorrelation = 
    CALCULATE(
        AVERAGE(FactFleetTrips[DelayMinutes]),
        FactFleetTrips[DelayReason] = "LOADING"
    )
RETURN
    HighGravityDwell * 0.6 + RouteDelayCorrelation * 0.4
```

### Row-Level Security

```dax
-- In Power BI, create a role "RegionalManager"
-- Add DAX filter on DimGeography:
[Region] = USERNAME()

-- Or for more complex scenarios:
VAR UserEmail = USERPRINCIPALNAME()
VAR UserRegion = LOOKUPVALUE(
    UserAccessTable[Region],
    UserAccessTable[Email], UserEmail
)
RETURN [Region] = UserRegion
```

## Automated Alerting

### Email Alerts for KPI Breaches

```sql
-- Configure Database Mail first, then create alert procedure
CREATE PROCEDURE SendKPIAlert
    @KPIName VARCHAR(100),
    @ThresholdValue DECIMAL(18,2),
    @ActualValue DECIMAL(18,2),
    @AlertLevel VARCHAR(20) = 'WARNING'
AS
BEGIN
    DECLARE @Subject NVARCHAR(255) = @AlertLevel + ': ' + @KPIName + ' Threshold Breach';
    DECLARE @Body NVARCHAR(MAX) = 
        'Alert: ' + @KPIName + ' has exceeded threshold.' + CHAR(13) + CHAR(10) +
        'Threshold: ' + CAST(@ThresholdValue AS VARCHAR(20)) + CHAR(13) + CHAR(10) +
        'Actual Value: ' + CAST(@ActualValue AS VARCHAR(20)) + CHAR(13) + CHAR(10) +
        'Timestamp: ' + CONVERT(VARCHAR(30), GETDATE(), 120);
    
    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = 'LogiFleetPulseAlerts',
        @recipients = '${ALERT_EMAIL_RECIPIENTS}',
        @subject = @Subject,
        @body = @Body;
END;

-- Scheduled job to check KPIs every 15 minutes
CREATE PROCEDURE MonitorKPIs
AS
BEGIN
    -- Check fleet idle time threshold
    DECLARE @AvgIdleTime DECIMAL(10,2);
    SELECT @AvgIdleTime = AVG(IdleTimeMinutes)
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE());
    
    IF @AvgIdleTime > 15.0
        EXEC SendKPIAlert 'Fleet Idle Time', 15.0, @AvgIdleTime, 'WARNING';
    
    -- Check warehouse dwell time for high-gravity items
    DECLARE @HighGravityDwell DECIMAL(10,2);
    SELECT @HighGravityDwell = AVG(wo.DwellTimeMinutes)
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity pg ON wo.ProductKey = pg.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE pg.GravityScore >= 7.0
      AND t.FullDateTime >= DATEADD(HOUR, -4, GETDATE());
    
    IF @HighGravityDwell > 72.0
        EXEC SendKPIAlert 'High-Gravity SKU Dwell Time', 72.0, @HighGravityDwell, 'CRITICAL';
END;

-- Create SQL Server Agent job to run every 15 minutes
USE msdb;
EXEC sp_add_job @job_name = 'LogiFleet_KPI_Monitor';
EXEC sp_add_jobstep 
    @job_name = 'LogiFleet_KPI_Monitor',
    @step_name = 'Check KPIs',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC MonitorKPIs';
EXEC sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;
EXEC sp_attach_schedule 
    @job_name = 'LogiFleet_KPI_Monitor',
    @schedule_name = 'Every15Minutes';
EXEC sp_add_jobserver @job_name = 'LogiFleet_KPI_Monitor';
```

## Common Patterns

### Pattern 1: Time-Phased Dimension Querying

```sql
-- Query operations by shift and day-of-week pattern
SELECT 
    t.DayOfWeek,
    CASE t.ShiftID 
        WHEN 1 THEN 'Morning (6AM-2PM)'
        WHEN 2 THEN 'Afternoon (2PM-10PM)'
        WHEN 3 THEN 'Night (10PM-6AM)'
    END AS Shift,
    COUNT(*) AS OperationCount,
    AVG(wo.PickRateUnitsPerHour) AS AvgPickRate
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE wo.OperationType = 'PICKING'
  AND t.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
GROUP BY t.DayOfWeek, t.ShiftID
ORDER BY t.DayOfWeek, t.ShiftID;
```

### Pattern 2: Gravity Zone Rebalancing

```sql
-- Recommend SKU relocations based on velocity drift
WITH CurrentZonePlacement AS (
    SELECT 
        ProductKey,
        SKU,
        RecommendedZone,
        GravityScore
    FROM DimProductGravity
),
ActualPickFrequency AS (
    SELECT 
        ProductKey,
        COUNT(*) / 30.0 AS RecentPicksPerDay
    FROM FactWarehouseOperations
    WHERE OperationType = 'PICKING'
      AND TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm'))
    GROUP BY ProductKey
)
SELECT 
    czp.SKU,
    czp.RecommendedZone AS CurrentZone,
    CASE 
        WHEN apf.RecentPicksPerDay >= 50 THEN 'HIGH'
        WHEN apf.RecentPicksPerDay >= 20 THEN 'MEDIUM'
        ELSE 'LOW'
    END AS SuggestedZone,
    apf.RecentPicksPerDay,
    CASE 
        WHEN czp.RecommendedZone != CASE 
            WHEN apf.RecentPicksPerDay >= 50 THEN 'HIGH'
            WHEN apf.RecentPicksPerDay >= 20 THEN 'MEDIUM'
            ELSE 'LOW'
        END THEN 'RELOCATE'
        ELSE 'KEEP'
    END AS Action
FROM CurrentZonePlacement czp
LEFT JOIN ActualPickFrequency apf ON czp.ProductKey = apf.ProductKey
WHERE czp.RecommendedZone != CASE 
    WHEN apf.RecentPicksPerDay >= 50 THEN 'HIGH'
    WHEN apf.RecentPicksPerDay >= 20 THEN 'MEDIUM'
    ELSE 'LOW'
END;
```

### Pattern 3: External Data Integration (Weather API)

```sql
-- Create external table for weather data (using Polybase or OPENROWSET)
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://api.weather.com/v3/weather',
    CREDENTIAL = WeatherAPICredential
);

-- Correlate weather delays with route performance
SELECT 
    ft.RouteKey,
    r.RouteName,
    t.FullDateTime,
    ft.DelayMinutes,
    ft.DelayReason,
    w.WeatherCondition,
    w.Temperature,
    w.Precipitation
FROM FactFleetTrips ft
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
OUTER APPLY (
    SELECT TOP 1 
        Condition AS WeatherCondition,
        Temp AS Temperature,
        Precip AS Precipitation
    FROM OPENROWSET(
        BULK 'https://api.weather.com/v3/weather/conditions',
        FORMAT = 'JSON'
    ) AS WeatherData
    WHERE WeatherData.Latitude = r.StartLatitude
      AND WeatherData.Longitude = r.StartLongitude
      AND WeatherData.Timestamp <= t.FullDateTime
    ORDER BY ABS(DATEDIFF(MINUTE, WeatherData.Timestamp, t.FullDateTime))
) w
WHERE ft.DelayReason = 'WEATHER'
  AND t.FullDateTime >= DATEADD(MONTH, -3, GETDATE());
```

## Troubleshooting

### Issue: Power BI Dashboard Refresh Timeout

**Symptom**: Dashboard refresh fails with "Query timeout expired"

**Solution**:
1. Check for missing indexes on fact tables:
```sql
-- Find missing indexes
SELECT 
    migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) AS improvement_measure,
    'CREATE INDEX IX_' + mid.statement + '_' + ISNULL(mid.equality_columns, '') + 
    ISNULL(mid.inequality_columns, '') + 
    ' ON ' + mid.statement + ' (' + ISNULL(mid.equality_columns, '') + 
    CASE WHEN mid.inequality_columns IS NOT NULL THEN ',' + mid.inequality_columns ELSE '' END + ')' AS create_index_statement
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_
