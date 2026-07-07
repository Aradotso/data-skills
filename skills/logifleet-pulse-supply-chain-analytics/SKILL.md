---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI analytics engine for logistics, fleet management, and supply chain intelligence
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "configure logistics data warehouse with sql server"
  - "implement power bi fleet management dashboard"
  - "create supply chain KPI tracking system"
  - "build warehouse operations analytics"
  - "deploy logicore analytics for logistics"
  - "integrate fleet telemetry with warehouse data"
  - "setup multi-fact star schema for supply chain"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence platform that combines:
- **MS SQL Server data warehouse** with multi-fact star schema architecture
- **Power BI dashboards** for real-time logistics visualization
- **Warehouse operations tracking** (receiving, putaway, picking, packing, shipping)
- **Fleet telemetry integration** (GPS, fuel consumption, maintenance, routes)
- **Cross-fact KPI harmonization** linking inventory, fleet, and supplier data
- **Predictive analytics** for bottleneck detection and resource optimization

The platform provides a unified semantic layer that connects warehouse micro-operations, fleet performance, inventory aging, and external signals (weather, traffic) into actionable insights.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (recommended: 2022)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Database permissions: CREATE TABLE, CREATE PROCEDURE, CREATE VIEW

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS or Azure Data Studio
-- Execute the schema deployment script
:r deploy_schema.sql

-- Verify table creation
SELECT TABLE_NAME 
FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_TYPE = 'BASE TABLE'
AND TABLE_NAME LIKE 'Fact%' OR TABLE_NAME LIKE 'Dim%';
```

### Step 3: Configure Data Connections

Create a configuration file (based on `config_sample.json`):

```json
{
  "connections": {
    "sql_server": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "username": "${SQL_USERNAME}",
      "password": "${SQL_PASSWORD}",
      "connection_timeout": 30
    },
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    }
  },
  "refresh_intervals": {
    "warehouse_operations": "00:15:00",
    "fleet_trips": "00:05:00",
    "cross_dock": "00:30:00"
  }
}
```

### Step 4: Import Power BI Template

```powershell
# Open Power BI Desktop
# File → Open → Select LogiFleet_Pulse_Master.pbit
# Enter connection parameters when prompted:
# - SQL Server: ${SQL_SERVER_HOST}
# - Database: LogiFleetPulse
```

## Core Data Model Architecture

### Fact Tables

#### FactWarehouseOperations

Tracks all warehouse activities with time-phased granularity.

```sql
-- Insert warehouse operation event
INSERT INTO FactWarehouseOperations (
    OperationID,
    TimeKey,
    WarehouseKey,
    ProductKey,
    OperationType,
    QuantityProcessed,
    DwellTimeMinutes,
    PickRatePerHour,
    ZoneGravityScore,
    OperatorID,
    BatchNumber
)
VALUES (
    NEWID(),
    (SELECT TimeKey FROM DimTime WHERE TimeStamp = DATEADD(MINUTE, DATEDIFF(MINUTE, 0, GETDATE())/15*15, 0)),
    @WarehouseKey,
    @ProductKey,
    'PICKING',
    @Quantity,
    @DwellTime,
    @PickRate,
    @GravityScore,
    @OperatorID,
    @BatchNumber
);

-- Query: Operations with high dwell time in high-gravity zones
SELECT 
    wo.OperationID,
    dt.TimeStamp,
    p.ProductName,
    wo.DwellTimeMinutes,
    wo.ZoneGravityScore,
    w.WarehouseName
FROM FactWarehouseOperations wo
JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
JOIN DimProduct p ON wo.ProductKey = p.ProductKey
JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
WHERE wo.DwellTimeMinutes > 72 * 60
AND wo.ZoneGravityScore > 0.8
AND dt.TimeStamp >= DATEADD(DAY, -30, GETDATE());
```

#### FactFleetTrips

Captures fleet movement, fuel consumption, and route performance.

```sql
-- Insert fleet trip segment
INSERT INTO FactFleetTrips (
    TripID,
    VehicleKey,
    DriverKey,
    RouteKey,
    DepartureTimeKey,
    ArrivalTimeKey,
    DistanceKM,
    FuelConsumedLiters,
    IdleTimeMinutes,
    LoadWeightKG,
    WeatherConditionKey,
    MaintenanceStatusKey
)
VALUES (
    NEWID(),
    @VehicleKey,
    @DriverKey,
    @RouteKey,
    @DepartureTimeKey,
    @ArrivalTimeKey,
    @Distance,
    @FuelConsumed,
    @IdleTime,
    @LoadWeight,
    @WeatherKey,
    @MaintenanceKey
);

