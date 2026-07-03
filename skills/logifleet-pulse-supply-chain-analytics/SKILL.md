---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-fact star schema for unified logistics, warehouse, and fleet intelligence with real-time KPI harmonization
triggers:
  - set up logifleet pulse logistics analytics
  - deploy supply chain data warehouse with power bi
  - configure multi-fact star schema for fleet and warehouse
  - implement logifleet pulse dashboard
  - create logistics intelligence platform
  - build warehouse gravity zone analytics
  - connect power bi to logistics data model
  - set up cross-modal supply chain tracking
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform combining MS SQL Server data warehousing with Power BI visualization. It provides a **multi-fact star schema** that unifies warehouse operations, fleet telemetry, inventory management, and external signals (weather, traffic) into a single semantic layer for real-time decision-making.

Key differentiators:
- **Cross-fact KPI harmonization** - Links warehouse and fleet metrics through shared dimensions
- **Warehouse Gravity Zones™** - Spatial optimization based on pick frequency, value, and lead time
- **Adaptive Fleet Triage Engine** - Prioritizes maintenance by revenue impact
- **Temporal Elasticity Modeling** - Time-phased scenario simulation
- **15-minute refresh granularity** - Near real-time operational awareness

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Network access to data sources (WMS, TMS, telemetry APIs)

### Step 1: Deploy SQL Schema

```sql
-- Create database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour INT NOT NULL,
    Minute INT NOT NULL,
    QuarterHourBucket INT NOT NULL, -- 0-95 (15-min intervals per day)
    DayOfWeek NVARCHAR(10),
    DayOfMonth INT,
    Month INT,
    Quarter INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
)
CREATE CLUSTERED INDEX IX_DimTime_FullDateTime ON DimTime(FullDateTime)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) UNIQUE NOT NULL,
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone NVARCHAR(50)
)
CREATE NONCLUSTERED INDEX IX_DimGeography_LocationID ON DimGeography(LocationID)
GO

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    Brand NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    UnitCost DECIMAL(18,2),
    UnitPrice DECIMAL(18,2),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    ShelfLifeDays INT,
    -- Gravity Zone Calculations
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * urgency
    VelocityClass NVARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueClass NVARCHAR(20) -- 'High', 'Medium', 'Low'
)
CREATE NONCLUSTERED INDEX IX_DimProduct_SKU ON DimProduct(SKU)
CREATE NONCLUSTERED INDEX IX_DimProduct_GravityScore ON DimProduct(GravityScore DESC)
GO

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    Country NVARCHAR(100),
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeDaysStdDev DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityScore DECIMAL(5,2) -- Calculated composite
)
GO

CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID NVARCHAR(50) UNIQUE NOT NULL,
    VehicleType NVARCHAR(50), -- 'Box Truck', 'Flatbed', 'Refrigerated', 'Van'
    Capacity DECIMAL(10,2),
    CapacityUnit NVARCHAR(20), -- 'kg', 'cubic meters', 'pallets'
    YearManufactured INT,
    FuelType NVARCHAR(50),
    CurrentMileage DECIMAL(12,1),
    LastMaintenanceDate DATE,
    NextMaintenanceDue DATE,
    Status NVARCHAR(50) -- 'Active', 'Maintenance', 'Retired'
)
GO

-- Fact Tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2), -- Time in storage
    StorageZone NVARCHAR(50),
    EmployeeID NVARCHAR(50),
    BatchNumber NVARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeKey ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_ProductKey ON FactWarehouseOperations(ProductKey)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyStart INT NOT NULL,
    TimeKeyEnd INT,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT,
    DistanceKm DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    NumberOfStops INT,
    DelayMinutes DECIMAL(10,2), -- Actual vs. planned
    DelayReason NVARCHAR(200),
    DriverID NVARCHAR(50),
    TripStatus NVARCHAR(50), -- 'Completed', 'In Progress', 'Delayed', 'Cancelled'
    FOREIGN KEY (TimeKeyStart) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)
CREATE NONCLUSTERED INDEX IX_FactFleet_TimeKeyStart ON FactFleetTrips(TimeKeyStart)
CREATE NONCLUSTERED INDEX IX_FactFleet_VehicleKey ON FactFleetTrips(VehicleKey)
GO

-- Bridge table for many-to-many between trips and products
CREATE TABLE BridgeTripProduct (
    BridgeKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripKey BIGINT NOT NULL,
    ProductKey INT NOT NULL,
    QuantityLoaded INT,
    FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)
GO
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime with 15-minute granularity
CREATE PROCEDURE sp_PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate
    DECLARE @QuarterHour INT
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        SET @QuarterHour = (DATEPART(HOUR, @CurrentDateTime) * 4) + (DATEPART(MINUTE, @CurrentDateTime) / 15)
        
        INSERT INTO DimTime (
            FullDateTime, DateKey, TimeOfDay, Hour, Minute, QuarterHourBucket,
            DayOfWeek, DayOfMonth, Month, Quarter, FiscalYear, IsWeekend, IsHoliday
        )
        VALUES (
            @CurrentDateTime,
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMdd') AS INT),
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime),
            @QuarterHour,
            DATENAME(WEEKDAY, @CurrentDateTime),
            DAY(@CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            YEAR(@CurrentDateTime), -- Simplification, adjust for fiscal year logic
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END,
            0 -- Holiday logic to be implemented separately
        )
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime)
    END
END
GO

-- Execute for 2 years of data
EXEC sp_PopulateDimTime @StartDate = '2025-01-01', @EndDate = '2026-12-31'
GO
```

