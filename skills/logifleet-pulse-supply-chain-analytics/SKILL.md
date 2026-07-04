---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet, warehouse, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logistics data warehouse with Power BI"
  - "configure multi-fact star schema for fleet operations"
  - "implement warehouse gravity zone optimization"
  - "create cross-modal supply chain dashboard"
  - "build logistics intelligence engine with SQL Server"
  - "integrate fleet telemetry with warehouse operations"
  - "setup predictive bottleneck detection for logistics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced MS SQL Server and Power BI-based data warehousing solution for logistics and supply chain management. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and inventory data
- **Cross-fact KPI harmonization** for unified logistics intelligence
- **Real-time dashboards** with 15-minute refresh granularity
- **Warehouse Gravity Zones™** spatial optimization
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Fleet triage engine** with proactive maintenance prioritization

The platform integrates data from WMS, GPS/telematics, supplier portals, weather APIs, and order management systems into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQLCMD or SQL Server Management Studio (SSMS)
- Network access to data sources (WMS, telematics, APIs)

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```bash
# Using SQLCMD (Windows)
sqlcmd -S YOUR_SQL_SERVER -d LogiFleetDB -i schema/01_create_database.sql
sqlcmd -S YOUR_SQL_SERVER -d LogiFleetDB -i schema/02_dimension_tables.sql
sqlcmd -S YOUR_SQL_SERVER -d LogiFleetDB -i schema/03_fact_tables.sql
sqlcmd -S YOUR_SQL_SERVER -d LogiFleetDB -i schema/04_bridge_tables.sql
sqlcmd -S YOUR_SQL_SERVER -d LogiFleetDB -i schema/05_stored_procedures.sql
sqlcmd -S YOUR_SQL_SERVER -d LogiFleetDB -i schema/06_indexes_partitions.sql

# Or using SSMS: Open and execute scripts in order
```

### Step 3: Configure Data Sources

```json
// config.json (copy from config_sample.json)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetDB",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_feed": "${TELEMATICS_STREAM_URL}",
    "weather_api_key": "${WEATHER_API_KEY}",
    "traffic_api_key": "${TRAFFIC_API_KEY}"
  },
  "refresh_intervals": {
    "warehouse_operations_minutes": 15,
    "fleet_trips_minutes": 5,
    "external_apis_minutes": 30
  }
}
```

### Step 4: Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection string when prompted
3. Configure row-level security roles (optional)
4. Publish to Power BI Service or save as .pbix

## Core Data Model Structure

### Fact Tables

#### FactWarehouseOperations

```sql
-- Key metrics: putaway cycles, pick rates, packing times, dwell spans
SELECT 
    TimeKey,
    GeographyKey,
    ProductKey,
    OperationTypeKey,
    QuantityHandled,
    CycleDurationMinutes,
    DwellTimeHours,
    PickAccuracyRate,
    PackingEfficiencyScore
FROM FactWarehouseOperations
WHERE TimeKey >= CONVERT(VARCHAR(8), DATEADD(DAY, -7, GETDATE()), 112);
```

#### FactFleetTrips

```sql
-- Route segments, fuel consumption, idle time, loading events
SELECT 
    TripID,
    TimeKey,
    VehicleKey,
    RouteKey,
    DriverKey,
    DistanceKM,
    FuelConsumedLiters,
    IdleTimeMinutes,
    LoadingDurationMinutes,
    UnloadingDurationMinutes,
    AverageSpeedKMH
FROM FactFleetTrips
WHERE DepartureDateTime >= DATEADD(DAY, -1, GETDATE());
```

#### FactCrossDock

```sql
-- Transfers between inbound/outbound without long-term storage
SELECT 
    TimeKey,
    InboundShipmentKey,
    OutboundShipmentKey,
    ProductKey,
    CrossDockDurationMinutes,
    QuantityTransferred
FROM FactCrossDock
WHERE CrossDockDurationMinutes > 120; -- Flag delays > 2 hours
```

### Dimension Tables

#### DimTime (15-minute granularity)

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME,
    Date DATE,
    HourOfDay TINYINT,
    MinuteQuarter TINYINT, -- 0, 15, 30, 45
    DayOfWeek VARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(6),
    INDEX IX_DimTime_Date (Date)
);
```

#### DimProductGravity (Warehouse Gravity Zones™)

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50),
    ProductName NVARCHAR(200),
    VelocityScore DECIMAL(5,2), -- Picks per day
    ValueScore DECIMAL(10,2),   -- Revenue per unit
    FragilityScore TINYINT,     -- 1-10 scale
    GravityZone VARCHAR(10),    -- HIGH, MEDIUM, LOW
    RecommendedWarehouseZone VARCHAR(20),
    LastRecalculatedDate DATETIME
);
```

