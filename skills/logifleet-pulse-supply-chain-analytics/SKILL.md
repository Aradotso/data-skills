---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehousing for fleet logistics and supply chain intelligence with multi-fact star schema modeling
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet data model"
  - "create multi-fact star schema for logistics"
  - "build Power BI logistics dashboard"
  - "implement cross-dock operations tracking"
  - "set up real-time fleet telemetry analytics"
  - "configure warehouse gravity zones"
  - "deploy logistics data warehouse"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It unifies warehouse operations, fleet telemetry, inventory management, and external signals into a semantic data layer using a multi-fact star schema architecture. The platform enables cross-fact KPI analysis (e.g., correlating dwell time with fleet idling costs) through time-phased dimensions and bridge tables.

## What It Does

- **Multi-Modal Data Integration**: Combines WMS, GPS/telematics, supplier portals, weather/traffic APIs, and customer orders
- **Cross-Fact Analytics**: Links warehouse metrics with fleet performance through shared dimensions
- **Predictive Intelligence**: Identifies bottlenecks, maintenance priorities, and capacity optimization opportunities
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item value, and fragility
- **Real-Time Dashboards**: Power BI visualizations with 15-minute refresh cycles
- **Role-Based Access**: Row-level security for different organizational levels

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (recommended for temporal tables and polybase support)
- Power BI Desktop (latest version)
- Access to data sources: WMS, fleet telemetry, ERP systems

### Step 1: Deploy SQL Schema

```sql
-- Execute the main schema creation script
-- This creates fact tables, dimensions, views, and stored procedures
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables first (time-aware with SCD Type 2)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    Minute TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL, -- 0-3 for 15-min buckets
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod VARCHAR(10) NOT NULL,
    FiscalQuarter VARCHAR(10) NOT NULL,
    FiscalYear SMALLINT NOT NULL
);

-- Index for common query patterns
CREATE NONCLUSTERED INDEX IX_DimTime_Date ON DimTime(Date);
CREATE NONCLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime);
GO

CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, RouteNode, CrossDock
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1,
    ValidFrom DATETIME2 DEFAULT SYSDATETIME(),
    ValidTo DATETIME2 DEFAULT '9999-12-31'
);

CREATE NONCLUSTERED INDEX IX_DimGeography_LocationCode ON DimGeography(LocationCode);
GO

CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite: velocity + value + fragility
    VelocityClass VARCHAR(20), -- Fast, Medium, Slow
    ValueTier VARCHAR(20), -- High, Medium, Low
    FragilityIndex DECIMAL(3,2), -- 0.0 to 1.0
    OptimalZone VARCHAR(50), -- Recommended warehouse zone
    ValidFrom DATETIME2 DEFAULT SYSDATETIME(),
    ValidTo DATETIME2 DEFAULT '9999-12-31'
);

CREATE NONCLUSTERED INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU);
CREATE NONCLUSTERED INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC);
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    LeadTimeMean DECIMAL(5,1), -- In days
    LeadTimeStdDev DECIMAL(5,2),
    DefectRate DECIMAL(5,4), -- Percentage
    ComplianceScore DECIMAL(3,2), -- 0.0 to 1.0
    RiskCategory VARCHAR(20), -- Low, Medium, High
    ValidFrom DATETIME2 DEFAULT SYSDATETIME(),
    ValidTo DATETIME2 DEFAULT '9999-12-31'
);
GO

-- Fact table for warehouse operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL, -- Warehouse location
    SupplierKey INT,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT NOT NULL,
    DwellTimeMinutes INT, -- Time spent in current stage
    ZoneAssignment VARCHAR(50),
    BatchNumber VARCHAR(50),
    OrderNumber VARCHAR(50),
    HandlingCost DECIMAL(10,2),
    DefectCount INT DEFAULT 0,
    CONSTRAINT FK_FactWarehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_FactWarehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWarehouse_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeKey ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_ProductKey ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_OperationType ON FactWarehouseOperations(OperationType);
GO

-- Fact table for fleet operations
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKeyStart INT NOT NULL,
    TimeKeyEnd INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    RouteSegmentID VARCHAR(100),
    DistanceKm DECIMAL(8,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(8,2),
    LoadWeightKg DECIMAL(10,2),
    DeliveryCount INT,
    OnTimeStatus BIT, -- 1 = On time, 0 = Delayed
    DelayReasonCode VARCHAR(50), -- Weather, Traffic, Maintenance, etc.
    CONSTRAINT FK_FactFleet_TimeStart FOREIGN KEY (TimeKeyStart) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_TimeEnd FOREIGN KEY (TimeKeyEnd) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeKeyStart ON FactFleetTrips(TimeKeyStart);
CREATE NONCLUSTERED INDEX IX_FactFleet_VehicleKey ON FactFleetTrips(VehicleKey);
GO

-- Cross-dock fact table (bridge between warehouse and fleet)
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    CrossDockGeographyKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferTimeMinutes INT,
    Quantity INT,
    CONSTRAINT FK_FactCrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactCrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_FactCrossDock_Geography FOREIGN KEY (CrossDockGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactCrossDock_InboundTrip FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_FactCrossDock_OutboundTrip FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey)
);
GO
```

