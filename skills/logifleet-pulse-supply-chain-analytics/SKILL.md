---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics analytics with multi-fact star schema, fleet telemetry, and warehouse operations
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure fleet and warehouse analytics schema
  - implement multi-fact star schema for logistics
  - create supply chain intelligence dashboard
  - integrate warehouse operations with fleet telemetry
  - build logistics kpi tracking system
  - set up cross-modal supply chain analytics
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is an advanced logistics intelligence platform built on MS SQL Server and Power BI. It provides a unified semantic layer combining warehouse operations, fleet telemetry, inventory management, and external data sources through a multi-fact star schema with time-phased dimensions. The platform enables cross-fact KPI analysis, predictive bottleneck detection, and real-time operational dashboards for supply chain optimization.

## Core Components

### Architecture Overview

- **MS SQL Server** (2019+): Transactional data warehouse with multi-fact star schema
- **Power BI**: Visualization layer with semantic model and real-time dashboards
- **Data Sources**: WMS, TMS, telematics APIs, supplier portals, weather/traffic feeds
- **Refresh Cycle**: 15-minute incremental loads with time-phased dimensions

### Key Data Model Elements

1. **Fact Tables**:
   - `FactWarehouseOperations`: putaway, picking, packing, shipping events
   - `FactFleetTrips`: route segments, fuel consumption, idle time
   - `FactCrossDock`: direct transfers without long-term storage

2. **Dimension Tables**:
   - `DimTime`: 15-minute granularity with fiscal periods
   - `DimGeography`: hierarchical location data
   - `DimProductGravity`: velocity-based warehouse zoning
   - `DimSupplierReliability`: lead time and quality metrics

3. **Bridge Tables**: Many-to-many relationships between routes and storage zones

## Installation & Setup

### 1. Deploy SQL Schema

```sql
-- Execute the schema creation script
-- Assumes SQL Server 2019+ with appropriate permissions

USE master;
GO

CREATE DATABASE LogiFleetPulse
ON PRIMARY (
    NAME = LogiFleetPulse_Data,
    FILENAME = 'C:\SQLData\LogiFleetPulse_Data.mdf',
    SIZE = 10GB,
    FILEGROWTH = 1GB
)
LOG ON (
    NAME = LogiFleetPulse_Log,
    FILENAME = 'C:\SQLData\LogiFleetPulse_Log.ldf',
    SIZE = 2GB,
    FILEGROWTH = 512MB
);
GO

USE LogiFleetPulse;
GO

-- Create dimension tables first (time dimension)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeStamp DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Day INT NOT NULL,
    Hour INT NOT NULL,
    Minute15Bucket INT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek INT NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    CONSTRAINT UQ_DimTime_DateTime UNIQUE (DateTimeStamp)
);

-- Create clustered columnstore index for fast aggregation
CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime;
GO

-- Create geography dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, DistributionCenter, Route, Customer
    Address NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE,
    CONSTRAINT UQ_DimGeography_LocationID UNIQUE (LocationID, EffectiveDate)
);
GO

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2) NOT NULL, -- Calculated: velocity * value * fragility
    VelocityClass VARCHAR(20), -- Fast, Medium, Slow
    ValueClass VARCHAR(20), -- High, Medium, Low
    FragilityIndex INT, -- 1-10
    OptimalZoneType VARCHAR(50), -- Cold, Ambient, Hazmat, etc.
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    ReorderPoint INT,
    LeadTimeDays INT,
    IsActive BIT DEFAULT 1,
    LastUpdated DATETIME2 DEFAULT GETDATE()
);

CREATE NONCLUSTERED INDEX IX_DimProduct_SKU ON DimProductGravity(SKU) INCLUDE (GravityScore, VelocityClass);
GO

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(200) NOT NULL,
    SupplierType VARCHAR(50), -- Direct, Distributor, Manufacturer
    LeadTimeAvgDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2), -- Percentage
    DefectRate DECIMAL(5,4), -- Percentage
    ComplianceScore INT, -- 1-100
    ReliabilityTier VARCHAR(20), -- Platinum, Gold, Silver, Bronze
    Country NVARCHAR(100),
    IsActive BIT DEFAULT 1,
    LastEvaluationDate DATE
);
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    LocationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID VARCHAR(100) NOT NULL,
    OrderID VARCHAR(100),
    BatchID VARCHAR(100),
    EmployeeID VARCHAR(50),
    ZoneID VARCHAR(50),
    BinLocation VARCHAR(50),
    Quantity INT NOT NULL,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2), -- Time item sat before next operation
    DistanceTraveled DECIMAL(10,2), -- Meters within warehouse
    TemperatureAtOperation DECIMAL(5,2),
    IsException BIT DEFAULT 0,
    ExceptionReason NVARCHAR(500),
    CreatedDate DATETIME2 DEFAULT GETDATE()
);

-- Partition by month for performance
CREATE PARTITION FUNCTION PF_OperationMonth (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01',
    '2026-05-01', '2026-06-01', '2026-07-01', '2026-08-01'
);

CREATE PARTITION SCHEME PS_OperationMonth
AS PARTITION PF_OperationMonth ALL TO ([PRIMARY]);
GO

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripID VARCHAR(100) NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginLocationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationLocationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    AvgSpeedKMH DECIMAL(5,2),
    MaxSpeedKMH DECIMAL(5,2),
    HarshBrakingCount INT DEFAULT 0,
    HarshAccelerationCount INT DEFAULT 0,
    PlannedRouteID VARCHAR(100),
    ActualRouteDeviation DECIMAL(10,2), -- KM off planned route
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes DECIMAL(10,2),
    OnTimeStatus VARCHAR(20), -- OnTime, Early, Late
    DeliveryWindowVarianceMinutes DECIMAL(10,2),
    IsException BIT DEFAULT 0,
    ExceptionReason NVARCHAR(500),
    CreatedDate DATETIME2 DEFAULT GETDATE()
);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Vehicle_StartTime 
ON FactFleetTrips(VehicleID, StartTimeKey) INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
GO
```

