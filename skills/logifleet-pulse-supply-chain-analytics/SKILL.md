---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing platform for logistics, fleet management, and supply chain analytics with multi-fact star schema modeling
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet logistics dashboard"
  - "implement multi-fact star schema for logistics"
  - "create Power BI supply chain visualization"
  - "deploy SQL Server logistics data warehouse"
  - "build warehouse gravity zone analytics"
  - "set up real-time fleet tracking dashboard"
  - "configure cross-modal supply chain KPI tracking"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and business intelligence platform for logistics and supply chain management. It combines:

- **Multi-fact star schema** data model for warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time operational insights
- **Power BI dashboards** for visualizing KPIs across warehouse velocity, fleet performance, and inventory aging
- **Predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Warehouse Gravity Zones™** — spatial optimization based on pick frequency, item value, and fragility
- **Cross-fact KPI harmonization** linking inventory turnover to fuel consumption and route optimization

The platform is designed for 3PL operators, retail chains, food distributors, and any organization managing complex warehouse-fleet ecosystems.

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, ERP systems

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

Execute the SQL schema scripts in this order:

```sql
-- 1. Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- 2. Run dimension table creation script
-- (assumes DDL_Dimensions.sql exists in repo)
:r DDL_Dimensions.sql

-- 3. Run fact table creation script
:r DDL_Facts.sql

-- 4. Create relationships and indexes
:r DDL_Relationships.sql

-- 5. Deploy stored procedures
:r SP_IncrementalLoad.sql
:r SP_AlertingEngine.sql
```

### Step 3: Configure Data Connections

Update connection strings in the configuration file:

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SQL",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telematics": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}"
    }
  },
  "refresh": {
    "intervalMinutes": 15
  }
}
```

### Step 4: Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection parameters when prompted
3. Configure row-level security based on user roles
4. Publish to Power BI Service for sharing

## Core Data Model Structure

### Dimension Tables

**DimTime** — Temporal dimension with 15-minute granularity

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayName VARCHAR(20),
    WeekOfYear TINYINT,
    MonthNumber TINYINT,
    MonthName VARCHAR(20),
    Quarter TINYINT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- Create clustered index for performance
CREATE CLUSTERED INDEX IX_DimTime_TimeKey ON DimTime(TimeKey);
```

**DimGeography** — Hierarchical location dimension

```sql
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Distribution Center'
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    WarehouseCapacitySqFt INT,
    ActiveFlag BIT DEFAULT 1
);

CREATE INDEX IX_DimGeography_LocationID ON DimGeography(LocationID);
```

**DimProductGravity** — Product classification with gravity scoring

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    IsPerishable BIT,
    ShelfLifeDays INT,
    StandardCost DECIMAL(18,2),
    GravityScore DECIMAL(5,2), -- Calculated: combines velocity, value, fragility
    RecommendedZone VARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    LastUpdated DATETIME DEFAULT GETDATE()
);

CREATE INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU);
CREATE INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC);
```

**DimSupplierReliability** — Supplier performance metrics

```sql
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    Country VARCHAR(100),
    AvgLeadTimeDays DECIMAL(5,1),
    LeadTimeVarianceDays DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100 scale
    ReliabilityRating VARCHAR(20), -- 'Excellent', 'Good', 'Fair', 'Poor'
    LastAuditDate DATE
);
```

### Fact Tables

**FactWarehouseOperations** — Warehouse activity grain: operation event

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderID VARCHAR(100),
    BatchID VARCHAR(100),
    QuantityHandled INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    StorageZone VARCHAR(50),
    ActualZone VARCHAR(50),
    RecommendedZone VARCHAR(50),
    ZoneMismatchFlag BIT COMPUTED (CASE WHEN ActualZone <> RecommendedZone THEN 1 ELSE 0 END),
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    ErrorFlag BIT DEFAULT 0,
    ErrorReason VARCHAR(500)
);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeKey ON FactWarehouseOperations(TimeKey) INCLUDE (OperationType, QuantityHandled);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_ProductKey ON FactWarehouseOperations(ProductKey);
```

