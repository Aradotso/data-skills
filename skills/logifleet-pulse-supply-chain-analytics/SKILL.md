---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine combining MS SQL Server data warehousing with Power BI dashboards for real-time supply chain optimization
triggers:
  - set up logifleet pulse analytics dashboard
  - configure supply chain data warehouse schema
  - deploy logistics intelligence sql server database
  - create power bi fleet tracking dashboard
  - implement warehouse gravity zone optimization
  - build cross-fact supply chain kpi reports
  - analyze fleet telemetry with power bi
  - setup real-time logistics alerting system
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a semantic data layer. Built on MS SQL Server with Power BI visualization, it uses a multi-fact star schema to enable cross-domain KPI analysis and predictive bottleneck detection.

**Core Capabilities:**
- Multi-fact star schema data warehousing
- Real-time fleet and warehouse operations dashboards
- Predictive bottleneck detection and fleet triage
- Warehouse Gravity Zones™ spatial optimization
- Temporal elasticity modeling for scenario planning
- Cross-fact KPI harmonization (inventory × fleet × supplier metrics)

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Read/write access to source systems (WMS, TMS, ERP)

### 1. Deploy SQL Schema

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Deploy dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    TimeStamp DATETIME2 NOT NULL,
    FifteenMinuteBucket INT,
    HourOfDay INT,
    DayOfWeek INT,
    FiscalPeriod VARCHAR(20),
    IsWorkingHour BIT,
    INDEX IX_TimeStamp (TimeStamp)
)
GO

CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode VARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'RouteNode', 'Dock'
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    ParentGeographyKey INT,
    INDEX IX_LocationCode (LocationCode)
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite score: velocity + value + fragility
    VelocityRating VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2),
    OptimalZone VARCHAR(50),
    LastRecalculatedDate DATETIME2,
    INDEX IX_GravityScore (GravityScore DESC)
)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200),
    LeadTimeMean DECIMAL(8,2),
    LeadTimeVariance DECIMAL(8,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    ReliabilityRating VARCHAR(20),
    LastUpdatedDate DATETIME2
)
GO

-- Deploy fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    DurationMinutes DECIMAL(10,2),
    QuantityHandled INT,
    DwellTimeHours DECIMAL(10,2),
    ZoneType VARCHAR(50),
    OperatorID VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    INDEX IX_TimeGeo (TimeKey, GeographyKey),
    INDEX IX_Product (ProductKey)
)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    RouteCode VARCHAR(100),
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    LoadWeight DECIMAL(10,2),
    TirePressureAvg DECIMAL(5,2),
    EngineHealthScore DECIMAL(5,2),
    DelayMinutes DECIMAL(10,2),
    DelayReason VARCHAR(200),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_TimeOrigin (TimeKey, OriginGeographyKey),
    INDEX IX_Vehicle (VehicleID, TripStartTime)
)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    DockDwellMinutes DECIMAL(10,2),
    QuantityTransferred INT,
    TransferEfficiency DECIMAL(5,2),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey),
    INDEX IX_TimeDock (TimeKey, GeographyKey)
)
GO
```

### 2. Configure Data Source Connections

Create a configuration file `config.json`:

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SQL",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "type": "REST_API",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}",
      "refreshIntervalMinutes": 15
    },
    "telematics": {
      "type": "ODBC",
      "connectionString": "${TELEMATICS_ODBC_CONNECTION}",
      "refreshIntervalMinutes": 5
    },
    "weather": {
      "type": "REST_API",
      "endpoint": "https://api.weatherapi.com/v1/current.json",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "alerting": {
    "smtpServer": "${SMTP_SERVER}",
    "fromEmail": "${ALERT_FROM_EMAIL}",
    "recipients": ["${ALERT_RECIPIENT_1}", "${ALERT_RECIPIENT_2}"]
  }
}
```

