---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics intelligence with multi-fact star schema, fleet optimization, and warehouse analytics
triggers:
  - set up logistics analytics database
  - implement supply chain data warehouse
  - create fleet management dashboard
  - build warehouse operations analytics
  - deploy logifleet pulse schema
  - configure logistics intelligence platform
  - integrate warehouse and fleet data
  - design multi-fact logistics schema
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers implement LogiFleet Pulse, an advanced logistics intelligence platform that combines MS SQL Server data warehousing with Power BI visualization for unified supply chain analytics.

## What LogiFleet Pulse Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that:

- **Unifies disparate data sources**: Warehouse operations, fleet telemetry, supplier data, and external APIs
- **Implements multi-fact star schema**: Cross-fact KPI harmonization with time-phased dimensions
- **Provides real-time dashboards**: Power BI visualizations refreshed every 15 minutes
- **Enables predictive analytics**: Bottleneck detection, fleet triage, and temporal elasticity modeling
- **Supports warehouse optimization**: Gravity zone mapping based on pick frequency and item value

The platform connects warehouse velocity, fleet aerodynamics, and demand elasticity through a semantic data layer.

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Access to data sources: WMS, TMS, ERP systems, or telemetry APIs

### Initial Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL Schema**:
```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the schema creation script
:r schema/01_CreateDatabase.sql
:r schema/02_CreateDimensions.sql
:r schema/03_CreateFacts.sql
:r schema/04_CreateViews.sql
:r schema/05_CreateStoredProcedures.sql
```

3. **Configure Data Sources**:
```json
// config.json (do not commit with real credentials)
{
  "sqlServer": {
    "host": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "apiUrl": "${WMS_API_URL}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telemetry": {
      "apiUrl": "${TELEMETRY_API_URL}",
      "apiKey": "${TELEMETRY_API_KEY}"
    }
  }
}
```

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    Date DATE,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfWeek INT,
    Hour INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) UNIQUE NOT NULL,
    NodeType VARCHAR(20), -- 'Warehouse', 'Route', 'Region'
    NodeName VARCHAR(100),
    ParentNodeID VARCHAR(50),
    Continent VARCHAR(50),
    Country VARCHAR(50),
    Region VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    ValidFrom DATETIME2,
    ValidTo DATETIME2,
    IsCurrent BIT DEFAULT 1
);

-- DimProductGravity: Product classification with gravity score
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Pick frequency normalized
    ValueScore DECIMAL(5,2), -- Dollar value normalized
    FragilityScore DECIMAL(5,2), -- Handling complexity
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2),
    OptimalZone VARCHAR(20), -- 'High', 'Medium', 'Low' gravity
    ValidFrom DATETIME2,
    ValidTo DATETIME2,
    IsCurrent BIT DEFAULT 1
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- 'Platinum', 'Gold', 'Silver', 'Bronze'
    ValidFrom DATETIME2,
    ValidTo DATETIME2,
    IsCurrent BIT DEFAULT 1
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID VARCHAR(50),
    Quantity INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    ZoneAssignment VARCHAR(20),
    BatchNumber VARCHAR(50),
    OperatorID VARCHAR(50),
    IsException BIT DEFAULT 0,
    ExceptionReason VARCHAR(500)
);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time 
    ON FactWarehouseOperations(TimeKey) 
    INCLUDE (OperationType, Quantity, DwellTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product 
    ON FactWarehouseOperations(ProductKey, TimeKey) 
    INCLUDE (Quantity, DwellTimeMinutes);

-- FactFleetTrips: Vehicle telemetry and route data
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    RouteID VARCHAR(50),
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TotalTripMinutes INT,
    LoadWeightLbs INT,
    IsDelayed BIT DEFAULT 0,
    DelayReason VARCHAR(500),
    WeatherCondition VARCHAR(50),
    TrafficDensity VARCHAR(20) -- 'Low', 'Medium', 'High'
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle 
    ON FactFleetTrips(VehicleID, TimeKey) 
    INCLUDE (FuelConsumedGallons, IdleTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactFleet_Route 
    ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey, TimeKey);

-- FactCrossDock: Direct transfer operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT,
    DockDwellMinutes INT,
    TransferEfficiencyScore DECIMAL(5,2)
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Load warehouse operations incrementally
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge from staging table
    MERGE INTO FactWarehouseOperations AS target
    USING (
        SELECT 
            t.TimeKey,
            g.GeographyKey,
            p.ProductKey,
            s.SupplierKey,
            stg.OperationType,
            stg.OperationID,
            stg.Quantity,
            stg.DwellTimeMinutes,
            stg.ProcessingTimeMinutes,
            stg.ZoneAssignment,
            stg.BatchNumber,
            stg.OperatorID,
            stg.IsException,
            stg.ExceptionReason
        FROM StagingWarehouseOperations stg
        INNER JOIN DimTime t ON stg.OperationDateTime = t.FullDateTime
        INNER JOIN DimGeography g ON stg.WarehouseID = g.NodeID AND g.IsCurrent = 1
        INNER JOIN DimProductGravity p ON stg.SKU = p.SKU AND p.IsCurrent = 1
        LEFT JOIN DimSupplierReliability s ON stg.SupplierID = s.SupplierID AND s.IsCurrent = 1
        WHERE stg.OperationDateTime BETWEEN @StartDateTime AND @EndDateTime
    ) AS source
    ON target.OperationID = source.OperationID
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, GeographyKey, ProductKey, SupplierKey, OperationType, 
                OperationID, Quantity, DwellTimeMinutes, ProcessingTimeMinutes,
                ZoneAssignment, BatchNumber, OperatorID, IsException, ExceptionReason)
        VALUES (source.TimeKey, source.GeographyKey, source.ProductKey, source.SupplierKey,
                source.OperationType, source.OperationID, source.Quantity, 
                source.DwellTimeMinutes, source.ProcessingTimeMinutes,
                source.ZoneAssignment, source.BatchNumber, source.OperatorID,
                source.IsException, source.ExceptionReason);
    
    -- Log load statistics
    INSERT INTO ETLLog (ProcedureName, RecordsProcessed, StartTime, EndTime)
    VALUES ('sp_LoadWarehouseOperations', @@ROWCOUNT, @StartDateTime, GETDATE());