**FactFleetTrips** — Fleet activity grain: trip segment

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripID VARCHAR(100) NOT NULL,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    TirePressureAvg DECIMAL(5,2),
    EngineHealthScore DECIMAL(5,2), -- 0-100
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT,
    OnTimeFlag BIT,
    DeliveryPriority VARCHAR(20), -- 'Standard', 'Expedited', 'Critical'
    RevenueImpactScore DECIMAL(10,2) -- Calculated based on load value and priority
);

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeKey ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_VehicleID ON FactFleetTrips(VehicleID);
```

**FactCrossDock** — Cross-dock transfers grain: transfer event

```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripID VARCHAR(100),
    OutboundTripID VARCHAR(100),
    TransferTimeMinutes INT,
    QuantityTransferred INT,
    TemperatureCompliant BIT,
    TemperatureActual DECIMAL(5,2),
    TemperatureThresholdMin DECIMAL(5,2),
    TemperatureThresholdMax DECIMAL(5,2)
);
```

## Key Analytical Queries

### Calculate Warehouse Gravity Zone Efficiency

```sql
-- Identify SKUs in suboptimal zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZone,
    w.ActualZone,
    COUNT(*) AS MisplacedOperations,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    SUM(w.ProcessingTimeMinutes) AS TotalProcessingTime
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE w.ZoneMismatchFlag = 1
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone, w.ActualZone
HAVING COUNT(*) > 10
ORDER BY TotalProcessingTime DESC;
```

### Fleet Idle Time Analysis by Route

```sql
-- Identify routes with excessive idle time
SELECT 
    og.LocationName AS OriginLocation,
    dg.LocationName AS DestinationLocation,
    COUNT(*) AS TripCount,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    AVG(f.FuelConsumedLiters) AS AvgFuelConsumed,
    AVG(f.DistanceKm) AS AvgDistance,
    AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF(f.DurationMinutes, 0) * 100) AS IdleTimePercentage
FROM FactFleetTrips f
INNER JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
INNER JOIN DimGeography dg ON f.DestinationGeographyKey = dg.GeographyKey
WHERE f.TimeKey >= (SELECT TOP 1 TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
GROUP BY og.LocationName, dg.LocationName
HAVING AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF(f.DurationMinutes, 0) * 100) > 15
ORDER BY IdleTimePercentage DESC;
```

### Cross-Fact KPI: Dwell Time vs Fleet Cost

```sql
-- Correlate warehouse dwell time with delivery delays
WITH WarehouseDwell AS (
    SELECT 
        w.OrderID,
        p.ProductKey,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        MAX(CASE WHEN w.DwellTimeMinutes > 72*60 THEN 1 ELSE 0 END) AS HighDwellFlag
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'Shipping'
    GROUP BY w.OrderID, p.ProductKey
),
FleetPerformance AS (
    SELECT 
        f.TripID,
        f.OnTimeFlag,
        f.TrafficDelayMinutes,
        f.FuelConsumedLiters
    FROM FactFleetTrips f
)
SELECT 
    wd.ProductKey,
    p.SKU,
    p.ProductName,
    COUNT(*) AS ShipmentCount,
    AVG(wd.AvgDwellTime) AS AvgWarehouseDwell,
    SUM(wd.HighDwellFlag) AS HighDwellCount,
    AVG(CAST(fp.OnTimeFlag AS FLOAT)) * 100 AS OnTimePercentage,
    AVG(fp.FuelConsumedLiters) AS AvgFuelPerTrip
FROM WarehouseDwell wd
INNER JOIN DimProductGravity p ON wd.ProductKey = p.ProductKey
INNER JOIN FleetPerformance fp ON 1=1 -- Simplified join; actual would link via order/shipment bridge table
GROUP BY wd.ProductKey, p.SKU, p.ProductName
ORDER BY AvgWarehouseDwell DESC;
```

### Predictive Bottleneck Index

```sql
-- Calculate bottleneck probability score
SELECT 
    g.LocationName AS WarehouseLocation,
    t.DayName,
    t.HourOfDay,
    COUNT(*) AS OperationCount,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    STDEV(w.DwellTimeMinutes) AS DwellTimeVariance,
    -- Bottleneck Index: high volume + high variance + high dwell
    (COUNT(*) * AVG(w.DwellTimeMinutes) * STDEV(w.DwellTimeMinutes)) / 10000.0 AS BottleneckIndex
