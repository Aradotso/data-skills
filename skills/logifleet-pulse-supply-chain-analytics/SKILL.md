---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for real-time logistics, fleet, and warehouse intelligence with multi-fact star schema
triggers:
  - "set up logistics analytics dashboard"
  - "create supply chain data warehouse"
  - "build fleet management power bi report"
  - "implement warehouse operations analytics"
  - "configure multi-fact star schema for logistics"
  - "connect power bi to logistics database"
  - "analyze fleet telemetry and warehouse data"
  - "deploy logifleet pulse analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced data warehousing and analytics platform built on MS SQL Server and Power BI for real-time logistics intelligence. It implements a multi-fact star schema that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for cross-modal supply chain orchestration.

## What It Does

- **Multi-Fact Data Modeling**: Implements star schema with FactWarehouseOperations, FactFleetTrips, FactCrossDock, and shared time-phased dimensions
- **Real-Time Dashboards**: Power BI reports refreshed every 15 minutes with KPI harmonization across logistics domains
- **Predictive Analytics**: Bottleneck detection, fleet maintenance triage, warehouse gravity zone optimization
- **Cross-Domain Queries**: Links inventory turnover with fleet fuel efficiency, dwell time with route optimization
- **Role-Based Security**: Row-level security for user, supervisor, and executive views

## Installation

### Prerequisites

- MS SQL Server 2019+ (Enterprise, Standard, or Developer Edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics APIs, ERP systems

### Step 1: Deploy SQL Schema

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create schema for staging and production
CREATE SCHEMA staging
GO
CREATE SCHEMA prod
GO

-- Deploy dimension tables first (order matters for foreign keys)
-- DimTime: 15-minute granularity time dimension
CREATE TABLE prod.DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeValue DATETIME2(0) NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot15Min SMALLINT NOT NULL, -- 0-95 (96 slots per day)
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod VARCHAR(10) NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    CONSTRAINT UQ_DimTime_DateTime UNIQUE (DateTimeValue)
)
GO

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON prod.DimTime(DateTimeValue)
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE prod.DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Route Node, Distribution Center
    ParentGeographyKey INT NULL,
    Region VARCHAR(100) NULL,
    Country VARCHAR(100) NOT NULL,
    Continent VARCHAR(50) NOT NULL,
    Latitude DECIMAL(9,6) NULL,
    Longitude DECIMAL(9,6) NULL,
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    EndDate DATE NULL,
    CONSTRAINT FK_DimGeography_Parent FOREIGN KEY (ParentGeographyKey) 
        REFERENCES prod.DimGeography(GeographyKey)
)
GO

CREATE INDEX IX_DimGeography_LocationID ON prod.DimGeography(LocationID)
GO

-- DimProductGravity: Product dimension with velocity-based gravity score
CREATE TABLE prod.DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName VARCHAR(500) NOT NULL,
    Category VARCHAR(100) NOT NULL,
    SubCategory VARCHAR(100) NULL,
    GravityScore DECIMAL(5,2) NOT NULL DEFAULT 50.0, -- 0-100 scale
    VelocityClass VARCHAR(20) NOT NULL, -- Fast, Medium, Slow
    ValueClass VARCHAR(20) NOT NULL, -- High, Medium, Low
    FragilityScore TINYINT NOT NULL DEFAULT 1, -- 1-10 scale
    IsPerishable BIT DEFAULT 0,
    OptimalStorageZone VARCHAR(50) NULL,
    LastGravityUpdate DATETIME2(0) NOT NULL,
    IsActive BIT DEFAULT 1
)
GO

CREATE UNIQUE INDEX IX_DimProductGravity_SKU ON prod.DimProductGravity(SKU) 
    WHERE IsActive = 1
GO

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE prod.DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200) NOT NULL,
    LeadTimeDaysAvg DECIMAL(6,2) NULL,
    LeadTimeVariance DECIMAL(6,2) NULL,
    DefectRatePercent DECIMAL(5,2) NULL,
    ComplianceScore DECIMAL(5,2) NULL, -- 0-100
    LastEvaluationDate DATE NOT NULL,
    IsActive BIT DEFAULT 1
)
GO

