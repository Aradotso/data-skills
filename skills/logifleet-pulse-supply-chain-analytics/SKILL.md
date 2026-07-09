---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for warehouse operations, fleet management, and cross-modal supply chain analytics
triggers:
  - set up logifleet pulse logistics dashboard
  - deploy supply chain analytics warehouse schema
  - configure power bi logistics intelligence
  - implement multi-fact star schema for fleet data
  - create warehouse gravity zone optimization
  - build real-time logistics kpi dashboard
  - integrate fleet telemetry with warehouse operations
  - setup cross-modal supply chain analytics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines:
- **MS SQL Server data warehouse** with multi-fact star schema for warehouse operations, fleet telemetry, and cross-dock transfers
- **Power BI dashboards** for real-time logistics visualization and KPI monitoring
- **Cross-fact analytics** linking warehouse metrics (dwell time, pick rates) with fleet metrics (fuel consumption, idle time)
- **Predictive algorithms** for bottleneck detection and maintenance prioritization
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and item value

The platform integrates data from WMS, telematics, supplier portals, and external APIs to provide unified supply chain visibility.

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Enterprise or Developer Edition recommended)
- Power BI Desktop (latest version)
- SQLCMD utility or SQL Server Management Studio (SSMS)
- Access to data sources (WMS, fleet telemetry, ERP)

### Step 1: Deploy SQL Schema

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create Time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME(0) NOT NULL,
    Hour TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL, -- 0-3 for each hour
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod VARCHAR(7) NOT NULL, -- YYYY-MM
    FiscalQuarter VARCHAR(6) NOT NULL -- YYYY-Q1
)
GO

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(FullDateTime)
CREATE NONCLUSTERED INDEX IX_DimTime_Date ON DimTime(Date)
GO

-- Create Product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductSKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100) NOT NULL,
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2), -- kg
    UnitVolume DECIMAL(10,4), -- cubic meters
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    UnitValue DECIMAL(10,2), -- for gravity calculation
    GravityScore DECIMAL(5,2), -- computed field
    GravityZone VARCHAR(20) -- HIGH, MEDIUM, LOW
)
GO

CREATE INDEX IX_DimProduct_Category ON DimProduct(Category, SubCategory)
CREATE INDEX IX_DimProduct_GravityZone ON DimProduct(GravityZone)
GO

-- Create Geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode VARCHAR(20) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- WAREHOUSE, DISTRIBUTION_CENTER, ROUTE_NODE
    Address NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50)
)
GO

CREATE INDEX IX_DimGeography_Type ON DimGeography(LocationType)
CREATE INDEX IX_DimGeography_Country ON DimGeography(Country, Region, City)
GO

-- Create Warehouse Operations Fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- RECEIVING, PUTAWAY, PICKING, PACKING, SHIPPING
    OperationStartTime DATETIME2(0) NOT NULL,
    OperationEndTime DATETIME2(0),
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime),
    Quantity INT NOT NULL,
    DwellTimeHours DECIMAL(10,2), -- time in storage before next operation
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50),
    OrderID VARCHAR(50),
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WarehouseOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

CREATE CLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey, ProductKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Operation ON FactWarehouseOperations(OperationType, OperationStartTime)
GO

-- Create Fleet dimension
CREATE TABLE DimFleet (
    FleetKey INT IDENTITY(1,1) PRIMARY KEY,
    VehicleID VARCHAR(50) NOT NULL UNIQUE,
    VehicleType VARCHAR(50) NOT NULL, -- TRUCK, VAN, CARGO_BIKE
    Capacity DECIMAL(10,2), -- cubic meters
    MaxWeight DECIMAL(10,2), -- kg
    FuelType VARCHAR(20), -- DIESEL, ELECTRIC, HYBRID
    YearManufactured INT,
    IsActive BIT DEFAULT 1
)
GO

