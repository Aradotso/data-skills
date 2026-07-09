---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse for fleet tracking, warehouse operations, and supply chain KPI analytics
triggers:
  - "set up logistics analytics dashboard"
  - "create supply chain data warehouse"
  - "build fleet tracking analytics with Power BI"
  - "implement warehouse operations reporting"
  - "configure logistics intelligence platform"
  - "deploy multi-fact star schema for supply chain"
  - "integrate fleet telemetry with warehouse data"
  - "create cross-modal logistics KPI dashboard"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It provides a multi-fact star schema data warehouse that unifies warehouse operations, fleet telemetry, inventory management, and supply chain KPIs into a single semantic layer. The platform enables real-time cross-fact analytics like correlating warehouse dwell time with fleet idle costs, predictive bottleneck detection, and adaptive fleet maintenance prioritization.

**Core Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse gravity zone optimization
- Fleet telemetry integration and route optimization
- Predictive bottleneck detection
- Cross-fact KPI harmonization
- Real-time Power BI dashboards with 15-minute refresh cycles

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to source systems: WMS, TMS, telematics APIs

### Database Schema Deployment

```sql
-- 1. Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- 2. Create dimension tables

-- Time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Day INT NOT NULL,
    Hour INT NOT NULL,
    Minute15Bucket INT NOT NULL,
    DayOfWeek VARCHAR(20),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(20),
    INDEX IX_DimTime_FullDateTime (FullDateTime)
);

-- Geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, RouteNode, CrossDock
    Address VARCHAR(500),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    Region VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    INDEX IX_DimGeography_LocationID (LocationID)
);

-- Product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitCost DECIMAL(12,2),
    UnitPrice DECIMAL(12,2),
    Weight DECIMAL(10,2),
    Volume DECIMAL(10,2),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility
    VelocityClass VARCHAR(20), -- Fast, Medium, Slow
    INDEX IX_DimProduct_SKU (SKU)
);

-- Supplier reliability dimension
CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200),
    AvgLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    OnTimeDeliveryRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier VARCHAR(20), -- Gold, Silver, Bronze
    INDEX IX_DimSupplier_SupplierID (SupplierID)
);

-- Fleet vehicle dimension
CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) NOT NULL,
    VehicleType VARCHAR(50),
    Make VARCHAR(100),
    Model VARCHAR(100),
    Year INT,
    Capacity DECIMAL(10,2),
    FuelType VARCHAR(50),
    LastMaintenanceDate DATE,
    NextMaintenanceDue DATE,
    MaintenancePriorityScore DECIMAL(5,2),
    INDEX IX_DimVehicle_VehicleID (VehicleID)
);

-- 3. Create fact tables

-- Warehouse operations fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT,
    DwellTimeMinutes INT,
    PickTimeSeconds INT,
    StorageZone VARCHAR(50),
    GravityZone VARCHAR(20), -- High, Medium, Low
    OperatorID VARCHAR(50),
    BatchID VARCHAR(100),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey),
    INDEX IX_FactWarehouse_Time_Geo (TimeKey, GeographyKey)
);

-- Fleet trips fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DriverID VARCHAR(50),
    RouteID VARCHAR(100),
    DistanceMiles DECIMAL(10,2),
    DurationMinutes INT,
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TotalLoadWeight DECIMAL(12,2),
    DelayMinutes INT,
    DelayReason VARCHAR(200),
    WeatherCondition VARCHAR(50),
    FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_FactFleet_StartTime_Vehicle (StartTimeKey, VehicleKey)
);

-- Cross-dock operations fact
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    InboundTimeKey INT NOT NULL,
    OutboundTimeKey INT,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    GeographyKey INT NOT NULL,
    Quantity INT,
    DockDurationMinutes INT,
    FOREIGN KEY (InboundTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OutboundTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_FactCrossDock_Time_Product (InboundTimeKey, ProductKey)
);
```

### Time Dimension Population