-- FactWarehouseOperations: Core warehouse activity fact table
CREATE TABLE prod.FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME2(0) NOT NULL,
    OperationEndTime DATETIME2(0) NOT NULL,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    Quantity INT NOT NULL,
    UnitsPerHour AS CASE 
        WHEN DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) > 0 
        THEN (Quantity * 60.0) / DATEDIFF(MINUTE, OperationStartTime, OperationEndTime)
        ELSE 0 END PERSISTED,
    DwellTimeMinutes INT NULL, -- Time between ops
    StorageZone VARCHAR(50) NULL,
    OperatorID VARCHAR(50) NULL,
    BatchID VARCHAR(100) NULL,
    ErrorFlag BIT DEFAULT 0,
    ErrorReason VARCHAR(500) NULL,
    CONSTRAINT FK_FactWH_Time FOREIGN KEY (TimeKey) 
        REFERENCES prod.DimTime(TimeKey),
    CONSTRAINT FK_FactWH_Geography FOREIGN KEY (GeographyKey) 
        REFERENCES prod.DimGeography(GeographyKey),
    CONSTRAINT FK_FactWH_Product FOREIGN KEY (ProductKey) 
        REFERENCES prod.DimProductGravity(ProductKey)
)
GO

CREATE CLUSTERED INDEX IX_FactWH_Time ON prod.FactWarehouseOperations(TimeKey, GeographyKey)
GO

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE prod.FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50) NULL,
    TripStartTime DATETIME2(0) NOT NULL,
    TripEndTime DATETIME2(0) NOT NULL,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceKm DECIMAL(10,2) NOT NULL,
    FuelConsumedLiters DECIMAL(10,2) NULL,
    FuelEfficiency AS CASE 
        WHEN FuelConsumedLiters > 0 THEN DistanceKm / FuelConsumedLiters 
        ELSE NULL END PERSISTED,
    IdleTimeMinutes INT NULL,
    LoadingTimeMinutes INT NULL,
    UnloadingTimeMinutes INT NULL,
    AverageSpeedKmh DECIMAL(6,2) NULL,
    WeatherCondition VARCHAR(50) NULL,
    TrafficDelayMinutes INT DEFAULT 0,
    MaintenanceAlert BIT DEFAULT 0,
    MaintenanceAlertReason VARCHAR(500) NULL,
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) 
        REFERENCES prod.DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) 
        REFERENCES prod.DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Dest FOREIGN KEY (DestinationGeographyKey) 
        REFERENCES prod.DimGeography(GeographyKey)
)
GO

CREATE CLUSTERED INDEX IX_FactFleet_Time ON prod.FactFleetTrips(TimeKey, OriginGeographyKey)
GO

-- FactCrossDock: Cross-dock operations linking inbound and outbound
CREATE TABLE prod.FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundTripKey BIGINT NULL,
    OutboundTripKey BIGINT NULL,
    ReceiptTime DATETIME2(0) NOT NULL,
    ShipmentTime DATETIME2(0) NOT NULL,
    CrossDockDurationMinutes AS DATEDIFF(MINUTE, ReceiptTime, ShipmentTime) PERSISTED,
    Quantity INT NOT NULL,
    QualityCheckPassed BIT DEFAULT 1,
    CONSTRAINT FK_FactCD_Time FOREIGN KEY (TimeKey) 
        REFERENCES prod.DimTime(TimeKey),
    CONSTRAINT FK_FactCD_Geography FOREIGN KEY (GeographyKey) 
        REFERENCES prod.DimGeography(GeographyKey),
    CONSTRAINT FK_FactCD_Product FOREIGN KEY (ProductKey) 
        REFERENCES prod.DimProductGravity(ProductKey),
    CONSTRAINT FK_FactCD_Supplier FOREIGN KEY (SupplierKey) 
        REFERENCES prod.DimSupplierReliability(SupplierKey),
    CONSTRAINT FK_FactCD_InboundTrip FOREIGN KEY (InboundTripKey) 
        REFERENCES prod.FactFleetTrips(TripKey),
    CONSTRAINT FK_FactCD_OutboundTrip FOREIGN KEY (OutboundTripKey) 
        REFERENCES prod.FactFleetTrips(TripKey)
)
GO