-- Create Fleet Trips Fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    FleetKey INT NOT NULL,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    StartGeographyKey INT NOT NULL,
    EndGeographyKey INT,
    TripStartTime DATETIME2(0) NOT NULL,
    TripEndTime DATETIME2(0),
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime),
    DistanceKm DECIMAL(10,2),
    FuelConsumed DECIMAL(10,2), -- liters or kWh
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AverageSpeed DECIMAL(5,2), -- km/h
    OrdersDelivered INT,
    DriverID VARCHAR(50),
    RouteID VARCHAR(50),
    CONSTRAINT FK_FleetTrip_Fleet FOREIGN KEY (FleetKey) REFERENCES DimFleet(FleetKey),
    CONSTRAINT FK_FleetTrip_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrip_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrip_StartGeo FOREIGN KEY (StartGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrip_EndGeo FOREIGN KEY (EndGeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

CREATE CLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(StartTimeKey, FleetKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(RouteID, TripStartTime)
GO

-- Create Cross-Dock Fact (many-to-many bridge)
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    GeographyKey INT NOT NULL,
    CrossDockStartTime DATETIME2(0) NOT NULL,
    CrossDockEndTime DATETIME2(0),
    DwellMinutes AS DATEDIFF(MINUTE, CrossDockStartTime, CrossDockEndTime),
    Quantity INT NOT NULL,
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime for next 5 years at 15-minute intervals
CREATE PROCEDURE PopulateDimTime
    @StartDate DATE = NULL,
    @EndDate DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @StartDate IS NULL SET @StartDate = '2025-01-01';
    IF @EndDate IS NULL SET @EndDate = DATEADD(YEAR, 5, @StartDate);
    
    DECLARE @CurrentDateTime DATETIME2(0) = @StartDate;
    DECLARE @TimeKey INT;
    
    WHILE @CurrentDateTime < @EndDate
    BEGIN
        SET @TimeKey = CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT);
        
        INSERT INTO DimTime (
            TimeKey, FullDateTime, Date, TimeOfDay, Hour, QuarterHour,
            DayOfWeek, DayName, IsWeekend, FiscalPeriod, FiscalQuarter
        )
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime) / 15,
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END,
            FORMAT(@CurrentDateTime, 'yyyy-MM'),
            CONCAT(YEAR(@CurrentDateTime), '-Q', DATEPART(QUARTER, @CurrentDateTime))
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END
GO

EXEC PopulateDimTime;
GO
```

### Step 3: Create Gravity Score Calculation

```sql
-- Stored procedure to calculate and update product gravity scores
CREATE PROCEDURE UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate gravity based on recent pick frequency, value, and fragility
    WITH RecentActivity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            AVG(DwellTimeHours) AS AvgDwellTime
        FROM FactWarehouseOperations
        WHERE OperationType = 'PICKING'
            AND OperationStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    )
    UPDATE p
    SET 
        GravityScore = 
            -- Velocity component (40%)
            (COALESCE(ra.PickCount, 0) * 0.4 / NULLIF((SELECT MAX(PickCount) FROM RecentActivity), 0)) +
            -- Value component (30%)
            (p.UnitValue * 0.3 / NULLIF((SELECT MAX(UnitValue) FROM DimProduct), 0)) +
            -- Inverse dwell time component (20%)
            (CASE WHEN ra.AvgDwellTime > 0 THEN (1.0 / ra.AvgDwellTime) * 0.2 ELSE 0 END) +
            -- Fragility/perishability component (10%)
            ((CAST(p.IsFragile AS INT) + CAST(p.IsPerishable AS INT)) * 0.05),
        GravityZone = 
            CASE 
                WHEN GravityScore > 0.7 THEN 'HIGH'
                WHEN GravityScore > 0.3 THEN 'MEDIUM'
                ELSE 'LOW'
            END
    FROM DimProduct p
    LEFT JOIN RecentActivity ra ON p.ProductKey = ra.ProductKey;
END
GO
```

### Step 4: Create Cross-Fact KPI Views

```sql
-- View: Warehouse-Fleet Integration Metrics
CREATE VIEW vw_WarehouseFleetIntegration AS
SELECT 
    t.Date,
    t.FiscalPeriod,
    g.LocationName AS Warehouse,
    p.Category AS ProductCategory,
    
    -- Warehouse metrics
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(wo.Quantity) AS TotalPickedQty,
    AVG(wo.DurationMinutes) AS AvgPickDurationMin,
    
    -- Fleet metrics (for same products/location)
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin,
    AVG(ft.FuelConsumed) AS AvgFuelPerTrip,
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    
    -- Cross-KPI
    AVG(wo.DwellTimeHours * ft.IdleTimeMinutes) AS DwellIdleProduct
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
LEFT JOIN FactCrossDock cd ON cd.ProductKey = p.ProductKey 
    AND cd.GeographyKey = g.GeographyKey
    AND CAST(cd.CrossDockStartTime AS DATE) = t.Date
LEFT JOIN FactFleetTrips ft ON ft.StartGeographyKey = g.GeographyKey
    AND CAST(ft.TripStartTime AS DATE) = t.Date
