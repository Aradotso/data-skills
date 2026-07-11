---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing solution for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up logistics analytics dashboard
  - create supply chain data warehouse
  - implement fleet management reporting
  - build warehouse operations analytics
  - deploy power bi logistics solution
  - configure sql server star schema for logistics
  - analyze fleet telemetry and warehouse data
  - create cross-modal supply chain dashboards
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics solution for logistics and supply chain management. It provides:

- **Multi-fact star schema** combining warehouse operations, fleet trips, and cross-dock activities
- **Power BI dashboards** for real-time logistics intelligence
- **MS SQL Server backend** with time-phased dimensions and bridge tables
- **Cross-fact KPI harmonization** linking inventory, fleet, and warehouse metrics
- **Predictive analytics** for bottleneck detection and maintenance scheduling

The solution integrates data from warehouse management systems, telematics, GPS feeds, supplier portals, and external APIs (weather, traffic) into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version recommended)
- Administrative access to SQL Server
- WMS/TMS data sources or sample data generators

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

The repository includes SQL scripts for creating the data warehouse structure:

```sql
-- Execute the main schema deployment script
-- Located in /sql/schema/deploy_warehouse.sql

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    TimeValue DATETIME2 NOT NULL,
    Hour INT,
    DayOfWeek INT,
    DayOfMonth INT,
    Month INT,
    Quarter INT,
    FiscalYear INT,
    FiscalPeriod INT,
    IsWeekend BIT,
    TimeBlock VARCHAR(20) -- '00:00-00:15', '00:15-00:30', etc.
);
GO

CREATE INDEX IX_DimTime_TimeValue ON DimTime(TimeValue);
CREATE INDEX IX_DimTime_FiscalYear_Quarter ON DimTime(FiscalYear, Quarter);
GO

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    ProductCategory NVARCHAR(100),
    ProductSubcategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- 0-100, higher = higher priority
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    FragilityIndex DECIMAL(3,2), -- 0-10
    UnitValue DECIMAL(10,2),
    OptimalZone VARCHAR(10), -- 'A', 'B', 'C' zones
    LastUpdated DATETIME2 DEFAULT GETDATE()
);
GO

CREATE INDEX IX_DimProduct_GravityScore ON DimProductGravity(GravityScore DESC);
CREATE INDEX IX_DimProduct_VelocityClass ON DimProductGravity(VelocityClass);
GO

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(20) NOT NULL UNIQUE,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    Address NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Timezone VARCHAR(50)
);
GO

CREATE INDEX IX_DimGeo_Country_Region ON DimGeography(Country, Region);
CREATE INDEX IX_DimGeo_LatLong ON DimGeography(Latitude, Longitude);
GO

-- Create fact table for warehouse operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    QuantityHandled INT,
    DwellTimeMinutes DECIMAL(10,2),
    AssignedZone VARCHAR(10),
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    BatchNumber VARCHAR(50),
    QualityScore DECIMAL(3,2), -- 0-10
    CostPerUnit DECIMAL(10,4)
);
GO

CREATE INDEX IX_FactWH_Time_Product ON FactWarehouseOperations(TimeKey, ProductKey);
CREATE INDEX IX_FactWH_Operation_Type ON FactWarehouseOperations(OperationType);
CREATE INDEX IX_FactWH_DwellTime ON FactWarehouseOperations(DwellTimeMinutes);
GO

-- Create fact table for fleet trips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    OriginKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    PlannedDistance DECIMAL(10,2), -- km
    ActualDistance DECIMAL(10,2),
    FuelConsumed DECIMAL(10,2), -- liters
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    AverageSpeed DECIMAL(6,2),
    MaxSpeed DECIMAL(6,2),
    DelayMinutes DECIMAL(10,2),
    DelayReason VARCHAR(100), -- 'Traffic', 'Weather', 'Mechanical', 'Loading'
    FuelEfficiency AS (ActualDistance / NULLIF(FuelConsumed, 0)) PERSISTED,
    TripCost DECIMAL(10,2)
);
GO

CREATE INDEX IX_FactFleet_Time_Vehicle ON FactFleetTrips(TimeKey, VehicleID);
CREATE INDEX IX_FactFleet_Route ON FactFleetTrips(OriginKey, DestinationKey);
CREATE INDEX IX_FactFleet_DelayReason ON FactFleetTrips(DelayReason) WHERE DelayMinutes > 0;
GO

-- Create bridge table for many-to-many relationships
CREATE TABLE BridgeRouteStorage (
    BridgeKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OperationKey BIGINT FOREIGN KEY REFERENCES FactWarehouseOperations(OperationKey),
    LinkType VARCHAR(50), -- 'Pickup', 'Delivery'
    LinkSequence INT
);
GO

CREATE INDEX IX_Bridge_Trip ON BridgeRouteStorage(TripKey);
CREATE INDEX IX_Bridge_Operation ON BridgeRouteStorage(OperationKey);
GO
```