CREATE CLUSTERED INDEX IX_FactCD_Time ON prod.FactCrossDock(TimeKey, GeographyKey)
GO
```

### Step 2: Populate Dimension Tables

```sql
-- Populate DimTime for current year + 1 year ahead
DECLARE @StartDate DATE = '2026-01-01'
DECLARE @EndDate DATE = '2027-12-31'
DECLARE @CurrentDate DATETIME2(0) = @StartDate

WHILE @CurrentDate <= @EndDate
BEGIN
    DECLARE @Minute INT = 0
    WHILE @Minute < 1440 -- 1440 minutes in a day
    BEGIN
        INSERT INTO prod.DimTime (
            DateTimeValue, DateKey, TimeSlot15Min, HourOfDay, 
            DayOfWeek, DayName, IsWeekend, FiscalPeriod, FiscalYear
        )
        VALUES (
            DATEADD(MINUTE, @Minute, @CurrentDate),
            CONVERT(INT, FORMAT(@CurrentDate, 'yyyyMMdd')),
            @Minute / 15,
            @Minute / 60,
            DATEPART(WEEKDAY, @CurrentDate),
            DATENAME(WEEKDAY, @CurrentDate),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1,7) THEN 1 ELSE 0 END,
            'Q' + CAST(DATEPART(QUARTER, @CurrentDate) AS VARCHAR(1)),
            YEAR(@CurrentDate)
        )
        SET @Minute = @Minute + 15
    END
    SET @CurrentDate = DATEADD(DAY, 1, @CurrentDate)
END
GO

-- Sample geography data (customize for your locations)
INSERT INTO prod.DimGeography (LocationID, LocationName, LocationType, ParentGeographyKey, Region, Country, Continent, Latitude, Longitude, EffectiveDate)
VALUES 
    ('WH-001', 'Central Distribution Center', 'Warehouse', NULL, 'Midwest', 'USA', 'North America', 41.8781, -87.6298, '2026-01-01'),
    ('WH-002', 'East Coast Hub', 'Warehouse', NULL, 'Northeast', 'USA', 'North America', 40.7128, -74.0060, '2026-01-01'),
    ('RN-001', 'Chicago Route Node', 'Route Node', 1, 'Midwest', 'USA', 'North America', 41.8781, -87.6298, '2026-01-01'),
    ('RN-002', 'New York Route Node', 'Route Node', 2, 'Northeast', 'USA', 'North America', 40.7128, -74.0060, '2026-01-01')
GO
```

### Step 3: Create Stored Procedures for Data Loading

```sql
-- Stored procedure to calculate product gravity scores
CREATE PROCEDURE prod.UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on recent warehouse operations
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            COUNT(*) AS PickFrequency,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
            SUM(wo.Quantity) AS TotalVolumeL30D
        FROM prod.DimProductGravity p
        LEFT JOIN prod.FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        WHERE wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
            AND wo.OperationType = 'Picking'
        GROUP BY p.ProductKey, p.SKU
    ),
    ScoreCalculation AS (
        SELECT 
            ProductKey,
            SKU,
            -- Gravity score: weighted combination of frequency, volume, and inverse dwell time
            (
                (PickFrequency * 0.4) + 
                (TotalVolumeL30D / 1000.0 * 0.3) + 
                (CASE WHEN AvgDwellTime > 0 THEN (1000.0 / AvgDwellTime) * 0.3 ELSE 0 END)
            ) AS RawScore
        FROM ProductMetrics
    )
    UPDATE p
    SET 
        GravityScore = CASE 
            WHEN sc.RawScore > 100 THEN 100 
            WHEN sc.RawScore < 0 THEN 0 
            ELSE sc.RawScore 
        END,
        VelocityClass = CASE 
            WHEN sc.RawScore >= 70 THEN 'Fast'
            WHEN sc.RawScore >= 40 THEN 'Medium'
            ELSE 'Slow'
        END,
        LastGravityUpdate = GETDATE()
    FROM prod.DimProductGravity p
    INNER JOIN ScoreCalculation sc ON p.ProductKey = sc.ProductKey
    
    PRINT 'Product gravity scores updated: ' + CAST(@@ROWCOUNT AS VARCHAR(10))