### Step 3: Create Views for Cross-Fact Analysis

```sql
-- View combining warehouse operations and fleet trips
CREATE VIEW vw_WarehouseFleetIntegration AS
SELECT 
    dt.FullDateTime,
    dt.DayOfWeek,
    dg.LocationName AS WarehouseLocation,
    dg.City,
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    fwo.OperationType,
    fwo.QuantityHandled,
    fwo.DwellTimeHours,
    fwo.StorageZone,
    -- Correlated fleet metrics
    (SELECT AVG(IdleTimeMinutes) 
     FROM FactFleetTrips fft 
     WHERE fft.TimeKeyStart = fwo.TimeKey 
       AND fft.OriginGeographyKey = fwo.GeographyKey) AS AvgFleetIdleTimeAtSameLocation,
    (SELECT COUNT(*) 
     FROM FactFleetTrips fft 
     WHERE fft.TimeKeyStart = fwo.TimeKey 
       AND fft.OriginGeographyKey = fwo.GeographyKey) AS ConcurrentTrips
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
GO

-- View for fleet efficiency by product gravity
CREATE VIEW vw_FleetEfficiencyByGravity AS
SELECT 
    dp.GravityScore,
    dp.VelocityClass,
    AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
    AVG(fft.FuelConsumedLiters / NULLIF(fft.DistanceKm, 0)) AS AvgFuelEfficiency,
    AVG(fft.DelayMinutes) AS AvgDelayMinutes,
    COUNT(fft.TripKey) AS TotalTrips
FROM FactFleetTrips fft
INNER JOIN BridgeTripProduct btp ON fft.TripKey = btp.TripKey
INNER JOIN DimProduct dp ON btp.ProductKey = dp.ProductKey
WHERE fft.TripStatus = 'Completed'
GROUP BY dp.GravityScore, dp.VelocityClass
GO
```

### Step 4: Configure Data Loading

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON
    
    -- This assumes you have a staging table populated from your WMS
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, 
        QuantityHandled, DurationMinutes, DwellTimeHours, 
        StorageZone, EmployeeID, BatchNumber
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        stg.OperationType,
        stg.QuantityHandled,
        stg.DurationMinutes,
        DATEDIFF(HOUR, stg.ReceivedDateTime, stg.ProcessedDateTime) AS DwellTimeHours,
        stg.StorageZone,
        stg.EmployeeID,
        stg.BatchNumber
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime dt ON stg.OperationDateTime BETWEEN dt.FullDateTime AND DATEADD(MINUTE, 14, dt.FullDateTime)
    INNER JOIN DimGeography dg ON stg.LocationID = dg.LocationID
    INNER JOIN DimProduct dp ON stg.SKU = dp.SKU
    WHERE stg.OperationDateTime > @LastLoadDateTime
      AND stg.IsProcessed = 0
    
    -- Mark staging records as processed
    UPDATE StagingWarehouseOps 
    SET IsProcessed = 1 
    WHERE OperationDateTime > @LastLoadDateTime
