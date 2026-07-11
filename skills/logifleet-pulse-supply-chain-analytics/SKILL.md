---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics, fleet, and warehouse operations analytics with multi-fact star schema
triggers:
  - "set up logifleet pulse analytics"
  - "deploy supply chain data warehouse"
  - "configure logistics dashboard with power bi"
  - "integrate warehouse and fleet data"
  - "build multi-fact star schema for logistics"
  - "implement cross-modal supply chain analytics"
  - "create warehouse gravity zone model"
  - "setup real-time fleet tracking dashboard"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, inventory data, and external signals into a unified Power BI dashboard. Built on MS SQL Server with a custom multi-fact star schema, it enables cross-fact KPI analysis like correlating warehouse dwell time with fleet idling costs.

## What It Does

- **Unified Semantic Layer**: Links warehouse, fleet, and external data through shared dimensions
- **Multi-Fact Star Schema**: Time-phased dimensions with cross-fact query capabilities
- **Real-Time Dashboards**: Power BI visualizations refreshed every 15 minutes
- **Predictive Analytics**: Bottleneck detection and fleet maintenance prioritization
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency and item value
- **Temporal Elasticity Modeling**: Simulation scenarios for capacity planning

## Installation & Setup

### 1. Deploy SQL Schema

The project includes SQL scripts for creating the data warehouse on MS SQL Server 2019+.

```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    FifteenMinuteBucket TINYINT,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    FiscalPeriod SMALLINT,
    INDEX IX_DateTime (DateTime)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- 'Warehouse', 'Route Node'
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    INDEX IX_LocationType (LocationType)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity + value + fragility
    Velocity NVARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier NVARCHAR(20), -- 'High', 'Medium', 'Low'
    Fragility DECIMAL(3,2),
    INDEX IX_GravityScore (GravityScore DESC)
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    INDEX IX_ComplianceScore (ComplianceScore DESC)
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRate DECIMAL(7,2), -- Units per hour
    PackingTimeMinutes INT,
    StorageZone NVARCHAR(50),
    QuantityHandled INT,
    INDEX IX_TimeGeo (TimeKey, GeographyKey),
    INDEX IX_Product (ProductKey)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    FuelConsumedLiters DECIMAL(7,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DistanceKM DECIMAL(9,2),
    TripDurationMinutes INT,
    INDEX IX_Time (TimeKey),
    INDEX IX_Vehicle (VehicleID)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT NOT NULL FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferTimeMinutes INT,
    QuantityTransferred INT,
    INDEX IX_TimeGeo (TimeKey, GeographyKey)
);
```

### 2. Configure Data Sources

Create a configuration file for data source connections:

```json
-- config.json (example structure - actual values from environment)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "integrated"
  },
  "data_sources": {
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
  },
  "refresh_interval_minutes": 15
}
```

### 3. Create Stored Procedures for Data Loading

```sql
-- Incremental loading procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartTime DATETIME2,
    @EndTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external table or staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        DwellTimeMinutes, PickRate, PackingTimeMinutes,
        StorageZone, QuantityHandled
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        stg.OperationType,
        stg.DwellTimeMinutes,
        stg.PickRate,
        stg.PackingTimeMinutes,
        stg.StorageZone,
        stg.QuantityHandled
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime t ON stg.OperationDateTime >= t.DateTime 
        AND stg.OperationDateTime < DATEADD(MINUTE, 15, t.DateTime)
    INNER JOIN DimGeography g ON stg.WarehouseID = g.LocationName
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    WHERE stg.OperationDateTime BETWEEN @StartTime AND @EndTime
        AND stg.LoadedToFact = 0;
    
    -- Mark as loaded
    UPDATE StagingWarehouseOps
    SET LoadedToFact = 1
    WHERE OperationDateTime BETWEEN @StartTime AND @EndTime;
END;
GO

-- Procedure for calculating warehouse gravity zones
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(*) AS OperationCount,
            AVG(CAST(wo.PickRate AS FLOAT)) AS AvgPickRate,
            SUM(wo.QuantityHandled) AS TotalHandled
        FROM FactWarehouseOperations wo
        INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
        GROUP BY p.ProductKey
    )
    UPDATE p
    SET GravityScore = 
        CASE 
            WHEN pm.AvgPickRate >= 50 THEN 0.4
            WHEN pm.AvgPickRate >= 20 THEN 0.25
            ELSE 0.1
        END +
        CASE 
            WHEN p.ValueTier = 'High' THEN 0.3
            WHEN p.ValueTier = 'Medium' THEN 0.15
            ELSE 0.05
        END +
        (p.Fragility * 0.3)
    FROM DimProductGravity p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
GO
```