### Step 3: Load Sample Data

```sql
-- Populate dimension tables with sample data
-- Time dimension generator (15-minute intervals for 2 years)
DECLARE @StartDate DATETIME2 = '2025-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2026-12-31 23:45:00';
DECLARE @CurrentTime DATETIME2 = @StartDate;

WHILE @CurrentTime <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, TimeValue, Hour, DayOfWeek, DayOfMonth, Month, Quarter, FiscalYear, FiscalPeriod, IsWeekend, TimeBlock)
    VALUES (
        CAST(FORMAT(@CurrentTime, 'yyyyMMddHHmm') AS INT),
        @CurrentTime,
        DATEPART(HOUR, @CurrentTime),
        DATEPART(WEEKDAY, @CurrentTime),
        DATEPART(DAY, @CurrentTime),
        DATEPART(MONTH, @CurrentTime),
        DATEPART(QUARTER, @CurrentTime),
        YEAR(@CurrentTime),
        MONTH(@CurrentTime),
        CASE WHEN DATEPART(WEEKDAY, @CurrentTime) IN (1, 7) THEN 1 ELSE 0 END,
        FORMAT(@CurrentTime, 'HH:mm') + '-' + FORMAT(DATEADD(MINUTE, 15, @CurrentTime), 'HH:mm')
    );
    
    SET @CurrentTime = DATEADD(MINUTE, 15, @CurrentTime);
END;
GO

-- Sample product data
INSERT INTO DimProductGravity (SKU, ProductName, ProductCategory, GravityScore, VelocityClass, FragilityIndex, UnitValue, OptimalZone)
VALUES
    ('SKU-001', 'Premium Widget A', 'Electronics', 92.5, 'Fast', 8.5, 249.99, 'A'),
    ('SKU-002', 'Standard Widget B', 'Electronics', 65.0, 'Medium', 6.0, 89.99, 'B'),
    ('SKU-003', 'Bulk Component C', 'Industrial', 35.0, 'Slow', 2.0, 12.50, 'C'),
    ('SKU-004', 'Fragile Glassware', 'Home Goods', 78.0, 'Fast', 9.8, 45.00, 'A'),
    ('SKU-005', 'Heavy Machinery Part', 'Industrial', 45.0, 'Slow', 1.5, 1200.00, 'C');
GO

-- Sample geography data
INSERT INTO DimGeography (LocationCode, LocationName, LocationType, City, Region, Country, Continent, Latitude, Longitude)
VALUES
    ('WH-NYC-01', 'New York Distribution Center', 'Warehouse', 'New York', 'NY', 'USA', 'North America', 40.7128, -74.0060),
    ('WH-LA-01', 'Los Angeles Hub', 'Warehouse', 'Los Angeles', 'CA', 'USA', 'North America', 34.0522, -118.2437),
    ('DC-CHI-01', 'Chicago Cross Dock', 'Distribution Center', 'Chicago', 'IL', 'USA', 'North America', 41.8781, -87.6298),
    ('NODE-TX-01', 'Dallas Route Node', 'Route Node', 'Dallas', 'TX', 'USA', 'North America', 32.7767, -96.7970);
GO
```

### Step 4: Configure Data Connections

Create a configuration file for your data sources:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "integrated",
    "connection_timeout": 30
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}",
      "refresh_interval_minutes": 15
    },
    "telemetry_feed": {
      "endpoint": "${TELEMETRY_ENDPOINT}",
      "api_key": "${TELEMETRY_API_KEY}",
      "refresh_interval_minutes": 5
    },
    "weather_api": {
      "endpoint": "https://api.weather.com/v3",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "alert_settings": {
    "smtp_server": "${SMTP_SERVER}",
    "smtp_port": 587,
    "from_address": "${ALERT_EMAIL_FROM}",
    "alert_recipients": ["${ALERT_EMAIL_TO}"]
  }
}
```

### Step 5: Set Up Incremental Loading

```sql
-- Create stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assuming external source table exists (via linked server or staging)
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        ProductKey,
        WarehouseKey,
        OperationType,
        OperationStartTime,
        OperationEndTime,
        QuantityHandled,
        DwellTimeMinutes,
        AssignedZone,
        OperatorID,
        EquipmentID,
        BatchNumber,
        QualityScore,
        CostPerUnit
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        src.OperationType,
        src.StartTime,
        src.EndTime,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, COALESCE(src.EndTime, GETDATE())),
        src.Zone,
        src.OperatorID,
        src.EquipmentID,
        src.BatchNumber,
        src.QualityScore,
        src.UnitCost
    FROM StagingWarehouseOps src
    INNER JOIN DimTime t ON t.TimeValue = DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, 0, src.StartTime) / 15) * 15, 0)
    INNER JOIN DimProductGravity p ON p.SKU = src.SKU
    INNER JOIN DimGeography g ON g.LocationCode = src.WarehouseCode
    WHERE src.LastModified > @LastLoadTime;
    
    -- Update metadata table
    UPDATE ETLMetadata
    SET LastLoadTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO

-- Create stored procedure for fleet trips
CREATE PROCEDURE usp_LoadFleetTrips
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey,
        VehicleID,
        DriverID,
        OriginKey,
        DestinationKey,
        TripStartTime,
        TripEndTime,
        PlannedDistance,
        ActualDistance,
        FuelConsumed,
        IdleTimeMinutes,
        LoadingTimeMinutes,
        UnloadingTimeMinutes,
        AverageSpeed,
        MaxSpeed,
        DelayMinutes,
        DelayReason,
        TripCost
    )
    SELECT
        t.TimeKey,
        src.VehicleID,
        src.DriverID,
        go.GeographyKey,
        gd.GeographyKey,
        src.StartTime,
        src.EndTime,
        src.PlannedKm,
        src.ActualKm,
        src.FuelLiters,
        src.IdleMinutes,
        src.LoadMinutes,
        src.UnloadMinutes,
        src.AvgSpeed,
        src.MaxSpeed,
        DATEDIFF(MINUTE, src.PlannedEndTime, src.EndTime),
        src.DelayReason,
        src.TotalCost
    FROM StagingFleetTrips src
    INNER JOIN DimTime t ON t.TimeValue = DATEADD(MINUTE,
        (DATEDIFF(MINUTE, 0, src.StartTime) / 15) * 15, 0)
    INNER JOIN DimGeography go ON go.LocationCode = src.OriginCode
    INNER JOIN DimGeography gd ON gd.LocationCode = src.DestinationCode
    WHERE src.LastModified > @LastLoadTime;
    
    UPDATE ETLMetadata
    SET LastLoadTime = GETDATE()
    WHERE TableName = 'FactFleetTrips';
END;
GO
```

## Power BI Configuration

### Step 1: Open the Template

Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop.

### Step 2: Configure Data Source Connection

When prompted, enter your SQL Server connection details:

- **Server**: Your SQL Server hostname
- **Database**: LogiFleetPulse

### Step 3: Key Measures and DAX Formulas

The template includes pre-built measures. Here are examples you can customize:

```dax
-- Measure: Average Dwell Time
Avg Dwell Time (Hours) = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60

-- Measure: Fleet Efficiency Score (composite)
Fleet Efficiency Score = 
VAR AvgIdlePercent = 
    DIVIDE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        AVERAGE(DATEDIFF(FactFleetTrips[TripStartTime], FactFleetTrips[TripEndTime], MINUTE))
    )
VAR FuelEfficiency = AVERAGE(FactFleetTrips[FuelEfficiency])
VAR OnTimePercent = 
    DIVIDE(
        CALCULATE(COUNT(FactFleetTrips[TripKey]), FactFleetTrips[DelayMinutes] <= 0),
        COUNT(FactFleetTrips[TripKey])
    )
RETURN
    (1 - AvgIdlePercent) * 0.3 + 
    DIVIDE(FuelEfficiency, 10) * 0.3 + 
    OnTimePercent * 0.4

-- Measure: Cross-Fact KPI - Cost per Delivered Unit
Cost Per Delivered Unit = 
VAR FleetCost = SUM(FactFleetTrips[TripCost])
VAR WarehouseCost = SUMX(FactWarehouseOperations, [QuantityHandled] * [CostPerUnit])
VAR TotalUnits = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
RETURN
    DIVIDE(FleetCost + WarehouseCost, TotalUnits)

-- Measure: Warehouse Gravity Zone Performance
Zone Performance Index = 
VAR ActualZone = SELECTEDVALUE(FactWarehouseOperations[AssignedZone])
VAR OptimalZone = 
    CALCULATE(
        SELECTEDVALUE(DimProductGravity[OptimalZone]),
        FILTER(
            DimProductGravity,
            DimProductGravity[ProductKey] = SELECTEDVALUE(FactWarehouseOperations[ProductKey])
        )
    )