### 2. Create ETL Stored Procedures

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Stage data from source system (replace with actual WMS connection)
        -- This example assumes an external table or linked server
        INSERT INTO FactWarehouseOperations (
            TimeKey, LocationKey, ProductKey, OperationType,
            OperationID, OrderID, Quantity, DurationMinutes,
            DwellTimeHours, DistanceTraveled, IsException
        )
        SELECT 
            dt.TimeKey,
            dg.GeographyKey,
            dp.ProductKey,
            src.OperationType,
            src.OperationID,
            src.OrderID,
            src.Quantity,
            src.DurationMinutes,
            src.DwellTimeHours,
            src.DistanceTraveled,
            CASE WHEN src.DurationMinutes > src.StandardDuration * 1.5 THEN 1 ELSE 0 END AS IsException
        FROM ExternalWarehouseData src
        INNER JOIN DimTime dt ON DATEADD(MINUTE, 
            (DATEPART(MINUTE, src.OperationTimestamp) / 15) * 15, 
            CAST(CAST(src.OperationTimestamp AS DATE) AS DATETIME2) + 
            CAST(CAST(DATEPART(HOUR, src.OperationTimestamp) AS VARCHAR) + ':00:00' AS TIME)
        ) = dt.DateTimeStamp
        INNER JOIN DimGeography dg ON src.LocationID = dg.LocationID 
            AND src.OperationTimestamp BETWEEN dg.EffectiveDate AND ISNULL(dg.ExpirationDate, '9999-12-31')
        INNER JOIN DimProductGravity dp ON src.SKU = dp.SKU AND dp.IsActive = 1
        WHERE src.OperationTimestamp > @LastLoadDate
            AND NOT EXISTS (
                SELECT 1 FROM FactWarehouseOperations 
                WHERE OperationID = src.OperationID
            );
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO

