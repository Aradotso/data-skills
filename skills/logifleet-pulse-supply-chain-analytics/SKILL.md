---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for fleet logistics, warehouse operations, and cross-modal supply chain intelligence with multi-fact star schema
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - create multi-fact star schema for fleet and warehouse
  - implement supply chain intelligence dashboard
  - configure logifleet pulse sql server
  - build warehouse gravity zone analytics
  - integrate fleet telemetry with warehouse operations
  - design logistics kpi harmonization model
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced data warehousing and analytics platform that unifies fleet logistics, warehouse operations, and supply chain intelligence into a single semantic layer. Built on MS SQL Server with Power BI visualization, it uses a multi-fact star schema with time-phased dimensions to enable cross-modal analytics (e.g., correlating warehouse dwell time with fleet idling costs, or inventory velocity with route optimization).

## Core Capabilities

- **Multi-fact star schema** linking warehouse operations, fleet trips, cross-dock activities
- **Time-phased dimensions** at 15-minute granularity for real-time analytics
- **Warehouse Gravity Zones** spatial modeling based on pick frequency, value, and fragility
- **Fleet triage engine** prioritizing maintenance by revenue impact
- **Cross-fact KPI harmonization** through composite shared dimensions
- **Predictive bottleneck detection** using historical multi-fact correlations
- **Role-based access control** with row-level security

## Installation & Setup

### 1. Deploy SQL Server Schema

```sql
-- Create database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create time dimension (15-minute buckets)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek NVARCHAR(20),
    FiscalPeriod NVARCHAR(20),
    IsBusinessHour BIT
)
GO

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- 'Warehouse', 'RouteNode', 'Depot'
    Continent NVARCHAR(50),
    Country NVARCHAR(50),
    Region NVARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1
)
GO

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(300),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Picks per day normalized
    ValueScore DECIMAL(5,2), -- Revenue per unit normalized
    FragilityScore DECIMAL(5,2), -- Handling complexity 0-10
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone NVARCHAR(50), -- 'HighGravity', 'MediumGravity', 'LowGravity'
    LastUpdated DATETIME2 DEFAULT GETDATE()
)
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    Quantity INT,
    DwellTimeHours DECIMAL(8,2), -- Time in warehouse before next operation
    StorageZone NVARCHAR(50),
    OperatorID NVARCHAR(50),
    BatchID NVARCHAR(100)
)
GO

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID NVARCHAR(50) NOT NULL,
    RouteID NVARCHAR(100),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    LoadingTimeMinutes DECIMAL(8,2),
    UnloadingTimeMinutes DECIMAL(8,2),
    LoadWeightKG DECIMAL(10,2),
    AverageSP DECIMAL(5,2), -- Speed km/h
    WeatherCondition NVARCHAR(50), -- From external API
    TrafficDelay BIT
)
GO

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200),
    LeadTimeDaysAvg DECIMAL(5,2),
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,4),
    OnTimeDeliveryRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    LastAuditDate DATE
)
GO

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time 
ON FactWarehouseOperations(TimeKey) INCLUDE (ProductKey, DurationMinutes, DwellTimeHours)
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_Time 
ON FactFleetTrips(TimeKey) INCLUDE (VehicleID, DurationMinutes, FuelConsumedLiters, IdleTimeMinutes)
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product 
ON FactWarehouseOperations(ProductKey) INCLUDE (TimeKey, OperationType, Quantity)
GO
```

### 2. Load Dimension Data with Stored Procedures