### 3. Create ETL Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, DurationMinutes,
        QuantityHandled, DwellTimeHours, ZoneType, OperatorID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        src.OperationType,
        src.StartTime,
        src.EndTime,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime),
        src.Quantity,
        src.DwellHours,
        src.Zone,
        src.OperatorID
    FROM ExternalWMSData src
    INNER JOIN DimTime t ON CAST(src.StartTime AS DATE) = CAST(t.TimeStamp AS DATE)
        AND DATEPART(HOUR, src.StartTime) = t.HourOfDay
    INNER JOIN DimGeography g ON src.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    WHERE src.LastModified > @LastLoadTime
    
    -- Update last load timestamp
    UPDATE ETL_ControlTable 
    SET LastLoadTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations'
END
GO

-- Calculate gravity scores
CREATE PROCEDURE sp_RecalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        (VelocityWeight * 
            CASE VelocityRating 
                WHEN 'Fast' THEN 10
                WHEN 'Medium' THEN 5
                WHEN 'Slow' THEN 1
            END) +
        (ValueWeight *
            CASE ValueTier
                WHEN 'High' THEN 10
                WHEN 'Medium' THEN 5
                WHEN 'Low' THEN 1
            END) +
        (FragilityWeight * FragilityIndex * 10)
    ) / (VelocityWeight + ValueWeight + FragilityWeight),
    OptimalZone = CASE
        WHEN GravityScore > 7 THEN 'High-Gravity-Zone-A'
        WHEN GravityScore > 4 THEN 'Medium-Gravity-Zone-B'
        ELSE 'Low-Gravity-Zone-C'
    END,
    LastRecalculatedDate = GETDATE()
    FROM DimProductGravity
    CROSS APPLY (SELECT 
        VelocityWeight = 0.5,
        ValueWeight = 0.3,
        FragilityWeight = 0.2
    ) weights
END
GO
```

### 4. Set Up Automated Alerts

```sql
CREATE PROCEDURE sp_CheckCriticalAlerts
AS
BEGIN
    -- Fleet idle time alert
    DECLARE @AlertThresholdIdlePercent DECIMAL(5,2) = 15.0
    
    SELECT 
        VehicleID,
        AVG((IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, TripStartTime, TripEndTime), 0)) * 100) as IdlePercent,
        COUNT(*) as TripCount
    INTO #IdleAlerts
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
    GROUP BY VehicleID
    HAVING AVG((IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, TripStartTime, TripEndTime), 0)) * 100) > @AlertThresholdIdlePercent
    
    IF EXISTS (SELECT 1 FROM #IdleAlerts)
    BEGIN
        -- Send email alert (integrate with SQL Server Database Mail)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_RECIPIENT_1}',
            @subject = 'ALERT: High Fleet Idle Time Detected',
            @body = 'Critical idle time threshold exceeded. Check attached report.',
            @query = 'SELECT * FROM #IdleAlerts',
            @attach_query_result_as_file = 1
    END
    
    -- Warehouse dwell time alert
    SELECT 
        p.ProductName,
        g.LocationName,
        AVG(DwellTimeHours) as AvgDwellHours,
        MAX(DwellTimeHours) as MaxDwellHours
    FROM FactWarehouseOperations f
    INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON f.GeographyKey = g.GeographyKey
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last week
        AND p.VelocityRating = 'Fast'
        AND DwellTimeHours > 72
    GROUP BY p.ProductName, g.LocationName
    
    DROP TABLE #IdleAlerts
END
GO
```

## Power BI Configuration

### 1. Connect to Data Source

Open `LogiFleet_Pulse_Master.pbit` and configure connection:

```m
// Power Query M formula for SQL connection
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse",
        [
            Query = null,
            CommandTimeout = #duration(0, 0, 30, 0),
            ConnectionTimeout = #duration(0, 0, 5, 0)
        ]
    )
in
    Source
```

### 2. Key DAX Measures

```dax
// Cross-fact KPI: Dwell Time vs Fleet Idle Cost
TotalDwellCost = 
VAR AvgDwellHours = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR WarehouseCostPerHour = 5.50 // USD
VAR TotalWarehouseCost = AvgDwellHours * WarehouseCostPerHour * 
    COUNTROWS(FactWarehouseOperations)

VAR AvgIdleMinutes = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR IdleCostPerMinute = 0.85 // USD
VAR TotalIdleCost = AvgIdleMinutes * IdleCostPerMinute * 
    COUNTROWS(FactFleetTrips)