END;
GO

-- Calculate product gravity scores
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity from last 90 days of activity
    WITH ProductMetrics AS (
        SELECT 
            p.SKU,
            COUNT(DISTINCT f.OperationKey) AS PickCount,
            AVG(f.ProcessingTimeMinutes) AS AvgProcessingTime,
            SUM(f.Quantity * p.ValueScore) AS TotalValue
        FROM FactWarehouseOperations f
        INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
            AND f.OperationType = 'Picking'
        GROUP BY p.SKU
    ),
    ScoreNormalization AS (
        SELECT 
            SKU,
            CAST(PERCENT_RANK() OVER (ORDER BY PickCount) * 100 AS DECIMAL(5,2)) AS VelocityScore,
            CAST(PERCENT_RANK() OVER (ORDER BY TotalValue) * 100 AS DECIMAL(5,2)) AS ValueScore
        FROM ProductMetrics
    )
    UPDATE p
    SET 
        VelocityScore = s.VelocityScore,
        ValueScore = s.ValueScore,
        OptimalZone = CASE 
            WHEN (s.VelocityScore * 0.5 + s.ValueScore * 0.3) > 75 THEN 'High'
            WHEN (s.VelocityScore * 0.5 + s.ValueScore * 0.3) > 40 THEN 'Medium'
            ELSE 'Low'
        END
    FROM DimProductGravity p
    INNER JOIN ScoreNormalization s ON p.SKU = s.SKU
    WHERE p.IsCurrent = 1;
END;
GO

-- Automated alerting for KPI thresholds
CREATE PROCEDURE sp_ProcessLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Fleet idling alert (> 15% of trip time)
    INSERT INTO AlertQueue (AlertType, Severity, Description, AffectedEntity)
    SELECT 
        'Fleet Idle Excessive',
        'High',
        'Vehicle ' + VehicleID + ' exceeded 15% idle time on route ' + RouteID,
        VehicleID
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE Date = CAST(GETDATE() AS DATE) AND Hour = DATEPART(HOUR, GETDATE()))
        AND (CAST(IdleTimeMinutes AS FLOAT) / NULLIF(TotalTripMinutes, 0)) > 0.15
        AND NOT EXISTS (
            SELECT 1 FROM AlertQueue a 
            WHERE a.AffectedEntity = FactFleetTrips.VehicleID 
                AND a.AlertType = 'Fleet Idle Excessive'
                AND a.CreatedDateTime > DATEADD(HOUR, -4, GETDATE())
        );
    
    -- Warehouse dwell time alert (> 72 hours)
    INSERT INTO AlertQueue (AlertType, Severity, Description, AffectedEntity)
    SELECT 
        'Warehouse Dwell Excessive',
        'Medium',
        'SKU ' + p.SKU + ' in zone ' + f.ZoneAssignment + ' exceeds 72hr dwell time',
        p.SKU
    FROM FactWarehouseOperations f
    INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
    WHERE f.DwellTimeMinutes > 4320
        AND f.OperationType IN ('Putaway', 'Receiving')
        AND NOT EXISTS (
            SELECT 1 FROM AlertQueue a 
            WHERE a.AffectedEntity = p.SKU 
                AND a.AlertType = 'Warehouse Dwell Excessive'
                AND a.CreatedDateTime > DATEADD(HOUR, -8, GETDATE())
        );