-- Query: Fuel efficiency by route with weather correlation
SELECT 
    r.RouteName,
    AVG(ft.FuelConsumedLiters / ft.DistanceKM) AS AvgFuelPerKM,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    wc.ConditionDescription,
    COUNT(*) AS TripCount
FROM FactFleetTrips ft
JOIN DimRoute r ON ft.RouteKey = r.RouteKey
JOIN DimWeatherCondition wc ON ft.WeatherConditionKey = wc.WeatherConditionKey
WHERE ft.DepartureTimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(DAY, -90, GETDATE()))
GROUP BY r.RouteName, wc.ConditionDescription
HAVING COUNT(*) > 10
ORDER BY AvgFuelPerKM DESC;
```

#### FactCrossDock

Handles transfers between inbound and outbound without long-term storage.

```sql
-- Cross-dock transfer event
INSERT INTO FactCrossDock (
    TransferID,
    InboundTimeKey,
    OutboundTimeKey,
    ProductKey,
    SupplierKey,
    DestinationKey,
    QuantityUnits,
    TransferDurationMinutes,
    QualityCheckStatus
)
VALUES (
    NEWID(),
    @InboundTimeKey,
    @OutboundTimeKey,
    @ProductKey,
    @SupplierKey,
    @DestinationKey,
    @Quantity,
    DATEDIFF(MINUTE, @InboundTime, @OutboundTime),
    @QualityStatus
);
```

### Dimension Tables

#### DimTime (Time-Phased Dimension)

```sql
-- Query time patterns by hour and day
SELECT 
    t.HourOfDay,
    t.DayOfWeek,
    AVG(wo.PickRatePerHour) AS AvgPickRate,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wo
JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.FiscalYear = 2026
GROUP BY t.HourOfDay, t.DayOfWeek
ORDER BY t.DayOfWeek, t.HourOfDay;
```

#### DimProductGravity (Warehouse Gravity Zones™)

```sql
-- Update product gravity score based on velocity and value
UPDATE DimProductGravity
SET 
    GravityScore = (
        (VelocityFactor * 0.4) + 
        (ValueFactor * 0.3) + 
        (FragilityFactor * 0.2) + 
        (ReplenishmentUrgency * 0.1)
    ),
    RecommendedZone = CASE 
        WHEN GravityScore > 0.8 THEN 'HIGH_GRAVITY'
        WHEN GravityScore > 0.5 THEN 'MEDIUM_GRAVITY'
        ELSE 'LOW_GRAVITY'
    END
WHERE LastRecalculation < DATEADD(DAY, -7, GETDATE());
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Load warehouse operations incrementally
EXEC sp_LoadWarehouseOperations 
    @LastLoadTimestamp = '2026-01-01 00:00:00',
    @BatchSize = 10000;

-- Load fleet trips with telemetry
EXEC sp_LoadFleetTrips 
    @StartDate = '2026-01-01',
    @EndDate = '2026-01-31';

-- Refresh product gravity scores
EXEC sp_RecalculateProductGravity 
    @LookbackDays = 90;
```

### Cross-Fact KPI Calculation

```sql
-- Calculate composite KPI: Inventory turnover vs fuel efficiency
CREATE PROCEDURE sp_CompositeKPI_InventoryFleetEfficiency
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SELECT 
        p.ProductCategory,
        SUM(wo.QuantityProcessed) / NULLIF(AVG(inv.OnHandQuantity), 0) AS TurnoverRate,
        AVG(ft.FuelConsumedLiters / ft.DistanceKM) AS AvgFuelEfficiency,
        -- Composite score: high turnover + low fuel consumption = optimal
        (SUM(wo.QuantityProcessed) / NULLIF(AVG(inv.OnHandQuantity), 0)) / 
        NULLIF(AVG(ft.FuelConsumedLiters / ft.DistanceKM), 0) AS CompositeEfficiencyScore
    FROM FactWarehouseOperations wo
    JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    JOIN FactInventorySnapshot inv ON p.ProductKey = inv.ProductKey
    JOIN FactFleetTrips ft ON wo.TimeKey = ft.DepartureTimeKey
    WHERE wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= @StartDate)
    AND wo.TimeKey <= (SELECT TimeKey FROM DimTime WHERE TimeStamp <= @EndDate)
    GROUP BY p.ProductCategory
    ORDER BY CompositeEfficiencyScore DESC;