GROUP BY t.Date, t.FiscalPeriod, g.LocationName, p.Category
GO
```

### Step 5: Set Up Data Ingestion

```sql
-- Example: Incremental load from external WMS system
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTime DATETIME2(0) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(HOUR, -1, GETDATE());
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        OperationStartTime, OperationEndTime, Quantity,
        DwellTimeHours, StorageZone, OperatorID, OrderID
    )
    SELECT 
        CAST(FORMAT(wms.OperationStartTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        p.ProductKey,
        g.GeographyKey,
        wms.OperationType,
        wms.OperationStartTime,
        wms.OperationEndTime,
        wms.Quantity,
        wms.DwellTimeHours,
        wms.StorageZone,
        wms.OperatorID,
        wms.OrderID
    FROM OPENQUERY(WMS_SERVER, 
        'SELECT * FROM warehouse_operations 
         WHERE operation_start_time > ''' + CONVERT(VARCHAR, @LastLoadTime, 120) + '''') wms
    INNER JOIN DimProduct p ON wms.ProductSKU = p.ProductSKU
    INNER JOIN DimGeography g ON wms.LocationCode = g.LocationCode;
    
    -- Update gravity scores after each load
    EXEC UpdateProductGravityScores;
END
GO
```

## Power BI Configuration

### Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `your-sql-server.database.windows.net` (or localhost)
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **Import** (for smaller datasets) or **DirectQuery** (for real-time)

### Create Relationships in Power BI Model

Power BI should auto-detect relationships, but verify:

```
FactWarehouseOperations[TimeKey] → DimTime[TimeKey]
FactWarehouseOperations[ProductKey] → DimProduct[ProductKey]
FactWarehouseOperations[GeographyKey] → DimGeography[GeographyKey]

FactFleetTrips[StartTimeKey] → DimTime[TimeKey]
FactFleetTrips[FleetKey] → DimFleet[FleetKey]
FactFleetTrips[StartGeographyKey] → DimGeography[GeographyKey]

FactCrossDock[TimeKey] → DimTime[TimeKey]
FactCrossDock[ProductKey] → DimProduct[ProductKey]
```

### Key DAX Measures

```dax
// Total Dwell Time (Hours)
Total Dwell Time = 
SUM(FactWarehouseOperations[DwellTimeHours])

// Average Pick Rate (items/hour)
Avg Pick Rate = 
DIVIDE(
    SUM(FactWarehouseOperations[Quantity]),
    SUM(FactWarehouseOperations[DurationMinutes]) / 60,
    0
)

// Fleet Utilization %
Fleet Utilization = 
DIVIDE(
    SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
)

// Fuel Efficiency (km per liter or kWh)
Fuel Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelConsumed]),
    0
)

// Cross-Dock Velocity (inverse of dwell time)
CrossDock Velocity = 
DIVIDE(
    COUNT(FactCrossDock[CrossDockKey]),
    SUM(FactCrossDock[DwellMinutes]) / 60,
    0
)

// Gravity-Weighted Throughput
Gravity Weighted Throughput = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[Quantity] * RELATED(DimProduct[GravityScore])
)

// Predictive Bottleneck Index (simplified)
Bottleneck Index = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    IF(
        AvgDwell > 48 && AvgIdle > 20,
        1.0, // High risk
        IF(AvgDwell > 24 || AvgIdle > 10, 0.5, 0.1) // Medium or low
    )
```

### Create Dashboard Visuals

**Page 1: Warehouse Operations**
- Card: Total Pick Rate
- Line chart: Dwell Time Trend (by Date, colored by Gravity Zone)
- Matrix: Operations by Product Category and Storage Zone
- Map: Geographic distribution of warehouse locations

**Page 2: Fleet Analytics**
- Card: Fleet Utilization %
- Bar chart: Fuel Efficiency by Vehicle Type
- Scatter plot: Distance vs Idle Time (size = Fuel Consumed)
- Table: Top 10 routes by delivery count

**Page 3: Cross-Modal Integration**
- KPI card: Bottleneck Index
- Waterfall chart: Dwell Time → Cross-Dock → Fleet Idle (cumulative delay)
- Heatmap: Product Category vs Geography (color = Gravity-Weighted Throughput)

## Configuration

### Environment Variables

```bash
# .env file for data source connections
SQL_SERVER=your-sql-server.database.windows.net
SQL_DATABASE=LogiFleetPulse
SQL_USERNAME=logifleet_user
SQL_PASSWORD=${SQL_PASSWORD}  # Set in secure environment

WMS_API_ENDPOINT=https://wms.company.com/api/v2
WMS_API_KEY=${WMS_API_KEY}

