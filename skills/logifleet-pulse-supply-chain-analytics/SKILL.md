---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine with MS SQL Server data warehousing and Power BI dashboards for supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "create logistics data warehouse with Power BI"
  - "implement multi-fact star schema for fleet tracking"
  - "build warehouse operations dashboard"
  - "configure supply chain KPI monitoring"
  - "deploy LogiFleet Pulse SQL schema"
  - "integrate fleet telemetry with warehouse data"
  - "create cross-modal logistics analytics"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet management, and supply chain analytics into a single semantic data layer. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema that enables cross-functional KPI analysis, predictive bottleneck detection, and real-time operational monitoring.

**Core capabilities:**
- Multi-fact data warehouse (warehouse operations, fleet trips, cross-dock transfers)
- Time-phased dimensional modeling with 15-minute granularity
- Warehouse Gravity Zones™ for spatial optimization
- Adaptive Fleet Triage Engine for maintenance prioritization
- Real-time Power BI dashboards with role-based security
- Predictive analytics for bottleneck detection

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources (WMS, TMS, telemetry APIs)

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the main schema deployment script
:r .\sql\01_CreateDatabase.sql
:r .\sql\02_CreateDimensions.sql
:r .\sql\03_CreateFacts.sql
:r .\sql\04_CreateViews.sql
:r .\sql\05_CreateStoredProcedures.sql
:r .\sql\06_CreateIndexes.sql
```

3. **Configure data sources:**
```json
{
  "connectionStrings": {
    "sqlServer": "Server=${SQL_SERVER};Database=LogiFleetPulse;User Id=${SQL_USER};Password=${SQL_PASSWORD};",
    "wmsApi": "${WMS_API_ENDPOINT}",
    "fleetTelemetry": "${FLEET_TELEMETRY_ENDPOINT}"
  },
  "refreshInterval": "15minutes",
  "timezone": "UTC"
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
- Enter SQL Server connection details when prompted
- Configure data refresh schedule (minimum 15-minute intervals recommended)

## Key Database Components

### Dimension Tables

**DimTime** - Temporal dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeValue DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot VARCHAR(5), -- e.g., '09:15'
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayName VARCHAR(10),
    WeekOfYear TINYINT,
    MonthNumber TINYINT,
    MonthName VARCHAR(10),
    Quarter TINYINT,
    Year INT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
);

-- Index for time-series queries
CREATE NONCLUSTERED INDEX IX_DimTime_DateTime 
ON DimTime(DateTimeValue) INCLUDE (TimeSlot, HourOfDay, DayOfWeek);
```

**DimProductGravity** - Product classification with warehouse velocity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT,
    IsPerishable BIT,
    ShelfLife INT, -- days
    -- Gravity Zone Scoring
    PickFrequencyScore INT, -- 1-100
    ValueScore INT, -- 1-100
    FragilityScore INT, -- 1-100
    ReplenishmentLeadDays INT,
    GravityZoneRank INT, -- Composite score
    RecommendedZone VARCHAR(20), -- 'HIGH', 'MEDIUM', 'LOW'
    LastRecalculated DATETIME
);
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationType VARCHAR(20), -- 'WAREHOUSE', 'ROUTE_NODE', 'CUSTOMER'
    LocationName NVARCHAR(200),
    Address NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT
);
```

### Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(20), -- 'RECEIVE', 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    Quantity INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    ZoneLocation VARCHAR(50),
    AssignedGravityZone VARCHAR(20),
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    BatchID VARCHAR(50),
    AnomalyFlag BIT,
    AnomalyReason NVARCHAR(500)
);

-- Covering index for time-based aggregations
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (OperationType, Quantity, DwellTimeMinutes);
```

**FactFleetTrips** - Fleet telemetry and trip data:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimRoute(RouteKey),
    OriginKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceKM DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    AverageSpeedKMH DECIMAL(5,2),
    WeatherCondition VARCHAR(50),
    DelayMinutes INT,
    DelayReason NVARCHAR(500),
    MaintenanceAlertFlag BIT,
    PriorityScore INT -- For triage engine
);
```

## Stored Procedures for Data Loading