-- Calculate product gravity scores (run daily)
CREATE PROCEDURE usp_RefreshProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH RecentActivity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS OperationCount,
            AVG(DurationMinutes) AS AvgHandlingTime,
            SUM(Quantity) AS TotalQuantityMoved
        FROM FactWarehouseOperations
        WHERE CreatedDate >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    ),
    VelocityCalc AS (
        SELECT 
            ProductKey,
            NTILE(3) OVER (ORDER BY TotalQuantityMoved DESC) AS VelocityTier
        FROM RecentActivity
    )
    UPDATE dp
    SET 
        GravityScore = (
            (CASE vc.VelocityTier WHEN 1 THEN 100 WHEN 2 THEN 50 ELSE 25 END) * 0.4 +
            (CASE dp.ValueClass WHEN 'High' THEN 100 WHEN 'Medium' THEN 50 ELSE 25 END) * 0.3 +
            (dp.FragilityIndex * 10) * 0.3
        ),
        VelocityClass = CASE vc.VelocityTier WHEN 1 THEN 'Fast' WHEN 2 THEN 'Medium' ELSE 'Slow' END,
        LastUpdated = GETDATE()
    FROM DimProductGravity dp
    INNER JOIN VelocityCalc vc ON dp.ProductKey = vc.ProductKey;
END;
GO
```

### 3. Configure Power BI Connection

```sql
-- Create view for Power BI semantic layer
CREATE VIEW vw_WarehousePerformance
AS
SELECT 
    dt.Date,
    dt.Hour,
    dt.DayOfWeek,
    dg.LocationName,
    dg.Region,
    dp.SKU,
    dp.ProductName,
    dp.Category,
    dp.GravityScore,
    dp.VelocityClass,
    fwo.OperationType,
    fwo.ZoneID,
    COUNT(*) AS OperationCount,
    SUM(fwo.Quantity) AS TotalQuantity,
    AVG(fwo.DurationMinutes) AS AvgDurationMinutes,
    AVG(fwo.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(fwo.DistanceTraveled) AS TotalDistanceTraveled,
    SUM(CASE WHEN fwo.IsException = 1 THEN 1 ELSE 0 END) AS ExceptionCount
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fwo.LocationKey = dg.GeographyKey
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
GROUP BY 
    dt.Date, dt.Hour, dt.DayOfWeek,
    dg.LocationName, dg.Region,
    dp.SKU, dp.ProductName, dp.Category, dp.GravityScore, dp.VelocityClass,
    fwo.OperationType, fwo.ZoneID;
GO

CREATE VIEW vw_FleetPerformance
AS
SELECT 
    dt.Date,
    dt.Hour,
    orig.LocationName AS OriginLocation,
    orig.Region AS OriginRegion,
    dest.LocationName AS DestinationLocation,
    dest.Region AS DestinationRegion,
    fft.VehicleID,
    fft.DriverID,
    COUNT(*) AS TripCount,
    SUM(fft.DistanceKM) AS TotalDistanceKM,
    SUM(fft.DurationMinutes) AS TotalDurationMinutes,
    SUM(fft.IdleTimeMinutes) AS TotalIdleTimeMinutes,
    SUM(fft.FuelConsumedLiters) AS TotalFuelConsumed,
    AVG(fft.LoadWeightKG) AS AvgLoadWeight,
    AVG(CAST(fft.IdleTimeMinutes AS FLOAT) / NULLIF(fft.DurationMinutes, 0)) AS AvgIdlePercentage,
    SUM(fft.TrafficDelayMinutes) AS TotalTrafficDelayMinutes,
    SUM(CASE WHEN fft.OnTimeStatus = 'OnTime' THEN 1 ELSE 0 END) AS OnTimeCount,
    SUM(CASE WHEN fft.IsException = 1 THEN 1 ELSE 0 END) AS ExceptionCount
FROM FactFleetTrips fft
INNER JOIN DimTime dt ON fft.StartTimeKey = dt.TimeKey
INNER JOIN DimGeography orig ON fft.OriginLocationKey = orig.GeographyKey
INNER JOIN DimGeography dest ON fft.DestinationLocationKey = dest.GeographyKey
GROUP BY 
    dt.Date, dt.Hour,
    orig.LocationName, orig.Region,
    dest.LocationName, dest.Region,
    fft.VehicleID, fft.DriverID;
GO

-- Cross-fact KPI view
CREATE VIEW vw_CrossModalAnalytics
AS
SELECT 
    dt.Date,
    dg.LocationName AS WarehouseLocation,
    dp.Category AS ProductCategory,
    dp.VelocityClass,
    warehouse.TotalQuantity AS WarehouseQuantityProcessed,
    warehouse.AvgDwellTimeHours,
    fleet.TripCount AS RelatedFleetTrips,
    fleet.TotalIdleTimeMinutes AS FleetIdleTime,
    fleet.TotalFuelConsumed,
    CASE 
        WHEN warehouse.AvgDwellTimeHours > 72 AND fleet.AvgIdlePercentage > 0.15 
        THEN 'Critical - High Dwell + High Idle'
        WHEN warehouse.AvgDwellTimeHours > 48 
        THEN 'Warning - High Dwell Time'
        WHEN fleet.AvgIdlePercentage > 0.20 
        THEN 'Warning - Excessive Idle'
        ELSE 'Normal'
    END AS OperationalStatus
FROM DimTime dt
CROSS JOIN DimGeography dg
CROSS JOIN DimProductGravity dp
LEFT JOIN (
    SELECT 
        TimeKey, LocationKey, ProductKey,
        SUM(Quantity) AS TotalQuantity,
        AVG(DwellTimeHours) AS AvgDwellTimeHours
    FROM FactWarehouseOperations
    GROUP BY TimeKey, LocationKey, ProductKey
) warehouse ON dt.TimeKey = warehouse.TimeKey 
    AND dg.GeographyKey = warehouse.LocationKey 
    AND dp.ProductKey = warehouse.ProductKey
LEFT JOIN (
    SELECT 
        StartTimeKey AS TimeKey,
        OriginLocationKey AS LocationKey,
        COUNT(*) AS TripCount,
        SUM(IdleTimeMinutes) AS TotalIdleTimeMinutes,
        SUM(FuelConsumedLiters) AS TotalFuelConsumed,
        AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0)) AS AvgIdlePercentage
    FROM FactFleetTrips
    GROUP BY StartTimeKey, OriginLocationKey
) fleet ON dt.TimeKey = fleet.TimeKey 
    AND dg.GeographyKey = fleet.LocationKey