TELEMETRY_API_ENDPOINT=https://fleet-telemetry.company.com/api
TELEMETRY_API_KEY=${TELEMETRY_API_KEY}

WEATHER_API_KEY=${WEATHER_API_KEY}
WEATHER_API_ENDPOINT=https://api.openweathermap.org/data/2.5
```

### Power BI Data Refresh Schedule

Set up automated refresh in Power BI Service:
1. Publish report to Power BI workspace
2. Go to Dataset settings → Scheduled refresh
3. Set refresh frequency: **Every 15 minutes** (requires Premium capacity) or **Every hour** (Pro)
4. Configure gateway connection to on-premises SQL Server if needed

## Common Patterns

### Pattern 1: Gravity Zone Rebalancing

```sql
-- Identify products that should move to different gravity zones
SELECT 
    ProductSKU,
    ProductName,
    CurrentGravityZone = GravityZone,
    RecommendedZone = 
        CASE 
            WHEN GravityScore > 0.7 THEN 'HIGH'
            WHEN GravityScore > 0.3 THEN 'MEDIUM'
            ELSE 'LOW'
        END,
    GravityScore,
    RecentPickCount = (
        SELECT COUNT(*) 
        FROM FactWarehouseOperations wo
        WHERE wo.ProductKey = p.ProductKey
            AND wo.OperationType = 'PICKING'
            AND wo.OperationStartTime >= DATEADD(DAY, -7, GETDATE())
    )
FROM DimProduct p
WHERE GravityZone <> 
    CASE 
        WHEN GravityScore > 0.7 THEN 'HIGH'
        WHEN GravityScore > 0.3 THEN 'MEDIUM'
        ELSE 'LOW'
    END
ORDER BY ABS(GravityScore - 
    CASE GravityZone 
        WHEN 'HIGH' THEN 0.85 
        WHEN 'MEDIUM' THEN 0.5 
        ELSE 0.15 
    END) DESC;
```

### Pattern 2: Fleet Maintenance Prioritization

```sql
-- Generate maintenance priority queue
WITH VehicleMetrics AS (
    SELECT 
        f.VehicleID,
        f.VehicleType,
        AVG(ft.FuelConsumed / NULLIF(ft.DistanceKm, 0)) AS AvgFuelPerKm,
        AVG(ft.IdleTimeMinutes * 1.0 / NULLIF(ft.TripDurationMinutes, 0)) AS IdleRatio,
        COUNT(*) AS TripCount,
        SUM(ft.DistanceKm) AS TotalDistance
    FROM DimFleet f
    INNER JOIN FactFleetTrips ft ON f.FleetKey = ft.FleetKey
    WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY f.FleetKey, f.VehicleID, f.VehicleType
),
AnomalyScores AS (
    SELECT 
        *,
        AnomalyScore = 
            -- High fuel consumption relative to fleet average
            (CASE WHEN AvgFuelPerKm > (SELECT AVG(AvgFuelPerKm) * 1.2 FROM VehicleMetrics) 
                  THEN 0.4 ELSE 0 END) +
            -- High idle ratio
            (CASE WHEN IdleRatio > 0.2 THEN 0.3 ELSE 0 END) +
            -- High mileage
            (CASE WHEN TotalDistance > 10000 THEN 0.3 ELSE 0 END)
    FROM VehicleMetrics
)
SELECT 
    VehicleID,
    VehicleType,
    ROUND(AnomalyScore, 2) AS MaintenancePriority,
    ROUND(AvgFuelPerKm, 3) AS FuelPerKm,
    ROUND(IdleRatio * 100, 1) AS IdlePercent,
    TotalDistance AS DistanceKm
FROM AnomalyScores
WHERE AnomalyScore > 0.3
ORDER BY AnomalyScore DESC;
```

### Pattern 3: Cross-Fact Correlation Analysis

```sql
-- Analyze correlation between warehouse dwell time and fleet delays
SELECT 
    t.Date,
    p.Category,
    AvgWarehouseDwell = AVG(wo.DwellTimeHours),
    AvgFleetIdle = AVG(ft.IdleTimeMinutes),
    TotalDeliveries = COUNT(DISTINCT ft.TripKey),
    DelayCorrelation = 
        CASE 
            WHEN AVG(wo.DwellTimeHours) > 48 AND AVG(ft.IdleTimeMinutes) > 15 
            THEN 'HIGH_CORRELATION'
            WHEN AVG(wo.DwellTimeHours) > 24 OR AVG(ft.IdleTimeMinutes) > 10
            THEN 'MODERATE'
            ELSE 'LOW'
        END
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactCrossDock cd ON cd.ProductKey = p.ProductKey 
    AND CAST(cd.CrossDockStartTime AS DATE) = t.Date