### Incremental Warehouse Operations Load
```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load new operations since last refresh
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        Quantity, DwellTimeMinutes, ProcessingTimeMinutes,
        ZoneLocation, AssignedGravityZone
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        ops.operation_type,
        ops.quantity,
        DATEDIFF(MINUTE, ops.start_time, ops.end_time) AS DwellTimeMinutes,
        ops.processing_minutes,
        ops.zone_location,
        p.RecommendedZone
    FROM ExternalWMS.dbo.Operations ops
    INNER JOIN DimTime t ON DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, 0, ops.operation_time) / 15) * 15, 0) = t.DateTimeValue
    INNER JOIN DimProductGravity p ON ops.sku = p.SKU
    INNER JOIN DimGeography g ON ops.warehouse_id = g.LocationID
    WHERE ops.operation_time > @LastLoadTime
        AND ops.is_processed = 0;
    
    -- Update processed flag
    UPDATE ExternalWMS.dbo.Operations
    SET is_processed = 1
    WHERE operation_time > @LastLoadTime;
END;
```

### Gravity Zone Recalculation
```sql
CREATE PROCEDURE sp_RecalculateGravityZones
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate composite gravity scores
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            -- Pick frequency (last 30 days)
            ISNULL(COUNT(CASE WHEN wo.OperationType = 'PICK' THEN 1 END), 0) AS PickCount,
            -- Average dwell time
            AVG(ISNULL(wo.DwellTimeMinutes, 0)) AS AvgDwellMinutes,
            -- Total value moved
            SUM(wo.Quantity * p.UnitWeight) AS TotalWeightMoved
        FROM DimProductGravity p
        LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        LEFT JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateTimeValue >= DATEADD(DAY, -30, GETDATE())
        GROUP BY p.ProductKey
    )
    UPDATE p
    SET 
        PickFrequencyScore = CASE 
            WHEN pm.PickCount >= 100 THEN 100
            WHEN pm.PickCount <= 0 THEN 1
            ELSE pm.PickCount
        END,
        GravityZoneRank = (
            (CASE WHEN pm.PickCount >= 100 THEN 100 ELSE pm.PickCount END) * 0.5 +
            p.ValueScore * 0.3 +
            p.FragilityScore * 0.2
        ),
        RecommendedZone = CASE 
            WHEN (pm.PickCount * 0.5 + p.ValueScore * 0.3 + p.FragilityScore * 0.2) >= 75 THEN 'HIGH'
            WHEN (pm.PickCount * 0.5 + p.ValueScore * 0.3 + p.FragilityScore * 0.2) >= 40 THEN 'MEDIUM'
            ELSE 'LOW'
        END,
        LastRecalculated = GETDATE()
    FROM DimProductGravity p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
```

## Key Analytical Views

### Cross-Fact KPI View
```sql
CREATE VIEW vw_CrossFactKPIs AS
SELECT 
    t.DateKey,
    t.Year,
    t.MonthName,
    g.Country,
    g.LocationName,
    -- Warehouse metrics
    COUNT(DISTINCT CASE WHEN wo.OperationType = 'PICK' THEN wo.OperationKey END) AS PickOperations,
    AVG(CASE WHEN wo.OperationType = 'PICK' THEN wo.ProcessingTimeMinutes END) AS AvgPickTime,
    SUM(CASE WHEN wo.DwellTimeMinutes > 72*60 THEN 1 ELSE 0 END) AS LongDwellCount,
    -- Fleet metrics
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    SUM(ft.FuelConsumedLiters) AS TotalFuelConsumed,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    SUM(ft.DelayMinutes) AS TotalDelayMinutes,
    -- Cross-fact correlation
    CASE 
        WHEN AVG(wo.DwellTimeMinutes) > 120 AND AVG(ft.IdleTimeMinutes) > 30 
        THEN 'HIGH_FRICTION'
        ELSE 'NORMAL'
    END AS OperationalFrictionLevel
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
LEFT JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
GROUP BY t.DateKey, t.Year, t.MonthName, g.Country, g.LocationName;
```

