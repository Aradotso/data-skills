---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI dashboards for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logistics data warehouse"
  - "create supply chain analytics dashboard"
  - "build fleet management reporting system"
  - "implement warehouse operations tracking"
  - "configure Power BI for logistics KPIs"
  - "design multi-fact star schema for supply chain"
  - "integrate fleet telemetry with warehouse data"
  - "deploy logistics intelligence platform"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics solution for logistics operations, combining MS SQL Server backend with Power BI visualization. It implements a multi-fact star schema that unifies warehouse operations, fleet management, inventory tracking, and external data sources into a single semantic layer for cross-functional supply chain analytics.

**Key capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse operations tracking (receiving, putaway, picking, packing, shipping)
- Fleet telemetry integration (GPS, fuel, driver behavior, maintenance)
- Cross-dock operations monitoring
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Real-time KPI dashboards with role-based access

## Installation & Setup

### Prerequisites

```bash
# Required software
- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) 18+
- Power BI Desktop (latest version)
- (Optional) Azure Synapse Analytics for external enrichment
```

### Database Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Connect to your SQL Server instance via SSMS
-- Run the schema deployment script
:r schema_deploy.sql

-- Verify table creation
SELECT TABLE_NAME 
FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_NAME;
```

3. **Configure data source connections:**

```json
// config.json (copy from config_sample.json)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "Windows",
    "connection_timeout": 30
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

## Core Data Model

### Fact Tables

#### FactWarehouseOperations

```sql
-- Track warehouse operations with granular metrics
CREATE TABLE dbo.FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DwellTimeMinutes DECIMAL(10,2),
    OperatorID INT,
    GravityZoneKey INT,
    ProcessingTimeSec DECIMAL(10,2),
    ErrorFlag BIT DEFAULT 0,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Create clustered columnstore index for analytics performance
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOps 
ON dbo.FactWarehouseOperations;

-- Create nonclustered indexes for common filters
CREATE NONCLUSTERED INDEX IX_TimeKey ON dbo.FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_ProductKey ON dbo.FactWarehouseOperations(ProductKey);
```

#### FactFleetTrips

```sql
-- Track fleet operations and telemetry
CREATE TABLE dbo.FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    TripDistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    AverageSpeedKph DECIMAL(10,2),
    MaxSpeedKph DECIMAL(10,2),
    RouteKey INT,
    WeatherCondition VARCHAR(50),
    DelayMinutes DECIMAL(10,2),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON dbo.FactFleetTrips;
```

### Dimension Tables

#### DimTime (Time-Phased)

```sql
-- 15-minute granularity time dimension
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT,
    TimeOfDay TIME,
    Hour TINYINT,
    QuarterHour TINYINT, -- 0, 15, 30, 45
    DayOfWeek TINYINT,
    DayName VARCHAR(20),
    IsWeekend BIT,
    IsHoliday BIT,
    FiscalYear INT,
    FiscalQuarter TINYINT,
    FiscalMonth TINYINT
);

-- Populate time dimension (example for one month)
DECLARE @StartDate DATETIME = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2026-02-01 00:00:00';
DECLARE @CurrentTime DATETIME = @StartDate;

WHILE @CurrentTime < @EndDate
BEGIN
    INSERT INTO dbo.DimTime (
        TimeKey, FullDateTime, DateKey, TimeOfDay, Hour, 
        QuarterHour, DayOfWeek, DayName, IsWeekend
    )
    VALUES (
        CAST(FORMAT(@CurrentTime, 'yyyyMMddHHmm') AS INT),
        @CurrentTime,
        CAST(FORMAT(@CurrentTime, 'yyyyMMdd') AS INT),
        CAST(@CurrentTime AS TIME),
        DATEPART(HOUR, @CurrentTime),
        (DATEPART(MINUTE, @CurrentTime) / 15) * 15,
        DATEPART(WEEKDAY, @CurrentTime),
        DATENAME(WEEKDAY, @CurrentTime),
        CASE WHEN DATEPART(WEEKDAY, @CurrentTime) IN (1, 7) THEN 1 ELSE 0 END
    );
    
    SET @CurrentTime = DATEADD(MINUTE, 15, @CurrentTime);
END;
```

#### DimProductGravity (Warehouse Gravity Zones)

