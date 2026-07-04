---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing platform for logistics intelligence, fleet optimization, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure logistics data warehouse with Power BI"
  - "implement multi-fact star schema for fleet management"
  - "build warehouse gravity zone optimization"
  - "create logistics intelligence dashboard"
  - "integrate fleet telemetry with warehouse operations"
  - "deploy supply chain analytics engine"
  - "analyze cross-modal logistics KPIs"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines:
- **MS SQL Server data warehouse** with multi-fact star schema for warehouse operations, fleet trips, and cross-dock activities
- **Power BI dashboards** for real-time visualization and analysis
- **Cross-fact KPI harmonization** linking inventory, fleet telemetry, and external data sources
- **Predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Warehouse gravity zone optimization** based on pick frequency, value, and replenishment patterns

The platform integrates data from WMS (Warehouse Management Systems), GPS/telematics feeds, supplier portals, and external APIs (weather, traffic) into a unified semantic layer.

## Installation and Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs

### Step 1: Deploy SQL Schema

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create fact table: Warehouse Operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes DECIMAL(10,2),
    CycleTimeMinutes DECIMAL(10,2),
    OperationCost DECIMAL(12,2),
    OperationTimestamp DATETIME2 NOT NULL,
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WarehouseOps_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WarehouseOps_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey)
);
GO

-- Create columnstore index for analytics performance
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOps_Analytics
ON FactWarehouseOperations (
    TimeKey, ProductKey, WarehouseKey, OperationType, 
    QuantityHandled, DwellTimeMinutes, CycleTimeMinutes, OperationCost
);
GO

-- Create fact table: Fleet Trips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripDistanceKm DECIMAL(10,2),
    TripDurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    DelayMinutes DECIMAL(10,2),
    DelayReason VARCHAR(100),
    TripCost DECIMAL(12,2),
    TripStartTimestamp DATETIME2 NOT NULL,
    TripEndTimestamp DATETIME2,
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrips_Driver FOREIGN KEY (DriverKey) REFERENCES DimDriver(DriverKey),
    CONSTRAINT FK_FleetTrips_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey),
    CONSTRAINT FK_FleetTrips_OriginGeo FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_DestGeo FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);
GO

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips_Analytics
ON FactFleetTrips (
    TimeKey, VehicleKey, RouteKey, TripDistanceKm, 
    IdleTimeMinutes, FuelConsumedLiters, DelayMinutes
);
GO

-- Create dimension: Time (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDatetime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    QuarterHourBucket TINYINT NOT NULL, -- 0-3 within each hour
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthOfYear TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    Year SMALLINT NOT NULL,
    FiscalPeriod VARCHAR(20),
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL,
    HolidayName VARCHAR(100)
);
GO

-- Create dimension: Product with Gravity Score
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductSKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitValue DECIMAL(12,2),
    WeightKg DECIMAL(10,3),
    VolumeM3 DECIMAL(10,4),
    FragilityScore TINYINT, -- 1-10
    TemperatureRequirement VARCHAR(50), -- 'Ambient', 'Refrigerated', 'Frozen'
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility
    EffectiveDate DATE NOT NULL,
    ExpiryDate DATE,
    IsCurrent BIT DEFAULT 1
);
GO

-- Create dimension: Warehouse with Gravity Zones
CREATE TABLE DimWarehouse (
    WarehouseKey INT IDENTITY(1,1) PRIMARY KEY,
    WarehouseCode VARCHAR(20) UNIQUE NOT NULL,
    WarehouseName VARCHAR(200) NOT NULL,
    GeographyKey INT NOT NULL,
    TotalCapacityM3 DECIMAL(12,2),
    ZoneCount INT,
    HasColdStorage BIT,
    HasCrossDock BIT,
    CONSTRAINT FK_Warehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);
GO

CREATE TABLE DimWarehouseZone (
    ZoneKey INT IDENTITY(1,1) PRIMARY KEY,
    WarehouseKey INT NOT NULL,
    ZoneCode VARCHAR(20) NOT NULL,
    ZoneName VARCHAR(100),
    GravityTier VARCHAR(20), -- 'High', 'Medium', 'Low'
    DistanceToShippingDockMeters DECIMAL(8,2),
    CapacityM3 DECIMAL(10,2),
    CONSTRAINT FK_Zone_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT UK_Zone UNIQUE (WarehouseKey, ZoneCode)
);
GO