END
GO
```

### Step 5: Calculate Gravity Scores

```sql
-- Stored procedure to recalculate product gravity scores
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON
    
    -- Calculate velocity based on last 90 days
    WITH VelocityCalc AS (
        SELECT 
            ProductKey,
            SUM(QuantityHandled) AS TotalMoved,
            COUNT(DISTINCT TimeKey) AS ActiveDays,
            SUM(QuantityHandled) * 1.0 / NULLIF(COUNT(DISTINCT TimeKey), 0) AS DailyVelocity
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -90, GETDATE()) ORDER BY TimeKey OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY)
          AND OperationType IN ('Picking', 'Shipping')
        GROUP BY ProductKey
    ),
    GravityCalc AS (
        SELECT 
            dp.ProductKey,
            ISNULL(vc.DailyVelocity, 0) AS Velocity,
            dp.UnitPrice AS Value,
            CASE 
                WHEN dp.RequiresRefrigeration = 1 THEN 1.5
                WHEN dp.IsFragile = 1 THEN 1.3
                ELSE 1.0
            END AS UrgencyMultiplier,
            -- Gravity = (Velocity * Value * Urgency) normalized to 0-100 scale
            (ISNULL(vc.DailyVelocity, 0) * dp.UnitPrice * 
             CASE WHEN dp.RequiresRefrigeration = 1 THEN 1.5 WHEN dp.IsFragile = 1 THEN 1.3 ELSE 1.0 END) AS RawGravity
        FROM DimProduct dp
        LEFT JOIN VelocityCalc vc ON dp.ProductKey = vc.ProductKey
    )
    UPDATE dp
    SET 
        GravityScore = CAST(
            (gc.RawGravity * 1.0 / NULLIF((SELECT MAX(RawGravity) FROM GravityCalc), 0)) * 100 
        AS DECIMAL(5,2)),
        VelocityClass = CASE 
            WHEN gc.Velocity >= (SELECT PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY Velocity) OVER() FROM GravityCalc) THEN 'Fast'
            WHEN gc.Velocity >= (SELECT PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY Velocity) OVER() FROM GravityCalc) THEN 'Medium'
            ELSE 'Slow'
        END,
        ValueClass = CASE 
            WHEN gc.Value >= (SELECT PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY Value) OVER() FROM GravityCalc) THEN 'High'
            WHEN gc.Value >= (SELECT PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY Value) OVER() FROM GravityCalc) THEN 'Medium'
            ELSE 'Low'
        END
    FROM DimProduct dp
    INNER JOIN GravityCalc gc ON dp.ProductKey = gc.ProductKey
END
GO

-- Schedule to run daily
-- USE SQL Server Agent or Azure Automation to schedule
```

## Power BI Configuration

### Step 1: Connect to SQL Server

1. Open Power BI Desktop
2. Get Data > SQL Server
3. Server: `your-server-name`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for better performance)
6. Load tables: `DimTime`, `DimGeography`, `DimProduct`, `DimVehicle`, `FactWarehouseOperations`, `FactFleetTrips`, `BridgeTripProduct`

### Step 2: Define Relationships

Power BI should auto-detect most relationships. Verify:

```
FactWarehouseOperations[TimeKey] → DimTime[TimeKey] (Many-to-One)
FactWarehouseOperations[GeographyKey] → DimGeography[GeographyKey] (Many-to-One)
FactWarehouseOperations[ProductKey] → DimProduct[ProductKey] (Many-to-One)

FactFleetTrips[TimeKeyStart] → DimTime[TimeKey] (Many-to-One)
FactFleetTrips[VehicleKey] → DimVehicle[VehicleKey] (Many-to-One)
FactFleetTrips[OriginGeographyKey] → DimGeography[GeographyKey] (Many-to-One)

BridgeTripProduct[TripKey] → FactFleetTrips[TripKey] (Many-to-One)
BridgeTripProduct[ProductKey] → DimProduct[ProductKey] (Many-to-One)
```

Set relationship cardinality carefully for the bridge table.

### Step 3: Create DAX Measures

```dax
-- Total Warehouse Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