END
GO

-- Stored procedure for incremental fact table loading
CREATE PROCEDURE prod.LoadWarehouseOperations
    @StartDateTime DATETIME2(0),
    @EndDateTime DATETIME2(0)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from staging to production (adjust source as needed)
    INSERT INTO prod.FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, Quantity,
        DwellTimeMinutes, StorageZone, OperatorID, BatchID,
        ErrorFlag, ErrorReason
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OperationStartTime,
        s.OperationEndTime,
        s.Quantity,
        s.DwellTimeMinutes,
        s.StorageZone,
        s.OperatorID,
        s.BatchID,
        s.ErrorFlag,
        s.ErrorReason
    FROM staging.WarehouseOperations s
    INNER JOIN prod.DimTime t ON t.DateTimeValue = DATEADD(MINUTE, 
        (DATEPART(MINUTE, s.OperationStartTime) / 15) * 15, 
        DATEADD(HOUR, DATEDIFF(HOUR, 0, s.OperationStartTime), 0))
    INNER JOIN prod.DimGeography g ON g.LocationID = s.LocationID AND g.IsActive = 1
    INNER JOIN prod.DimProductGravity p ON p.SKU = s.SKU AND p.IsActive = 1
    WHERE s.OperationStartTime >= @StartDateTime 
        AND s.OperationStartTime < @EndDateTime
        AND s.IsProcessed = 0
    
    -- Mark staging records as processed
    UPDATE staging.WarehouseOperations
    SET IsProcessed = 1
    WHERE OperationStartTime >= @StartDateTime 
        AND OperationStartTime < @EndDateTime
    
    PRINT 'Warehouse operations loaded: ' + CAST(@@ROWCOUNT AS VARCHAR(10))
END
GO
```

### Step 4: Configure Power BI Connection

```sql
-- Create a view for Power BI consumption (optimized for cross-fact queries)
CREATE VIEW prod.vw_LogisticsKPIs AS
SELECT 
    -- Time dimensions
    t.DateTimeValue,
    t.DateKey,
    t.HourOfDay,
    t.DayOfWeek,
    t.DayName,
    t.FiscalPeriod,
    
    -- Geography
    g.LocationName,
    g.LocationType,
    g.Region,
    g.Country,
    
    -- Product
    p.SKU,
    p.ProductName,
    p.Category,
    p.GravityScore,
    p.VelocityClass,
    
    -- Warehouse metrics
    wo.OperationType,
    wo.DurationMinutes AS WH_DurationMinutes,
    wo.UnitsPerHour AS WH_UnitsPerHour,
    wo.DwellTimeMinutes,
    wo.StorageZone,
    
    -- Fleet metrics (joined via time and geography)
    ft.VehicleID,
    ft.TripDurationMinutes AS Fleet_TripDurationMinutes,
    ft.DistanceKm,
    ft.FuelEfficiency,
    ft.IdleTimeMinutes,
    ft.TrafficDelayMinutes,
    
    -- Cross-dock metrics
    cd.CrossDockDurationMinutes,
    cd.Quantity AS CD_Quantity,
    
    -- Calculated cross-fact KPIs
    CASE 
        WHEN wo.DwellTimeMinutes > 72 * 60 AND ft.IdleTimeMinutes > 30 
        THEN 1 ELSE 0 
    END AS HighRiskFlag,
    
    CASE 
        WHEN p.GravityScore > 70 AND wo.StorageZone NOT LIKE '%Priority%'
        THEN 1 ELSE 0
    END AS SuboptimalPlacementFlag