WHERE dg.LocationType = 'Warehouse';
GO
```

### 4. Power BI Setup

Open Power BI Desktop and create a new connection:

**Connection String Format:**
```
Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;Trusted_Connection=True;
```

**DAX Measures for Power BI:**

```dax
// Warehouse Efficiency Score
WarehouseEfficiency = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR ExceptionOps = CALCULATE(COUNTROWS(FactWarehouseOperations), FactWarehouseOperations[IsException] = TRUE())
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR EfficiencyScore = 
    (1 - (ExceptionOps / TotalOps)) * 0.5 +
    (1 - MIN(AvgDwell / 48, 1)) * 0.5  // 48 hours is baseline
RETURN EfficiencyScore * 100

// Fleet Utilization Rate
FleetUtilization = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Fuel Efficiency (KM per Liter)
FuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Predictive Bottleneck Index
BottleneckIndex = 
VAR DwellScore = MIN(AVERAGE(FactWarehouseOperations[DwellTimeHours]) / 24, 3) * 33.33
VAR IdleScore = MIN(AVERAGE(FactFleetTrips[IdleTimeMinutes]) / 60, 3) * 33.33
VAR ExceptionScore = 
    (CALCULATE(COUNTROWS(FactWarehouseOperations), FactWarehouseOperations[IsException] = TRUE()) +
     CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[IsException] = TRUE())) / 
    (COUNTROWS(FactWarehouseOperations) + COUNTROWS(FactFleetTrips)) * 100
RETURN DwellScore + IdleScore + ExceptionScore

// On-Time Delivery Percentage
OnTimeDeliveryRate = 
DIVIDE(
    CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeStatus] = "OnTime"),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Cross-Modal Impact Score