```sql
-- Assign gravity scores to products based on velocity, value, fragility
CREATE TABLE dbo.DimProductGravity (
    GravityKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductKey INT NOT NULL,
    VelocityScore DECIMAL(5,2), -- Pick frequency normalized 0-100
    ValueScore DECIMAL(5,2), -- Product value normalized 0-100
    FragilityScore DECIMAL(5,2), -- Handling complexity 0-100
    CompositeGravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2),
    RecommendedZone VARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    DistanceFromDock DECIMAL(10,2), -- Meters
    EffectiveDate DATE,
    ExpiryDate DATE,
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Create function to calculate optimal zone assignment
CREATE FUNCTION dbo.fn_CalculateOptimalZone(@GravityScore DECIMAL(5,2))
RETURNS VARCHAR(50)
AS
BEGIN
    RETURN CASE 
        WHEN @GravityScore >= 75 THEN 'High-Gravity'
        WHEN @GravityScore >= 50 THEN 'Medium-Gravity'
        ELSE 'Low-Gravity'
    END;
END;
```

## Data Loading Patterns

### Incremental Load Stored Procedure

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @LastLoadTimeKey INT = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get last loaded time if not provided
    IF @LastLoadTimeKey IS NULL
    BEGIN
        SELECT @LastLoadTimeKey = ISNULL(MAX(TimeKey), 0)
        FROM dbo.FactWarehouseOperations;
    END;
    
    -- Insert new operations from staging
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        QuantityHandled, DwellTimeMinutes, OperatorID,
        GravityZoneKey, ProcessingTimeSec
    )
    SELECT 
        t.TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.OperatorID,
        pg.GravityKey,
        DATEDIFF(SECOND, s.StartTime, s.EndTime) AS ProcessingTimeSec
    FROM staging.WarehouseOperations s
    INNER JOIN DimTime t ON t.FullDateTime = DATEADD(MINUTE, 
        (DATEPART(MINUTE, s.StartTime) / 15) * 15, 
        CAST(CAST(s.StartTime AS DATE) AS DATETIME) + 
        CAST(CAST(DATEPART(HOUR, s.StartTime) AS VARCHAR) + ':00' AS TIME)
    )
    INNER JOIN DimWarehouse w ON w.WarehouseCode = s.WarehouseCode
    INNER JOIN DimProduct p ON p.SKU = s.ProductSKU
    LEFT JOIN DimProductGravity pg ON pg.ProductKey = p.ProductKey
        AND GETDATE() BETWEEN pg.EffectiveDate AND ISNULL(pg.ExpiryDate, '9999-12-31')
    WHERE t.TimeKey > @LastLoadTimeKey
        AND s.LoadedFlag = 0;
    
    -- Mark staging records as loaded
    UPDATE staging.WarehouseOperations
    SET LoadedFlag = 1
    WHERE LoadedFlag = 0;
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' new warehouse operations.';
END;
```

### Fleet Telemetry Integration

```sql
-- Stored procedure to process fleet telemetry stream
CREATE PROCEDURE dbo.usp_ProcessFleetTelemetry
AS
BEGIN
    -- Process raw GPS/telemetry data into trip segments
    WITH TripSegments AS (
        SELECT 
            VehicleID,
            DriverID,
            MIN(Timestamp) AS TripStart,
            MAX(Timestamp) AS TripEnd,
            SUM(DistanceKm) AS TotalDistance,
            SUM(FuelUsedLiters) AS TotalFuel,
            SUM(CASE WHEN Speed = 0 AND EngineOn = 1 
                THEN DurationMinutes ELSE 0 END) AS IdleTime,
            AVG(Speed) AS AvgSpeed,
            MAX(Speed) AS MaxSpeed
        FROM staging.FleetTelemetry
        WHERE ProcessedFlag = 0
        GROUP BY VehicleID, DriverID, 
            DATEDIFF(HOUR, Timestamp, '2000-01-01') / 24 -- Group by trip session
    )
    INSERT INTO dbo.FactFleetTrips (
        VehicleKey, DriverKey, StartTimeKey, EndTimeKey,
        TripDistanceKm, FuelConsumedLiters, IdleTimeMinutes,
        AverageSpeedKph, MaxSpeedKph
    )
    SELECT 
        v.VehicleKey,
        d.DriverKey,
        tStart.TimeKey AS StartTimeKey,
        tEnd.TimeKey AS EndTimeKey,
        ts.TotalDistance,
        ts.TotalFuel,
        ts.IdleTime,
        ts.AvgSpeed,
        ts.MaxSpeed
    FROM TripSegments ts
    INNER JOIN DimVehicle v ON v.VehicleID = ts.VehicleID
    INNER JOIN DimDriver d ON d.DriverID = ts.DriverID
    INNER JOIN DimTime tStart ON tStart.FullDateTime = 
        DATEADD(MINUTE, (DATEPART(MINUTE, ts.TripStart) / 15) * 15,
        CAST(CAST(ts.TripStart AS DATE) AS DATETIME) + 
        CAST(CAST(DATEPART(HOUR, ts.TripStart) AS VARCHAR) + ':00' AS TIME))
    INNER JOIN DimTime tEnd ON tEnd.FullDateTime = 
        DATEADD(MINUTE, (DATEPART(MINUTE, ts.TripEnd) / 15) * 15,
        CAST(CAST(ts.TripEnd AS DATE) AS DATETIME) + 
        CAST(CAST(DATEPART(HOUR, ts.TripEnd) AS VARCHAR) + ':00' AS TIME));