END;
GO
```

## Cross-Fact Analytics Views

```sql
-- Unified logistics performance view
CREATE VIEW vw_UnifiedLogisticsKPI AS
SELECT 
    t.Date,
    t.Hour,
    g.Region,
    g.NodeName AS Warehouse,
    
    -- Warehouse metrics
    SUM(CASE WHEN w.OperationType = 'Picking' THEN w.Quantity ELSE 0 END) AS TotalPicks,
    AVG(CASE WHEN w.OperationType = 'Picking' THEN w.ProcessingTimeMinutes ELSE NULL END) AS AvgPickTime,
    SUM(w.DwellTimeMinutes) / NULLIF(COUNT(DISTINCT w.ProductKey), 0) AS AvgDwellTime,
    
    -- Fleet metrics
    SUM(f.DistanceMiles) AS TotalMiles,
    SUM(f.FuelConsumedGallons) AS TotalFuel,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    SUM(CASE WHEN f.IsDelayed = 1 THEN 1 ELSE 0 END) * 100.0 / NULLIF(COUNT(f.TripKey), 0) AS DelayPercentage,
    
    -- Cross-fact KPIs
    SUM(f.FuelConsumedGallons) / NULLIF(SUM(CASE WHEN w.OperationType = 'Shipping' THEN w.Quantity ELSE 0 END), 0) AS FuelPerShipment,
    AVG(w.DwellTimeMinutes) * AVG(f.IdleTimeMinutes) / 100.0 AS InefficiencyIndex
    
FROM DimTime t
LEFT JOIN FactWarehouseOperations w ON t.TimeKey = w.TimeKey
LEFT JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY t.Date, t.Hour, g.Region, g.NodeName;
GO

-- Warehouse gravity zone analysis
CREATE VIEW vw_WarehouseGravityAnalysis AS
SELECT 
    p.OptimalZone AS RecommendedZone,
    w.ZoneAssignment AS CurrentZone,
    COUNT(DISTINCT w.ProductKey) AS ProductCount,
    SUM(w.Quantity) AS TotalVolume,
    AVG(w.ProcessingTimeMinutes) AS AvgProcessingTime,
    SUM(w.DwellTimeMinutes) / NULLIF(COUNT(*), 0) AS AvgDwellTime,
    CASE 
        WHEN p.OptimalZone = w.ZoneAssignment THEN 'Aligned'
        ELSE 'Misaligned'
    END AS ZoneAlignment
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE p.IsCurrent = 1
    AND w.TimeKey >= (SELECT TimeKey FROM DimTime WHERE Date = DATEADD(DAY, -7, GETDATE()))
GROUP BY p.OptimalZone, w.ZoneAssignment;
GO
```

## Power BI Integration

### Connection String Setup

```powerquery
// Power BI M Query - SQL Server connection
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"),
        "LogiFleetPulse",
        [
            Query = null,
            CommandTimeout = #duration(0, 0, 10, 0)
        ]
    ),
    Navigation = Source{[Schema="dbo"]}[Data]
in
    Navigation
```

### DAX Measures for Cross-Fact Analysis

```dax
// Total Warehouse Throughput
Warehouse Throughput = 
CALCULATE(
    SUM(FactWarehouseOperations[Quantity]),
    FactWarehouseOperations[OperationType] IN {"Picking", "Shipping"}
)

// Fleet Efficiency Score
Fleet Efficiency = 
VAR TotalMiles = SUM(FactFleetTrips[DistanceMiles])
VAR TotalIdle = SUM(FactFleetTrips[IdleTimeMinutes]) / 60
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedGallons])
RETURN
DIVIDE(TotalMiles, TotalFuel + TotalIdle, 0)

// Cross-Fact: Inventory Velocity Impact on Fuel
Velocity-to-Fuel Ratio = 
VAR WarehouseVelocity = 
    CALCULATE(
        AVERAGEX(
            DimProductGravity,
            DimProductGravity[VelocityScore]
        ),
        DimProductGravity[IsCurrent] = TRUE
    )