END;
```

### Predictive Bottleneck Detection

```sql
-- Identify probable congestion points
CREATE PROCEDURE sp_PredictBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    WITH HistoricalPatterns AS (
        SELECT 
            w.WarehouseName,
            t.HourOfDay,
            t.DayOfWeek,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
            STDEV(wo.DwellTimeMinutes) AS StdDevDwellTime,
            COUNT(*) AS SampleCount
        FROM FactWarehouseOperations wo
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
        WHERE t.TimeStamp >= DATEADD(DAY, -90, GETDATE())
        GROUP BY w.WarehouseName, t.HourOfDay, t.DayOfWeek
        HAVING COUNT(*) > 30
    )
    SELECT 
        hp.WarehouseName,
        hp.HourOfDay,
        hp.DayOfWeek,
        hp.AvgDwellTime,
        hp.AvgDwellTime + (2 * hp.StdDevDwellTime) AS PredictedMaxDwellTime,
        CASE 
            WHEN hp.AvgDwellTime > 120 THEN 'HIGH_RISK'
            WHEN hp.AvgDwellTime > 60 THEN 'MEDIUM_RISK'
            ELSE 'LOW_RISK'
        END AS BottleneckRisk
    FROM HistoricalPatterns hp
    WHERE hp.AvgDwellTime > 60
    ORDER BY hp.AvgDwellTime DESC;
END;
```

## Power BI DAX Measures

### Fleet Idle Time Percentage

```dax
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + 
    SUMX(FactFleetTrips, FactFleetTrips[DistanceKM] / 60 * 60), // Approximate driving time
    0
) * 100
```

### Warehouse Gravity Zone Efficiency

```dax
Gravity Zone Efficiency = 
AVERAGEX(
    FactWarehouseOperations,
    DIVIDE(
        FactWarehouseOperations[PickRatePerHour],
        FactWarehouseOperations[DwellTimeMinutes] / 60,
        0
    ) * FactWarehouseOperations[ZoneGravityScore]
)
```

### Cross-Fact Delay Correlation

```dax
Weather-Delayed Shipments with High Dwell = 
CALCULATE(
    COUNT(FactFleetTrips[TripID]),
    FILTER(
        FactFleetTrips,
        RELATED(DimWeatherCondition[ImpactLevel]) = "HIGH"
    ),
    FILTER(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] > 4320 // 72 hours
    )
)
```

### Temporal Elasticity Simulation

```dax
Simulated Capacity Impact = 
VAR CurrentCapacity = SUM(FactWarehouseOperations[QuantityProcessed])
VAR TargetCapacityMultiplier = 1.2 // 20% increase
VAR HistoricalFleetUtilization = AVERAGE(FactFleetTrips[IdleTimeMinutes]) / 60

RETURN
    CurrentCapacity * TargetCapacityMultiplier * 
    (1 - (HistoricalFleetUtilization * 0.05)) // Penalty factor for idle time
```

## Configuration Patterns

### Scheduled Data Refresh

```sql
-- Create SQL Server Agent job for incremental refresh
USE msdb;
GO

EXEC sp_add_job 
    @job_name = N'LogiFleet_IncrementalRefresh',
    @enabled = 1,
    @description = N'Refresh warehouse and fleet data every 15 minutes';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalRefresh',
    @step_name = N'Load Operations',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_LoadWarehouseOperations @BatchSize = 5000;',
    @retry_attempts = 3,
    @retry_interval = 5;

EXEC sp_add_jobschedule
    @job_name = N'LogiFleet_IncrementalRefresh',
    @name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_add_jobserver 
    @job_name = N'LogiFleet_IncrementalRefresh';
```

### Row-Level Security (Power BI)

```dax
-- Create role: RegionalManager
[RegionFilter] = 
    VAR UserRegion = LOOKUPVALUE(
        DimUsers[Region],
        DimUsers[Email], USERPRINCIPALNAME()
    )
    RETURN
        DimGeography[Region] = UserRegion