### Step 2: Create Vehicle and Driver Dimensions

```sql
-- Additional dimensions for fleet tracking
CREATE TABLE DimVehicle (
    VehicleKey INT IDENTITY(1,1) PRIMARY KEY,
    VehicleID VARCHAR(50) NOT NULL UNIQUE,
    VehicleType VARCHAR(50), -- Truck, Van, Refrigerated
    Capacity DECIMAL(10,2),
    CapacityUnit VARCHAR(20), -- Kg, m³
    YearManufactured SMALLINT,
    MaintenanceScore DECIMAL(3,2), -- 0.0 to 1.0
    IsActive BIT DEFAULT 1
);

CREATE TABLE DimDriver (
    DriverKey INT IDENTITY(1,1) PRIMARY KEY,
    DriverID VARCHAR(50) NOT NULL UNIQUE,
    DriverName VARCHAR(200),
    SafetyScore DECIMAL(3,2), -- 0.0 to 1.0
    ExperienceYears DECIMAL(4,1),
    IsActive BIT DEFAULT 1
);

-- Add foreign keys to FactFleetTrips
ALTER TABLE FactFleetTrips
ADD CONSTRAINT FK_FactFleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey);

ALTER TABLE FactFleetTrips
ADD CONSTRAINT FK_FactFleet_Driver FOREIGN KEY (DriverKey) REFERENCES DimDriver(DriverKey);
GO
```

### Step 3: Create Stored Procedures for Data Loading

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external staging table
    -- Assumes staging table exists with source data
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes,
        ZoneAssignment, BatchNumber, OrderNumber, HandlingCost, DefectCount
    )
    SELECT 
        dt.TimeKey,
        dp.ProductKey,
        dg.GeographyKey,
        ds.SupplierKey,
        stg.OperationType,
        stg.Quantity,
        stg.DwellTimeMinutes,
        stg.ZoneAssignment,
        stg.BatchNumber,
        stg.OrderNumber,
        stg.HandlingCost,
        stg.DefectCount
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime dt ON CAST(stg.OperationDateTime AS DATETIME2) = dt.DateTime
    INNER JOIN DimProductGravity dp ON stg.SKU = dp.SKU AND dp.ValidTo = '9999-12-31'
    INNER JOIN DimGeography dg ON stg.WarehouseCode = dg.LocationCode AND dg.ValidTo = '9999-12-31'
    LEFT JOIN DimSupplierReliability ds ON stg.SupplierCode = ds.SupplierCode AND ds.ValidTo = '9999-12-31'
    WHERE stg.OperationDateTime > @LastLoadDateTime;
END;
GO