### Predictive Bottleneck Detection
```sql
CREATE VIEW vw_PredictiveBottlenecks AS
WITH HourlyMetrics AS (
    SELECT 
        t.DateKey,
        t.HourOfDay,
        g.LocationName,
        COUNT(*) AS OperationCount,
        AVG(wo.ProcessingTimeMinutes) AS AvgProcessingTime,
        STDEV(wo.ProcessingTimeMinutes) AS StdDevProcessingTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
    WHERE t.DateTimeValue >= DATEADD(DAY, -7, GETDATE())
    GROUP BY t.DateKey, t.HourOfDay, g.LocationName
)
SELECT 
    LocationName,
    HourOfDay,
    AVG(OperationCount) AS AvgOperationsPerHour,
    AVG(AvgProcessingTime) AS AvgProcessingTime,
    AVG(StdDevProcessingTime) AS ProcessingVariability,
    CASE 
        WHEN AVG(OperationCount) > 100 AND AVG(AvgProcessingTime) > 15 THEN 'HIGH_RISK'
        WHEN AVG(OperationCount) > 75 AND AVG(AvgProcessingTime) > 12 THEN 'MODERATE_RISK'
        ELSE 'LOW_RISK'
    END AS BottleneckRiskLevel
FROM HourlyMetrics
GROUP BY LocationName, HourOfDay;
```

## Power BI DAX Measures

### Fleet Efficiency Score
```dax
Fleet Efficiency Score = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKM])
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedLiters])
VAR IdlePercentage = DIVIDE(SUM(FactFleetTrips[IdleTimeMinutes]), SUM(FactFleetTrips[DurationMinutes]), 0)
VAR FuelEfficiency = DIVIDE(TotalDistance, TotalFuel, 0)
VAR BaseScore = FuelEfficiency * 10
VAR IdlePenalty = IdlePercentage * 100
RETURN
    MAX(0, MIN(100, BaseScore - IdlePenalty))
```

### Warehouse Velocity Index
```dax
Warehouse Velocity Index = 
VAR PickOperations = CALCULATE(COUNT(FactWarehouseOperations[OperationKey]), FactWarehouseOperations[OperationType] = "PICK")
VAR AvgPickTime = CALCULATE(AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes]), FactWarehouseOperations[OperationType] = "PICK")
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR VelocityScore = DIVIDE(PickOperations, AvgPickTime, 0) * 100
VAR DwellPenalty = MIN(50, AvgDwellTime / 60)
RETURN
    MAX(0, VelocityScore - DwellPenalty)
```

### Cross-Modal Impact Metric
```dax
Cross-Modal Impact = 
VAR WarehouseDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR FleetIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR CorrelationFactor = 
    IF(
        WarehouseDwell > 120 && FleetIdle > 30,
        1.5,
        IF(WarehouseDwell > 60 && FleetIdle > 20, 1.2, 1.0)
    )
RETURN
    (WarehouseDwell + FleetIdle) * CorrelationFactor
```

## Automated Alerting System

### Configure Alert Thresholds
```sql
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(100) NOT NULL,
    MetricType VARCHAR(50), -- 'DWELL_TIME', 'IDLE_TIME', 'FUEL_EFFICIENCY'
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    Severity VARCHAR(20), -- 'CRITICAL', 'WARNING', 'INFO'
    NotificationChannels VARCHAR(200), -- 'EMAIL,SMS,TEAMS'
    IsActive BIT DEFAULT 1
);

-- Example alert configuration
INSERT INTO AlertConfiguration (AlertName, MetricType, ThresholdValue, ComparisonOperator, Severity, NotificationChannels)
VALUES 
    ('Long Dwell Time Alert', 'DWELL_TIME', 72, '>', 'WARNING', 'EMAIL,TEAMS'),
    ('High Fleet Idle Alert', 'IDLE_TIME', 30, '>', 'CRITICAL', 'EMAIL,SMS,TEAMS'),
    ('Low Fuel Efficiency', 'FUEL_EFFICIENCY', 8.0, '<', 'WARNING', 'EMAIL');
```

