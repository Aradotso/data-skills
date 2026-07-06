---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI multi-modal logistics intelligence platform for warehouse, fleet, and supply chain analytics with predictive bottleneck detection
triggers:
  - "set up LogiFleet Pulse supply chain dashboard"
  - "configure warehouse and fleet analytics database"
  - "implement multi-fact star schema for logistics"
  - "create Power BI logistics intelligence dashboard"
  - "deploy warehouse gravity zone optimization"
  - "build real-time fleet tracking analytics"
  - "integrate supply chain data warehouse"
  - "setup cross-modal logistics KPI dashboard"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse, fleet, and supply chain operations
- **Real-time dashboards** with 15-minute data refresh cycles
- **Predictive bottleneck detection** using cross-fact KPI harmonization
- **Warehouse Gravity Zones™** for optimal storage layout
- **Adaptive fleet triage** based on real-time telemetry and revenue impact
- **Temporal elasticity modeling** for scenario simulation

The platform ingests data from WMS, GPS/telematics, supplier portals, weather APIs, and order history to provide unified logistics intelligence.

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Enterprise or Standard edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate database credentials with CREATE DATABASE and CREATE TABLE permissions

### Step 1: Deploy SQL Schema

```sql
-- Create the main database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayOfMonth INT NOT NULL,
    Hour INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    IsBusinessHour BIT NOT NULL,
    FiscalPeriod VARCHAR(20),
    INDEX IX_DimTime_DateTime NONCLUSTERED (DateTime),
    INDEX IX_DimTime_Date NONCLUSTERED (Date)
)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(255) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, RouteNode, CrossDock
    Continent VARCHAR(100),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1,
    INDEX IX_DimGeography_LocationID NONCLUSTERED (LocationID),
    INDEX IX_DimGeography_Type NONCLUSTERED (LocationType)
)
GO

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(255) NOT NULL,
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    WeightKG DECIMAL(10,3),
    VolumeM3 DECIMAL(10,4),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    RequiresRefrigeration BIT DEFAULT 0,
    GravityScore DECIMAL(5,2), -- Computed: velocity * value / fragility
    GravityZone VARCHAR(50), -- HighGravity, MediumGravity, LowGravity
    UnitValue DECIMAL(12,2),
    INDEX IX_DimProduct_SKU NONCLUSTERED (SKU),
    INDEX IX_DimProduct_GravityZone NONCLUSTERED (GravityZone)
)
GO

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(255) NOT NULL,
    Country VARCHAR(100),
    LeadTimeDays INT,
    LeadTimeVarianceDays DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- Gold, Silver, Bronze
    INDEX IX_DimSupplier_SupplierID NONCLUSTERED (SupplierID)
)
GO

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) NOT NULL UNIQUE,
    VehicleType VARCHAR(50), -- Truck, Van, Refrigerated
    Capacity_KG DECIMAL(10,2),
    Capacity_M3 DECIMAL(10,2),
    FuelType VARCHAR(50),
    YearManufactured INT,
    MaintenanceStatus VARCHAR(50),
    LastMaintenanceDate DATE,
    NextMaintenanceDue DATE,
    IsActive BIT DEFAULT 1,
    INDEX IX_DimFleet_VehicleID NONCLUSTERED (VehicleID)
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME2 NOT NULL,
    OperationEndTime DATETIME2,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    QuantityHandled INT,
    ErrorCount INT DEFAULT 0,
    EmployeeID VARCHAR(50),
    StorageZone VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    INDEX IX_FactWH_Time NONCLUSTERED (TimeKey),
    INDEX IX_FactWH_Product NONCLUSTERED (ProductKey),
    INDEX IX_FactWH_Geography NONCLUSTERED (GeographyKey),
    INDEX IX_FactWH_OperationType NONCLUSTERED (OperationType)
)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    FleetKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripStartTime DATETIME2 NOT NULL,
    TripEndTime DATETIME2,
    PlannedDurationMinutes INT,
    ActualDurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    DistanceKM DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    DelayMinutes DECIMAL(10,2),
    DelayReason VARCHAR(255),
    WeatherCondition VARCHAR(100),
    DriverID VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (FleetKey) REFERENCES DimFleet(FleetKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_FactFleet_Time NONCLUSTERED (TimeKey),
    INDEX IX_FactFleet_Fleet NONCLUSTERED (FleetKey),
    INDEX IX_FactFleet_Origin NONCLUSTERED (OriginGeographyKey),
    INDEX IX_FactFleet_Destination NONCLUSTERED (DestinationGeographyKey)
)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    ArrivalTime DATETIME2 NOT NULL,
    DepartureTime DATETIME2,
    DwellMinutes DECIMAL(10,2),
    QuantityUnits INT,
    TemperatureMin DECIMAL(5,2),
    TemperatureMax DECIMAL(5,2),
    QualityCheckPassed BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey),
    INDEX IX_FactCD_Time NONCLUSTERED (TimeKey),
    INDEX IX_FactCD_Product NONCLUSTERED (ProductKey),
    INDEX IX_FactCD_Supplier NONCLUSTERED (SupplierKey)
)
GO
```