## Key Stored Procedures

### Incremental Data Load

```sql
-- Execute every 15 minutes via SQL Server Agent job
EXEC dbo.usp_IncrementalLoadWarehouseOperations 
    @LastLoadTimestamp = '2026-07-04 09:00:00';

EXEC dbo.usp_IncrementalLoadFleetTrips 
    @LastLoadTimestamp = '2026-07-04 09:05:00';
```

### Calculate Gravity Scores

```sql
-- Recalculate warehouse gravity zones weekly
EXEC dbo.usp_RecalculateProductGravityScores 
    @DaysOfHistory = 90,
    @VelocityWeight = 0.5,
    @ValueWeight = 0.3,
    @FragilityWeight = 0.2;
```

### Predictive Bottleneck Detection

```sql
-- Run daily for next-day predictions
EXEC dbo.usp_PredictBottlenecks 
    @PredictionHorizonHours = 24,
    @ConfidenceThreshold = 0.75,
    @OutputTable = 'TempBottleneckPredictions';

-- Query results
SELECT 
    PredictedDateTime,
    BottleneckType, -- 'WAREHOUSE_CONGESTION', 'FLEET_CAPACITY', 'CROSS_DOCK_DELAY'
    AffectedArea,
    ProbabilityScore,
    RecommendedAction
FROM TempBottleneckPredictions
WHERE ProbabilityScore > 0.8
ORDER BY PredictedDateTime, ProbabilityScore DESC;
```

## Common Query Patterns

### Cross-Fact KPI: Dwell Time vs. Fleet Idling Cost

```sql
-- Find correlation between warehouse dwell and fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        f.ProductKey,
        d.Date,
        AVG(f.DwellTimeHours) AS AvgDwellHours
    FROM FactWarehouseOperations f
    JOIN DimTime d ON f.TimeKey = d.TimeKey
    WHERE d.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY f.ProductKey, d.Date
),
FleetIdle AS (
    SELECT 
        r.ProductKey, -- Assumes route-product bridge table
        d.Date,
        SUM(f.IdleTimeMinutes) AS TotalIdleMinutes,
        SUM(f.IdleTimeMinutes * v.HourlyCostUSD / 60) AS TotalIdleCostUSD
    FROM FactFleetTrips f
    JOIN DimTime d ON f.TimeKey = d.TimeKey
    JOIN BridgeRouteProduct r ON f.RouteKey = r.RouteKey
    JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    WHERE d.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY r.ProductKey, d.Date
)
SELECT 
    wd.ProductKey,
    p.ProductName,
    AVG(wd.AvgDwellHours) AS AvgDwellHours,
    AVG(fi.TotalIdleMinutes) AS AvgIdleMinutes,
    AVG(fi.TotalIdleCostUSD) AS AvgIdleCostUSD,
    CORR(wd.AvgDwellHours, fi.TotalIdleMinutes) AS Correlation
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.ProductKey = fi.ProductKey AND wd.Date = fi.Date
JOIN DimProductGravity p ON wd.ProductKey = p.ProductKey
GROUP BY wd.ProductKey, p.ProductName
HAVING COUNT(*) >= 20 -- Minimum sample size
ORDER BY Correlation DESC;
```

### Fleet Utilization by Gravity Zone

```sql
-- Analyze how product gravity affects delivery efficiency
SELECT 
    pg.GravityZone,
    COUNT(DISTINCT ft.TripID) AS TotalTrips,
    AVG(ft.DistanceKM) AS AvgDistanceKM,
    AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(ft.FuelConsumedLiters) AS TotalFuelLiters,
    SUM(ft.FuelConsumedLiters) / SUM(ft.DistanceKM) AS LitersPer100KM
FROM FactFleetTrips ft
JOIN BridgeRouteProduct brp ON ft.RouteKey = brp.RouteKey
JOIN DimProductGravity pg ON brp.ProductKey = pg.ProductKey
JOIN DimTime dt ON ft.TimeKey = dt.TimeKey
WHERE dt.Date >= DATEADD(DAY, -7, GETDATE())
GROUP BY pg.GravityZone
ORDER BY pg.GravityZone;
```

### Anomaly Detection with User Tags