```sql
-- Stored procedure: Populate time dimension
CREATE PROCEDURE sp_PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @EndDateTime DATETIME2 = DATEADD(DAY, 1, @EndDate);
    
    WHILE @CurrentDateTime < @EndDateTime
    BEGIN
        INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeOfDay, Hour, MinuteBucket, DayOfWeek, FiscalPeriod, IsBusinessHour)
        VALUES (
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
            @CurrentDateTime,
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMdd') AS INT),
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15,
            DATENAME(WEEKDAY, @CurrentDateTime),
            CONCAT('FY', YEAR(@CurrentDateTime), 'Q', DATEPART(QUARTER, @CurrentDateTime)),
            CASE 
                WHEN DATEPART(HOUR, @CurrentDateTime) BETWEEN 8 AND 17 
                     AND DATEPART(WEEKDAY, @CurrentDateTime) NOT IN (1, 7) 
                THEN 1 
                ELSE 0 
            END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END
GO

-- Stored procedure: Update product gravity scores
CREATE PROCEDURE sp_UpdateProductGravityScores
    @DaysWindow INT = 30
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity score based on recent picks
    WITH VelocityCalc AS (
        SELECT 
            ProductKey,
            COUNT(*) AS TotalPicks,
            COUNT(*) * 1.0 / @DaysWindow AS PicksPerDay
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND OperationStartTime >= DATEADD(DAY, -@DaysWindow, GETDATE())
        GROUP BY ProductKey
    ),
    NormalizedVelocity AS (
        SELECT 
            ProductKey,
            (PicksPerDay - MIN(PicksPerDay) OVER()) / 
            NULLIF(MAX(PicksPerDay) OVER() - MIN(PicksPerDay) OVER(), 0) * 10 AS VelocityScore
        FROM VelocityCalc
    )
    UPDATE p
    SET 
        p.VelocityScore = ISNULL(nv.VelocityScore, 0),
        p.RecommendedZone = CASE 
            WHEN p.GravityScore >= 7 THEN 'HighGravity'
            WHEN p.GravityScore >= 4 THEN 'MediumGravity'
            ELSE 'LowGravity'
        END,
        p.LastUpdated = GETDATE()
    FROM DimProductGravity p
    LEFT JOIN NormalizedVelocity nv ON p.ProductKey = nv.ProductKey;
END
GO
```

### 3. Create Cross-Fact Analytics Views

```sql
-- View: Unified logistics performance
CREATE VIEW vw_UnifiedLogisticsPerformance AS
SELECT 
    t.FullDateTime,
    t.FiscalPeriod,
    g.LocationName AS WarehouseName,
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZone,
    
    -- Warehouse metrics
    AVG(wo.DurationMinutes) AS AvgOperationDurationMin,
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(wo.Quantity) AS TotalQuantityProcessed,
    
    -- Fleet metrics (for products shipped from this warehouse)
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelEfficiency,
    
    -- Cross-fact KPI: Cost per unit moved
    (AVG(ft.FuelConsumedLiters) * 1.5) / NULLIF(SUM(wo.Quantity), 0) AS EstimatedFuelCostPerUnit
    
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON ft.OriginKey = wo.WarehouseKey 
    AND ft.TimeKey >= wo.TimeKey 
    AND ft.TimeKey <= wo.TimeKey + 96 -- Within 24 hours
GROUP BY 
    t.FullDateTime, t.FiscalPeriod, g.LocationName, 
    p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone
GO

-- View: Fleet triage priority queue
CREATE VIEW vw_FleetTriagePriorityQueue AS
SELECT 
    ft.VehicleID,
    ft.TripStartTime,
    ft.RouteID,
    p.SKU,
    p.ValueScore,
    
    -- Calculate priority score
    (
        (ft.IdleTimeMinutes / NULLIF(ft.DurationMinutes, 0)) * 30 + -- Idle % weight
        (p.ValueScore * 0.5) + -- Product value weight
        CASE WHEN p.Category = 'Perishables' THEN 20 ELSE 0 END -- Urgency boost
    ) AS TriagePriorityScore,
    
    CASE 
        WHEN ft.IdleTimeMinutes / NULLIF(ft.DurationMinutes, 0) > 0.15 THEN 'High Idle'
        WHEN ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0) > 0.25 THEN 'Fuel Inefficient'
        WHEN ft.LoadWeightKG > 8000 THEN 'Overweight Risk'
        ELSE 'Normal'
    END AS AlertType
    
FROM FactFleetTrips ft
INNER JOIN FactWarehouseOperations wo ON wo.WarehouseKey = ft.OriginKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE ft.TripStartTime >= DATEADD(DAY, -7, GETDATE())
GO
```

## Power BI Integration

### Configure Data Connection

```javascript
// Power BI M query for incremental refresh
let
    Source = Sql.Database("${SQL_SERVER}", "LogiFleetPulse"),
    
    // Parameter-based filtering for incremental refresh
    FilteredData = Table.SelectRows(
        Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
        each [OperationStartTime] >= RangeStart and [OperationStartTime] < RangeEnd
    ),
    
    // Join with dimensions
    JoinedWithTime = Table.NestedJoin(
        FilteredData, {"TimeKey"}, 
        DimTime, {"TimeKey"}, 
        "TimeDetails", JoinKind.Inner
    ),
    
    ExpandedTime = Table.ExpandTableColumn(
        JoinedWithTime, "TimeDetails", 
        {"FiscalPeriod", "DayOfWeek", "IsBusinessHour"}
    )
in
    ExpandedTime
```