-- Procedure to calculate gravity scores
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on recent activity
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(*) AS OperationCount,
            AVG(CAST(w.DwellTimeMinutes AS FLOAT)) AS AvgDwellTime,
            SUM(w.Quantity) AS TotalVolume
        FROM DimProductGravity p
        LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
        WHERE w.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days (96 intervals/day * 30)
        GROUP BY p.ProductKey
    )
    UPDATE p
    SET 
        GravityScore = (
            (CASE WHEN pm.TotalVolume > 1000 THEN 1.0 ELSE pm.TotalVolume / 1000.0 END) * 0.4 + -- Velocity
            (CASE WHEN p.ValueTier = 'High' THEN 1.0 WHEN p.ValueTier = 'Medium' THEN 0.5 ELSE 0.2 END) * 0.3 + -- Value
            (1.0 - ISNULL(p.FragilityIndex, 0)) * 0.3 -- Inverse fragility
        ),
        VelocityClass = CASE 
            WHEN pm.TotalVolume >= 1000 THEN 'Fast'
            WHEN pm.TotalVolume >= 500 THEN 'Medium'
            ELSE 'Slow'
        END
    FROM DimProductGravity p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey
    WHERE p.ValidTo = '9999-12-31';
END;
GO
```

### Step 4: Create Analytical Views

```sql
-- Cross-fact view: Warehouse dwell time vs fleet idle time
CREATE VIEW vw_DwellTimeVsFleetIdle AS
SELECT 
    dt.Date,
    dt.Hour,
    dg.Region,
    dp.Category,
    AVG(w.DwellTimeMinutes) AS AvgWarehouseDwellTime,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleTime,
    CORR(w.DwellTimeMinutes, f.IdleTimeMinutes) OVER (
        PARTITION BY dt.Date, dg.Region
    ) AS DwellIdleCorrelation
FROM FactWarehouseOperations w
INNER JOIN DimTime dt ON w.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON w.GeographyKey = dg.GeographyKey
INNER JOIN DimProductGravity dp ON w.ProductKey = dp.ProductKey
LEFT JOIN FactCrossDock cd ON w.ProductKey = cd.ProductKey AND w.TimeKey = cd.TimeKey
LEFT JOIN FactFleetTrips f ON cd.OutboundTripKey = f.TripKey
WHERE w.OperationType = 'Shipping'
    AND f.TripKey IS NOT NULL
GROUP BY dt.Date, dt.Hour, dg.Region, dp.Category;
GO

-- View for predictive bottleneck index
CREATE VIEW vw_BottleneckIndex AS
SELECT 
    dg.LocationName,
    dt.Date,
    dt.Hour,
    COUNT(DISTINCT w.ProductKey) AS DistinctSKUs,
    SUM(w.Quantity) AS TotalVolume,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    STDEV(w.DwellTimeMinutes) AS DwellTimeVariance,
    -- Bottleneck score: higher when volume is high, dwell time is high, and variance is high
    (SUM(w.Quantity) / 100.0) * (AVG(w.DwellTimeMinutes) / 60.0) * (1 + ISNULL(STDEV(w.DwellTimeMinutes), 0) / 100.0) AS BottleneckScore
FROM FactWarehouseOperations w
INNER JOIN DimTime dt ON w.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON w.GeographyKey = dg.GeographyKey
WHERE w.OperationType IN ('Picking', 'Packing')
GROUP BY dg.LocationName, dt.Date, dt.Hour;
GO
```

## Power BI Configuration

### Step 1: Import Template

1. Download `LogiFleet_Pulse_Master.pbit` from the repository
2. Open in Power BI Desktop
3. When prompted, enter SQL Server connection details:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server (use service account credentials)

### Step 2: Configure Data Refresh

```powerquery
// Power Query M code for incremental refresh
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    LastRefresh = DateTime.LocalNow(),
    FilteredRows = Table.SelectRows(
        Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
        each [TimeKey] >= (Number.From(LastRefresh) - 2880) // Last 30 days
    )