-- Create dimension: Vehicle
CREATE TABLE DimVehicle (
    VehicleKey INT IDENTITY(1,1) PRIMARY KEY,
    VehicleID VARCHAR(50) UNIQUE NOT NULL,
    VehicleType VARCHAR(50), -- 'Van', 'Truck', 'Semi'
    MaxLoadKg DECIMAL(10,2),
    FuelType VARCHAR(30),
    YearManufactured SMALLINT,
    MaintenanceScore TINYINT, -- 1-100
    TelematchicsDeviceID VARCHAR(100)
);
GO

-- Create dimension: Geography (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50), -- 'Warehouse', 'CustomerSite', 'RouteNode'
    Address VARCHAR(500),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    RegionCode VARCHAR(50),
    ContinentCode VARCHAR(10)
);
GO
```

### Step 2: Create Stored Procedures for Data Loading

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, SupplierKey,
        OperationType, QuantityHandled, DwellTimeMinutes,
        CycleTimeMinutes, OperationCost, OperationTimestamp
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        s.SupplierKey,
        stg.OperationType,
        stg.Quantity,
        stg.DwellTimeMinutes,
        stg.CycleTimeMinutes,
        stg.OperationCost,
        stg.OperationTimestamp
    FROM Staging_WarehouseOperations stg
    INNER JOIN DimTime t ON t.FullDatetime = DATEADD(MINUTE, 
        (DATEPART(MINUTE, stg.OperationTimestamp) / 15) * 15,
        DATEADD(HOUR, DATEPART(HOUR, stg.OperationTimestamp),
        CAST(CAST(stg.OperationTimestamp AS DATE) AS DATETIME2)))
    INNER JOIN DimProduct p ON p.ProductSKU = stg.ProductSKU AND p.IsCurrent = 1
    INNER JOIN DimWarehouse w ON w.WarehouseCode = stg.WarehouseCode
    LEFT JOIN DimSupplier s ON s.SupplierCode = stg.SupplierCode
    WHERE stg.OperationTimestamp > @LastLoadTimestamp
        AND stg.IsProcessed = 0;
    
    -- Update staging table processed flag
    UPDATE Staging_WarehouseOperations
    SET IsProcessed = 1, ProcessedTimestamp = GETUTCDATE()
    WHERE OperationTimestamp > @LastLoadTimestamp
        AND IsProcessed = 0;
END;
GO

-- Procedure to calculate and update product gravity scores
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductVelocity AS (
        SELECT 
            p.ProductKey,
            COUNT(*) AS PickCount,
            SUM(f.QuantityHandled) AS TotalQuantity,
            DATEDIFF(DAY, MIN(f.OperationTimestamp), MAX(f.OperationTimestamp)) AS DaySpan
        FROM FactWarehouseOperations f
        INNER JOIN DimProduct p ON f.ProductKey = p.ProductKey
        WHERE f.OperationType = 'Picking'
            AND f.OperationTimestamp >= DATEADD(MONTH, -3, GETDATE())
        GROUP BY p.ProductKey
    )
    UPDATE p
    SET GravityScore = 
        (COALESCE(pv.TotalQuantity / NULLIF(pv.DaySpan, 0), 0) * 0.4) + -- Velocity weight
        (COALESCE(p.UnitValue, 0) / 100.0 * 0.4) + -- Value weight
        (COALESCE(p.FragilityScore, 0) * 0.2) -- Fragility weight
    FROM DimProduct p
    LEFT JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
    WHERE p.IsCurrent = 1;
END;
GO

-- Procedure for fleet maintenance prioritization
CREATE PROCEDURE usp_GenerateFleetMaintenanceQueue
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        v.VehicleID,
        v.VehicleType,
        v.MaintenanceScore,
        AVG(f.IdleTimeMinutes / NULLIF(f.TripDurationMinutes, 0)) AS AvgIdlePercentage,
        SUM(f.TripDistanceKm) AS TotalDistanceLast30Days,
        SUM(f.FuelConsumedLiters) AS TotalFuelLast30Days,
        COUNT(CASE WHEN f.DelayMinutes > 30 THEN 1 END) AS MajorDelayCount,
        -- Risk score: combines maintenance score, idle time, and high-value cargo exposure
        (100 - v.MaintenanceScore) * 0.5 +
        (AVG(f.IdleTimeMinutes / NULLIF(f.TripDurationMinutes, 0)) * 100 * 0.3) +
        (COUNT(CASE WHEN f.DelayMinutes > 30 THEN 1 END) * 5 * 0.2) AS MaintenancePriorityScore
    FROM DimVehicle v
    LEFT JOIN FactFleetTrips f ON v.VehicleKey = f.VehicleKey
        AND f.TripStartTimestamp >= DATEADD(DAY, -30, GETDATE())
    GROUP BY v.VehicleID, v.VehicleType, v.MaintenanceScore
    HAVING MaintenancePriorityScore > 40
    ORDER BY MaintenancePriorityScore DESC;
END;
GO
```