LEFT JOIN FactFleetTrips ft ON ft.OutboundTripKey = cd.CrossDockKey
WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY t.Date, p.Category
HAVING COUNT(DISTINCT ft.TripKey) > 0
ORDER BY t.Date DESC, AvgWarehouseDwell DESC;
```

## Troubleshooting

### Issue: Power BI relationship errors

**Symptom**: "Can't create relationship between tables"

**Solution**: 
1. Verify foreign key columns have matching data types
2. Check for NULL values in key columns:
```sql
SELECT COUNT(*) FROM FactWarehouseOperations WHERE TimeKey IS NULL;
```
3. Ensure TimeKey format matches (YYYYMMDDHHMM as INT)
4. In Power BI Model view, manually create relationships if auto-detect fails

### Issue: Slow query performance

**Symptom**: Dashboard takes >30 seconds to refresh

**Solution**:
```sql
-- Add columnstore indexes for fact tables (SQL Server 2016+)
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_CS 
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, Quantity, DwellTimeHours);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_CS
ON FactFleetTrips (StartTimeKey, FleetKey, StartGeographyKey, DistanceKm, FuelConsumed);

-- Partition large fact tables by date
-- (Requires partitioning function and scheme setup)
```

### Issue: Gravity scores not updating

**Symptom**: All products show NULL or 0 for GravityScore

**Solution**:
```sql
-- Verify data exists in FactWarehouseOperations
SELECT COUNT(*) FROM FactWarehouseOperations WHERE OperationType = 'PICKING';

-- Manually trigger gravity score update
EXEC UpdateProductGravityScores;

-- Check for division by zero issues
SELECT MAX(PickCount) FROM (
    SELECT ProductKey, COUNT(*) AS PickCount
    FROM FactWarehouseOperations
    WHERE OperationType = 'PICKING'
    GROUP BY ProductKey
) x;
```

### Issue: Cross-fact KPIs show unexpected values

**Symptom**: Bottleneck Index always shows 1.0 or 0

**Solution**:
- Verify date ranges align between fact tables
- Check for time zone mismatches in timestamps
- Use `INNER JOIN` instead of `LEFT JOIN` if both metrics are required:

```sql
-- Debug cross-fact data availability
SELECT 
    'Warehouse' AS Source,
    MIN(OperationStartTime) AS EarliestRecord,
    MAX(OperationStartTime) AS LatestRecord,
    COUNT(*) AS RecordCount
FROM FactWarehouseOperations
UNION ALL
SELECT 
    'Fleet' AS Source,
    MIN(TripStartTime),
    MAX(TripStartTime),
    COUNT(*)
FROM FactFleetTrips;
```

### Issue: Power BI DirectQuery mode limitations

**Symptom**: "This visual can't be used with DirectQuery"

**Solution**:
- Use Import mode for complex DAX measures
- Pre-calculate aggregations in SQL views:
```sql
CREATE VIEW vw_DailyWarehouseSummary AS
SELECT 
    CAST(OperationStartTime AS DATE) AS OperationDate,
    ProductKey,
    GeographyKey,
    SUM(Quantity) AS TotalQty,
    AVG(DwellTimeHours) AS AvgDwell,
    AVG(DurationMinutes) AS AvgDuration
FROM FactWarehouseOperations
GROUP BY CAST(OperationStartTime AS DATE), ProductKey, GeographyKey;
```
- Use aggregation tables in Power BI (Premium feature)

## API Integration Examples

### Ingest Telemetry Data via REST API

```sql
-- Enable SQL Server to call external APIs (requires CLR or external scripts)
-- Alternative: Use Azure Data Factory or Power Automate

-- Example Python script to load telemetry data
-- Run via SQL Server Agent job or external scheduler

/*
import pyodbc
import requests
import os
from datetime import datetime

conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER')};"
    f"DATABASE={os.getenv('SQL_DATABASE')};"
    f"UID={os.getenv('SQL_USERNAME')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)

response = requests.get(
    f"{os.getenv('TELEMETRY_API_ENDPOINT')}/trips/recent",
    headers={'Authorization': f"Bearer {os.getenv('TELEMETRY_API_KEY')}"}
)

trips = response.json()

cursor = conn.cursor()
for trip in trips:
    cursor.execute("""
        INSERT INTO FactFleetTrips (
            FleetKey, StartTimeKey, StartGeography