RETURN TotalWarehouseCost + TotalIdleCost

// Predictive Bottleneck Index
BottleneckRiskScore = 
VAR DwellFactor = 
    DIVIDE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        48,
        0
    ) * 0.4

VAR FleetDelayFactor = 
    DIVIDE(
        SUM(FactFleetTrips[DelayMinutes]),
        SUM(FactFleetTrips[DistanceKm]) * 2,
        0
    ) * 0.3

VAR SupplierReliabilityFactor = 
    (1 - AVERAGE(DimSupplierReliability[ComplianceScore]) / 100) * 0.3

RETURN (DwellFactor + FleetDelayFactor + SupplierReliabilityFactor) * 100

// Fleet Maintenance Priority Score
MaintenancePriority = 
VAR EngineHealth = AVERAGE(FactFleetTrips[EngineHealthScore])
VAR TirePressure = AVERAGE(FactFleetTrips[TirePressureAvg])
VAR LoadIntensity = DIVIDE(
    SUM(FactFleetTrips[LoadWeight]),
    COUNT(FactFleetTrips[TripKey]),
    0
)
VAR RevenueImpact = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[LoadWeight] * 
        RELATED(DimProductGravity[GravityScore]) * 
        10 // Revenue factor
    )

RETURN 
    (10 - EngineHealth) * 0.3 +
    (35 - TirePressure) * 0.2 +
    LoadIntensity * 0.2 +
    (RevenueImpact / 1000) * 0.3

// Warehouse Gravity Zone Efficiency
GravityZoneEfficiency = 
DIVIDE(
    SUMX(
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[ZoneType] = 
                RELATED(DimProductGravity[OptimalZone])
        ),
        FactWarehouseOperations[QuantityHandled]
    ),
    SUM(FactWarehouseOperations[QuantityHandled]),
    0
) * 100
```

### 3. Create Dynamic Visuals

Key dashboard components:

1. **Fleet Heatmap**: Map visual showing vehicle locations colored by idle time percentage
2. **Gravity Zone Matrix**: Matrix showing SKU distribution vs. optimal zone assignment
3. **Bottleneck Timeline**: Line chart with predictive bottleneck score over time
4. **Cross-Fact Decomposition Tree**: Explore dwell time by product → location → supplier reliability
5. **Maintenance Queue Table**: Ranked list of vehicles by maintenance priority score

## Common Usage Patterns

### Pattern 1: Identify Misallocated Inventory

```sql
-- Find high-gravity products in low-gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone,
    g.LocationName as CurrentZone,
    AVG(f.DwellTimeHours) as AvgDwellTime,
    SUM(f.QuantityHandled) as TotalVolume
FROM FactWarehouseOperations f
INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON f.GeographyKey = g.GeographyKey
WHERE p.GravityScore > 7
    AND g.LocationName LIKE '%Low-Gravity%'
    AND f.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZone, g.LocationName
ORDER BY TotalVolume DESC
```

### Pattern 2: Correlate Fleet Delays with Weather

```sql
-- Join fleet delays with external weather data
SELECT 
    f.TripKey,
    f.VehicleID,
    f.DelayMinutes,
    f.DelayReason,
    og.LocationName as Origin,
    dg.LocationName as Destination,
    w.Temperature,
    w.Precipitation,
    w.WindSpeed
FROM FactFleetTrips f
INNER JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
INNER JOIN DimGeography dg ON f.DestinationGeographyKey = dg.GeographyKey
LEFT JOIN ExternalWeatherData w ON 
    CAST(f.TripStartTime AS DATE) = w.WeatherDate
    AND og.LocationCode = w.LocationCode
WHERE f.DelayMinutes > 30
    AND f.TimeKey >= (SELECT MAX(TimeKey) - 168 FROM DimTime)