### Step 3: Populate Time Dimension

```sql
-- Populate DimTime with 15-minute granularity for 5 years
DECLARE @StartDate DATETIME2 = '2024-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2029-12-31 23:45:00';
DECLARE @CurrentDate DATETIME2 = @StartDate;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey, FullDatetime, DateKey, TimeOfDay, QuarterHourBucket,
        HourOfDay, DayOfWeek, DayOfMonth, WeekOfYear, MonthOfYear,
        Quarter, Year, IsWeekend, IsHoliday
    )
    VALUES (
        CAST(FORMAT(@CurrentDate, 'yyyyMMddHHmm') AS INT),
        @CurrentDate,
        CAST(FORMAT(@CurrentDate, 'yyyyMMdd') AS INT),
        CAST(@CurrentDate AS TIME),
        DATEPART(MINUTE, @CurrentDate) / 15,
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATEPART(DAY, @CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        DATEPART(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        YEAR(@CurrentDate),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END,
        0 -- Update separately for holidays
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
END;
GO
```

### Step 4: Configure Power BI Connection

Create a `config.json` file (do not commit with actual credentials):

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}",
    "port": 1433
  },
  "refresh_schedule": {
    "interval_minutes": 15,
    "full_refresh_hour": 2
  },
  "external_apis": {
    "weather_api_key": "${WEATHER_API_KEY}",
    "traffic_api_key": "${TRAFFIC_API_KEY}"
  }
}
```

## Key SQL Queries and Patterns

### Cross-Fact Analysis: Dwell Time vs. Fleet Idle Time

```sql
-- Find SKUs with high dwell time that correlate with fleet delays
WITH WarehouseDwell AS (
    SELECT 
        p.ProductSKU,
        p.ProductName,
        p.GravityScore,
        w.WarehouseName,
        AVG(f.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations f
    INNER JOIN DimProduct p ON f.ProductKey = p.ProductKey
    INNER JOIN DimWarehouse w ON f.WarehouseKey = w.WarehouseKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FullDatetime >= DATEADD(MONTH, -1, GETDATE())
        AND f.OperationType IN ('Putaway', 'Picking')
    GROUP BY p.ProductSKU, p.ProductName, p.GravityScore, w.WarehouseName
),
FleetIdle AS (
    SELECT 
        og.LocationName AS OriginWarehouse,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        AVG(f.DelayMinutes) AS AvgDelay
    FROM FactFleetTrips f
    INNER JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FullDatetime >= DATEADD(MONTH, -1, GETDATE())
        AND og.LocationType = 'Warehouse'
    GROUP BY og.LocationName
)
SELECT 
    wd.ProductSKU,
    wd.ProductName,
    wd.GravityScore,
    wd.WarehouseName,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.AvgDelay,
    -- Correlation indicator
    CASE 
        WHEN wd.AvgDwellTime > 180 AND fi.AvgIdleTime > 30 THEN 'High Risk'
        WHEN wd.AvgDwellTime > 120 OR fi.AvgIdleTime > 20 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS RiskLevel
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.WarehouseName = fi.OriginWarehouse
WHERE wd.AvgDwellTime > 60
ORDER BY wd.AvgDwellTime DESC, fi.AvgIdleTime DESC;
```

### Warehouse Gravity Zone Optimization Recommendation

```sql
-- Identify products in wrong gravity zones
WITH ProductActivity AS (
    SELECT 
        p.ProductKey,
        p.ProductSKU,
        p.ProductName,
        p.GravityScore,
        wz.ZoneKey,
        wz.ZoneName,
        wz.GravityTier,
        wz.DistanceToShippingDockMeters,
        COUNT(*) AS PickFrequency,
        AVG(f.CycleTimeMinutes) AS AvgCycleTime
    FROM FactWarehouseOperations f
    INNER JOIN DimProduct p ON f.ProductKey = p.ProductKey
    INNER JOIN DimWarehouse w ON f.WarehouseKey = w.WarehouseKey
    INNER JOIN DimWarehouseZone wz ON w.WarehouseKey = wz.WarehouseKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE f.OperationType = 'Picking'
        AND t.FullDatetime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.ProductKey, p.ProductSKU, p.ProductName, p.GravityScore,
             wz.ZoneKey, wz.ZoneName, wz.GravityTier, wz.DistanceToShippingDockMeters
)
SELECT 
    ProductSKU,
    ProductName,
    GravityScore,
    ZoneName AS CurrentZone,
    GravityTier AS CurrentTier,
    PickFrequency,
    AvgCycleTime,
    DistanceToShippingDockMeters,
    CASE 
        WHEN GravityScore > 7 AND GravityTier != 'High' THEN 'Move to High Gravity Zone'
        WHEN GravityScore BETWEEN 4 AND 7 AND GravityTier != 'Medium' THEN 'Move to Medium Gravity Zone'
        WHEN GravityScore < 4 AND GravityTier != 'Low' THEN 'Move to Low Gravity Zone'
        ELSE 'Correctly Placed'
    END AS Recommendation,
    CASE 
        WHEN GravityScore > 7 AND GravityTier != 'High' THEN AvgCycleTime * 0.25
        WHEN GravityScore BETWEEN 4 AND 7 AND GravityTier != 'Medium' THEN AvgCycleTime * 0.15
        WHEN GravityScore < 4 AND GravityTier != 'Low' THEN AvgCycleTime * 0.10
        ELSE 0
    END AS EstimatedCycleTimeSavingsMinutes
FROM ProductActivity
WHERE GravityScore IS NOT NULL
ORDER BY EstimatedCycleTimeSavingsMinutes DESC;
```

### Route Efficiency with Weather Correlation

```sql
-- Analyze routes with delays correlated to weather conditions
SELECT 
    r.RouteName,
    dg.City AS DestinationCity,
    t.Year,
    t.MonthOfYear,
    COUNT(*) AS TripCount,
    AVG(f.TripDurationMinutes) AS AvgDuration,
    AVG(f.DelayMinutes) AS AvgDelay,
    SUM(CASE WHEN f.DelayReason LIKE '%Weather%' THEN 1 ELSE 0 END) AS WeatherDelayCount,
    AVG(f.FuelConsumedLiters / NULLIF(f.TripDistanceKm, 0)) AS AvgFuelEfficiency
FROM FactFleetTrips f
INNER JOIN DimRoute r ON f.RouteKey = r.RouteKey
INNER JOIN DimGeography dg ON f.DestinationGeographyKey = dg.GeographyKey
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.FullDatetime >= DATEADD(YEAR, -1, GETDATE())
GROUP BY r.RouteName, dg.City, t.Year, t.MonthOfYear
HAVING AVG(f.DelayMinutes) > 15
ORDER BY WeatherDelayCount DESC, AvgDelay DESC;
```

## Power BI DAX Measures

### Key Performance Indicators

Create these measures in Power BI:

```dax
// Total Warehouse Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time
Avg Dwell Time (Hours) = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60

// Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUMX(FactFleetTrips, FactFleetTrips[TripDurationMinutes] - FactFleetTrips[IdleTimeMinutes]),
    SUMX(FactFleetTrips, FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// On-Time Delivery Rate
On-Time Delivery % = 
DIVIDE(
    CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[DelayMinutes] <= 15),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Warehouse Capacity Utilization
Capacity Utilization % = 
VAR CurrentVolume = 
    SUMX(
        FILTER(FactWarehouseOperations, FactWarehouseOperations[OperationType] = "Putaway"),
        RELATED(DimProduct[VolumeM3]) * FactWarehouseOperations[QuantityHandled]
    )
VAR TotalCapacity = SUM(DimWarehouse[TotalCapacityM3])
RETURN DIVIDE(CurrentVolume, TotalCapacity, 0) * 100

// Cross-Fact: Cost per Delivered Unit
Cost per Delivered Unit = 
VAR WarehouseCost = SUM(FactWarehouseOperations[OperationCost])
VAR FleetCost = SUM(FactFleetTrips[TripCost])
VAR TotalUnits = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
RETURN DIVIDE(WarehouseCost + FleetCost, TotalUnits, 0)

// Predictive Bottleneck Score (simplified)
Bottleneck Risk Score = 
VAR HighDwellCount = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[DwellTimeMinutes] > 180
    )
VAR HighIdleCount = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[IdleTimeMinutes] > 45
    )
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
RETURN (HighDwellCount + HighIdleCount) / TotalOperations * 100
```

### Time Intelligence Measures

```dax
// Warehouse Operations vs Previous Period
Operations vs Prior Period % = 
VAR CurrentPeriod = [Total Operations]
VAR PriorPeriod = 
    CALCULATE(
        [Total Operations],
        DATEADD(DimTime[FullDatetime], -1, MONTH)
    )