### Create DAX Measures

```dax
// Measure: Warehouse efficiency index
Warehouse Efficiency Index = 
VAR AvgDuration = AVERAGE(FactWarehouseOperations[DurationMinutes])
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR Benchmark_Duration = 45 // minutes
VAR Benchmark_Dwell = 24 // hours
RETURN
    (Benchmark_Duration / AvgDuration * 50) + 
    (Benchmark_Dwell / AvgDwell * 50)

// Measure: Fleet idle cost impact
Fleet Idle Cost Impact = 
VAR IdleMinutes = SUM(FactFleetTrips[IdleTimeMinutes])
VAR HourlyCost = 35 // Average operational cost per hour
VAR ProductValue = AVERAGE(DimProductGravity[ValueScore]) * 1000
RETURN
    (IdleMinutes / 60) * HourlyCost * (1 + ProductValue / 10000)

// Measure: Cross-fact bottleneck index
Bottleneck Prediction Index = 
VAR HighDwellProducts = 
    COUNTROWS(
        FILTER(
            FactWarehouseOperations, 
            FactWarehouseOperations[DwellTimeHours] > 72
        )
    )
VAR HighIdleTrips = 
    COUNTROWS(
        FILTER(
            FactFleetTrips, 
            FactFleetTrips[IdleTimeMinutes] / FactFleetTrips[DurationMinutes] > 0.15
        )
    )
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
RETURN
    ((HighDwellProducts + HighIdleTrips * 1.5) / TotalOperations) * 100

// Measure: Gravity zone compliance
Gravity Zone Compliance % = 
DIVIDE(
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[StorageZone] = 
                RELATED(DimProductGravity[RecommendedZone])
        )
    ),
    COUNTROWS(FactWarehouseOperations),
    0
) * 100
```

## Configuration & ETL Patterns

### Incremental Data Loading

```sql
-- Stored procedure: Incremental load warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations_Incremental
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assuming external staging table from WMS integration
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, Quantity,
        DwellTimeHours, StorageZone, OperatorID, BatchID
    )
    SELECT 
        CAST(FORMAT(stg.OperationStartTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        g.GeographyKey,
        p.ProductKey,
        stg.OperationType,
        stg.OperationStartTime,
        stg.OperationEndTime,
        stg.Quantity,
        stg.DwellTimeHours,
        stg.StorageZone,
        stg.OperatorID,
        stg.BatchID
    FROM StagingWarehouseOperations stg
    INNER JOIN DimGeography g ON stg.WarehouseCode = g.LocationCode
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    WHERE stg.OperationStartTime > @LastLoadTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations fo
            WHERE fo.BatchID = stg.BatchID 
                AND fo.OperationStartTime = stg.OperationStartTime
        );
    
    -- Update last load timestamp
    UPDATE ETL_Control
    SET LastLoadTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END
GO
```

### External Data Integration (Weather/Traffic)

```sql
-- Create external table for Azure-hosted traffic data
CREATE EXTERNAL DATA SOURCE TrafficDataSource
WITH (
    TYPE = HADOOP,
    LOCATION = 'wasbs://trafficdata@${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net',
    CREDENTIAL = AzureStorageCredential
);
GO

CREATE EXTERNAL FILE FORMAT CSVFileFormat
WITH (
    FORMAT_TYPE = DELIMITEDTEXT,
    FORMAT_OPTIONS (FIELD_TERMINATOR = ',', STRING_DELIMITER = '"')
);
GO

CREATE EXTERNAL TABLE ExtTrafficDelays (
    RouteID NVARCHAR(100),
    Timestamp DATETIME2,
    DelayMinutes INT
)
WITH (
    LOCATION = '/delays/',
    DATA_SOURCE = TrafficDataSource,
    FILE_FORMAT = CSVFileFormat
);
GO

-- Merge traffic delays into fleet trips
UPDATE ft
SET ft.TrafficDelay = CASE WHEN etd.DelayMinutes > 15 THEN 1 ELSE 0 END
FROM FactFleetTrips ft
INNER JOIN ExtTrafficDelays etd 
    ON ft.RouteID = etd.RouteID
    AND ABS(DATEDIFF(MINUTE, ft.TripStartTime, etd.Timestamp)) < 30;
```