VAR FleetFuel = 
    AVERAGE(FactFleetTrips[FuelConsumedGallons])
RETURN
DIVIDE(WarehouseVelocity, FleetFuel, 0)

// Time Intelligence: YoY Comparison
Throughput YoY % = 
VAR CurrentPeriod = [Warehouse Throughput]
VAR PriorPeriod = 
    CALCULATE(
        [Warehouse Throughput],
        DATEADD(DimTime[Date], -1, YEAR)
    )
RETURN
DIVIDE(CurrentPeriod - PriorPeriod, PriorPeriod, 0)

// Predictive Bottleneck Index
Bottleneck Index = 
VAR DwellScore = 
    DIVIDE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        4320, // 72 hours threshold
        0
    )
VAR IdleScore = 
    DIVIDE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        AVERAGE(FactFleetTrips[TotalTripMinutes]),
        0
    )
RETURN
(DwellScore * 0.6) + (IdleScore * 0.4)
```

### Power BI Template Structure

```
LogiFleet_Pulse_Master.pbit
├── Data Model
│   ├── FactWarehouseOperations
│   ├── FactFleetTrips
│   ├── FactCrossDock
│   ├── DimTime
│   ├── DimGeography
│   ├── DimProductGravity
│   └── DimSupplierReliability
├── Reports
│   ├── Executive Overview (high-level KPIs)
│   ├── Warehouse Operations (gravity zones, dwell analysis)
│   ├── Fleet Management (route efficiency, fuel analysis)
│   ├── Cross-Modal Analysis (unified performance)
│   └── Predictive Alerts (bottleneck heatmaps)
└── Security
    └── Row-Level Security (by region, by role)
```

## Common Patterns

### Pattern 1: Daily ETL Pipeline

```sql
-- Execute daily at 2 AM
DECLARE @YesterdayStart DATETIME2 = CAST(DATEADD(DAY, -1, GETDATE()) AS DATE);
DECLARE @YesterdayEnd DATETIME2 = DATEADD(SECOND, -1, DATEADD(DAY, 1, @YesterdayStart));

-- Load warehouse operations
EXEC sp_LoadWarehouseOperations @YesterdayStart, @YesterdayEnd;

-- Load fleet trips
EXEC sp_LoadFleetTrips @YesterdayStart, @YesterdayEnd;

-- Update gravity scores weekly
IF DATEPART(WEEKDAY, GETDATE()) = 1
BEGIN
    EXEC sp_UpdateProductGravityScores;
END

-- Process alerts every 15 minutes
EXEC sp_ProcessLogisticsAlerts;
```

### Pattern 2: Real-Time Dashboard Refresh

```powerquery
// Power BI incremental refresh partition
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"),
        "LogiFleetPulse"
    ),
    FactTable = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FilteredRows = Table.SelectRows(
        FactTable, 
        each [TimeKey] >= RangeStart and [TimeKey] < RangeEnd
    )
in
    FilteredRows
```

### Pattern 3: Custom Gravity Zone Calculation

```sql
-- Recalculate zones based on real-time activity
WITH RealtimeMetrics AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        COUNT(*) AS RecentPicks,
        AVG(f.ProcessingTimeMinutes) AS AvgProcessing
    FROM FactWarehouseOperations f
    INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
        AND f.OperationType = 'Picking'
    GROUP BY p.ProductKey, p.SKU
)
SELECT 
    SKU,
    CASE 
        WHEN RecentPicks > 20 THEN 'High'
        WHEN RecentPicks > 5 THEN 'Medium'
        ELSE 'Low'
    END AS SuggestedZone,
    RecentPicks,
    AvgProcessing
FROM RealtimeMetrics
ORDER BY RecentPicks DESC;
```

## Configuration Best Practices

### SQL Server Indexing Strategy

```sql
-- Partition fact tables by date for performance
CREATE PARTITION FUNCTION pf_LogiFleetDate (DATE)
AS RANGE RIGHT FOR VALUES (
    '2025-01-01', '2025-02-01', '2025-03-01', '2025-04-01',
    '2025-05-01', '2025-06-01', '2025-07-01', '2025-08-01',
    '2025-09-01', '2025-10-01', '2025-11-01', '2025-12-01'
);

CREATE PARTITION SCHEME ps_LogiFleetDate
AS PARTITION pf_LogiFleetDate
ALL TO ([PRIMARY]);