FROM FactWarehouseOperations w
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.TimeKey >= (SELECT TOP 1 TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -90, GETDATE()))
GROUP BY g.LocationName, t.DayName, t.HourOfDay
HAVING COUNT(*) > 50
ORDER BY BottleneckIndex DESC;
```

## Stored Procedures

### Incremental Data Load

```sql
CREATE PROCEDURE SP_IncrementalLoad_WarehouseOperations
    @LastLoadTimeKey INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operations since last load
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        GeographyKey,
        ProductKey,
        SupplierKey,
        OperationType,
        OrderID,
        QuantityHandled,
        DwellTimeMinutes,
        ProcessingTimeMinutes,
        StorageZone,
        ActualZone,
        RecommendedZone,
        OperatorID,
        ErrorFlag
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        src.OperationType,
        src.OrderID,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTime,
        src.ProcessingMinutes,
        src.Zone,
        src.Zone AS ActualZone,
        p.RecommendedZone,
        src.OperatorID,
        src.ErrorFlag
    FROM StagingWarehouseOperations src -- External staging table from WMS
    INNER JOIN DimTime t ON CAST(src.EventDateTime AS DATE) = CAST(t.FullDateTime AS DATE) 
        AND DATEPART(HOUR, src.EventDateTime) = t.HourOfDay
        AND DATEPART(MINUTE, src.EventDateTime) / 15 * 15 = DATEPART(MINUTE, t.TimeOfDay)
    INNER JOIN DimGeography g ON src.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON src.SupplierID = s.SupplierID
    WHERE t.TimeKey > @LastLoadTimeKey;
    
    -- Update last load timestamp
    UPDATE ETLControl SET LastLoadTimeKey = (SELECT MAX(TimeKey) FROM DimTime WHERE FullDateTime <= GETDATE());
    
END;
GO
```

### Automated Alerting Engine

```sql
CREATE PROCEDURE SP_GenerateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentTimeKey INT = (SELECT TOP 1 TimeKey FROM DimTime WHERE FullDateTime <= GETDATE() ORDER BY TimeKey DESC);
    
    -- Alert 1: Fleet idling > 15% of trip duration
    INSERT INTO AlertLog (AlertType, Severity, EntityID, Message, GeneratedAt)
    SELECT 
        'FleetIdling' AS AlertType,
        'Warning' AS Severity,
        VehicleID AS EntityID,
        'Vehicle ' + VehicleID + ' idling time is ' + CAST(AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0) * 100) AS VARCHAR) + '% over last 24 hours' AS Message,
        GETDATE()
    FROM FactFleetTrips
    WHERE TimeKey >= @CurrentTimeKey - 96 -- Last 24 hours at 15-min intervals
    GROUP BY VehicleID
    HAVING AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0) * 100) > 15;
    
    -- Alert 2: High dwell time in warehouse
    INSERT INTO AlertLog (AlertType, Severity, EntityID, Message, GeneratedAt)
    SELECT 
        'HighDwellTime' AS AlertType,
        'Critical' AS Severity,
        p.SKU AS EntityID,
        'SKU ' + p.SKU + ' has average dwell time of ' + CAST(AVG(w.DwellTimeMinutes) AS VARCHAR) + ' minutes' AS Message,
        GETDATE()
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.TimeKey >= @CurrentTimeKey - 288 -- Last 72 hours
        AND p.IsPerishable = 1
    GROUP BY p.SKU
    HAVING AVG(w.DwellTimeMinutes) > 72 * 60;
    
    -- Alert 3: Temperature non-compliance in cross-dock
    INSERT INTO AlertLog (AlertType, Severity, EntityID, Message, GeneratedAt)
    SELECT 
        'TemperatureViolation' AS AlertType,
        'Critical' AS Severity,
        p.SKU AS EntityID,
        'SKU ' + p.SKU + ' had temperature non-compliance at ' + g.LocationName AS Message,
        GETDATE()
    FROM FactCrossDock cd
    INNER JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON cd.GeographyKey = g.GeographyKey
    WHERE cd.TimeKey >= @CurrentTimeKey - 96
        AND cd.TemperatureCompliant = 0;
        