### 4. Power BI Integration

Connect Power BI to the SQL Server database:

```powerquery
// Power Query M script for connecting to LogiFleet data
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse"
    ),
    
    // Load fact tables
    FactWarehouse = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FactFleet = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
    FactCrossDock = Source{[Schema="dbo",Item="FactCrossDock"]}[Data],
    
    // Load dimensions
    DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
    DimGeo = Source{[Schema="dbo",Item="DimGeography"]}[Data],
    DimProduct = Source{[Schema="dbo",Item="DimProductGravity"]}[Data],
    DimSupplier = Source{[Schema="dbo",Item="DimSupplierReliability"]}[Data]
in
    Source
```

## Key DAX Measures for Power BI

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```dax
// Measure: Average Dwell Time
Avg Dwell Time = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

// Measure: Fleet Idle Time Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// Measure: Cross-Fact Correlation Score
Dwell-Idle Correlation = 
VAR DwellByProduct = 
    SUMMARIZE(
        FactWarehouseOperations,
        DimProductGravity[SKU],
        "AvgDwell", AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
    )
VAR IdleByProduct = 
    SUMMARIZE(
        RELATEDTABLE(FactCrossDock),
        DimProductGravity[SKU],
        "LinkedIdle", CALCULATE(SUM(FactFleetTrips[IdleTimeMinutes]))
    )
RETURN
    AVERAGEX(
        DwellByProduct,
        [AvgDwell] * RELATED(IdleByProduct[LinkedIdle])
    )

// Measure: Warehouse Gravity Score
Warehouse Gravity = 
CALCULATE(
    AVERAGE(DimProductGravity[GravityScore]),
    FILTER(
        DimProductGravity,
        DimProductGravity[Velocity] IN {"Fast", "Medium"}
    )
)

// Measure: Fleet Utilization Rate
Fleet Utilization = 
VAR ActiveTime = SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalTime = SUM(FactFleetTrips[TripDurationMinutes])
RETURN
DIVIDE(ActiveTime, TotalTime, 0) * 100
```

## Common Analytical Patterns

### 1. Identify High-Dwell Products in Wrong Zones

```sql
-- Query: Products with high gravity in low-gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    wo.StorageZone,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE p.GravityScore >= 0.7
    AND wo.StorageZone NOT IN ('A1', 'A2', 'B1') -- High-priority zones
GROUP BY p.SKU, p.ProductName, p.GravityScore, wo.StorageZone
HAVING AVG(wo.DwellTimeMinutes) > 60
ORDER BY AvgDwellTime DESC;
```

### 2. Fleet Triage by Revenue Impact

```sql
-- Query: Prioritize fleet maintenance by cargo value
WITH FleetCargoValue AS (
    SELECT 
        ft.VehicleID,
        SUM(cd.QuantityTransferred * 
            CASE p.ValueTier 
                WHEN 'High' THEN 100 
                WHEN 'Medium' THEN 50 
                ELSE 20 
            END
        ) AS EstimatedCargoValue,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(CASE WHEN ft.IdleTimeMinutes > 30 THEN 1 ELSE 0 END) AS HighIdleEvents
    FROM FactFleetTrips ft
    LEFT JOIN FactCrossDock cd ON ft.TripKey = cd.OutboundTripKey
    LEFT JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    WHERE ft.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days
    GROUP BY ft.VehicleID
)
SELECT 
    VehicleID,
    EstimatedCargoValue,
    AvgIdleTime,
    HighIdleEvents,
    (EstimatedCargoValue / 1000.0) * (AvgIdleTime / 60.0) AS MaintenancePriorityScore
FROM FleetCargoValue
WHERE HighIdleEvents >= 3
ORDER BY MaintenancePriorityScore DESC;
```