-- Average Dwell Time
Avg Dwell Time (Hours) = AVERAGE(FactWarehouseOperations[DwellTimeHours])

-- Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

-- Cross-Fact KPI: Dwell Time Impact on Fleet Delay
Dwell-Delay Correlation = 
VAR DwellTable = 
    SUMMARIZE(
        FactWarehouseOperations,
        FactWarehouseOperations[ProductKey],
        "AvgDwell", AVERAGE(FactWarehouseOperations[DwellTimeHours])
    )
VAR FleetTable = 
    SUMMARIZE(
        FactFleetTrips,
        FactFleetTrips[TripKey],
        "AvgDelay", AVERAGE(FactFleetTrips[DelayMinutes])
    )
RETURN
    -- Simplified correlation indicator (full calculation would use LINESTX)
    AVERAGEX(
        DwellTable,
        [AvgDwell]
    ) * 
    AVERAGEX(
        FleetTable,
        [AvgDelay]
    )

-- Gravity-Weighted Inventory Value
Gravity-Weighted Value = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[QuantityHandled] * 
    RELATED(DimProduct[UnitPrice]) * 
    RELATED(DimProduct[GravityScore]) / 100
)

-- Predictive Bottleneck Index (simplified heuristic)
Bottleneck Risk Score = 
VAR HighDwellCount = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[DwellTimeHours] > 72
    )
VAR HighIdleCount = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[IdleTimeMinutes] > (FactFleetTrips[DurationMinutes] * 0.15)
    )
VAR TotalOps = COUNTROWS(FactWarehouseOperations) + COUNTROWS(FactFleetTrips)
RETURN
    DIVIDE(HighDwellCount + HighIdleCount, TotalOps, 0) * 100

-- Fuel Efficiency by Gravity Zone
Fuel Efficiency (L/km) = 
DIVIDE(
    SUM(FactFleetTrips[FuelConsumedLiters]),
    SUM(FactFleetTrips[DistanceKm]),
    0
)
```

### Step 4: Build Core Dashboards

**Page 1: Warehouse Operations Overview**
- KPI Cards: Total Operations, Avg Dwell Time, Gravity-Weighted Value
- Line Chart: Operations Volume by Hour (DimTime[Hour] x Total Operations)
- Bar Chart: Top 10 Products by Gravity Score
- Heatmap: Dwell Time by Storage Zone and Day of Week
- Table: Products with Dwell Time > 72 hours

**Page 2: Fleet Performance**
- KPI Cards: Fleet Utilization %, Fuel Efficiency, Avg Delay Minutes
- Map: Trip routes with bubble size = delay minutes
- Column Chart: Idle Time % by Vehicle Type
- Line Chart: Fuel Consumption trend over time
- Table: Vehicles due for maintenance (DimVehicle[NextMaintenanceDue] < TODAY)

**Page 3: Cross-Modal Intelligence**
- Scatter Chart: Dwell Time (X) vs. Fleet Delay (Y), bubble size = product value
- Clustered Column: Fleet Efficiency by Product Velocity Class
- Matrix: Warehouse Location (rows) x Fleet Origin (columns) with Total Trips
- Line + Column: Bottleneck Risk Score over time with operation volumes

### Step 5: Set Up Alerts

Use Power BI service (after publishing):
1. Publish report to workspace
2. Pin key KPI cards to dashboard
3. Configure data alerts on tiles:
   - Alert when Avg Dwell Time > 72 hours
   - Alert when Fleet Utilization < 70%
   - Alert when Bottleneck Risk Score > 25

## Configuration File Structure

```json
{
  "data_sources": {
    "sql_server": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "authentication": {
        "type": "SQL",
        "username": "${SQL_USERNAME}",
        "password": "${SQL_PASSWORD}"
      }
    },
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}",
      "poll_interval_minutes": 15
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}",
      "poll_interval_minutes": 5
    },
    "weather_api": {
      "provider": "openweather",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "etl_schedule": {
    "time_dimension": "daily at 00:00",
    "warehouse_operations": "every 15 minutes",
    "fleet_trips": "every 5 minutes",
    "gravity_scores": "daily at 02:00"
  },
  "alert_thresholds": {
    "dwell_time_hours": 72,
    "fleet_utilization_percent": 70,
    "idle_time_percent": 15,
    "bottleneck_risk_score": 25
  },
  "notification_channels": {
    "email": "${ALERT_EMAIL_RECIPIENTS}",
    "teams_webhook": "${TEAMS_WEBHOOK_URL}",
    "sms_gateway": "${SMS_GATEWAY_ENDPOINT}"
  }
}
```

## Common Patterns & Workflows

### Pattern 1: Daily Operational Review

```sql
-- Run morning summary query
SELECT 
    dg.LocationName,
    COUNT(DISTINCT fwo.OperationKey) AS TotalOperations,
    AVG(fwo.DwellTimeHours) AS AvgDwell,
    SUM(CASE WHEN fwo.DwellTimeHours > 72 THEN 1 ELSE 0 END) AS HighDwellCount
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
WHERE dt.DateKey = CAST(FORMAT(GETDATE() - 1, 'yyyyMMdd') AS INT)
GROUP BY dg.LocationName
ORDER BY AvgDwell DESC
```

### Pattern 2: Identify Storage Zone Optimization Opportunities

```sql
-- Find products in wrong gravity zones
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.VelocityClass,
    fwo.StorageZone,
    COUNT(*) AS OperationCount,
    AVG(fwo.DurationMinutes) AS AvgPickTime