END;
GO
```

## Power BI DAX Measures

### Warehouse Utilization Percentage

```dax
Warehouse Utilization % = 
VAR TotalCapacity = SUM(DimGeography[WarehouseCapacitySqFt])
VAR UsedSpace = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[QuantityHandled] * RELATED(DimProductGravity[UnitVolume])
    )
RETURN
DIVIDE(UsedSpace, TotalCapacity, 0) * 100
```

### Fleet Efficiency Score

```dax
Fleet Efficiency Score = 
VAR AvgIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR AvgFuelPerKm = DIVIDE(AVERAGE(FactFleetTrips[FuelConsumedLiters]), AVERAGE(FactFleetTrips[DistanceKm]))
VAR OnTimeRate = DIVIDE(COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[OnTimeFlag] = TRUE())), COUNTROWS(FactFleetTrips))
VAR HealthScore = AVERAGE(FactFleetTrips[EngineHealthScore])

RETURN
(OnTimeRate * 40) + ((100 - AvgIdleTime) * 0.3) + (HealthScore * 0.3)
```

### Gravity Zone Compliance Rate

```dax
Gravity Zone Compliance % = 
DIVIDE(
    COUNTROWS(FILTER(FactWarehouseOperations, FactWarehouseOperations[ZoneMismatchFlag] = 0)),
    COUNTROWS(FactWarehouseOperations),
    0
) * 100
```

### Revenue Impact of Delays

```dax
Delayed Shipment Revenue Impact = 
SUMX(
    FILTER(FactFleetTrips, FactFleetTrips[OnTimeFlag] = FALSE()),
    FactFleetTrips[RevenueImpactScore] * FactFleetTrips[TrafficDelayMinutes] * 0.01
)
```

## Configuration Best Practices

### SQL Server Indexing Strategy

```sql
-- Columnstore index for analytical queries on large fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse_Analytics
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, QuantityHandled, DwellTimeMinutes, ProcessingTimeMinutes);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleet_Analytics
ON FactFleetTrips (TimeKey, VehicleID, DistanceKm, FuelConsumedLiters, IdleTimeMinutes);

-- Partition fact tables by date for performance
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT PK_FactWarehouse_Partitioned PRIMARY KEY (OperationKey, TimeKey);

-- Create partition function (yearly partitions)
CREATE PARTITION FUNCTION PF_YearlyPartition (INT)
AS RANGE RIGHT FOR VALUES (20250101, 20260101, 20270101);

CREATE PARTITION SCHEME PS_YearlyPartition
AS PARTITION PF_YearlyPartition
ALL TO ([PRIMARY]);
```

### Power BI Refresh Schedule

Set up incremental refresh in Power BI:

1. Create parameters in Power BI Desktop:
   - `RangeStart` (Date/Time)
   - `RangeEnd` (Date/Time)

2. Filter fact tables using parameters:

```powerquery
= Table.SelectRows(
    FactWarehouseOperations,
    each [EventDateTime] >= RangeStart and [EventDateTime] < RangeEnd
)
```

3. Configure incremental refresh policy:
   - Refresh last 7 days
   - Store 2 years of historical data
   - Detect data changes using `LastUpdated` column

### Row-Level Security

```dax
-- RLS filter for regional managers (applies to all tables through relationships)
[Region] = USERNAME()

-- RLS filter for warehouse supervisors
[WarehouseLocation] IN (
    LOOKUPVALUE(
        UserWarehouseAccess[WarehouseLocation],
        UserWarehouseAccess[UserEmail],
        USERNAME()
    )
)
```

## Common Patterns

### Pattern 1: Time Intelligence with 15-Minute Buckets

```dax
-- Current period vs previous period (15-min intervals)
Current 15-Min Operations = 
CALCULATE(
    COUNT(FactWarehouseOperations[OperationKey]),
    FILTER(DimTime, DimTime[TimeKey] = MAX(DimTime[TimeKey]))
)

Previous 15-Min Operations = 
CALCULATE(
    COUNT(FactWarehouseOperations[OperationKey]),
    FILTER(DimTime, DimTime[TimeKey] = MAX(DimTime[TimeKey]) - 1)
)

Operations Change % = 
DIVIDE(
    [Current 15-Min Operations] - [Previous 15-Min Operations],
    [Previous 15-Min Operations],
    0
) * 100
```

### Pattern 2: Bridge Table for Many-to-Many

```sql
-- Bridge table linking routes to storage zones
CREATE TABLE BridgeRouteZone (
    RouteID VARCHAR(100),
    StorageZone VARCHAR(50),
    AllocationPercentage DECIMAL(5,2)
);