### Alert Execution Procedure
```sql
CREATE PROCEDURE sp_ExecuteAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertResults TABLE (
        AlertName VARCHAR(100),
        MetricValue DECIMAL(10,2),
        Threshold DECIMAL(10,2),
        Severity VARCHAR(20),
        AffectedEntity VARCHAR(200)
    );
    
    -- Check dwell time alerts
    INSERT INTO @AlertResults
    SELECT 
        ac.AlertName,
        AVG(wo.DwellTimeMinutes) / 60 AS AvgDwellHours,
        ac.ThresholdValue,
        ac.Severity,
        g.LocationName
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    CROSS JOIN AlertConfiguration ac
    WHERE ac.MetricType = 'DWELL_TIME'
        AND ac.IsActive = 1
        AND t.DateTimeValue >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY ac.AlertName, ac.ThresholdValue, ac.Severity, g.LocationName
    HAVING AVG(wo.DwellTimeMinutes) / 60 > ac.ThresholdValue;
    
    -- Check fleet idle alerts
    INSERT INTO @AlertResults
    SELECT 
        ac.AlertName,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        ac.ThresholdValue,
        ac.Severity,
        v.VehicleID
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    CROSS JOIN AlertConfiguration ac
    WHERE ac.MetricType = 'IDLE_TIME'
        AND ac.IsActive = 1
        AND t.DateTimeValue >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY ac.AlertName, ac.ThresholdValue, ac.Severity, v.VehicleID
    HAVING AVG(ft.IdleTimeMinutes) > ac.ThresholdValue;
    
    -- Return alert summary
    SELECT * FROM @AlertResults;
    
    -- Log alerts to history table
    INSERT INTO AlertHistory (AlertName, MetricValue, Threshold, Severity, AffectedEntity, AlertTime)
    SELECT AlertName, MetricValue, Threshold, Severity, AffectedEntity, GETDATE()
    FROM @AlertResults;
END;
```

## Common Patterns

### Time-Based Aggregation with Fiscal Calendar
```sql
-- Monthly warehouse performance by fiscal period
SELECT 
    t.FiscalPeriod,
    t.Year,
    t.MonthName,
    g.LocationName,
    COUNT(DISTINCT wo.ProductKey) AS UniqueProducts,
    SUM(wo.Quantity) AS TotalUnitsProcessed,
    AVG(wo.ProcessingTimeMinutes) AS AvgProcessingTime,
    SUM(CASE WHEN wo.AnomalyFlag = 1 THEN 1 ELSE 0 END) AS AnomalyCount
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
WHERE t.Year >= 2025
GROUP BY t.FiscalPeriod, t.Year, t.MonthName, g.LocationName
ORDER BY t.Year DESC, t.FiscalPeriod DESC;
```

### Gravity Zone Migration Analysis
```sql
-- Track products that changed gravity zones
WITH ZoneHistory AS (
    SELECT 
        p.SKU,
        p.ProductName,
        p.RecommendedZone AS CurrentZone,
        LAG(p.RecommendedZone) OVER (PARTITION BY p.SKU ORDER BY p.LastRecalculated) AS PreviousZone,
        p.LastRecalculated
    FROM DimProductGravity p
)
SELECT 
    SKU,
    ProductName,
    PreviousZone,
    CurrentZone,
    LastRecalculated,
    CASE 
        WHEN CurrentZone = 'HIGH' AND PreviousZone IN ('MEDIUM', 'LOW') THEN 'UPGRADED'
        WHEN CurrentZone = 'LOW' AND PreviousZone IN ('MEDIUM', 'HIGH') THEN 'DOWNGRADED'
        ELSE 'STABLE'
    END AS MigrationDirection
FROM ZoneHistory
WHERE PreviousZone IS NOT NULL 
    AND PreviousZone <> CurrentZone
ORDER BY LastRecalculated DESC;
```

### Fleet Route Optimization Analysis
```sql
-- Identify routes with consistently high delays
SELECT 
    r.RouteName,
    r.PlannedDistanceKM,
    COUNT(*) AS TripCount,
    AVG(ft.DelayMinutes) AS AvgDelay,
    AVG(ft.FuelConsumedLiters) AS AvgFuel,
    AVG(ft.IdleTimeMinutes) AS AvgIdle,
    -- Calculate efficiency vs. baseline
    AVG(ft.DistanceKM / NULLIF(ft.FuelConsumedLiters, 0)) AS ActualKMPerLiter,
    r.BaselineKMPerLiter,
    CASE 
        WHEN AVG(ft.DelayMinutes) > 30 THEN 'NEEDS_OPTIMIZATION'
        WHEN AVG(ft.IdleTimeMinutes) > 25 THEN 'REVIEW_STOPS'
        ELSE 'ACCEPTABLE'
    END AS OptimizationRecommendation
FROM FactFleetTrips ft
INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.DateTimeValue >= DATEADD(DAY, -30, GETDATE())
GROUP BY r.RouteName, r.PlannedDistanceKM, r.BaselineKMPerLiter
HAVING COUNT(*) >= 10
ORDER BY AvgDelay DESC;
```