VAR ZoneMatch = IF(ActualZone = OptimalZone, 1, 0)
VAR AvgDwellInZone = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FactWarehouseOperations[AssignedZone] = ActualZone
    )
RETURN
    ZoneMatch * 100 - AvgDwellInZone * 0.5

-- Measure: Predictive Bottleneck Index
Bottleneck Risk Index = 
VAR HighDwellCount = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[DwellTimeMinutes] > 240 -- 4 hours
    )
VAR HighIdleFleet = 
    CALCULATE(
        COUNT(FactFleetTrips[TripKey]),
        FactFleetTrips[IdleTimeMinutes] > AVERAGE(FactFleetTrips[IdleTimeMinutes]) * 1.5
    )
VAR DelayTrend = 
    VAR CurrentPeriod = CALCULATE(AVERAGE(FactFleetTrips[DelayMinutes]), DATESINPERIOD(DimTime[TimeValue], MAX(DimTime[TimeValue]), -7, DAY))
    VAR PriorPeriod = CALCULATE(AVERAGE(FactFleetTrips[DelayMinutes]), DATESINPERIOD(DimTime[TimeValue], MAX(DimTime[TimeValue]), -14, DAY))
    RETURN DIVIDE(CurrentPeriod - PriorPeriod, PriorPeriod, 0)
RETURN
    (HighDwellCount * 10) + (HighIdleFleet * 8) + (DelayTrend * 100)
```

### Step 4: Row-Level Security (RLS)

Configure role-based access:

```dax
-- Create role: Regional Manager (sees only their region)
[Region] = USERNAME()

-- Create role: Warehouse Manager (sees only their warehouse)
[LocationCode] IN {"WH-NYC-01", "WH-LA-01"} -- Replace with dynamic lookup

-- Create role: Executive (sees everything)
1=1
```

Apply roles in Power BI Desktop → Modeling → Manage Roles.

## Common Analytical Queries

### Query 1: Top 10 High-Gravity Products by Dwell Time

```sql
SELECT TOP 10
    p.SKU,
    p.ProductName,
    p.GravityScore,
    AVG(f.DwellTimeMinutes) AS AvgDwellMinutes,
    COUNT(*) AS OperationCount,
    f.AssignedZone,
    p.OptimalZone,
    CASE 
        WHEN f.AssignedZone = p.OptimalZone THEN 'Optimal'
        ELSE 'Suboptimal'
    END AS ZoneAlignment
FROM FactWarehouseOperations f
INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
WHERE f.OperationType IN ('Putaway', 'Picking')
    AND f.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, f.AssignedZone, p.OptimalZone
ORDER BY p.GravityScore DESC, AvgDwellMinutes DESC;
```

### Query 2: Fleet Utilization by Hour of Day

```sql
SELECT 
    t.Hour,
    COUNT(DISTINCT f.VehicleID) AS ActiveVehicles,
    AVG(f.ActualDistance) AS AvgDistanceKm,
    AVG(f.FuelEfficiency) AS AvgFuelEfficiency,
    AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(CASE WHEN f.DelayMinutes > 0 THEN 1 ELSE 0 END) AS DelayedTrips,
    AVG(f.TripCost) AS AvgTripCost
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE f.TripStartTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY t.Hour
ORDER BY t.Hour;
```

### Query 3: Cross-Fact Analysis - Shipment Delays Correlated with Warehouse Dwell

```sql
WITH WarehouseDwell AS (
    SELECT 
        w.BatchNumber,
        w.ProductKey,
        MAX(w.DwellTimeMinutes) AS MaxDwellMinutes,
        MIN(w.OperationStartTime) AS FirstOperation
    FROM FactWarehouseOperations w
    WHERE w.OperationType = 'Shipping'
        AND w.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY w.BatchNumber, w.ProductKey
),
LinkedTrips AS (
    SELECT 
        f.TripKey,
        f.VehicleID,
        f.DelayMinutes,
        f.DelayReason,
        b.OperationKey,
        w.BatchNumber,
        w.MaxDwellMinutes
    FROM FactFleetTrips f
    INNER JOIN BridgeRouteStorage b ON f.TripKey = b.TripKey
    INNER JOIN FactWarehouseOperations wo ON b.OperationKey = wo.OperationKey
    INNER JOIN WarehouseDwell w ON wo.BatchNumber = w.BatchNumber
    WHERE b.LinkType = 'Pickup'
)
SELECT 
    CASE 
        WHEN MaxDwellMinutes < 60 THEN '< 1 hour'
        WHEN MaxDwellMinutes < 240 THEN '1-4 hours'
        WHEN MaxDwellMinutes < 1440 THEN '4-24 hours'
        ELSE '> 24 hours'
    END AS DwellBucket,
    AVG(DelayMinutes) AS AvgDeliveryDelay,
    COUNT(*) AS TripCount,
    SUM(CASE WHEN DelayMinutes > 30 THEN 1 ELSE 0 END) AS SignificantDelays