```

### Alert Threshold Configuration

```sql
-- Configure automated alerts
INSERT INTO AlertThresholds (
    MetricName,
    ThresholdValue,
    ComparisonOperator,
    AlertPriority,
    NotificationChannels
)
VALUES 
    ('FleetIdleTimePercent', 15.0, '>', 'HIGH', 'EMAIL,TEAMS'),
    ('WarehouseDwellTimeHours', 72.0, '>', 'MEDIUM', 'EMAIL'),
    ('FuelEfficiencyDeviation', 20.0, '>', 'MEDIUM', 'TEAMS'),
    ('CrossDockTransferMinutes', 45.0, '>', 'LOW', 'EMAIL');

-- Stored procedure to check thresholds
CREATE PROCEDURE sp_CheckAlertThresholds
AS
BEGIN
    DECLARE @AlertLog TABLE (
        MetricName NVARCHAR(100),
        CurrentValue DECIMAL(18,2),
        ThresholdValue DECIMAL(18,2),
        AlertPriority NVARCHAR(20),
        Timestamp DATETIME
    );

    -- Check fleet idle time
    INSERT INTO @AlertLog
    SELECT 
        'FleetIdleTimePercent',
        (SUM(CAST(IdleTimeMinutes AS DECIMAL)) / SUM(CAST(DistanceKM AS DECIMAL) / 60 * 60)) * 100,
        at.ThresholdValue,
        at.AlertPriority,
        GETDATE()
    FROM FactFleetTrips
    CROSS JOIN AlertThresholds at
    WHERE at.MetricName = 'FleetIdleTimePercent'
    AND DepartureTimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(HOUR, -1, GETDATE()))
    HAVING (SUM(CAST(IdleTimeMinutes AS DECIMAL)) / SUM(CAST(DistanceKM AS DECIMAL) / 60 * 60)) * 100 > at.ThresholdValue;

    -- Send alerts
    SELECT * FROM @AlertLog;
END;
```

## Common Usage Patterns

### Pattern 1: Daily Operations Dashboard

```sql
-- Query for morning briefing report
SELECT 
    'Warehouse' AS Category,
    COUNT(DISTINCT wo.OperationID) AS EventCount,
    AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
    SUM(wo.QuantityProcessed) AS TotalUnitsProcessed
FROM FactWarehouseOperations wo
JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.TimeStamp >= CAST(GETDATE() AS DATE)

UNION ALL

SELECT 
    'Fleet' AS Category,
    COUNT(DISTINCT ft.TripID) AS EventCount,
    AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(ft.DistanceKM) AS TotalDistanceKM
FROM FactFleetTrips ft
JOIN DimTime t ON ft.DepartureTimeKey = t.TimeKey
WHERE t.TimeStamp >= CAST(GETDATE() AS DATE);
```

### Pattern 2: Supplier Performance Analysis

```sql
-- Analyze supplier reliability impact on warehouse operations
SELECT 
    s.SupplierName,
    s.LeadTimeVarianceDays,
    s.DefectPercentage,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwell,
    COUNT(DISTINCT wo.OperationID) AS OperationCount
FROM FactWarehouseOperations wo
JOIN DimProduct p ON wo.ProductKey = p.ProductKey
JOIN DimSupplierReliability s ON p.SupplierKey = s.SupplierKey
WHERE wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(DAY, -30, GETDATE()))
GROUP BY s.SupplierName, s.LeadTimeVarianceDays, s.DefectPercentage
HAVING COUNT(DISTINCT wo.OperationID) > 100
ORDER BY s.DefectPercentage DESC, AvgWarehouseDwell DESC;
```

### Pattern 3: Route Optimization Candidate Identification

```sql
-- Find routes with consistently high idle time for optimization
WITH RoutePerformance AS (
    SELECT 
        r.RouteKey,
        r.RouteName,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(ft.FuelConsumedLiters / ft.DistanceKM) AS AvgFuelEfficiency,
        COUNT(*) AS TripCount,
        STDEV(ft.IdleTimeMinutes) AS StdDevIdleTime
    FROM FactFleetTrips ft
    JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    WHERE ft.DepartureTimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(DAY, -90, GETDATE()))
    GROUP BY r.RouteKey, r.RouteName
    HAVING COUNT(*) > 50
)
SELECT 
    RouteName,
    AvgIdleTime,
    AvgFuelEfficiency,
    TripCount,
    CASE 
        WHEN AvgIdleTime > 30 AND AvgFuelEfficiency > 0.12 THEN 'HIGH_PRIORITY'
        WHEN AvgIdleTime > 20 OR AvgFuelEfficiency > 0.10 THEN 'MEDIUM_PRIORITY'
        ELSE 'LOW_PRIORITY'
    END AS OptimizationPriority