-- Apply to fact table
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- columns as before
) ON ps_LogiFleetDate(OperationDate);

-- Columnstore index for analytical queries
CREATE COLUMNSTORE INDEX IX_FactWarehouse_Columnstore
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, OperationType, 
    Quantity, DwellTimeMinutes, ProcessingTimeMinutes
);
```

### Row-Level Security

```sql
-- Create security predicate
CREATE FUNCTION fn_SecurityPredicateRegion(@Region NVARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS Result
    WHERE @Region IN (
        SELECT Value FROM STRING_SPLIT(
            (SELECT RegionAccess FROM Users WHERE Username = USER_NAME()),
            ','
        )
    )
    OR IS_MEMBER('db_owner') = 1
);
GO

-- Apply to geography dimension
CREATE SECURITY POLICY RegionSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicateRegion(Region)
ON dbo.DimGeography
WITH (STATE = ON);
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom**: Queries joining multiple fact tables take > 30 seconds

**Solution**:
```sql
-- Create indexed views for common joins
CREATE VIEW vw_WarehouseFleetBridge
WITH SCHEMABINDING
AS
SELECT 
    w.TimeKey,
    w.GeographyKey,
    w.ProductKey,
    f.VehicleID,
    SUM(w.Quantity) AS TotalQuantity,
    COUNT_BIG(*) AS RecordCount
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.FactFleetTrips f ON w.TimeKey = f.TimeKey AND w.GeographyKey = f.OriginGeographyKey
WHERE w.OperationType = 'Shipping'
GROUP BY w.TimeKey, w.GeographyKey, w.ProductKey, f.VehicleID;

CREATE UNIQUE CLUSTERED INDEX IX_WarehouseFleetBridge
ON vw_WarehouseFleetBridge (TimeKey, GeographyKey, ProductKey, VehicleID);
```

### Issue: Power BI Refresh Timeout

**Symptom**: "Query timeout expired" during scheduled refresh

**Solution**:
```powerquery
// Increase timeout in M query
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"),
        "LogiFleetPulse",
        [
            CommandTimeout = #duration(0, 1, 0, 0), // 1 hour
            Query = "EXEC sp_LoadWarehouseOperations @StartDateTime, @EndDateTime"
        ]
    )
in
    Source
```

### Issue: Gravity Scores Not Updating

**Symptom**: OptimalZone recommendations remain static

**Solution**:
```sql
-- Verify stored procedure execution
SELECT 
    ProcedureName,
    RecordsProcessed,
    StartTime,
    EndTime
FROM ETLLog
WHERE ProcedureName = 'sp_UpdateProductGravityScores'
ORDER BY StartTime DESC;

-- Manually trigger if needed
EXEC sp_UpdateProductGravityScores;

-- Check for null values blocking calculation
SELECT SKU, VelocityScore, ValueScore, FragilityScore
FROM DimProductGravity
WHERE IsCurrent = 1
    AND (VelocityScore IS NULL OR ValueScore IS NULL);
```

### Issue: Alert Duplicates

**Symptom**: Same alert fires multiple times within threshold window

**Solution**:
```sql
-- Add unique constraint to alert queue
ALTER TABLE AlertQueue
ADD CONSTRAINT UQ_AlertDeduplication 
UNIQUE (AlertType, AffectedEntity, CreatedDate);

-- Modify alert procedure to use MERGE
MERGE INTO AlertQueue AS target
USING (
    SELECT 
        'Fleet Idle Excessive' AS AlertType,
        VehicleID AS AffectedEntity,
        CAST(GETDATE() AS DATE) AS CreatedDate
    FROM FactFleetTrips
    WHERE IdleTimeMinutes / NULLIF(TotalTripMinutes, 0) > 0.15
) AS source
ON target.AlertType = source.AlertType 
    AND target.AffectedEntity = source.AffectedEntity
    AND target.CreatedDate = source.CreatedDate
WHEN NOT MATCHED THEN
    INSERT (AlertType, AffectedEntity, CreatedDate, Severity, Description)
    VALUES (source.AlertType, source.AffectedEntity, source.CreatedDate, 'High', 'Excessive idle time');
```

## Advanced Usage

### Temporal Elasticity Simulation

```sql
-- Simulate impact of warehouse capacity changes on fleet utilization
WITH BaselineMetrics AS (
    SELECT 
        AVG(w.DwellTimeMinutes) AS CurrentDwell,
        AVG(f.IdleTimeMinutes) AS CurrentIdle,
        COUNT(DISTINCT f.VehicleID)