```sql
-- Populate DimTime with 15-minute buckets for 5 years
DECLARE @StartDate DATETIME2 = '2024-01-01';
DECLARE @EndDate DATETIME2 = '2029-12-31';
DECLARE @CurrentDate DATETIME2 = @StartDate;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey,
        FullDateTime,
        Year,
        Quarter,
        Month,
        Day,
        Hour,
        Minute15Bucket,
        DayOfWeek,
        IsWeekend,
        FiscalPeriod
    )
    VALUES (
        CAST(FORMAT(@CurrentDate, 'yyyyMMddHHmm') AS INT),
        @CurrentDate,
        YEAR(@CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        MONTH(@CurrentDate),
        DAY(@CurrentDate),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(MINUTE, @CurrentDate) / 15,
        DATENAME(WEEKDAY, @CurrentDate),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END,
        'FY' + CAST(YEAR(DATEADD(MONTH, 6, @CurrentDate)) AS VARCHAR(4)) + '-Q' + CAST(DATEPART(QUARTER, DATEADD(MONTH, 6, @CurrentDate)) AS VARCHAR(1))
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
END;
```

## Configuration

### Connection String Setup

Create a `config.json` file for your data source connections:

```json
{
  "database": {
    "server": "ENV:SQL_SERVER_HOST",
    "database": "LogiFleetPulse",
    "username": "ENV:SQL_USERNAME",
    "password": "ENV:SQL_PASSWORD",
    "encrypt": true,
    "trustServerCertificate": false
  },
  "dataSources": {
    "wmsApi": {
      "endpoint": "ENV:WMS_API_ENDPOINT",
      "apiKey": "ENV:WMS_API_KEY",
      "refreshIntervalMinutes": 15
    },
    "telematicsApi": {
      "endpoint": "ENV:TELEMATICS_API_ENDPOINT",
      "apiKey": "ENV:TELEMATICS_API_KEY",
      "refreshIntervalMinutes": 5
    },
    "weatherApi": {
      "endpoint": "ENV:WEATHER_API_ENDPOINT",
      "apiKey": "ENV:WEATHER_API_KEY",
      "refreshIntervalMinutes": 30
    }
  },
  "powerBI": {
    "workspaceId": "ENV:POWERBI_WORKSPACE_ID",
    "datasetId": "ENV:POWERBI_DATASET_ID",
    "refreshSchedule": "*/15 * * * *"
  }
}
```

### ETL Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        GeographyKey,
        ProductKey,
        SupplierKey,
        OperationType,
        Quantity,
        DwellTimeMinutes,
        PickTimeSeconds,
        StorageZone,
        GravityZone,
        OperatorID,
        BatchID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        stg.PickTimeSeconds,
        stg.StorageZone,
        CASE 
            WHEN p.GravityScore >= 80 THEN 'High'
            WHEN p.GravityScore >= 50 THEN 'Medium'
            ELSE 'Low'
        END AS GravityZone,
        stg.OperatorID,
        stg.BatchID
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON FORMAT(stg.OperationTime, 'yyyyMMddHHmm') = t.TimeKey
    INNER JOIN DimGeography g ON stg.WarehouseID = g.LocationID
    INNER JOIN DimProduct p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplier s ON stg.SupplierID = s.SupplierID
    WHERE stg.LoadedTimestamp > @LastLoadTimestamp;
    
    -- Update gravity scores based on recent velocity
    UPDATE p
    SET GravityScore = (
        SELECT 
            (COUNT(*) * 1.0 / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0)) * -- Velocity
            p.UnitPrice * -- Value
            (CASE WHEN p.IsFragile = 1 THEN 1.5 ELSE 1.0 END) -- Fragility factor
        FROM FactWarehouseOperations f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE f.ProductKey = p.ProductKey
            AND t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    )
    FROM DimProduct p;
END;
GO