FROM RoutePerformance
WHERE AvgIdleTime > 15 OR AvgFuelEfficiency > 0.09
ORDER BY AvgIdleTime DESC, AvgFuelEfficiency DESC;
```

### Pattern 4: Seasonal Demand Forecasting

```sql
-- Extract seasonal patterns for predictive modeling
SELECT 
    t.FiscalQuarter,
    t.MonthName,
    p.ProductCategory,
    SUM(wo.QuantityProcessed) AS TotalProcessed,
    AVG(wo.PickRatePerHour) AS AvgPickRate,
    LAG(SUM(wo.QuantityProcessed), 1) OVER (
        PARTITION BY p.ProductCategory 
        ORDER BY t.FiscalYear, t.FiscalQuarter
    ) AS PreviousQuarterVolume,
    (SUM(wo.QuantityProcessed) - LAG(SUM(wo.QuantityProcessed), 1) OVER (
        PARTITION BY p.ProductCategory 
        ORDER BY t.FiscalYear, t.FiscalQuarter
    )) / NULLIF(LAG(SUM(wo.QuantityProcessed), 1) OVER (
        PARTITION BY p.ProductCategory 
        ORDER BY t.FiscalYear, t.FiscalQuarter
    ), 0) * 100 AS GrowthPercentage
FROM FactWarehouseOperations wo
JOIN DimTime t ON wo.TimeKey = t.TimeKey
JOIN DimProduct p ON wo.ProductKey = p.ProductKey
WHERE t.FiscalYear >= 2024
GROUP BY t.FiscalYear, t.FiscalQuarter, t.MonthName, p.ProductCategory
ORDER BY p.ProductCategory, t.FiscalYear, t.FiscalQuarter;
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom**: Queries joining multiple fact tables take >10 seconds

**Solution**: Create indexed views for common cross-fact joins

```sql
CREATE VIEW vw_WarehouseFleetCorrelation
WITH SCHEMABINDING
AS
SELECT 
    wo.ProductKey,
    wo.TimeKey,
    wo.WarehouseKey,
    SUM(wo.QuantityProcessed) AS TotalQuantity,
    AVG(wo.DwellTimeMinutes) AS AvgDwell,
    COUNT_BIG(*) AS RowCount
FROM dbo.FactWarehouseOperations wo
GROUP BY wo.ProductKey, wo.TimeKey, wo.WarehouseKey;

CREATE UNIQUE CLUSTERED INDEX IDX_WarehouseFleetCorrelation
ON vw_WarehouseFleetCorrelation (ProductKey, TimeKey, WarehouseKey);
```

### Issue: Power BI Refresh Timeout

**Symptom**: Power BI dataset refresh fails after 2+ hours

**Solution**: Implement incremental refresh policy

```powerquery
// In Power BI Query Editor
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    FilteredRows = Table.SelectRows(
        Source, 
        each [TimeStamp] >= RangeStart and [TimeStamp] < RangeEnd
    )
in
    FilteredRows
```

Configure incremental refresh in Power BI Desktop:
- Store data: 3 years
- Refresh data: Last 7 days
- Detect data changes: TimeStamp column

### Issue: Incorrect Gravity Score Calculations

**Symptom**: Products assigned to wrong warehouse zones

**Solution**: Validate input factors and recalibrate weights