FROM prod.FactWarehouseOperations wo
INNER JOIN prod.DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN prod.DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN prod.DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN prod.FactFleetTrips ft ON 
    ft.TimeKey = t.TimeKey AND 
    (ft.OriginGeographyKey = g.GeographyKey OR ft.DestinationGeographyKey = g.GeographyKey)
LEFT JOIN prod.FactCrossDock cd ON 
    cd.TimeKey = t.TimeKey AND 
    cd.GeographyKey = g.GeographyKey AND
    cd.ProductKey = p.ProductKey
GO
```

### Step 5: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template (`.pbit`)
3. Enter connection parameters:
   - SQL Server: `${SQL_SERVER_INSTANCE}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server (use stored credentials, not embedded)
4. The template will auto-load:
   - Pre-built star schema relationships
   - DAX measures for cross-fact KPIs
   - Dashboard pages for warehouse, fleet, and executive views

## Key Configuration

### Connection String for External Data Sources

```sql
-- Configure polybase for external CSV/JSON ingestion (optional)
-- Requires SQL Server Enterprise Edition
CREATE EXTERNAL DATA SOURCE WMS_DataFeed
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${AZURE_BLOB_CONNECTION_STRING}',
    CREDENTIAL = AzureStorageCredential
)
GO

CREATE EXTERNAL FILE FORMAT CSV_Format
WITH (
    FORMAT_TYPE = DELIMITEDTEXT,
    FORMAT_OPTIONS (
        FIELD_TERMINATOR = ',',
        STRING_DELIMITER = '"',
        FIRST_ROW = 2,
        USE_TYPE_DEFAULT = FALSE
    )
)
GO

CREATE EXTERNAL TABLE staging.WarehouseOperations (
    OperationID VARCHAR(100),
    LocationID VARCHAR(50),
    SKU VARCHAR(100),
    OperationType VARCHAR(50),
    OperationStartTime DATETIME2(0),
    OperationEndTime DATETIME2(0),
    Quantity INT,
    DwellTimeMinutes INT,
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50),
    BatchID VARCHAR(100),
    ErrorFlag BIT,
    ErrorReason VARCHAR(500),
    IsProcessed BIT DEFAULT 0
)
WITH (
    LOCATION = '/wms/operations/',
    DATA_SOURCE = WMS_DataFeed,
    FILE_FORMAT = CSV_Format
)
GO
```

### Scheduled Data Refresh (SQL Agent Job)

```sql
-- Create SQL Agent job for 15-minute incremental refresh
USE msdb
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_IncrementalRefresh',
    @enabled = 1,
    @description = N'Load warehouse and fleet data every 15 minutes'
GO

EXEC dbo.sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalRefresh',
    @step_name = N'Load Warehouse Operations',
    @subsystem = N'TSQL',
    @command = N'
        DECLARE @EndTime DATETIME2(0) = GETDATE()
        DECLARE @StartTime DATETIME2(0) = DATEADD(MINUTE, -15, @EndTime)
        EXEC prod.LoadWarehouseOperations @StartTime, @EndTime
        EXEC prod.UpdateProductGravityScores
    ',
    @database_name = N'LogiFleetPulse',
    @retry_attempts = 3,
    @retry_interval = 2
GO

EXEC dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15
GO

EXEC dbo.sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalRefresh',
    @schedule_name = N'Every15Minutes'
GO

EXEC dbo.sp_add_jobserver
    @job_name = N'LogiFleet_IncrementalRefresh'
GO
```

## Common Usage Patterns

### Cross-Fact Query: Dwell Time vs Fleet Idle Time