FROM LinkedTrips
GROUP BY 
    CASE 
        WHEN MaxDwellMinutes < 60 THEN '< 1 hour'
        WHEN MaxDwellMinutes < 240 THEN '1-4 hours'
        WHEN MaxDwellMinutes < 1440 THEN '4-24 hours'
        ELSE '> 24 hours'
    END
ORDER BY AvgDeliveryDelay DESC;
```

### Query 4: Warehouse Zone Rebalancing Recommendations

```sql
WITH ProductMovement AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.GravityScore,
        p.OptimalZone,
        f.AssignedZone AS CurrentZone,
        COUNT(*) AS OperationFrequency,
        AVG(f.DwellTimeMinutes) AS AvgDwell,
        SUM(f.QuantityHandled) AS TotalVolume
    FROM FactWarehouseOperations f
    INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
    WHERE f.OperationStartTime >= DATEADD(DAY, -90, GETDATE())
        AND f.OperationType = 'Picking'
    GROUP BY p.ProductKey, p.SKU, p.GravityScore, p.OptimalZone, f.AssignedZone
)
SELECT 
    SKU,
    GravityScore,
    CurrentZone,
    OptimalZone,
    OperationFrequency,
    AvgDwell,
    TotalVolume,
    CASE 
        WHEN CurrentZone <> OptimalZone AND OperationFrequency > 50 THEN 'High Priority'
        WHEN CurrentZone <> OptimalZone AND OperationFrequency > 20 THEN 'Medium Priority'
        WHEN CurrentZone <> OptimalZone THEN 'Low Priority'
        ELSE 'No Action'
    END AS RebalancePriority,
    CAST((OperationFrequency * 0.5 + TotalVolume * 0.3 + (100 - AvgDwell) * 0.2) AS DECIMAL(10,2)) AS RebalanceScore
FROM ProductMovement
WHERE CurrentZone <> OptimalZone
ORDER BY RebalanceScore DESC;
```

## Automated Alerting System

### Set Up Alert Stored Procedure

```sql
CREATE PROCEDURE usp_GenerateLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @AlertSubject NVARCHAR(200);
    
    -- Alert 1: Excessive Fleet Idle Time
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips 
        WHERE TripStartTime >= DATEADD(HOUR, -4, GETDATE())
            AND IdleTimeMinutes > (DATEDIFF(MINUTE, TripStartTime, TripEndTime) * 0.20)
    )
    BEGIN
        SET @AlertSubject = 'ALERT: High Fleet Idle Time Detected';
        SET @AlertMessage = 'Multiple vehicles exceeded 20% idle time threshold in the last 4 hours. Review required.';
        
        -- Insert into AlertLog table
        INSERT INTO AlertLog (AlertType, AlertSubject, AlertMessage, AlertTime)
        VALUES ('FleetIdle', @AlertSubject, @AlertMessage, GETDATE());
        
        -- Send email (requires Database Mail configured)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_TO}',
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END;
    
    -- Alert 2: High-Gravity Products in Wrong Zone
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations f
        INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
        WHERE f.OperationStartTime >= DATEADD(HOUR, -8, GETDATE())
            AND p.GravityScore > 80
            AND f.AssignedZone <> p.OptimalZone
            AND f.OperationType = 'Picking'
        GROUP BY f.AssignedZone, p.OptimalZone
        HAVING COUNT(*) > 10
    )
    BEGIN
        SET @AlertSubject = 'ALERT: High-Gravity SKUs Misplaced';
        SET @AlertMessage = 'High-priority products are being picked from suboptimal zones. Zone rebalancing recommended.';
        
        INSERT INTO AlertLog (AlertType, AlertSubject, AlertMessage, AlertTime)
        VALUES ('ZoneMisalignment', @AlertSubject, @AlertMessage, GETDATE());
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_TO}',
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END;
    
    -- Alert 3: Bottleneck Prediction
    DECLARE @BottleneckIndex DECIMAL(10,2);
    
    SELECT @BottleneckIndex = 
        (SELECT COUNT(*) FROM FactWarehouseOperations WHERE DwellTimeMinutes > 240 AND OperationStartTime >= DATEADD(HOUR, -4, GET