FROM FactWarehouseOperations fwo
INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
WHERE fwo.OperationType = 'Picking'
  AND (
      (dp.VelocityClass = 'Fast' AND fwo.StorageZone LIKE '%Back%') OR
      (dp.VelocityClass = 'Slow' AND fwo.StorageZone LIKE '%Front%')
  )
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, dp.VelocityClass, fwo.StorageZone
HAVING COUNT(*) > 10
ORDER BY dp.GravityScore DESC
```

### Pattern 3: Fleet Maintenance Prioritization

```sql
-- Generate proactive maintenance queue
WITH TripLoadValue AS (
    SELECT 
        fft.VehicleKey,
        SUM(btp.QuantityLoaded * dp.UnitPrice) AS TotalCargoValue,
        COUNT(DISTINCT fft.TripKey) AS RecentTrips
    FROM FactFleetTrips fft
    INNER JOIN BridgeTripProduct btp ON fft.TripKey = btp.TripKey
    INNER JOIN DimProduct dp ON btp.ProductKey = dp.ProductKey
    WHERE fft.TimeKeyStart >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()) ORDER BY TimeKey OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY)
    GROUP BY fft.VehicleKey
)
SELECT 
    dv.VehicleID,
    dv.VehicleType,
    DATEDIFF(DAY, dv.LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
    DATEDIFF(DAY, GETDATE(), dv.NextMaintenanceDue) AS DaysUntilDue,
    tlv.TotalCargoValue,
    tlv.RecentTrips,
    -- Priority Score: (days overdue * cargo value * trip frequency)
    CASE 
        WHEN DATEDIFF(DAY, GETDATE(), dv.NextMaintenanceDue) < 0 
        THEN ABS(DATEDIFF(DAY, GETDATE(), dv.NextMaintenanceDue)) * tlv.TotalCargoValue * tlv.RecentTrips
        ELSE 0
    END AS MaintenancePriorityScore
FROM DimVehicle dv
LEFT JOIN TripLoadValue tlv ON dv.VehicleKey = tlv.VehicleKey
WHERE dv.Status = 'Active'
ORDER BY MaintenancePriorityScore DESC
```

### Pattern 4: Cross-Modal Anomaly Investigation

```sql
-- Investigate correlation between warehouse dwell and delivery delay
WITH WarehouseDwell AS (
    SELECT 
        dt.DateKey,
        dg.LocationID,
        AVG(fwo.DwellTimeHours) AS AvgDwell
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    WHERE dt.DateKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY dt.DateKey, dg.LocationID
),
FleetDelay AS (
    SELECT 
        dt.DateKey,
        dg.LocationID,
        AVG(fft.DelayMinutes) AS AvgDelay
    FROM FactFleetTrips fft
    INNER JOIN DimTime