RETURN DIVIDE(CurrentPeriod - PriorPeriod, PriorPeriod, 0) * 100

// Rolling 30-Day Fleet Efficiency
Fleet Efficiency 30D = 
CALCULATE(
    [Fleet Utilization %],
    DATESINPERIOD(
        DimTime[FullDatetime],
        LASTDATE(DimTime[FullDatetime]),
        -30,
        DAY
    )
)
```

## Common Integration Patterns

### Pattern 1: Real-Time Telemetry Ingestion

```sql
-- Create external table for streaming telemetry data (requires PolyBase)
CREATE EXTERNAL DATA SOURCE TelematicsStream
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://${STORAGE_ACCOUNT}.blob.core.windows.net/telematics',
    CREDENTIAL = AzureStorageCredential
);
GO

CREATE EXTERNAL FILE FORMAT TelematicsJSON
WITH (
    FORMAT_TYPE = JSON
);
GO

CREATE EXTERNAL TABLE Staging_TelematicsRaw (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2)
)
WITH (
    LOCATION = '/realtime/',
    DATA_SOURCE = TelematicsStream,
    FILE_FORMAT = TelematicsJSON
);
GO

-- Scheduled job to process telemetry into fleet trips
CREATE PROCEDURE usp_ProcessTelematicsStream
AS
BEGIN
    -- Aggregate GPS points into trip segments
    INSERT INTO FactFleetTrips (...)
    SELECT ...
    FROM Staging_TelematicsRaw
    -- Complex logic to detect trip start/end, calculate metrics