## Troubleshooting

### Performance Issues

**Slow dashboard refresh:**
```sql
-- Check for missing indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.database_id = DB_ID('LogiFleetPulse')
ORDER BY s.user_seeks + s.user_scans + s.user_lookups DESC;

-- Add columnstore index for large fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOps_Columnstore
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, Quantity, DwellTimeMinutes);
```

**Partition large fact tables by date:**
```sql
-- Create partition function for monthly partitioning
CREATE PARTITION FUNCTION PF_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (20250101, 20250201, 20250301, 20250401, 20250501);

CREATE PARTITION SCHEME PS_MonthlyPartition
AS PARTITION PF_MonthlyPartition
ALL TO ([PRIMARY]);

-- Rebuild fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey BIGINT IDENTITY(1,1),
    TimeKey INT,
    DateKey INT, -- Partition key
    -- ... other columns
) ON PS_MonthlyPartition(DateKey);
```

### Data Quality Issues

**Detect anomalies in dwell time:**
```sql
-- Flag statistical outliers
WITH DwellStats AS (
    SELECT 
        AVG(DwellTimeMinutes) AS AvgDwell,
        STDEV(DwellTimeMinutes) AS StdDwell
    FROM FactWarehouseOperations
    WHERE DwellTimeMinutes > 0
)
UPDATE wo
SET 
    AnomalyFlag = 1,
    AnomalyReason = 'Dwell time exceeds 3 standard deviations from mean'
FROM FactWarehouseOperations wo
CROSS JOIN DwellStats ds
WHERE wo.DwellTimeMinutes > (ds.AvgDwell + (3 * ds.StdDwell))
    AND wo.AnomalyFlag = 0;
```

### Connection Issues

**Test external data source connectivity:**
```sql
-- Verify linked server connection
EXEC sp_testlinkedserver 'ExternalWMS';

-- Check external table permissions
SELECT * 
FROM OPENQUERY(ExternalWMS, 'SELECT COUNT(*) FROM Operations WHERE operation_time > DATEADD(HOUR, -1, GETDATE())');
```

### Power BI Optimization

**Reduce model size with aggregation tables:**
```sql
-- Create daily aggregation table
CREATE TABLE FactWarehouseOperations_Daily (
    DateKey INT,
    ProductKey INT,
    WarehouseKey INT,
    TotalQuantity INT,
    AvgDwellTimeMinutes DECIMAL(10,2),
    AvgProcessingTimeMinutes DECIMAL(10,2),
    OperationCount INT,
    PRIMARY KEY (DateKey, ProductKey, WarehouseKey)
);

-- Populate aggregation table
INSERT INTO FactWarehouseOperations_Daily
SELECT 
    t.DateKey,
    wo.ProductKey,
    wo.WarehouseKey,
    SUM(wo.Quantity) AS TotalQuantity,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    AVG(wo.ProcessingTimeMinutes) AS AvgProcessingTimeMinutes,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
GROUP BY t.DateKey, wo.ProductKey, wo.WarehouseKey;
```

## Best Practices

1. **Always use parameterized queries** for data loading to prevent SQL injection
2. **Implement incremental refresh** in Power BI for fact tables over 1M rows
3. **Schedule gravity zone recalculation** weekly during off-peak hours
4. **Monitor query execution plans** for views used in dashboards
5. **Use row-level security** in Power BI to restrict data access by user role
6. **Archive historical data** beyond 2 years to separate tables
7. **Document custom DAX measures** with business context comments
8. **Test alert thresholds** in non-production environment before deployment

## Environment Variables Reference

Required environment variables for configuration:

- `SQL_SERVER` - SQL Server instance name or IP
- `SQL_USER` - Database username
- `SQL_PASSWORD` - Database password
- `WMS_API_ENDPOINT` - Warehouse Management System API URL
- `FLEET_TELEMETRY_ENDPOINT` - Fleet tracking API URL
- `WEATHER_API_KEY` - External weather service API key
- `SMTP_SERVER` - Email server for alerts
- `SMTP_USER` - Email authentication username
- `SMTP_PASSWORD` - Email authentication password