### Step 2: Create Utility Views and Stored Procedures

```sql
-- Cross-fact KPI view: Dwell time vs Fleet utilization
CREATE VIEW vw_DwellTimeFleetUtilization AS
SELECT 
    dt.Date,
    dg.LocationName,
    dp.Category AS ProductCategory,
    AVG(fw.DwellTimeHours) AS AvgDwellTimeHours,
    AVG(ff.IdleTimeMinutes) AS AvgIdleTimeMinutes,
    AVG(ff.ActualDurationMinutes / NULLIF(ff.PlannedDurationMinutes, 0)) AS EfficiencyRatio,
    COUNT(DISTINCT fw.OperationKey) AS WarehouseOperations,
    COUNT(DISTINCT ff.TripKey) AS FleetTrips
FROM FactWarehouseOperations fw
INNER JOIN DimTime dt ON fw.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fw.GeographyKey = dg.GeographyKey
INNER JOIN DimProduct dp ON fw.ProductKey = dp.ProductKey
LEFT JOIN FactFleetTrips ff ON dt.Date = (SELECT Date FROM DimTime WHERE TimeKey = ff.TimeKey)
    AND dg.GeographyKey IN (ff.OriginGeographyKey, ff.DestinationGeographyKey)
GROUP BY dt.Date, dg.LocationName, dp.Category
GO

-- Stored procedure: Calculate gravity score for products
CREATE PROCEDURE sp_UpdateProductGravityScores AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity from warehouse operations
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS OperationCount,
            AVG(DurationMinutes) AS AvgHandlingTime
        FROM FactWarehouseOperations
        WHERE OperationStartTime >= DATEADD(MONTH, -3, GETDATE())
        GROUP BY ProductKey
    )
    UPDATE dp
    SET 
        GravityScore = CASE 
            WHEN pv.OperationCount IS NULL THEN 1.0
            ELSE (pv.OperationCount * dp.UnitValue) / 
                 (CASE WHEN dp.IsFragile = 1 THEN 2.0 ELSE 1.0 END * 
                  CASE WHEN dp.IsPerishable = 1 THEN 1.5 ELSE 1.0 END)
        END,
        GravityZone = CASE
            WHEN (pv.OperationCount * dp.UnitValue) / 
                 (CASE WHEN dp.IsFragile = 1 THEN 2.0 ELSE 1.0 END) > 50000 THEN 'HighGravity'
            WHEN (pv.OperationCount * dp.UnitValue) / 
                 (CASE WHEN dp.IsFragile = 1 THEN 2.0 ELSE 1.0 END) > 10000 THEN 'MediumGravity'
            ELSE 'LowGravity'
        END
    FROM DimProduct dp
    LEFT JOIN ProductVelocity pv ON dp.ProductKey = pv.ProductKey
END
GO

-- Stored procedure: Predictive bottleneck detection
CREATE PROCEDURE sp_DetectBottlenecks 
    @ThresholdMultiplier DECIMAL(5,2) = 1.5
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Detect warehouse operations exceeding average duration
    SELECT 
        'Warehouse' AS BottleneckType,
        dg.LocationName,
        dp.SKU,
        fw.OperationType,
        AVG(fw.DurationMinutes) AS CurrentAvgDuration,
        (SELECT AVG(DurationMinutes) 
         FROM FactWarehouseOperations 
         WHERE ProductKey = fw.ProductKey 
           AND OperationType = fw.OperationType) AS HistoricalAvgDuration,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations fw
    INNER JOIN DimTime dt ON fw.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fw.GeographyKey = dg.GeographyKey
    INNER JOIN DimProduct dp ON fw.ProductKey = dp.ProductKey
    WHERE dt.DateTime >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY dg.LocationName, dp.SKU, fw.OperationType, fw.ProductKey
    HAVING AVG(fw.DurationMinutes) > @ThresholdMultiplier * 
        (SELECT AVG(DurationMinutes) 
         FROM FactWarehouseOperations 
         WHERE ProductKey = fw.ProductKey 
           AND OperationType = fw.OperationType)
    
    UNION ALL
    
    -- Detect fleet trips with excessive idle time
    SELECT 
        'Fleet' AS BottleneckType,
        dg.LocationName AS Location,
        df.VehicleID AS Identifier,
        'Idle Time' AS Activity,
        AVG(ff.IdleTimeMinutes) AS CurrentMetric,
        (SELECT AVG(IdleTimeMinutes) FROM FactFleetTrips WHERE FleetKey = ff.FleetKey) AS HistoricalMetric,
        COUNT(*) AS EventCount
    FROM FactFleetTrips ff
    INNER JOIN DimTime dt ON ff.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON ff.OriginGeographyKey = dg.GeographyKey
    INNER JOIN DimFleet df ON ff.FleetKey = df.FleetKey
    WHERE dt.DateTime >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY dg.LocationName, df.VehicleID, ff.FleetKey
    HAVING AVG(ff.IdleTimeMinutes) > @ThresholdMultiplier * 
        (SELECT AVG(IdleTimeMinutes) FROM FactFleetTrips WHERE FleetKey = ff.FleetKey)
    
    ORDER BY CurrentAvgDuration DESC
END
GO
```