```sql
-- Query tagged anomalies for ML training
SELECT 
    a.AnomalyID,
    a.DetectedDateTime,
    a.AnomalyType,
    a.AffectedEntityType,
    a.AffectedEntityID,
    a.SeverityScore,
    a.SystemReasonCode,
    ut.UserVerification, -- 'TRUE_POSITIVE', 'FALSE_POSITIVE', 'PENDING'
    ut.UserNotes,
    ut.RootCause
FROM dbo.DetectedAnomalies a
LEFT JOIN dbo.UserAnomalyTags ut ON a.AnomalyID = ut.AnomalyID
WHERE a.DetectedDateTime >= DATEADD(DAY, -30, GETDATE())
ORDER BY a.DetectedDateTime DESC;
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

```dax
// Composite KPI: Weighted Logistics Efficiency Score
LogisticsEfficiencyScore = 
VAR WarehouseScore = 
    DIVIDE(
        [TotalQuantityHandled],
        [TotalCycleDuration],
        0
    ) * 0.4
VAR FleetScore = 
    DIVIDE(
        [TotalDistanceKM],
        [TotalIdleTime] + [TotalDistanceKM] / 60,
        0
    ) * 0.6
RETURN
    WarehouseScore + FleetScore
```

```dax
// Time-phased dwell time with previous period comparison
DwellTimeChange% = 
VAR CurrentPeriod = [AvgDwellTimeHours]
VAR PreviousPeriod = 
    CALCULATE(
        [AvgDwellTimeHours],
        DATEADD(DimTime[Date], -7, DAY)
    )
RETURN
    DIVIDE(
        CurrentPeriod - PreviousPeriod,
        PreviousPeriod,
        0
    )
```

### Row-Level Security Configuration

```sql
-- Create security roles in SQL Server
CREATE TABLE dbo.UserRoleSecurity (
    UserEmail VARCHAR(100),
    SecurityRole VARCHAR(50), -- 'EXECUTIVE', 'SUPERVISOR', 'OPERATOR'
    AllowedGeographyKey INT,
    AllowedWarehouseID INT
);

-- Power BI RLS DAX filter
[UserEmail] = USERPRINCIPALNAME() 
    && [GeographyKey] IN VALUES(UserRoleSecurity[AllowedGeographyKey])
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Clustered indexes on fact tables (time-based)
CREATE CLUSTERED INDEX IX_Fact_TimeKey 
    ON FactWarehouseOperations(TimeKey);

-- Non-clustered indexes for frequent joins
CREATE NONCLUSTERED INDEX IX_Fact_ProductKey 
    ON FactWarehouseOperations(ProductKey)
    INCLUDE (QuantityHandled, DwellTimeHours);

-- Filtered index for active trips
CREATE NONCLUSTERED INDEX IX_Fleet_Active 
    ON FactFleetTrips(VehicleKey, TimeKey)
    WHERE TripStatusID = 1; -- Active trips only
```

### Partitioning for Large Datasets

```sql
-- Partition fact tables by month
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601
);

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly
ALL TO ([PRIMARY]);

-- Apply to fact table
CREATE TABLE FactWarehouseOperations (
    TimeKey INT NOT NULL,
    -- other columns
) ON PS_Monthly(TimeKey);
```

### Automated Refresh Jobs

```sql
-- SQL Server Agent job (T-SQL step)
USE LogiFleetDB;

DECLARE @LastLoad DATETIME;
SELECT @LastLoad = MAX(LoadTimestamp) FROM dbo.ETLAuditLog;

BEGIN TRY
    EXEC dbo.usp_IncrementalLoadWarehouseOperations @LastLoad;
    EXEC dbo.usp_IncrementalLoadFleetTrips @LastLoad;
    
    INSERT INTO dbo.ETLAuditLog (LoadTimestamp, Status)
    VALUES (GETDATE(), 'SUCCESS');
END TRY
BEGIN CATCH
    INSERT INTO dbo.ETLAuditLog (LoadTimestamp, Status, ErrorMessage)
    VALUES (GETDATE(), 'FAILED', ERROR_MESSAGE());
    
    THROW;
END CATCH
```

## Troubleshooting

### Power BI Connection Issues

```powershell
# Test SQL Server connectivity
Test-NetConnection -ComputerName YOUR_SQL_SERVER -Port 1433

# Check SQL Server authentication
sqlcmd -S YOUR_SQL_SERVER -U ${SQL_USER} -P ${SQL_PASSWORD} -Q "SELECT @@VERSION"
```

### Slow Query Performance

```sql
-- Identify missing indexes
SELECT 
    migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) AS ImprovementMeasure,
    mid.statement AS TableName,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_group_stats migs