END;
GO
```

### Pattern 2: WMS Integration via REST API

Python script to extract data from WMS API and load into SQL:

```python
import requests
import pyodbc
import os
from datetime import datetime, timedelta

# Configuration from environment variables
SQL_SERVER = os.getenv('SQL_SERVER_HOST')
SQL_DATABASE = 'LogiFleetPulse'
SQL_USER = os.getenv('SQL_SERVER_USER')
SQL_PASSWORD = os.getenv('SQL_SERVER_PASSWORD')
WMS_API_URL = os.getenv('WMS_API_URL')
WMS_API_KEY = os.getenv('WMS_API_KEY')

def extract_warehouse_operations(start_time, end_time):
    """Extract operations from WMS API"""
    headers = {'Authorization': f'Bearer {WMS_API_KEY}'}
    params = {
        'start_time': start_time.isoformat(),
        'end_time': end_time.isoformat()
    }
    
    response = requests.get(
        f'{WMS_API_URL}/api/v1/operations',
        headers=headers,
        params=params
    )
    response.raise_for_status()
    return response.json()

def load_to_staging(operations):
    """Load operations into SQL staging table"""
    conn_str = (
        f'DRIVER={{ODBC Driver 17 for SQL Server}};'
        f'SERVER={SQL_SERVER};DATABASE={SQL_DATABASE};'
        f'UID={SQL_USER};PWD={SQL_PASSWORD}'
    )
    
    with pyodbc.connect(conn_str) as conn:
        cursor = conn.cursor()
        
        # Bulk insert into staging
        for op in operations:
            cursor.execute("""
                INSERT INTO Staging_WarehouseOperations (
                    OperationType, ProductSKU, WarehouseCode, SupplierCode,