CrossModalImpact = 
VAR HighDwellCount = CALCULATE(COUNTROWS(FactWarehouseOperations), FactWarehouseOperations[DwellTimeHours] > 72)
VAR HighIdleCount = CALCULATE(COUNTROWS(FactFleetTrips), 
    DIVIDE(FactFleetTrips[IdleTimeMinutes], FactFleetTrips[DurationMinutes]) > 0.15)
VAR TotalRecords = COUNTROWS(FactWarehouseOperations) + COUNTROWS(FactFleetTrips)
RETURN (HighDwellCount + HighIdleCount) / TotalRecords * 100
```

### 5. Automated Alerting Configuration

```sql
-- Create alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName NVARCHAR(200) NOT NULL,
    MetricType VARCHAR(50) NOT NULL, -- DwellTime, IdleTime, FuelConsumption, etc.
    ThresholdValue DECIMAL(10,2) NOT NULL,
    ThresholdOperator VARCHAR(10) NOT NULL, -- >, <, =, >=, <=
    EvaluationWindowMinutes INT NOT NULL,
    RecipientEmails NVARCHAR(MAX) NOT NULL, -- Semicolon separated
    AlertPriority VARCHAR(20) NOT NULL, -- Critical, Warning, Info
    IsActive BIT DEFAULT 1,
    CreatedDate DATETIME2 DEFAULT GETDATE()
);
GO

-- Insert sample alert configurations
INSERT INTO AlertConfiguration (AlertName, MetricType, ThresholdValue, ThresholdOperator, EvaluationWindowMinutes, RecipientEmails, AlertPriority)
VALUES 
    ('High Warehouse Dwell Time', 'DwellTime', 72.0, '>', 60, '${LOGISTICS_MANAGER_EMAIL}', 'Warning'),
    ('Excessive Fleet Idle Time', 'IdlePercentage', 15.0, '>', 60, '${FLEET_MANAGER_EMAIL}', 'Warning'),
    ('Fuel Efficiency Drop', 'FuelEfficiency', 8.0, '<', 240, '${OPERATIONS_DIRECTOR_EMAIL}', 'Critical'),
    ('Late Delivery Spike', 'LateDeliveryRate', 10.0, '>', 30, '${CUSTOMER_SERVICE_EMAIL}', 'Critical');
GO

-- Alert evaluation stored procedure
CREATE PROCEDURE usp_EvaluateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessages TABLE (
        AlertName NVARCHAR(200),
        CurrentValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2),
        Recipients NVARCHAR(MAX),
        Priority VARCHAR(20)
    );
    
    -- Evaluate dwell time alerts
    INSERT INTO @AlertMessages
    SELECT 
        ac.AlertName,
        AVG(fwo.DwellTimeHours) AS CurrentValue,
        ac.ThresholdValue,
        ac.RecipientEmails,
        ac.AlertPriority
    FROM AlertConfiguration ac
    CROSS APPLY (
        SELECT AVG(DwellTimeHours) AS AvgDwell
        FROM FactWarehouseOperations
        WHERE CreatedDate >= DATEADD(MINUTE, -ac.EvaluationWindowMinutes, GETDATE())
    ) fwo
    WHERE ac.MetricType = 'DwellTime'
        AND ac.IsActive = 1
        AND (
            (ac.ThresholdOperator = '>' AND fwo.AvgDwell > ac.ThresholdValue) OR
            (ac.ThresholdOperator = '<' AND fwo.AvgDwell < ac.ThresholdValue)
        );
    
    -- Evaluate idle time alerts
    INSERT INTO @AlertMessages
    SELECT 
        ac.AlertName,
        AVG(CAST(fft.IdleTimeMinutes AS FLOAT) / NULLIF(fft.DurationMinutes, 0) * 100) AS CurrentValue,
        ac.ThresholdValue,
        ac.RecipientEmails,
        ac.AlertPriority
    FROM AlertConfiguration ac
    CROSS APPLY (
        SELECT 
            AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0) * 100) AS AvgIdlePercentage
        FROM FactFleetTrips
        WHERE CreatedDate >= DATEADD(MINUTE, -ac.EvaluationWindowMinutes, GETDATE())
    ) fft
    WHERE ac.MetricType = 'IdlePercentage'
        AND ac.IsActive = 1
        AND (
            (ac.ThresholdOperator = '>' AND fft.AvgIdlePercentage > ac.ThresholdValue) OR
            (ac.ThresholdOperator = '<' AND fft.AvgIdlePercentage < ac.ThresholdValue)
        );
    
    -- Send alerts (integrate with your email system or messaging API)
    -- This is a placeholder - implement using sp_send_dbmail or external API
    SELECT 
        AlertName,
        CurrentValue,
        ThresholdValue,
        Recipients,
        Priority,
        GETDATE() AS AlertTimestamp
    FROM @AlertMessages;