END;
```

## Cross-Fact KPI Queries

### Warehouse Dwell Time vs Fleet Idle Time Correlation

```sql
-- Analyze correlation between warehouse dwell and fleet idle time
SELECT 
    t.DateKey,
    t.DayName,
    w.WarehouseName,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdle,
    COUNT(DISTINCT wo.OperationID) AS WarehouseOps,
    COUNT(DISTINCT ft.TripID) AS FleetTrips,
    -- Composite inefficiency score
    (AVG(wo.DwellTimeMinutes) / 60.0) * (AVG(ft.IdleTimeMinutes) / 60.0) AS InefficiencyScore
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN dbo.DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
LEFT JOIN dbo.FactFleetTrips ft ON ft.OriginGeographyKey = w.GeographyKey
    AND ft.StartTimeKey BETWEEN t.TimeKey - 100 AND t.TimeKey + 100 -- Within ~25 hours
WHERE t.DateKey BETWEEN 20260601 AND 20260630
GROUP BY t.DateKey, t.DayName, w.WarehouseName
HAVING COUNT(DISTINCT wo.OperationID) > 10
ORDER BY InefficiencyScore DESC;
```

### Gravity Zone Optimization Analysis

```sql
-- Identify products that should be reassigned to different gravity zones
WITH CurrentPerformance AS (
    SELECT 
        p.ProductKey,
        p.ProductName,
        pg.RecommendedZone AS CurrentZone,
        pg.CompositeGravityScore AS CurrentScore,
        COUNT(*) AS PickCount,
        AVG(wo.ProcessingTimeSec) AS AvgPickTime,
        AVG(pg.DistanceFromDock) AS AvgDistance
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN dbo.DimProductGravity pg ON wo.GravityZoneKey = pg.GravityKey
    WHERE wo.OperationType = 'Picking'
        AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY p.ProductKey, p.ProductName, pg.RecommendedZone, 
             pg.CompositeGravityScore, pg.DistanceFromDock
),
RecalculatedScores AS (
    SELECT 
        ProductKey,
        ProductName,
        CurrentZone,
        CurrentScore,
        PickCount,
        AvgPickTime,
        AvgDistance,
        -- Recalculate velocity score based on recent activity
        (PickCount / 30.0) * 10 AS NewVelocityScore, -- Picks per day * 10
        dbo.fn_CalculateOptimalZone(
            (PickCount / 30.0) * 10 * 0.5 + CurrentScore * 0.5
        ) AS RecommendedZone
    FROM CurrentPerformance
)
SELECT 
    ProductKey,
    ProductName,
    CurrentZone,
    RecommendedZone,
    PickCount,
    AvgPickTime,
    AvgDistance,
    CASE 
        WHEN CurrentZone <> RecommendedZone THEN 'Reassign Needed'
        ELSE 'Optimal'
    END AS ActionRequired,
    -- Estimated time savings (in seconds per pick)
    CASE 
        WHEN RecommendedZone = 'High-Gravity' THEN (AvgDistance / 1.5) * 0.1
        WHEN RecommendedZone = 'Medium-Gravity' THEN 0
        ELSE -((AvgDistance / 1.5) * 0.05)
    END AS EstimatedTimeSavings