-- Load fleet trips with delay analysis
CREATE PROCEDURE sp_LoadFleetTrips
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        StartTimeKey,
        EndTimeKey,
        VehicleKey,
        OriginGeographyKey,
        DestinationGeographyKey,
        DriverID,
        RouteID,
        DistanceMiles,
        DurationMinutes,
        FuelConsumedGallons,
        IdleTimeMinutes,
        LoadingTimeMinutes,
        UnloadingTimeMinutes,
        TotalLoadWeight,
        DelayMinutes,
        DelayReason,
        WeatherCondition
    )
    SELECT 
        tStart.TimeKey,
        tEnd.TimeKey,
        v.VehicleKey,
        gOrigin.GeographyKey,
        gDest.GeographyKey,
        stg.DriverID,
        stg.RouteID,
        stg.DistanceMiles,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime),
        stg.FuelConsumedGallons,
        stg.IdleTimeMinutes,
        stg.LoadingTimeMinutes,
        stg.UnloadingTimeMinutes,
        stg.TotalLoadWeight,
        CASE 
            WHEN DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) > stg.PlannedDurationMinutes 
            THEN DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) - stg.PlannedDurationMinutes
            ELSE 0
        END AS DelayMinutes,
        stg.DelayReason,
        stg.WeatherCondition
    FROM StagingFleetTrips stg
    INNER JOIN DimTime tStart ON FORMAT(stg.StartTime, 'yyyyMMddHHmm') = tStart.TimeKey
    INNER JOIN DimTime tEnd ON FORMAT(stg.EndTime, 'yyyyMMddHHmm') = tEnd.TimeKey
    INNER JOIN DimVehicle v ON stg.VehicleID = v.VehicleID
    INNER JOIN DimGeography gOrigin ON stg.OriginLocationID = gOrigin.LocationID
    INNER JOIN DimGeography gDest ON stg.DestinationLocationID = gDest.LocationID
    WHERE stg.LoadedTimestamp > @LastLoadTimestamp;
END;
GO
```

## Power BI Integration

### Creating the Semantic Model

1. **Open Power BI Desktop** and connect to your SQL Server:

```powerquery
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"),
        "LogiFleetPulse",
        [
            Query = "SELECT * FROM FactWarehouseOperations",
            CreateNavigationProperties = false
        ]
    )
in
    Source
```

2. **Define relationships** in Power BI Model view:
   - FactWarehouseOperations[TimeKey] → DimTime[TimeKey] (Many-to-One)
   - FactWarehouseOperations[GeographyKey] → DimGeography[GeographyKey] (Many-to-One)
   - FactWarehouseOperations[ProductKey] → DimProduct[ProductKey] (Many-to-One)
   - FactFleetTrips[StartTimeKey] → DimTime[TimeKey] (Many-to-One, role-playing)
   - FactFleetTrips[VehicleKey] → DimVehicle[VehicleKey] (Many-to-One)

3. **Create cross-fact measures** using DAX:

```dax
// Total warehouse dwell time in hours
TotalDwellTimeHours = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Fleet idle cost (assuming $50/hour average)
FleetIdleCost = 
SUMX(
    FactFleetTrips,
    (FactFleetTrips[IdleTimeMinutes] / 60) * 50
)

// Cross-fact KPI: Dwell time per SKU vs Fleet idle cost per route
DwellToIdleCostRatio = 
DIVIDE(
    [TotalDwellTimeHours],
    [FleetIdleCost],
    0
)

// Warehouse gravity zone efficiency
GravityZoneEfficiency = 
VAR HighGravityPickTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[PickTimeSeconds]),
        FactWarehouseOperations[GravityZone] = "High"
    )
VAR LowGravityPickTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[PickTimeSeconds]),
        FactWarehouseOperations[GravityZone] = "Low"
    )
RETURN
    DIVIDE(LowGravityPickTime, HighGravityPickTime, 1)

// Predictive bottleneck score
BottleneckRiskScore = 
VAR CurrentDwellTime = [TotalDwellTimeHours]
VAR HistoricalAvgDwell = 
    CALCULATE(
        [TotalDwellTimeHours],
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -30, DAY)
    ) / 30
VAR DwellVariance = CurrentDwellTime - HistoricalAvgDwell
VAR FleetUtilization = 
    DIVIDE(
        SUMX(FactFleetTrips, FactFleetTrips[DurationMinutes] - FactFleetTrips[IdleTimeMinutes]),
        SUMX(FactFleetTrips, FactFleetTrips[DurationMinutes]),
        0
    )
RETURN
    (DwellVariance * 0.6) + ((1 - FleetUtilization) * 100 * 0.4)