JOIN sys.dm_db_missing_index_groups mig ON migs.group_handle = mig.index_group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE mid.database_id = DB_ID('LogiFleetDB')
ORDER BY ImprovementMeasure DESC;
```

### Data Quality Checks

```sql
-- Validate fact table completeness
DECLARE @ExpectedRecords INT, @ActualRecords INT;

-- Expect one record per 15-minute interval per warehouse
SELECT @ExpectedRecords = 
    DATEDIFF(MINUTE, DATEADD(DAY, -1, GETDATE()), GETDATE()) / 15 
    * (SELECT COUNT(*) FROM DimGeography WHERE GeographyType = 'WAREHOUSE');

SELECT @ActualRecords = COUNT(*) 
FROM FactWarehouseOperations f
JOIN DimTime dt ON f.TimeKey = dt.TimeKey
WHERE dt.Date >= DATEADD(DAY, -1, GETDATE());

IF @ActualRecords < @ExpectedRecords * 0.95
    RAISERROR('Data completeness below 95%% threshold', 16, 1);
```

### Dashboard Performance Optimization

```dax
// Use variables to avoid recalculating expressions
TotalLogisticsCost = 
VAR WarehouseCosts = SUMX(FactWarehouseOperations, [LaborCost])
VAR FleetCosts = SUMX(FactFleetTrips, [FuelCost] + [MaintenanceCost])
VAR CrossDockCosts = SUMX(FactCrossDock, [HandlingCost])
RETURN
    WarehouseCosts + FleetCosts + CrossDockCosts

// Use SUMMARIZE for aggregations instead of nested CALCULATE
ProductPerformance = 
SUMMARIZE(
    FactWarehouseOperations,
    DimProductGravity[GravityZone],
    "TotalQuantity", SUM(FactWarehouseOperations[QuantityHandled]),
    "AvgDwell", AVERAGE(FactWarehouseOperations[DwellTimeHours])
)
```

## External API Integration

### Weather Correlation Analysis

```sql
-- Stored procedure to import weather data
CREATE PROCEDURE dbo.usp_ImportWeatherData
    @APIKey VARCHAR(100) = NULL
AS
BEGIN
    -- Use API key from environment or parameter
    DECLARE @WeatherAPIKey VARCHAR(100) = COALESCE(@APIKey, '${WEATHER_API_KEY}');
    
    -- Call external REST API (requires CLR integration or linked server)
    -- Store results in staging table
    INSERT INTO dbo.StagingWeatherData (LocationKey, Timestamp, Temperature, Precipitation, WindSpeed)
    -- API call implementation here
    
    -- Merge into dimension table
    MERGE DimWeather AS target
    USING dbo.StagingWeatherData AS source
    ON target.LocationKey = source.LocationKey AND target.Timestamp = source.Timestamp
    WHEN MATCHED THEN UPDATE SET /* update columns */
    WHEN NOT MATCHED THEN INSERT /* insert new rows */;
END
```

## Advanced Analytics Scenarios

### Temporal Elasticity Simulation

```sql
-- What-if analysis: 95% warehouse capacity vs. current
DECLARE @CurrentCapacity DECIMAL(5,2) = 0.80;
DECLARE @SimulatedCapacity DECIMAL(5,2) = 0.95;

WITH HistoricalPerformance AS (
    SELECT 
        CAST(DATEPART(HOUR, dt.FullDateTime) AS TINYINT) AS HourOfDay,
        AVG(wo.QuantityHandled) AS AvgQuantity,
        AVG(wo.CycleDurationMinutes) AS AvgCycleDuration,
        STDEV(wo.CycleDurationMinutes) AS StdDevCycleDuration
    FROM FactWarehouseOperations wo
    JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.Date >= DATEADD(DAY, -90, GETDATE())
    GROUP BY DATEPART(HOUR, dt.FullDateTime)
)
SELECT 
    HourOfDay,
    AvgQuantity,
    AvgCycleDuration AS CurrentCycleDuration,
    AvgCycleDuration * POWER((@SimulatedCapacity / @CurrentCapacity), 1.5) AS ProjectedCycleDuration,
    StdDevCycleDuration * (@SimulatedCapacity / @CurrentCapacity) AS ProjectedVariance
FROM HistoricalPerformance
ORDER BY HourOfDay;
```

This skill enables AI agents to help developers deploy, configure, and query the LogiFleet Pulse logistics analytics platform with SQL Server and Power BI.