ORDER BY f.DelayMinutes DESC
```

### Pattern 3: Simulate Zone Reallocation Impact

```sql
-- Temporal elasticity model: What-if scenario
WITH SimulatedReallocation AS (
    SELECT 
        f.*,
        p.OptimalZone as RecommendedZone,
        CASE 
            WHEN f.ZoneType = p.OptimalZone THEN f.DurationMinutes
            ELSE f.DurationMinutes * 0.75 -- Assume 25% efficiency gain
        END as ProjectedDuration
    FROM FactWarehouseOperations f
    INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
    WHERE f.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
)
SELECT 
    'Current State' as Scenario,
    AVG(DurationMinutes) as AvgOperationMinutes,
    SUM(QuantityHandled) as TotalQuantity
FROM SimulatedReallocation
UNION ALL
SELECT 
    'Optimized Zones' as Scenario,
    AVG(ProjectedDuration) as AvgOperationMinutes,
    SUM(QuantityHandled) as TotalQuantity
FROM SimulatedReallocation
```

### Pattern 4: Fleet Triage Queue Generation

```sql
-- Ranked maintenance priority list
WITH VehicleHealth AS (
    SELECT 
        VehicleID,
        AVG(EngineHealthScore) as AvgEngineHealth,
        AVG(TirePressureAvg) as AvgTirePressure,
        AVG(LoadWeight) as AvgLoad,
        SUM(LoadWeight * 
            (SELECT AVG(GravityScore) FROM DimProductGravity)
        ) as RevenueImpact
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 336 FROM DimTime) -- Last 3.5 days
    GROUP BY VehicleID
)
SELECT 
    VehicleID,
    AvgEngineHealth,
    AvgTirePressure,
    AvgLoad,
    RevenueImpact,
    (
        (10 - AvgEngineHealth) * 0.3 +
        (35 - AvgTirePressure) * 0.2 +
        (AvgLoad / 1000) * 0.2 +
        (RevenueImpact / 100000) * 0.3
    ) as PriorityScore
FROM VehicleHealth
WHERE AvgEngineHealth < 8 OR AvgTirePressure < 32
ORDER BY PriorityScore DESC
```

## Row-Level Security Configuration

```sql
-- Create role-based security
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY,
    RoleName VARCHAR(50),
    AccessLevel VARCHAR(20) -- 'Executive', 'Supervisor', 'Operator'
)

CREATE TABLE UserRoles (
    UserEmail VARCHAR(200) PRIMARY KEY,
    RoleID INT FOREIGN KEY REFERENCES SecurityRoles(RoleID),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey)
)

-- RLS function for geography-based filtering
CREATE FUNCTION fn_SecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 as AccessPermitted
    FROM dbo.UserRoles ur
    INNER JOIN dbo.SecurityRoles sr ON ur.RoleID = sr.RoleID
    WHERE ur.UserEmail = USER_NAME()
        AND (
            sr.AccessLevel = 'Executive' -- Full access
            OR ur.GeographyKey = @GeographyKey -- Location-specific
        )
)
GO

-- Apply security policy
CREATE SECURITY POLICY GeographySecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(GeographyKey)
    ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(OriginGeographyKey)
    ON dbo.FactFleetTrips
WITH (STATE = ON)
GO
```

## Troubleshooting

### Issue: Power BI Refresh Failures

**Symptom**: "Data source error" or timeout during scheduled refresh

**Solution**:
```sql
-- Check for long-running queries
SELECT 
    session_id,
    start_time,
    status,
    command,
    wait_type,
    wait_time,
    blocking_session_id,
    SUBSTRING(text, 1, 200) as query_text
FROM sys.dm_exec_requests
CROSS APPLY sys.dm_exec_sql_text(sql_handle)
WHERE database_id = DB_ID('LogiFleetPulse')
ORDER BY wait_time DESC

-- Optimize indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey)
INCLUDE (QuantityHandled, DwellTimeHours)

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeVehicle
ON FactFleetTrips(TimeKey, VehicleID)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters)
```

### Issue: Incorrect Gravity Scores

**Symptom**: Products showing wrong zone recommendations

**Solution**:
```sql
-- Verify dimension data quality
SELECT 
    p.SKU,
    p.VelocityRating,
    p.ValueTier,
    p.FragilityIndex,
    p.GravityScore,
    COUNT(f.OperationKey) as OperationCount,
    AVG(f.QuantityHandled) as AvgQuantity
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations f ON p.ProductKey = f.ProductKey
WHERE p.LastRecalculatedDate < DATEADD(DAY, -7, GETDATE())
GROUP BY p.SKU, p.VelocityRating, p.ValueTier, p.FragilityIndex, p.GravityScore
HAVING COUNT(f.OperationKey) > 100