// Fleet maintenance priority
MaintenancePriorityIndex = 
SUMX(
    VALUES(DimVehicle[VehicleKey]),
    VAR VehicleRevenue = 
        CALCULATE(
            SUMX(
                FactFleetTrips,
                FactFleetTrips[TotalLoadWeight] * 2.5 // Assume $2.5 revenue per lb
            ),
            FILTER(
                FactFleetTrips,
                FactFleetTrips[VehicleKey] = DimVehicle[VehicleKey]
            )
        )
    VAR MaintenanceScore = 
        CALCULATE(
            MAX(DimVehicle[MaintenancePriorityScore]),
            FILTER(
                DimVehicle,
                DimVehicle[VehicleKey] = EARLIER(DimVehicle[VehicleKey])
            )
        )
    RETURN
        VehicleRevenue * MaintenanceScore
)
```

### Dashboard Examples

**Page 1: Warehouse Operations Dashboard**
- KPI cards: Total operations, avg dwell time, gravity zone distribution
- Line chart: Dwell time trend by gravity zone (last 30 days)
- Heatmap: Storage zone efficiency matrix
- Table: Top 10 slow-moving SKUs with recommendations

**Page 2: Fleet Analytics Dashboard**
- KPI cards: Total trips, fuel consumption, idle time percentage
- Map visual: Route density with delay hotspots
- Bar chart: Top 10 routes by delay minutes
- Scatter plot: Distance vs fuel efficiency by vehicle type

**Page 3: Cross-Modal Intelligence**
- Combined metric: Warehouse dwell impact on fleet delivery windows
- Waterfall chart: Bottleneck risk score decomposition
- Matrix: Product category vs delivery performance
- Gauge: Fleet utilization vs warehouse capacity correlation

## Key SQL Analytics Patterns

### Finding Delayed Shipments from High-Dwell Zones

```sql
-- Shipments delayed due to weather that originated from cold-storage 
-- zones with >72 hours dwell time in last quarter
SELECT 
    ft.RouteID,
    g.LocationName AS Origin,
    p.ProductName,
    wo.DwellTimeMinutes / 60.0 AS DwellTimeHours,
    ft.DelayMinutes,
    ft.WeatherCondition,
    t.FullDateTime AS ShipmentDate
FROM FactFleetTrips ft
INNER JOIN FactWarehouseOperations wo 
    ON ft.OriginGeographyKey = wo.GeographyKey
    AND wo.ProductKey IN (
        SELECT ProductKey 
        FROM DimProduct 
        WHERE RequiresRefrigeration = 1
    )
INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
WHERE 
    wo.DwellTimeMinutes > 4320 -- 72 hours
    AND ft.DelayMinutes > 0
    AND ft.WeatherCondition IS NOT NULL
    AND t.FullDateTime >= DATEADD(QUARTER, -1, GETDATE())
ORDER BY ft.DelayMinutes DESC;
```

### Calculating Optimal Gravity Zone Assignments

```sql
-- Recommend storage zone reassignments based on gravity score drift
WITH ProductVelocity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.GravityScore AS CurrentGravityScore,
        COUNT(*) AS OperationsLast30Days,
        AVG(wo.PickTimeSeconds) AS AvgPickTime,
        wo.StorageZone AS CurrentZone
    FROM DimProduct p
    INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.GravityScore, wo.StorageZone
),
ZonePerformance AS (
    SELECT 
        StorageZone,
        AVG(PickTimeSeconds) AS AvgZonePickTime
    FROM FactWarehouseOperations
    GROUP BY StorageZone
)
SELECT 
    pv.SKU,
    pv.CurrentZone,
    pv.CurrentGravityScore,
    pv.OperationsLast30Days,
    pv.AvgPickTime,
    zp.AvgZonePickTime,
    CASE 
        WHEN pv.CurrentGravityScore >= 80 AND pv.AvgPickTime > zp.AvgZonePickTime * 1.2 
            THEN 'Move to High-Gravity Zone'
        WHEN pv.CurrentGravityScore < 50 AND pv.AvgPickTime < zp.AvgZonePickTime * 0.8 
            THEN 'Move to Low-Gravity Zone'
        ELSE 'No Change Needed'
    END AS Recommendation