### 3. Temporal Elasticity Simulation

```sql
-- Query: Simulate warehouse capacity impact on fleet utilization
WITH CapacityScenarios AS (
    SELECT 0.80 AS CapacityFactor UNION ALL
    SELECT 0.85 UNION ALL
    SELECT 0.90 UNION ALL
    SELECT 0.95
),
CurrentMetrics AS (
    SELECT 
        AVG(CAST(wo.DwellTimeMinutes AS FLOAT)) AS AvgDwell,
        AVG(CAST(ft.IdleTimeMinutes AS FLOAT)) AS AvgIdle
    FROM FactWarehouseOperations wo
    CROSS JOIN FactFleetTrips ft
    WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
)
SELECT 
    cs.CapacityFactor,
    cm.AvgDwell * (1.0 / cs.CapacityFactor) AS ProjectedDwellTime,
    cm.AvgIdle * (1.0 + (1.0 - cs.CapacityFactor) * 2) AS ProjectedIdleTime,
    (cm.AvgDwell * (1.0 / cs.CapacityFactor)) - cm.AvgDwell AS DwellDelta
FROM CapacityScenarios cs
CROSS JOIN CurrentMetrics cm
ORDER BY cs.CapacityFactor;
```

## Automated Alerting

### Setup Alert Stored Procedure

```sql
CREATE PROCEDURE sp_CheckThresholdsAndAlert
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertLog TABLE (
        AlertType NVARCHAR(100),
        Severity NVARCHAR(20),
        Message NVARCHAR(500),
        MetricValue DECIMAL(10,2)
    );
    
    -- Check: Fleet idle time threshold
    INSERT INTO @AlertLog
    SELECT 
        'Fleet Idle Threshold Breach',
        'High',
        'Vehicle ' + VehicleID + ' exceeded 15% idle time',
        (IdleTimeMinutes * 1.0 / TripDurationMinutes) * 100
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
        AND (IdleTimeMinutes * 1.0 / TripDurationMinutes) > 0.15;
    
    -- Check: Warehouse dwell time anomaly
    INSERT INTO @AlertLog
    SELECT 
        'Warehouse Dwell Anomaly',
        'Medium',
        'SKU ' + p.SKU + ' in zone ' + wo.StorageZone + ' has excessive dwell time',
        AVG(CAST(wo.DwellTimeMinutes AS DECIMAL(10,2)))
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime)
        AND p.GravityScore >= 0.7
    GROUP BY p.SKU, wo.StorageZone
    HAVING AVG(wo.DwellTimeMinutes) > 72;
    
    -- Output alerts (integrate with email/SMS service)
    SELECT * FROM @AlertLog;
END;
GO

-- Schedule this procedure via SQL Agent or external scheduler
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Create non-clustered indexes for common query patterns
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, PickRate);

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeVehicle 
ON FactFleetTrips(TimeKey, VehicleID) 
INCLUDE (IdleTimeMinutes, FuelConsumedLiters);

-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCIX_FactWarehouse_Analytics
ON FactWarehouseOperations(TimeKey, GeographyKey, ProductKey, DwellTimeMinutes, QuantityHandled);
```

### Row-Level Security