FROM RecalculatedScores
WHERE CurrentZone <> RecommendedZone
ORDER BY PickCount DESC, EstimatedTimeSavings DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks using time-series pattern analysis
CREATE VIEW dbo.vw_BottleneckPrediction AS
WITH HourlyMetrics AS (
    SELECT 
        t.DateKey,
        t.Hour,
        w.WarehouseKey,
        w.WarehouseName,
        COUNT(wo.OperationID) AS OperationCount,
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        STDEV(wo.DwellTimeMinutes) AS StdDevDwell,
        AVG(wo.ProcessingTimeSec) AS AvgProcessing
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN dbo.DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    WHERE t.DateKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY t.DateKey, t.Hour, w.WarehouseKey, w.WarehouseName
),
MovingAverages AS (
    SELECT 
        *,
        AVG(OperationCount) OVER (
            PARTITION BY WarehouseKey, Hour 
            ORDER BY DateKey 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS MA7_OperationCount,
        AVG(AvgDwell) OVER (
            PARTITION BY WarehouseKey, Hour 
            ORDER BY DateKey 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS MA7_AvgDwell
    FROM HourlyMetrics
)
SELECT 
    WarehouseKey,
    WarehouseName,
    DateKey,
    Hour,
    OperationCount,
    MA7_OperationCount,
    AvgDwell,
    MA7_AvgDwell,
    -- Bottleneck score: deviation from moving average
    CASE 
        WHEN OperationCount > MA7_OperationCount * 1.3 
             AND AvgDwell > MA7_AvgDwell * 1.2 
        THEN 'High Risk'
        WHEN OperationCount > MA7_OperationCount * 1.15 
             OR AvgDwell > MA7_AvgDwell * 1.1
        THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM MovingAverages
WHERE DateKey = CAST(FORMAT(GETDATE(), 'yyyyMMdd') AS INT);
```

## Power BI Integration

### Connect to SQL Server

```powerquery
// Power Query M code for SQL connection
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse",
        [
            Query = "SELECT * FROM dbo.FactWarehouseOperations WHERE TimeKey >= " & 
                    Text.From(Number.From(DateTime.AddDays(DateTime.LocalNow(), -90))),
            CommandTimeout = #duration(0, 0, 5, 0)
        ]
    )
in
    Source
```

### Key DAX Measures

```dax
// Total Warehouse Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time with Time Intelligence
Avg Dwell Time = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    USERELATIONSHIP(FactWarehouseOperations[TimeKey], DimTime[TimeKey])
)

// Dwell Time vs Previous Period
Dwell Time vs PP = 
VAR CurrentDwell = [Avg Dwell Time]
VAR PreviousDwell = 
    CALCULATE(
        [Avg Dwell Time],
        DATEADD(DimTime[FullDateTime], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentDwell - PreviousDwell, PreviousDwell, 0)

// Fleet Idle Time %
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + 
    SUM(FactFleetTrips[TripDistanceKm]) / AVERAGE(FactFleetTrips[AverageSpeedKph]) * 60,
    0
)

// Cross-Fact Inefficiency Score
Logistics Inefficiency Score = 
VAR WarehouseDwell = [Avg Dwell Time] / 60
VAR FleetIdle = [Fleet Idle %]
RETURN
    WarehouseDwell * FleetIdle * 100

// Gravity Zone Compliance
Gravity Zone Compliance % = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[ErrorFlag] = FALSE
    ),
    COUNTROWS(FactWarehouseOperations),
    0
)

// Predictive Bottleneck Alert
Bottleneck Alert = 
VAR CurrentOps = [Total Operations]
VAR MA7 = 
    CALCULATE(
        [Total Operations],
        DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -7, DAY)
    ) / 7
RETURN
    IF(CurrentOps > MA7 * 1.3, "⚠️ High Risk", "✓ Normal")
```

### Row-Level Security

```dax
// RLS for warehouse managers (restrict to their warehouse)
[WarehouseKey] = LOOKUPVALUE(
    DimUser[AssignedWarehouseKey],
    DimUser[Username], USERNAME()
)

// RLS for regional managers (restrict to their region)
DimWarehouse[RegionKey] IN 
    CALCULATETABLE(
        VALUES(DimUser[AssignedRegionKey]),
        DimUser[Username] = USERNAME()
    )