-- Force recalculation
EXEC sp_RecalculateProductGravity
```

### Issue: Alert Fatigue

**Symptom**: Too many low-priority alerts

**Solution**:
```sql
-- Implement dynamic thresholds based on historical patterns
ALTER PROCEDURE sp_CheckCriticalAlerts
AS
BEGIN
    -- Calculate statistical threshold (mean + 2 standard deviations)
    DECLARE @IdleThreshold DECIMAL(5,2)
    
    SELECT @IdleThreshold = AVG(IdlePercent) + (2 * STDEV(IdlePercent))
    FROM (
        SELECT 
            AVG((IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, TripStartTime, TripEndTime), 0)) * 100) as IdlePercent
        FROM FactFleetTrips
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
        GROUP BY CAST(TripStartTime AS DATE)
    ) daily_stats
    
    -- Only alert on statistical anomalies
    SELECT VehicleID, IdlePercent
    FROM (
        SELECT 
            VehicleID,
            AVG((IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, TripStartTime, TripEndTime), 0)) * 100) as IdlePercent
        FROM FactFleetTrips
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime)
        GROUP BY VehicleID
    ) current_performance
    WHERE IdlePercent > @IdleThreshold
END
GO
```

## Performance Optimization

### Partitioning Large Fact Tables

```sql
-- Partition by month for FactWarehouseOperations
CREATE PARTITION FUNCTION pf_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, 202604, 202605, 202606)

CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition ALL TO ([PRIMARY])

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey BIGINT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    -- ... other columns ...
    MonthKey INT NOT NULL,
    CONSTRAINT PK_FactWarehouse_Partitioned 
        PRIMARY KEY (MonthKey, OperationKey)
) ON ps_MonthlyPartition(MonthKey)
```

### Columnstore Indexes for Analytics

```sql
-- Add columnstore for read-heavy analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse_Analytics
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey,
    QuantityHandled, DwellTimeHours, DurationMinutes
)

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleet_Analytics
ON FactFleetTrips (
    TimeKey, OriginGeographyKey, DestinationGeographyKey,
    VehicleID, DistanceKm, FuelConsumedLiters, IdleTimeMinutes
)
```

## Best Practices

1. **Incremental Loads**: Always use watermark-based incremental loading to minimize data transfer
2. **Dimension Updates**: Use SCD Type 2 for supplier reliability and product gravity changes
3. **Time Granularity**: Balance 15-minute buckets with daily aggregates for performance
4. **Alerting Tiers**: Implement critical/warning/info severity levels to reduce noise
5. **Data Validation**: Create audit tables to track ETL success/failure rates
6. **Backup Strategy**: Schedule nightly backups with 30-day retention for compliance

## Advanced: Machine Learning Integration

```python
# Example: Predict bottleneck probability using Azure ML
import pyodbc
import pandas as pd
from azure.ai.ml import MLClient
from azure.identity import DefaultAzureCredential

# Connect to LogiFleet database
conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.environ['SQL_SERVER_HOST']};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.environ['SQL_SERVER_USER']};"
    f"PWD={os.environ['SQL_SERVER_PASSWORD']}"
)

# Extract features for ML model
query = """
SELECT 
    f.TimeKey,
    AVG(f.DwellTimeHours) as AvgDwell,
    COUNT(DISTINCT f.ProductKey) as SKUDiversity,
    SUM(f.QuantityHandled) as TotalVolume,
    AVG(ft.IdleTimeMinutes) as AvgFleetIdle,
    AVG(s.LeadTimeVariance) as SupplierVariance
FROM FactWarehouseOperations f
INNER JOIN FactFleetTrips ft ON f.TimeKey = ft.TimeKey
INNER JOIN FactCrossDock cd ON f.ProductKey = cd.ProductKey
INNER JOIN DimSupplier