## Common Patterns & Use Cases

### Pattern 1: Identify Misplaced High-Gravity Products

```sql
-- Find products in wrong gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZone,
    wo.StorageZone AS CurrentZone,
    COUNT(*) AS OperationCount,
    AVG(wo.DurationMinutes) AS AvgPickDuration
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
    AND wo.StorageZone <> p.RecommendedZone
    AND wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone, wo.StorageZone
HAVING COUNT(*) > 50
ORDER BY p.GravityScore DESC;
```

### Pattern 2: Correlate Fleet Idle with Warehouse Delays

```sql
-- Cross-fact analysis: Does warehouse dwell predict fleet idle?
WITH WarehouseDwell AS (
    SELECT 
        wo.WarehouseKey,
        AVG(wo.DwellTimeHours) AS AvgDwellHours,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    WHERE wo.OperationStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY wo.WarehouseKey
),
FleetIdle AS (
    SELECT 
        ft.OriginKey AS WarehouseKey,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    WHERE ft.TripStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY ft.OriginKey
)
SELECT 
    g.LocationName,
    wd.AvgDwellHours,
    fi.AvgIdleMinutes,
    -- Correlation indicator
    CASE 
        WHEN wd.AvgDwellHours > 48 AND fi.AvgIdleMinutes > 30 THEN 'High Risk'
        WHEN wd.AvgDwellHours > 24 AND fi.AvgIdleMinutes > 20 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.WarehouseKey = fi.WarehouseKey
INNER JOIN DimGeography g ON wd.WarehouseKey = g.GeographyKey
ORDER BY wd.AvgDwellHours DESC, fi.AvgIdleMinutes DESC;
```

### Pattern 3: Automated Alert Generation

```sql
-- Stored procedure: Generate daily alerts
CREATE PROCEDURE sp_GenerateDailyAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Alert 1: High-gravity products in low-gravity zones
    INSERT INTO AlertQueue (AlertType, Severity, Description, CreatedAt)
    SELECT 
        'GravityMismatch',
        'High',
        CONCAT('Product ', p.SKU, ' (gravity: ', p.GravityScore, 
               ') found in ', wo.StorageZone, ' instead of ', p.RecommendedZone),
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.StorageZone <> p.RecommendedZone
        AND p.GravityScore >= 7
        AND wo.OperationStartTime >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY p.SKU, p.GravityScore, wo.StorageZone, p.RecommendedZone
    HAVING COUNT(*) > 5;
    
    -- Alert 2: Fleet vehicles with excessive idle time
    INSERT INTO AlertQueue (AlertType, Severity, Description, CreatedAt)
    SELECT 
        'ExcessiveIdle',
        'Medium',
        CONCAT('Vehicle ', ft.VehicleID, ' idle for ', 
               ft.IdleTimeMinutes, ' minutes (', 
               ROUND(ft.IdleTimeMinutes * 100.0 / ft.DurationMinutes, 1), '% of trip)'),
        GETDATE()
    FROM FactFleetTrips ft
    WHERE ft.TripStartTime >= DATEADD(HOUR, -24, GETDATE())
        AND ft.IdleTimeMinutes / NULLIF(ft.DurationMinutes, 0) > 0.15;
    
    -- Alert 3: Supplier reliability degradation
    INSERT INTO AlertQueue (AlertType, Severity, Description, CreatedAt)
    SELECT 
        'SupplierReliability',
        'High',
        CONCAT('Supplier ', s.SupplierName, ' on-time rate dropped to ', 
               ROUND(s.OnTimeDeliveryRate * 100, 1), '%'),
        GETDATE()
    FROM DimSupplierReliability s
    WHERE s.OnTimeDeliveryRate < 0.85
        AND s.LastAuditDate >= DATEADD(DAY, -30, GETDATE());
END
GO

-- Schedule with SQL Agent
EXEC sp_GenerateDailyAlerts;
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Solution**: Ensure proper indexing on time and geography keys

```sql
-- Add columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Analytics
ON FactWarehouseOperations (TimeKey, WarehouseKey, ProductKey, DurationMinutes, DwellTimeHours);
GO

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_Analytics
ON FactFleetTrips (TimeKey, OriginKey, DestinationKey, IdleTimeMinutes, FuelConsumedLiters);
GO