### Step 3: Configure Power BI Template

1. Open the provided `.pbit` template file in Power BI Desktop
2. When prompted, enter your SQL Server connection details:
   - Server: `your-server-name.database.windows.net` or `localhost\SQLEXPRESS`
   - Database: `LogiFleetPulse`
3. Choose authentication method (Windows or SQL Server)
4. The template will auto-detect relationships based on foreign keys

## Key Configuration Patterns

### Environment Variables for Connection Strings

```sql
-- Use SQL Server environment variables or Azure Key Vault
-- Example: Parameterized connection in Power BI
-- Data Source Properties > Advanced Options > Connection String:
-- Data Source=$(SQL_SERVER);Initial Catalog=$(SQL_DATABASE);Integrated Security=SSPI;

-- For external API integrations, use stored procedures with environment lookup:
CREATE PROCEDURE sp_GetWeatherData
    @Location VARCHAR(100),
    @Date DATE
AS
BEGIN
    DECLARE @ApiKey VARCHAR(255)
    
    -- Retrieve from secure table or environment
    SELECT @ApiKey = ConfigValue 
    FROM SystemConfig 
    WHERE ConfigKey = 'WEATHER_API_KEY'
    
    -- Use @ApiKey in external API call (via CLR or linked server)
END
GO
```

### Data Ingestion Pattern (Incremental Load)