```sql
-- Create security policy for regional access
CREATE SCHEMA Security;
GO

CREATE FUNCTION Security.fn_RegionPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS Result
WHERE @GeographyKey IN (
    SELECT GeographyKey 
    FROM dbo.DimGeography g
    INNER JOIN dbo.UserRegionAccess ura ON g.Region = ura.Region
    WHERE ura.UserName = USER_NAME()
);
GO

CREATE SECURITY POLICY RegionalDataAccess
ADD FILTER PREDICATE Security.fn_RegionPredicate(GeographyKey) 
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE Security.fn_RegionPredicate(OriginGeographyKey) 
ON dbo.FactFleetTrips
WITH (STATE = ON);
```

## Troubleshooting

### Issue: Power BI Refresh Failures

**Symptom**: Scheduled refresh fails with timeout error.

**Solution**: Implement incremental refresh in Power BI:

```powerquery
// Define RangeStart and RangeEnd parameters
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse",
        [Query="
            SELECT * FROM FactWarehouseOperations
            WHERE TimeKey >= " & Number.ToText(RangeStart) & "
            AND TimeKey < " & Number.ToText(RangeEnd)
        ]
    )
in
    Source
```

### Issue: Cross-Fact Query Performance

**Symptom**: Queries joining multiple fact tables run slowly.

**Solution**: Use bridge tables or materialized views:

```sql
-- Create materialized view for common cross-fact analysis
CREATE VIEW vw_WarehouseFleetCorrelation
WITH SCHEMABINDING
AS
SELECT 
    wo.TimeKey,
    wo.ProductKey,
    wo.GeographyKey,
    AVG(wo.DwellTimeMinutes) AS AvgDwell,
    COUNT_BIG(*) AS OpCount,
    (SELECT AVG(IdleTimeMinutes) 
     FROM dbo.FactFleetTrips ft
     WHERE ft.TimeKey = wo.TimeKey) AS ConcurrentAvgIdle
FROM dbo.FactWarehouseOperations wo
GROUP BY wo.TimeKey, wo.ProductKey, wo.GeographyKey;
GO

CREATE UNIQUE CLUSTERED INDEX IX_MaterializedView 
ON vw_WarehouseFleetCorrelation(TimeKey, ProductKey, GeographyKey);
```

### Issue: Dimension Table SCD Handling

**Symptom**: Historical product categorizations lost after updates.

**Solution**: Implement Type 2 SCD for DimProductGravity:

```sql
ALTER TABLE DimProductGravity ADD 
    ValidFrom DATETIME2 DEFAULT GETDATE(),
    ValidTo DATETIME2 DEFAULT '9999-12-31',
    IsCurrent BIT DEFAULT 1;

CREATE PROCEDURE sp_UpdateProductDimension
    @SKU NVARCHAR(50),
    @NewGravityScore DECIMAL(5,2)
AS
BEGIN
    -- Expire current record
    UPDATE DimProductGravity
    SET ValidTo = GETDATE(), IsCurrent = 0
    WHERE SKU = @SKU AND IsCurrent = 1;
    
    -- Insert new record
    INSERT INTO DimProductGravity (SKU, GravityScore, ValidFrom, IsCurrent)
    SELECT SKU, @NewGravityScore, GETDATE(), 1
    FROM DimProductGravity
    WHERE SKU = @SKU AND ValidTo = GETDATE();
END;
```

## Environment Variables Reference

Required environment variables for deployment:

- `SQL_SERVER_HOST` — MS SQL Server hostname or IP
- `SQL_SERVER_DATABASE` — Database name (default: LogiFleetPulse)
- `WMS_API_ENDPOINT` — Warehouse Management System API URL
- `WMS_API_KEY` — WMS authentication key
- `TELEMATICS_API_ENDPOINT` — Fleet telematics service URL
- `TELEMATICS_API_KEY` — Telematics authentication key
- `WEATHER_API_ENDPOINT` — External weather data service
- `WEATHER_API_KEY` — Weather service API key
- `POWERBI_WORKSPACE_ID` — Power BI workspace for deployment
- `ALERT_EMAIL_SMTP` — SMTP server for email alerts
- `ALERT_EMAIL_FROM` — Sender email address for alerts