```

## Automated Alerting

### SQL Server Agent Job for Alerts

```sql
-- Create alert procedure
CREATE PROCEDURE dbo.usp_CheckAndSendAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @AlertRecipients NVARCHAR(500) = COALESCE('${ALERT_EMAIL}', 'logistics@company.com');
    
    -- Check for high idle time
    IF EXISTS (
        SELECT 1 
        FROM dbo.FactFleetTrips ft
        INNER JOIN dbo.DimTime t ON ft.StartTimeKey = t.TimeKey
        WHERE t.DateKey = CAST(FORMAT(GETDATE(), 'yyyyMMdd') AS INT)
        GROUP BY ft.VehicleKey
        HAVING AVG(ft.IdleTimeMinutes) > 0.15 * AVG(ft.TripDistanceKm / ft.AverageSpeedKph * 60)
    )
    BEGIN
        SET @AlertMessage = 'ALERT: One or more vehicles exceeded 15% idle time threshold today.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @AlertRecipients,
            @subject = 'LogiFleet Pulse: High Idle Time Alert',
            @body = @AlertMessage;
    END;
    
    -- Check for warehouse bottlenecks
    IF EXISTS (
        SELECT 1 
        FROM dbo.vw_BottleneckPrediction
        WHERE BottleneckRisk = 'High Risk'
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Warehouse bottleneck predicted in the next hour. Review dwell times.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @AlertRecipients,
            @subject = 'LogiFleet Pulse: Bottleneck Warning',
            @body = @AlertMessage;
    END;
END;

-- Schedule job to run every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleet_Alert_Monitor',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet_Alert_Monitor',
    @step_name = N'Check_Alerts',
    @subsystem = N'TSQL',
    @command = N'EXEC dbo.usp_CheckAndSendAlerts',
    @database_name = N'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'LogiFleet_Alert_Monitor',
    @schedule_name = N'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = N'LogiFleet_Alert_Monitor';
```

## Common Troubleshooting

### Issue: Power BI Dashboard Slow Refresh

**Solution 1: Optimize fact table indexes**
```sql
-- Add filtered nonclustered indexes for common date ranges
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_TimeKey_Includes
ON dbo.FactWarehouseOperations (TimeKey)
INCLUDE (ProductKey, WarehouseKey, DwellTimeMinutes, ProcessingTimeSec)
WHERE TimeKey >= 202601010000; -- Adjust based on retention policy

-- Update statistics
UPDATE STATISTICS dbo.FactWarehouseOperations WITH FULLSCAN;
```

**Solution 2: Use aggregated summary tables**
```sql
-- Create daily aggregates for historical data
CREATE TABLE dbo.FactWarehouseOperations_Daily (
    DateKey INT,
    WarehouseKey INT,
    ProductKey INT,
    TotalOperations INT,
    AvgDwellTime DECIMAL(10,2),
    AvgProcessingTime DECIMAL(10,2),
    PRIMARY KEY (DateKey, WarehouseKey, ProductKey)
);

-- Populate daily aggregates (run nightly)
INSERT INTO dbo.FactWarehouseOperations_Daily
SELECT 
    t.DateKey,
    wo.WarehouseKey,
    wo.ProductKey,
    COUNT(*) AS TotalOperations,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(wo.ProcessingTimeSec) AS AvgProcessingTime
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateKey = CAST(FORMAT(DATEADD(DAY, -1, GETDATE()), 'yyyyMMdd') AS INT)
GROUP BY t.DateKey, wo.WarehouseKey, wo.ProductKey;
```

### Issue: Time Dimension Misalignment

**Problem:** Operations not mapping to 15-minute time buckets correctly.

**Solution:**
```sql
-- Create function to round to nearest 15-minute interval
CREATE FUNCTION dbo.fn_RoundTo15Minutes(@InputTime DATETIME)
RETURNS DATETIME
AS
BEGIN
    RETURN DATEADD(MINUTE, 
        (DATEPART(MINUTE, @InputTime) / 15) * 15 - DATEPART(MINUTE, @InputTime),
        DATEADD(SECOND, -DATEPART(SECOND, @InputTime), @InputTime)
    );
END;

-- Use in ETL
SELECT 
    dbo.fn_RoundTo15Minutes(s.OperationTimestamp) AS RoundedTime,
    t.TimeKey
FROM staging.WarehouseOperations s
INNER JOIN DimTime t ON t.FullDateTime = dbo.fn_RoundTo15Minutes(s.OperationTimestamp);
```

### Issue: External API Data Missing

**Problem:** Weather or traffic API calls failing, leaving NULL values.

**Solution:**
```sql
-- Create fallback logic for missing external data
UPDATE dbo.FactFleetTrips
SET WeatherCondition = 'Unknown'
WHERE WeatherCondition IS NULL 
    AND StartTimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT);

-- Log missing data for monitoring
INSERT INTO dbo.DataQualityLog (TableName, I