```sql
-- Incremental ETL from source systems
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge from staging table (populated by SSIS, Azure Data Factory, or custom ETL)
    MERGE FactWarehouseOperations AS target
    USING (
        SELECT 
            t.TimeKey,
            g.GeographyKey,
            p.ProductKey,
            stg.OperationType,
            stg.OperationStartTime,
            stg.OperationEndTime,
            DATEDIFF(MINUTE, stg.OperationStartTime, stg.OperationEndTime) AS DurationMinutes,
            stg.DwellTimeHours,
            stg.QuantityHandled,
            stg.ErrorCount,
            stg.EmployeeID,
            stg.StorageZone
        FROM Staging_WarehouseOperations stg
        INNER JOIN DimTime t ON CAST(stg.OperationStartTime AS DATE) = t.Date 
            AND DATEPART(HOUR, stg.OperationStartTime) = t.Hour
            AND (DATEPART(MINUTE, stg.OperationStartTime) / 15) * 15 = t.MinuteBucket
        INNER JOIN DimGeography g ON stg.LocationID = g.LocationID
        INNER JOIN DimProduct p ON stg.SKU = p.SKU
        WHERE stg.LastModified > @LastLoadTimestamp
    ) AS source
    ON 1=0 -- Always insert (append-only fact table)
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, GeographyKey, ProductKey, OperationType, OperationStartTime, 
                OperationEndTime, DurationMinutes, DwellTimeHours, QuantityHandled, 
                ErrorCount, EmployeeID, StorageZone)
        VALUES (source.TimeKey, source.GeographyKey, source.ProductKey, source.OperationType,
                source.OperationStartTime, source.OperationEndTime, source.DurationMinutes,
                source.DwellTimeHours, source.QuantityHandled, source.ErrorCount,
                source.EmployeeID, source.StorageZone);
    
    -- Update last load timestamp
    UPDATE SystemConfig 
    SET ConfigValue = CONVERT(VARCHAR(50), GETDATE(), 126)
    WHERE ConfigKey = 'LAST_WH_LOAD_TIMESTAMP'
END
GO
```

## Common Usage Patterns

### Pattern 1: Cross-Fact Analysis (Warehouse + Fleet)

```sql
-- Find products with high dwell time that also cause fleet delays
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityZone,
    AVG(fw.DwellTimeHours) AS AvgDwellTime,
    AVG(ff.DelayMinutes) AS AvgFleetDelay,
    COUNT(DISTINCT fw.OperationKey) AS WarehouseOps,
    COUNT(DISTINCT ff.TripKey) AS FleetTrips
FROM FactWarehouseOperations fw
INNER JOIN DimProduct dp ON fw.ProductKey = dp.ProductKey
INNER JOIN DimTime dt ON fw.TimeKey = dt.TimeKey
INNER JOIN FactFleetTrips ff ON dt.Date = (SELECT Date FROM DimTime WHERE TimeKey = ff.TimeKey)
LEFT JOIN (
    -- Link products to trips via cross-dock or similar bridge table
    SELECT DISTINCT ProductKey, TripKey 
    FROM FactCrossDock fcd
    INNER JOIN FactFleetTrips fft ON fcd.OutboundTripKey = fft.TripKey
) bridge ON dp.ProductKey = bridge.ProductKey AND ff.TripKey = bridge.TripKey
WHERE fw.DwellTimeHours > 48
  AND ff.DelayMinutes > 30
  AND dt.Date >= DATEADD(MONTH, -1, GETDATE())
GROUP BY dp.SKU, dp.ProductName, dp.GravityZone
HAVING COUNT(DISTINCT fw.OperationKey) > 5
ORDER BY AvgDwellTime DESC, AvgFleetDelay DESC
```

### Pattern 2: Temporal Elasticity Simulation