in
    FilteredRows
```

### Step 3: Row-Level Security Setup

```dax
// DAX measure for user-based filtering
// Create a security table mapping users to regions
[RegionFilter] = 
VAR CurrentUser = USERPRINCIPALNAME()
VAR AllowedRegions = 
    CALCULATETABLE(
        VALUES(SecurityMapping[Region]),
        SecurityMapping[UserEmail] = CurrentUser
    )
RETURN
    IF(
        ISEMPTY(AllowedRegions),
        FALSE(), // No access if not in security table
        DimGeography[Region] IN AllowedRegions
    )
```

### Step 4: Key DAX Measures

```dax
// Cross-fact measure: Fuel cost per unit shipped
[Fuel Cost Per Unit] = 
VAR TotalFuelCost = 
    CALCULATE(
        SUM(FactFleetTrips[FuelConsumedLiters]) * 1.5, // Assume $1.5/liter
        ALL(DimTime)
    )
VAR TotalUnitsShipped = 
    CALCULATE(
        SUM(FactWarehouseOperations[Quantity]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
RETURN
    DIVIDE(TotalFuelCost, TotalUnitsShipped, 0)

// Warehouse gravity zone compliance
[Gravity Zone Compliance %] = 
VAR ActualPlacements = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[ZoneAssignment] = DimProductGravity[OptimalZone]
    )
VAR TotalPlacements = COUNT(FactWarehouseOperations[OperationKey])
RETURN
    DIVIDE(ActualPlacements, TotalPlacements, 0)

// Predictive delay risk score
[Delay Risk Score] = 
VAR AvgDelayRate = 
    CALCULATE(
        AVERAGE(FactFleetTrips[OnTimeStatus]),
        ALL(DimTime)
    )
VAR CurrentHourDelayRate = 
    AVERAGE(FactFleetTrips[OnTimeStatus])
VAR WeatherFactor = 
    IF(
        RELATED(ExternalWeatherData[SevereWeather]) = 1,
        1.5,
        1.0
    )
RETURN
    (1 - CurrentHourDelayRate) * WeatherFactor * 100
```

## Data Source Integration

### Connecting to WMS API

```sql
-- Example: Using SQL Server external table for REST API
-- Requires polybase configuration
CREATE EXTERNAL DATA SOURCE WMS_API
WITH (
    TYPE = GENERIC,
    LOCATION = '${WMS_API_ENDPOINT}',
    CREDENTIAL = WMS_API_Credential
);

-- Create staging table for API data
CREATE EXTERNAL TABLE StagingWarehouseOperations (
    OperationID VARCHAR(50),
    OperationDateTime DATETIME2,
    SKU VARCHAR(50),
    WarehouseCode VARCHAR(50),
    OperationType VARCHAR(50),
    Quantity INT,
    DwellTimeMinutes INT,
    ZoneAssignment VARCHAR(50),
    BatchNumber VARCHAR(50),
    OrderNumber VARCHAR(50),
    HandlingCost DECIMAL(10,2),
    DefectCount INT,
    SupplierCode VARCHAR(50)
)
WITH (
    DATA_SOURCE = WMS_API,
    LOCATION = '/operations/export',
    FILE_FORMAT = JSON_Format
);
```

### Fleet Telemetry Ingestion

```sql
-- Stream processing for GPS telemetry (use SQL Server BULK INSERT or Azure Stream Analytics)
CREATE TABLE StagingFleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineStatus VARCHAR(50),
    LoadWeight DECIMAL(10,2)
);

-- Stored procedure to aggregate telemetry into trips
CREATE PROCEDURE sp_AggregateFleetTrips
    @BatchDate DATE