END;
GO

-- Schedule the alert procedure (SQL Server Agent Job)
-- Run this manually or via Agent:
-- EXEC usp_EvaluateAlerts;
```

## Configuration

### Environment Variables

Set these in your deployment environment:

```bash
# SQL Server Connection
SQL_SERVER_HOST=your-sql-server.database.windows.net
SQL_SERVER_DATABASE=LogiFleetPulse
SQL_SERVER_USER=${SQL_SERVER_USER}
SQL_SERVER_PASSWORD=${SQL_SERVER_PASSWORD}

# External Data Sources
WMS_API_ENDPOINT=${WMS_API_ENDPOINT}
WMS_API_KEY=${WMS_API_KEY}
TELEMATICS_API_ENDPOINT=${TELEMATICS_API_ENDPOINT}
TELEMATICS_API_KEY=${TELEMATICS_API_KEY}
WEATHER_API_KEY=${WEATHER_API_KEY}

# Alert Recipients
LOGISTICS_MANAGER_EMAIL=logistics@company.com
FLEET_MANAGER_EMAIL=fleet@company.com
OPERATIONS_DIRECTOR_EMAIL=ops@company.com
CUSTOMER_SERVICE_EMAIL=cs@company.com

# Power BI
POWERBI_WORKSPACE_ID=${POWERBI_WORKSPACE_ID}
POWERBI_DATASET_ID=${POWERBI_DATASET_ID}
```

### Connection to External Systems

**Example: Connecting to WMS via Linked Server**

```sql
-- Create linked server to WMS database
EXEC sp_addlinkedserver 
    @server = 'WMS_SOURCE',
    @srvproduct = '',
    @provider = 'SQLNCLI',
    @datasrc = '${WMS_SERVER_HOST}',
    @catalog = 'WMSDatabase';

EXEC sp_addlinkedsrvlogin 
    @rmtsrvname = 'WMS_SOURCE',
    @useself = 'FALSE',
    @rmtuser = '${WMS_USER}',
    @rmtpassword = '${WMS_PASSWORD}';

-- Create external table for warehouse operations
CREATE EXTERNAL DATA SOURCE WMS_ExternalSource
WITH (
    TYPE = RDBMS,
    LOCATION = '${WMS_SERVER_HOST}',
    DATABASE_NAME = 'WMSDatabase',
    CREDENTIAL = WMS_Credential
);

CREATE EXTERNAL TABLE ExternalWarehouseData (
    OperationID VARCHAR(100),
    OperationTimestamp DATETIME2,
    OperationType VARCHAR(50),
    LocationID VARCHAR(50),
    SKU VARCHAR(100),
    OrderID VARCHAR(100),
    Quantity INT,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    DistanceTraveled DECIMAL(10,2),
    StandardDuration DECIMAL(10,2)
)
WITH (
    DATA_SOURCE = WMS_ExternalSource,
    SCHEMA_NAME = 'dbo',
    OBJECT_NAME = 'WarehouseOperations'
);
```

## Common Usage Patterns

### Query Warehouse Performance by Product Gravity

```sql
-- Find slow-moving products in high-traffic zones
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.VelocityClass,
    fwo.ZoneID,
    COUNT(*) AS OperationCount,
    AV