```sql
-- Simulate impact of increased warehouse capacity utilization
WITH BaselineMetrics AS (
    SELECT 
        AVG(DurationMinutes) AS BaselineAvgDuration,
        AVG(DwellTimeHours) AS BaselineAvgDwell,
        COUNT(*) AS BaselineOperationCount
    FROM FactWarehouseOperations
    WHERE TimeKey IN (
        SELECT TimeKey FROM DimTime 
        WHERE Date >= DATEADD(MONTH, -3, GETDATE())
    )
),
SimulatedLoad AS (
    SELECT 
        GeographyKey,
        ProductKey,
        DurationMinutes * 1.25 AS ProjectedDuration, -- 25% capacity increase
        DwellTimeHours * 1.15 AS ProjectedDwell
    FROM FactWarehouseOperations
    WHERE TimeKey IN (
        SELECT TimeKey FROM DimTime 
        WHERE Date >= DATEADD(WEEK, -1, GETDATE())
    )
)
SELECT 
    'Baseline' AS Scenario,
    bm.BaselineAvgDuration AS AvgDuration,
    bm.BaselineAvgDwell AS AvgDwell,
    bm.BaselineOperationCount AS OperationCount
FROM BaselineMetrics bm

UNION ALL

SELECT 
    'Simulated +25% Capacity' AS Scenario,
    AVG(sl.ProjectedDuration) AS AvgDuration,
    AVG(sl.ProjectedDwell) AS AvgDwell,
    COUNT(*) AS OperationCount
FROM SimulatedLoad sl
```

### Pattern 3: Adaptive Fleet Triage Scoring

```sql
-- Priority queue for fleet maintenance based on revenue impact
WITH FleetRisk AS (
    SELECT 
        df.VehicleID,
        df.VehicleType,
        df.NextMaintenanceDue,
        DATEDIFF(DAY, GETDATE(), df.NextMaintenanceDue) AS DaysUntilMaintenance,
        AVG(ff.LoadWeightKG * dp.UnitValue) AS AvgCargoValue,
        SUM(ff.DelayMinutes) AS TotalDelayMinutes,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ff
    INNER JOIN DimFleet df ON ff.FleetKey = df.FleetKey
    LEFT JOIN FactCrossDock fcd ON ff.TripKey = fcd.OutboundTripKey
    LEFT JOIN DimProduct dp ON fcd.ProductKey = dp.ProductKey
    WHERE ff.TripStartTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY df.VehicleID, df.VehicleType, df.NextMaintenanceDue
)
SELECT 
    VehicleID,
    VehicleType,
    DaysUntilMaintenance,
    AvgCargoValue,
    TotalDelayMinutes,
    -- Priority score: higher cargo value + closer to maintenance + more delays = higher priority
    (AvgCargoValue / 1000.0) * 
    (1.0 / NULLIF(DaysUntilMaintenance, 0)) * 
    (TotalDelayMinutes / 60.0) AS PriorityScore
FROM FleetRisk
WHERE DaysUntilMaintenance <= 30 OR TotalDelayMinutes > 120
ORDER BY PriorityScore DESC
```

## Power BI DAX Measures

### Custom Measures for Dashboard

```dax
// Measure: Fleet Efficiency Ratio
Fleet Efficiency % = 
DIVIDE(
    AVERAGE(FactFleetTrips[ActualDurationMinutes]),
    AVERAGE(FactFleetTrips[PlannedDurationMinutes]),
    1
) * 100

// Measure: Warehouse Throughput (operations per hour)
Warehouse Throughput = 
DIVIDE(
    COUNTROWS(FactWarehouseOperations),
    DISTINCTCOUNT(DimTime[Hour])
)

// Measure: Cross-Modal Dwell Time Impact
Dwell-to-Delay Correlation = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgDelay = AVERAGE(FactFleetTrips[DelayMinutes])
RETURN
    AvgDwell * 0.4 + AvgDelay * 0.6 // Weighted composite score

// Measure: Gravity Zone Compliance %
Gravity Compliance % = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProduct[GravityZone] = "HighGravity",
        FactWarehouseOperations[StorageZone] IN {"A1", "A2", "B1"} // Near dock zones
    ),
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProduct[GravityZone] = "HighGravity"
    )
) * 100

// Measure: Predictive Bottleneck Index (0-100)
Bottleneck Index = 
VAR CurrentMetrics = 
    SUMMARIZE(
        FactWarehouseOperations,
        DimProduct[ProductKey],
        "CurrentAvg", AVERAGE(FactWarehouseOperations[DurationMinutes])
    )
VAR HistoricalBaseline = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DurationMinutes]),
        DATEADD(DimTime[Date], -90, DAY)
    )
RETURN
    DIVIDE([CurrentAvg] - HistoricalBaseline, HistoricalBaseline, 0) * 100
```