AS
BEGIN
    -- Use windowing functions to identify trip segments
    WITH TripSegments AS (
        SELECT 
            VehicleID,
            Timestamp,
            Latitude,
            Longitude,
            Speed,
            FuelLevel,
            LoadWeight,
            LAG(Timestamp) OVER (PARTITION BY VehicleID ORDER BY Timestamp) AS PrevTimestamp,
            CASE 
                WHEN DATEDIFF(MINUTE, LAG(Timestamp) OVER (PARTITION BY VehicleID ORDER BY Timestamp), Timestamp) > 30
                THEN 1 ELSE 0
            END AS NewTripFlag
        FROM StagingFleetTelemetry
        WHERE CAST(Timestamp AS DATE) = @BatchDate
    ),
    TripGrouped AS (
        SELECT 
            *,
            SUM(NewTripFlag) OVER (PARTITION BY VehicleID ORDER BY Timestamp) AS TripGroupID
        FROM TripSegments
    )
    INSERT INTO FactFleetTrips (
        TimeKeyStart, TimeKeyEnd, VehicleKey, DriverKey,
        OriginGeographyKey, DestinationGeographyKey,
        DistanceKm, DurationMinutes, IdleTimeMinutes,
        FuelConsumedLiters, LoadWeightKg, OnTimeStatus
    )
    SELECT 
        dtStart.TimeKey,
        dtEnd.TimeKey,
        dv.VehicleKey,
        1, -- Default driver if unknown
        dgOrigin.GeographyKey,
        dgDest.GeographyKey,
        SUM(geography::Point(t.Latitude, t.Longitude, 4326).STDistance(
            geography::Point(LAG(t.Latitude) OVER (PARTITION BY t.VehicleID, t.TripGroupID ORDER BY t.Timestamp), 
                           LAG(t.Longitude) OVER (PARTITION BY t.VehicleID, t.TripGroupID ORDER BY t.Timestamp), 4326)
        )) / 1000.0 AS DistanceKm,
        DATEDIFF(MINUTE, MIN(t.Timestamp), MAX(t.Timestamp)) AS DurationMinutes,
        SUM(CASE WHEN t.Speed < 5 THEN 1 ELSE 0 END) AS IdleTimeMinutes,
        MAX(FuelLevel) - MIN(FuelLevel) AS FuelConsumedLiters,
        AVG(LoadWeight) AS LoadWeightKg,
        1 AS OnTimeStatus -- To be updated based on delivery records
    FROM TripGrouped t
    INNER JOIN DimTime dtStart ON CAST(MIN(t.Timestamp) AS DATETIME2) = dtStart.DateTime
    INNER JOIN DimTime dtEnd ON CAST(MAX(t.Timestamp) AS DATETIME2) = dtEnd.DateTime
    INNER JOIN DimVehicle dv ON t.VehicleID = dv.VehicleID
    -- Geography lookup based on start/end coordinates (simplified)
    CROSS APPLY (
        SELECT TOP 1 GeographyKey 
        FROM DimGeography 
        WHERE LocationType = 'RouteNode'
        ORDER BY geography::Point(Latitude, Longitude, 4326).STDistance(
            geography::Point(MIN(t.Latitude), MIN(t.Longitude), 4326)
        )
    ) dgOrigin
    CROSS APPLY (
        SELECT TOP 1 GeographyKey 
        FROM DimGeography 
        WHERE LocationType = 'RouteNode'
        ORDER BY geography::Point(Latitude, Longitude, 4326).STDistance(
            geography::Point(MAX(t.Latitude), MAX(t.Longitude), 4326)
        )
    ) dgDest
    GROUP BY t.VehicleID, t.TripGroupID, dv.VehicleKey,
             dtStart.TimeKey, dtEnd.TimeKey,
             dgOrigin.GeographyKey, dgDest.GeographyKey;
END;
GO
```

## Alert Configuration

### Setting Up Automated Alerts

```sql
-- Create alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName VARCHAR(200) NOT NULL,
    MetricName VARCHAR(100) NOT NULL,
    ThresholdValue DECIMAL(18,4),
    ThresholdOperator VARCHAR(10), -- >, <, =, >=, <=
    NotificationEmail VARCHAR(500),
    NotificationSMS VARCHAR(200),
    IsActive BIT DEFAULT 1
);