```sql
-- Find products with high warehouse dwell time AND high fleet idle time
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdle,
    COUNT(DISTINCT wo.OperationKey) AS WarehouseOps,
    COUNT(DISTINCT ft.TripKey) AS FleetTrips
FROM prod.DimProductGravity p
INNER JOIN prod.FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
INNER JOIN prod.FactFleetTrips ft ON 
    ft.TimeKey = wo.TimeKey AND
    DATEDIFF(HOUR, wo.OperationEndTime, ft.TripStartTime) BETWEEN 0 AND 24
WHERE wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    AND wo.DwellTimeMinutes > 60
GROUP BY p.SKU, p.ProductName, p.GravityScore
HAVING AVG(wo.DwellTimeMinutes) > 120 
    AND AVG(ft.IdleTimeMinutes) > 20
ORDER BY AVG(wo.DwellTimeMinutes) DESC
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be relocated to higher-priority zones
WITH CurrentZonePerformance AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.GravityScore,
        wo.StorageZone,
        COUNT(*) AS PickCount,
        AVG(wo.DurationMinutes) AS AvgPickTime,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY wo.DurationMinutes) 
            OVER (PARTITION BY wo.StorageZone) AS MedianZonePickTime
    FROM prod.DimProductGravity p
    INNER JOIN prod.FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    WHERE wo.OperationType = 'Picking'
        AND wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.GravityScore, wo.StorageZone
)
SELECT 
    SKU,
    GravityScore,
    StorageZone AS CurrentZone,
    AvgPickTime,
    MedianZonePickTime,
    CASE 
        WHEN GravityScore >= 70 THEN 'Priority-A'
        WHEN GravityScore >= 40 THEN 'Priority-B'
        ELSE 'Standard'
    END AS RecommendedZone,
    CASE 
        WHEN GravityScore >= 70 AND StorageZone NOT LIKE 'Priority-A%' 
        THEN 'Relocate to Priority-A'
        WHEN GravityScore >= 40 AND StorageZone LIKE 'Standard%'
        THEN 'Relocate to Priority-B'
        ELSE 'Keep Current'
    END AS Action
FROM CurrentZonePerformance
WHERE PickCount >= 10 -- Minimum sample size
ORDER BY GravityScore DESC, AvgPickTime DESC
```

### Fleet Maintenance Alert Query

```sql
-- Generate maintenance priority queue based on revenue impact
WITH VehicleRisk AS (
    SELECT 
        ft.VehicleID,
        COUNT(*) AS TripCount,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(ft.DistanceKm) AS TotalDistanceKm,
        SUM(CASE WHEN p.ValueClass = 'High' THEN cd.Quantity ELSE 0 END) AS HighValueCargo,
        MAX(ft.MaintenanceAlert) AS HasAlert
    FROM prod.FactFleetTrips ft
    LEFT JOIN prod.FactCrossDock cd ON ft.TripKey = cd.OutboundTripKey
    LEFT JOIN prod.DimProductGravity p ON cd.ProductKey = p.ProductKey
    WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ft.VehicleID
)
SELECT 
    VehicleID,
    TripCount,
    AvgIdleTime,
    TotalDistanceKm,
    HighValueCargo,
    (
        (AvgIdleTime * 0.3) + 
        (TotalDistanceKm / 100.0 * 0.3) + 
        (HighValueCargo / 1000.0 * 0.4)
    ) AS MaintenancePriorityScore,
    CASE 
        WHEN HasAlert = 1 THEN 'URGENT'
        WHEN (AvgIdleTime * 0.3) + (TotalDistanceKm / 100.0 * 0.3) + (HighValueCargo / 1000.0 * 0.4) > 50 
        THEN 'HIGH'
        ELSE 'NORMAL'
    END AS PriorityLevel
FROM VehicleRisk
ORDER BY MaintenancePriorityScore DESC
```

## Power BI DAX Measures

### Cross-Fact KPI: Total Logistics Cost

```dax
Total Logistics Cost = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DurationMinutes] * 0.50 // $0.50 per minute labor cost
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        (FactFleetTrips[FuelConsumedLiters] * 1.20) + // Fuel cost
        (FactFleetTrips[IdleTimeMinutes] * 0.30) // Idle cost
    )
RETURN WarehouseCost + FleetCost
```

### Warehouse Efficiency Score

```dax
Warehouse Efficiency Score = 
DIVIDE(
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[UnitsPerHour] > 100
        )
    ),
    COUNTROWS(FactWarehouseOperations),
    0
) * 100
```

###