```sql
-- Diagnostic query for gravity score outliers
SELECT 
    pg.ProductKey,
    p.ProductName,
    pg.GravityScore,
    pg.VelocityFactor,
    pg.ValueFactor,
    pg.FragilityFactor,
    pg.RecommendedZone,
    COUNT(wo.OperationID) AS ActualOperations,
    AVG(wo.PickRatePerHour) AS ActualPickRate
FROM DimProductGravity pg
JOIN DimProduct p ON pg.ProductKey = p.ProductKey
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
WHERE wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(DAY, -30, GETDATE()))
GROUP BY pg.ProductKey, p.ProductName, pg.GravityScore, pg.VelocityFactor, 
         pg.ValueFactor, pg.FragilityFactor, pg.RecommendedZone
HAVING (pg.GravityScore > 0.7 AND AVG(wo.PickRatePerHour) < 50)
    OR (pg.GravityScore < 0.3 AND AVG(wo.PickRatePerHour) > 100);

-- Recalibrate if needed
EXEC sp_RecalculateProductGravity @LookbackDays = 180;
```

### Issue: Missing Telemetry Data

**Symptom**: Fleet trips show NULL values for GPS or fuel data

**Solution**: Implement fallback and data quality checks

```sql
-- Create data quality monitoring view
CREATE VIEW vw_DataQualityMetrics
AS
SELECT 
    'FleetTrips' AS SourceTable,
    COUNT(*) AS TotalRecords,
    SUM(CASE WHEN FuelConsumedLiters IS NULL THEN 1 ELSE 0 END) AS MissingFuel,
    SUM(CASE WHEN DistanceKM IS NULL OR DistanceKM = 0 THEN 1 ELSE 0 END) AS MissingDistance,
    SUM(CASE WHEN IdleTimeMinutes IS NULL THEN 1 ELSE 0 END) AS MissingIdleTime,
    CAST(GETDATE() AS DATE) AS ReportDate
FROM FactFleetTrips
WHERE DepartureTimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(DAY, -1, GETDATE()));

-- Alert if missing data exceeds threshold
IF EXISTS (
    SELECT 1 FROM vw_DataQualityMetrics 
    WHERE CAST(MissingFuel AS FLOAT) / TotalRecords > 0.05
)
BEGIN
    RAISERROR('Data quality alert: >5%% missing fuel data in fleet trips', 16, 1);
END;
```

### Issue: Alert Fatigue

**Symptom**: Too many low-priority alerts being generated

**Solution**: Implement smart alert throttling

```sql
-- Modify alert procedure with time-based throttling
ALTER PROCEDURE sp_CheckAlertThresholds
AS
BEGIN
    -- Only send alerts if threshold breached for >30 minutes consecutively
    WITH AlertCandidates AS (
        SELECT 
            MetricName,
            CurrentValue,
            ThresholdValue,
            Timestamp,
            ROW_NUMBER() OVER (PARTITION BY MetricName ORDER BY Timestamp DESC) AS RecentRank
        FROM AlertHistory
        WHERE Timestamp >= DATEADD(MINUTE, -30, GETDATE())
        AND CurrentValue > ThresholdValue
    )
    SELECT MetricName, MAX(CurrentValue) AS MaxValue
    FROM AlertCandidates
    WHERE RecentRank <= 2 -- At least 2 consecutive breaches
    GROUP BY MetricName
    HAVING COUNT(*) >= 2;
END;
```

## Performance Optimization

### Partitioning Strategy

```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION FUNCTION PF_MonthlyOperations (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01',
    '2026-05-01', '2026-06-01', '2026-07-01', '2026-08-01',
    '2026-09-01', '2026-10-01', '2026-11-01', '2026-12-01'
);

CREATE PARTITION SCHEME PS_MonthlyOperations
AS PARTITION PF_MonthlyOperations
ALL TO ([PRIMARY]);

-- Create partitioned table
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationID UNIQUEIDENTIFIER PRIMARY KEY,
    TimeKey INT NOT NULL,
    -- ... other columns
    OperationTimestamp DATETIME NOT NULL
) ON PS_MonthlyOperations(OperationTimestamp);
```

### Columnstore Indexes for Analytics

```sql
-- Create columnstore index for aggregation-heavy queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips_Analytics
ON FactFleetTrips (
    DepartureTimeKey,
    VehicleKey,
    RouteKey,
    DistanceKM,
    FuelConsumedLiters,
    IdleTimeMinutes
);
```

This skill provides comprehensive coverage of the LogiFleet Pulse platform, enabling AI agents to help developers deploy, configure, query, and optimize supply chain analytics solutions using MS SQL Server and Power BI.