-- Insert sample alert
INSERT INTO AlertConfiguration (AlertName, MetricName, ThresholdValue, ThresholdOperator, NotificationEmail)
VALUES ('High Fleet Idle Time', 'IdleTimePercentage', 15.0, '>', '${LOGISTICS_MANAGER_EMAIL}');

-- Stored procedure to check alerts and send notifications
CREATE PROCEDURE sp_CheckAlerts
AS
BEGIN
    DECLARE @AlertCursor CURSOR;
    DECLARE @AlertName VARCHAR(200), @MetricName VARCHAR(100), 
            @Threshold DECIMAL(18,4), @Operator VARCHAR(10), @Email VARCHAR(500);
    
    SET @AlertCursor = CURSOR FOR
    SELECT AlertName, MetricName, ThresholdValue, ThresholdOperator, NotificationEmail
    FROM AlertConfiguration
    WHERE IsActive = 1;
    
    OPEN @AlertCursor;
    FETCH NEXT FROM @AlertCursor INTO @AlertName, @MetricName, @Threshold, @Operator, @Email;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Example: Check fleet idle time
        IF @MetricName = 'IdleTimePercentage'
        BEGIN
            DECLARE @CurrentIdlePct DECIMAL(5,2);
            SELECT @CurrentIdlePct = AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0)) * 100
            FROM FactFleetTrips
            WHERE TimeKeyStart >= (SELECT MAX(TimeKey) - 96 FROM DimTime); -- Last 24 hours
            
            IF (@Operator = '>' AND @CurrentIdlePct > @Threshold)
            BEGIN
                -- Send email via SQL Server Database Mail
                EXEC msdb.dbo.sp_send_dbmail
                    @recipients = @Email,
                    @subject = @AlertName,
                    @body = 'Current idle time percentage is ' + CAST(@CurrentIdlePct AS VARCHAR(10)) + '%, exceeding threshold of ' + CAST(@Threshold AS VARCHAR(10)) + '%';
            END
        END
        
        FETCH NEXT FROM @AlertCursor INTO @AlertName, @MetricName, @Threshold, @Operator, @Email;
    END
    
    CLOSE @AlertCursor;
    DEALLOCATE @AlertCursor;
END;
GO

-- Schedule via SQL Server Agent
-- CREATE JOB 'LogiFleet_Alert_Check' to run sp_CheckAlerts every 15 minutes
```

## Common Patterns

### Pattern 1: Time-Phased Dimension Lookup

```sql
-- Get product attributes as of a specific date
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone
FROM DimProductGravity p
WHERE p.SKU = 'SKU-12345'
    AND '2026-06-15' BETWEEN p.ValidFrom AND p.ValidTo;
```

### Pattern 2: Cross-Fact Correlation Analysis

```sql
-- Find products with high dwell time that also cause fleet delays
WITH HighDwellProducts AS (
    SELECT 
        ProductKey,
        AVG(DwellTimeMinutes) AS AvgDwell
    FROM FactWarehouseOperations
    WHERE OperationType = 'Picking'
    GROUP BY ProductKey
    HAVING AVG(DwellTimeMinutes) > 120
)
SELECT 
    dp.SKU,
    dp.ProductName,
    hdp.AvgDwell,
    COUNT(DISTINCT f.TripKey) AS AffectedTrips,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdle
FROM HighDwellProducts hdp
INNER JOIN DimProductGravity dp ON hdp.ProductKey = dp.ProductKey
INNER JOIN FactCrossDock cd ON hdp.ProductKey = cd.ProductKey
INNER JOIN FactFleetTrips f ON cd.OutboundTripKey = f.TripKey
GROUP BY dp.SKU, dp.ProductName, hdp.AvgDwell
ORDER BY AvgFleetIdle DESC;
```

### Pattern 3: Warehouse Zone Rebalancing Recommendation

```sql
-- Identify products in wrong gravity zones
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    