FROM ProductVelocity pv
INNER JOIN ZonePerformance zp ON pv.CurrentZone = zp.StorageZone
WHERE pv.OperationsLast30Days > 10 -- Only consider active SKUs
ORDER BY pv.CurrentGravityScore DESC;
```

### Fleet Route Optimization Analysis

```sql
-- Identify inefficient routes by comparing actual vs theoretical performance
WITH RouteMetrics AS (
    SELECT 
        ft.RouteID,
        COUNT(*) AS TotalTrips,
        AVG(ft.DistanceMiles) AS AvgDistance,
        AVG(ft.DurationMinutes) AS AvgDuration,
        AVG(ft.FuelConsumedGallons) AS AvgFuel,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(ft.DelayMinutes) AS AvgDelay,
        SUM(ft.TotalLoadWeight) AS TotalCargoWeight
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY ft.RouteID
),
TheoreticalPerformance AS (
    SELECT 
        RouteID,
        AvgDistance,
        AvgDistance / 55.0 AS TheoreticalDurationHours, -- Assume 55 mph avg
        AvgDistance / 8.0 AS TheoreticalFuelGallons, -- Assume 8 mpg
        TotalTrips
    FROM RouteMetrics
)
SELECT 
    rm.RouteID,
    rm.TotalTrips,
    rm.AvgDistance,
    rm.AvgDuration / 60.0 AS AvgDurationHours,
    tp.TheoreticalDurationHours,
    (rm.AvgDuration / 60.0 - tp.TheoreticalDurationHours) AS DurationVarianceHours,
    rm.AvgFuel,
    tp.TheoreticalFuelGallons,
    (rm.AvgFuel - tp.TheoreticalFuelGallons) AS FuelVarianceGallons,
    rm.AvgIdleTime,
    rm.AvgDelay,
    (rm.AvgIdleTime + rm.AvgDelay) / rm.AvgDuration * 100 AS WastePercentage
FROM RouteMetrics rm
INNER JOIN TheoreticalPerformance tp ON rm.RouteID = tp.RouteID
WHERE (rm.AvgDuration / 60.0 - tp.TheoreticalDurationHours) > 1 -- More than 1 hour variance
    OR (rm.AvgFuel - tp.TheoreticalFuelGallons) > 3 -- More than 3 gallons variance
ORDER BY WastePercentage DESC;
```

## Automated Alerting System

### Setting Up Threshold Alerts

```sql
-- Create alerts configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(200),
    MetricType VARCHAR(100), -- DwellTime, IdleTime, DelayRate, etc.
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- >, <, =, !=
    EvaluationWindowMinutes INT,
    AlertLevel VARCHAR(20), -- Info, Warning, Critical
    RecipientEmails VARCHAR(MAX),
    IsActive BIT
);

-- Stored procedure to evaluate alerts
CREATE PROCEDURE sp_EvaluateAlerts
AS
BEGIN
    DECLARE @AlertID INT, @MetricType VARCHAR(100), @ThresholdValue DECIMAL(10,2);
    DECLARE @ComparisonOperator VARCHAR(10), @WindowMinutes INT, @AlertLevel VARCHAR(20);
    DECLARE @Recipients VARCHAR(MAX), @CurrentValue DECIMAL(10,2), @AlertMessage NVARCHAR(MAX);
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, MetricType, ThresholdValue, ComparisonOperator, 
           EvaluationWindowMinutes, AlertLevel, RecipientEmails
    FROM AlertConfiguration
    WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricType, @ThresholdValue, 
                                       @ComparisonOperator, @WindowMinutes, 
                                       @AlertLevel, @Recipients;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Calculate current metric value based on type
        IF @MetricType = 'AvgDwellTimeHours'
        BEGIN
            SELECT @CurrentValue = AVG(DwellTimeMinutes) / 60.0
            FROM FactWarehouseOperations wo
            INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
            WHERE t.FullDateTime >= DATEADD(MINUTE, -@WindowMinutes, GETDATE());
        END
        ELSE IF @MetricType = 'FleetIdlePercentage'
        BEGIN
            SELECT @CurrentValue = 
                SUM(IdleTimeMinutes) * 100.0 / NULLIF(SUM(DurationMinutes), 0)
            FROM FactFleetTrips ft
            INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
            WHERE t.FullDateTime >= DATEADD(MINUTE, -@WindowMinutes, GETDATE());
        END
        
        -- Evaluate threshold condition
        IF (@ComparisonOperator = '>' AND @CurrentValue > @ThresholdValue)
            OR (@ComparisonOperator = '<' AND @CurrentValue < @ThresholdValue)
        BEGIN
            SET @AlertMessage = 
                N'ALERT [' + @AlertLevel + N']: ' + @MetricType + 
                N' is ' + CAST(@CurrentValue AS NVARCHAR(20)) + 
                N' (Threshold: ' + CAST(@ThresholdValue AS NVARCHAR(20)) + N')';
            
            -- Log alert
            INSERT INTO AlertLog (AlertID, TriggerTime, MetricValue, AlertMessage)
            VALUES (@AlertID, GETDATE(), @CurrentValue, @AlertMessage);
            
            --