## Troubleshooting

### Issue: Slow Query Performance on Large Fact Tables

**Solution**: Implement columnstore indexes for analytical queries

```sql
-- Create columnstore index on fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, OperationType, 
    DurationMinutes, DwellTimeHours, QuantityHandled
)
GO

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON FactFleetTrips (
    TimeKey, FleetKey, OriginGeographyKey, DestinationGeographyKey,
    ActualDurationMinutes, IdleTimeMinutes, FuelConsumedLiters, DelayMinutes
)
GO

-- Partition large tables by date range
ALTER TABLE FactWarehouseOperations 
SWITCH PARTITION 1 TO FactWarehouseOperations_Archive PARTITION 1
```

### Issue: Power BI Refresh Timeout

**Solution**: Implement incremental refresh policy

```powerquery
// In Power BI Query Editor, add parameters
let
    Source = Sql.Database(#"SQL_SERVER", #"SQL_DATABASE"),
    FilteredRows = Table.SelectRows(Source, each [OperationStartTime] >= RangeStart and [OperationStartTime] < RangeEnd)
in
    FilteredRows
```

Then configure incremental refresh in Power BI:
- Store last 2 years, refresh last 7 days
- Detect data changes using `OperationStartTime` column

### Issue: Dimension Table Slowly Changing Dimensions (SCD Type 2)

**Solution**: Implement versioning for product attributes

```sql
ALTER TABLE DimProduct
ADD EffectiveDate DATE DEFAULT GETDATE(),
    ExpirationDate DATE DEFAULT '9999-12-31',
    IsCurrent BIT DEFAULT 1
GO

-- Procedure to handle SCD Type 2 updates
CREATE PROCEDURE sp_UpdateProductSCD
    @SKU VARCHAR(100),
    @NewGravityScore DECIMAL(5,2),
    @NewGravityZone VARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Expire current version
    UPDATE DimProduct
    SET ExpirationDate = GETDATE(),
        IsCurrent = 0
    WHERE SKU = @SKU AND IsCurrent = 1
    
    -- Insert new version
    INSERT INTO DimProduct (SKU, ProductName, Category, Subcategory, WeightKG, 
                            VolumeM3, IsFragile, IsPerishable, RequiresRefrigeration,
                            GravityScore, GravityZone, UnitValue, EffectiveDate, IsCurrent)
    SELECT SKU, ProductName, Category, Subcategory, WeightKG,
           VolumeM3, IsFragile, IsPerishable, RequiresRefrigeration,
           @NewGravityScore, @NewGravityZone, UnitValue, GETDATE(), 1
    FROM DimProduct
    WHERE SKU = @SKU AND IsCurrent = 0
      AND ExpirationDate = CAST(GETDATE() AS DATE)
END
GO
```

### Issue: Cross-Fact Relationships Not Working in Power BI

**Solution**: Use bridge tables for many-to-many relationships

```sql
-- Create bridge table
CREATE TABLE BridgeProductTrip (
    ProductKey INT NOT NULL,
    TripKey BIGINT NOT NULL,
    QuantityLoaded INT,
    LoadSequence INT,
    PRIMARY KEY (ProductKey, TripKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey)
)
GO

-- Populate from cross-dock data
INSERT INTO BridgeProductTrip (ProductKey, TripKey, QuantityLoaded)
SELECT 
    fcd.ProductKey,
    fcd.OutboundTripKey,
    fcd.QuantityUnits
FROM FactCrossDock fcd
WHERE fcd.OutboundTripKey IS NOT NULL
GO
```

In Power BI, set the bridge table cardinality to "Many-to-Many" and hide it from report view.

## Alert Configuration Example

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(255) NOT NULL,