-- Partition large fact tables by date
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT PK_FactWarehouse_Partitioned PRIMARY KEY CLUSTERED (OperationKey, OperationStartTime)
ON DateRangePartitionScheme(OperationStartTime);
```

### Issue: Power BI Refresh Timeouts

**Solution**: Use incremental refresh and aggregations

```dax
// Create aggregation table in Power BI
WarehouseOperations_Agg = 
SUMMARIZECOLUMNS(
    DimTime[FiscalPeriod],
    DimGeography[LocationName],
    DimProductGravity[Category],
    "AvgDuration", AVERAGE(FactWarehouseOperations[DurationMinutes]),
    "TotalQuantity", SUM(FactWarehouseOperations[Quantity]),
    "AvgDwell", AVERAGE(FactWarehouseOperations[DwellTimeHours])
)
```

### Issue: Gravity Scores Not Updating

**Solution**: Check stored procedure execution and time window

```sql
-- Verify last gravity update
SELECT ProductKey, SKU, VelocityScore, LastUpdated
FROM DimProductGravity
WHERE LastUpdated < DATEADD(DAY, -7, GETDATE())
ORDER BY GravityScore DESC;

-- Force recalculation with longer window
EXEC sp_UpdateProductGravityScores @DaysWindow = 90;
```

### Issue: Missing Fleet-Warehouse Correlations

**Solution**: Validate time key alignment (15-minute buckets)

```sql
-- Check for orphaned records
SELECT 'Warehouse' AS Source, COUNT(*) AS OrphanCount
FROM FactWarehouseOperations wo
WHERE NOT EXISTS (SELECT 1 FROM DimTime t WHERE t.TimeKey = wo.TimeKey)

UNION ALL

SELECT 'Fleet', COUNT(*)
FROM FactFleetTrips ft
WHERE NOT EXISTS (SELECT 1 FROM DimTime t WHERE t.TimeKey = ft.TimeKey);

-- Correct time keys with rounding
UPDATE FactWarehouseOperations
SET TimeKey = CAST(
    FORMAT(
        DATEADD(
            MINUTE, 
            (DATEPART(MINUTE, OperationStartTime) / 15) * 15, 
            DATEADD(MINUTE, -DATEPART(MINUTE, OperationStartTime), OperationStartTime)
        ), 
        'yyyyMMddHHmm'
    ) AS INT
)
WHERE TimeKey NOT IN (SELECT TimeKey FROM DimTime);
```

## Security & Compliance

### Row-Level Security Implementation

```sql
-- Create security table
CREATE TABLE UserWarehouseAccess (
    UserEmail NVARCHAR(200),
    WarehouseKey INT
);
GO

-- Create security function
CREATE FUNCTION fn_WarehouseSecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS AccessGranted
    WHERE @WarehouseKey IN (
        SELECT WarehouseKey 
        FROM dbo.UserWarehouseAccess 
        WHERE UserEmail = USER_NAME()
    )
    OR IS_MEMBER('LogisticsAdmin') = 1;
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_WarehouseSecurityPredicate(WarehouseKey) 
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_WarehouseSecurityPredicate(OriginKey) 
ON dbo.FactFleetTrips
WITH (STATE = ON);
GO
```

## Environment Variables

All sensitive configuration should use environment variables:

- `${SQL_SERVER}` - SQL Server hostname
- `${SQL_DATABASE}` - Database name (default: LogiFleetPulse)
- `${SQL_USER}` - Database username
- `${SQL_PASSWORD}` - Database password
- `${AZURE_STORAGE_ACCOUNT}` - Azure storage for external data
- `${POWERBI_WORKSPACE}` - Power BI workspace ID
- `${ALERT_EMAIL_SMTP}` - SMTP server for alerts
- `${ALERT_EMAIL_FROM}` - Sender email address

## Advanced: Predictive Modeling Integration

```sql
-- Create staging table for ML predictions
CREATE TABLE ML_BottleneckPredictions (
    PredictionID INT PRIMARY KEY IDENTITY(1,1),
    WarehouseKey INT,
    ProductKey INT,
    PredictionDate DATETIME2,
    PredictedDwellHours DECIMAL(8,2),
    ConfidenceScore DECIMAL(5,4),
    ModelVersion