-- Query using bridge table
SELECT 
    f.TripID,
    b.StorageZone,
    SUM(w.QuantityHandled * b.AllocationPercentage / 100) AS AllocatedQuantity
FROM FactFleetTrips f
INNER JOIN BridgeRouteZone b ON f.TripID LIKE b.RouteID + '%'
INNER JOIN FactWarehouseOperations w ON w.StorageZone = b.StorageZone
GROUP BY f.TripID, b.StorageZone;
```

### Pattern 3: Slowly Changing Dimension (Type 2)

```sql
-- Track historical changes to product gravity scores
ALTER TABLE DimProductGravity ADD 
    EffectiveStartDate DATE DEFAULT '2025-01-01',
    EffectiveEndDate DATE DEFAULT '9999-12-31',
    IsCurrent BIT DEFAULT 1;

-- Update procedure for SCD Type 2
CREATE PROCEDURE SP_UpdateProductGravity
    @SKU VARCHAR(100),
    @NewGravityScore DECIMAL(5,2),
    @NewRecommendedZone VARCHAR(50)
AS
BEGIN
    -- Expire current record
    UPDATE DimProductGravity
    SET EffectiveEndDate = GETDATE(),
        IsCurrent = 0
    WHERE SKU = @SKU AND IsCurrent = 1;
    
    -- Insert new record
    INSERT INTO DimProductGravity (SKU, ProductName, GravityScore, RecommendedZone, EffectiveStartDate, IsCurrent)
    SELECT SKU, ProductName, @NewGravityScore, @NewRecommendedZone, GETDATE(), 1
    FROM DimProductGravity
    WHERE SKU = @SKU AND EffectiveEndDate = GETDATE();
END;
```

## Troubleshooting

### Issue: Power BI Refresh Times Out

**Solution**: Implement incremental refresh or partition fact tables by date range.

```sql
-- Create indexed view for pre-aggregated metrics
CREATE VIEW vw_DailyWarehouseSummary
WITH SCHEMABINDING
AS
SELECT 
    t.DateKey,
    w.GeographyKey,
    w.ProductKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(w.QuantityHandled) AS TotalQuantity,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.DimTime t ON w.TimeKey = t.TimeKey
GROUP BY t.DateKey, w.GeographyKey, w.ProductKey;

CREATE UNIQUE CLUSTERED INDEX IX_DailyWarehouseSummary 
ON vw_DailyWarehouseSummary (DateKey, GeographyKey, ProductKey);
```

### Issue: Duplicate Records in Fact Tables

**Solution**: Add unique constraints and use MERGE for upserts.

```sql
ALTER TABLE FactFleetTrips ADD CONSTRAINT UQ_TripSegment UNIQUE (TripID, TimeKey, OriginGeographyKey);

-- Use MERGE instead of INSERT
MERGE INTO FactFleetTrips AS target
USING StagingFleetTrips AS source
ON target.TripID = source.TripID AND target.TimeKey = source.TimeKey
WHEN NOT MATCHED THEN
    INSERT (TripID, TimeKey, VehicleID, DistanceKm, FuelConsumedLiters)
    VALUES (source.TripID, source.TimeKey, source.VehicleID, source.DistanceKm, source.FuelConsumedLiters);
```

### Issue: Gravity Score Not Updating Automatically

**Solution**: Create calculated column with trigger.

```sql
CREATE TRIGGER TR_UpdateGravityScore
ON DimProductGravity
AFTER INSERT, UPDATE
AS
BEGIN
    UPDATE p
    SET GravityScore = 
        (ISNULL(velocity.PickFrequency, 0) * 0.4) +
        (ISNULL(p.StandardCost, 0) / 100 * 0.3) +
        (CASE WHEN p.IsFragile = 1 THEN 20 ELSE 0 END) +
        (CASE WHEN p.IsPerishable = 1 THEN 15 ELSE 0 END),
    RecommendedZone = 
        CASE 
            WHEN GravityScore >= 70 THEN 'High-Gravity'
            WHEN GravityScore >= 40
