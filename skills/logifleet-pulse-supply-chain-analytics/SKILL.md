---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain intelligence with multi-fact star schema
triggers:
  - set up logifleet pulse supply chain analytics
  - configure power bi logistics dashboard
  - implement multi-fact star schema for warehouse data
  - deploy sql server logistics data warehouse
  - create fleet management analytics with power bi
  - build supply chain intelligence dashboard
  - integrate warehouse and fleet telemetry data
  - set up logistics kpi tracking system
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced data warehousing and analytics template for logistics and supply chain management. It provides a multi-fact star schema in MS SQL Server combined with Power BI dashboards to unify warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for real-time logistics intelligence.

## What This Project Does

- **Multi-fact star schema** linking warehouse operations, fleet trips, cross-dock transfers, and supplier reliability
- **Time-phased dimensions** with 15-minute granularity for real-time operational awareness
- **Power BI templates** for logistics KPI visualization and predictive bottleneck detection
- **Cross-fact KPI harmonization** to correlate warehouse dwell time with fleet performance
- **Warehouse gravity zones** for spatial optimization based on pick frequency and item value
- **Adaptive fleet triage** using weighted scoring from real-time telemetry

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, ERP systems
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Step 1: Deploy the SQL Schema

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create the time dimension table (15-minute buckets)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    Hour INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    IsWeekend BIT NOT NULL,
    FiscalPeriod INT NOT NULL,
    INDEX IX_DimTime_Date (Date),
    INDEX IX_DimTime_DateTime (DateTime)
);

-- Create geography dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Route Node, Distribution Center
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10, 7),
    Longitude DECIMAL(10, 7),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1,
    INDEX IX_DimGeography_LocationID (LocationID),
    INDEX IX_DimGeography_Type (LocationType)
);

-- Create product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10, 3),
    UnitVolume DECIMAL(10, 3),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    StorageTemperature VARCHAR(50), -- Ambient, Refrigerated, Frozen
    GravityScore DECIMAL(5, 2), -- Calculated: velocity × value × fragility
    OptimalZoneType VARCHAR(50), -- High-Gravity, Medium-Gravity, Low-Gravity
    INDEX IX_DimProduct_SKU (SKU),
    INDEX IX_DimProduct_GravityScore (GravityScore DESC)
);

-- Create supplier dimension
CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    Country VARCHAR(100),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5, 2), -- Standard deviation
    DefectRate DECIMAL(5, 4), -- Percentage
    ComplianceScore DECIMAL(5, 2), -- 0-100
    ReliabilityScore DECIMAL(5, 2), -- Calculated composite
    IsActive BIT DEFAULT 1,
    INDEX IX_DimSupplier_SupplierID (SupplierID)
);

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT NOT NULL,
    DwellTimeMinutes INT, -- Time in warehouse before next operation
    CycleTimeMinutes INT, -- Time to complete operation
    ZoneID VARCHAR(50),
    ZoneGravity VARCHAR(50), -- High, Medium, Low
    OperatorID VARCHAR(50),
    BatchID VARCHAR(100),
    OrderID VARCHAR(100),
    OperationCost DECIMAL(10, 2),
    OperationTimestamp DATETIME NOT NULL,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey),
    INDEX IX_FactWarehouse_Time (TimeKey),
    INDEX IX_FactWarehouse_Geography (GeographyKey),
    INDEX IX_FactWarehouse_Product (ProductKey),
    INDEX IX_FactWarehouse_OperationType (OperationType)
);

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    RouteID VARCHAR(100),
    TripDistanceKm DECIMAL(10, 2),
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10, 3),
    LoadWeightKg DECIMAL(10, 2),
    LoadVolumeM3 DECIMAL(10, 3),
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(200),
    TripCost DECIMAL(10, 2),
    TripStartTimestamp DATETIME NOT NULL,
    TripEndTimestamp DATETIME,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_FactFleet_Time (TimeKey),
    INDEX IX_FactFleet_Vehicle (VehicleID),
    INDEX IX_FactFleet_Route (RouteID)
);

-- Create cross-dock fact table (many-to-many bridge)
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    Quantity INT NOT NULL,
    CrossDockDurationMinutes INT,
    TransferTimestamp DATETIME NOT NULL,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    INDEX IX_FactCrossDock_Time (TimeKey),
    INDEX IX_FactCrossDock_Geography (GeographyKey)
);
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime for a date range
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    DECLARE @MinuteBuckets TABLE (Bucket INT);
    INSERT INTO @MinuteBuckets VALUES (0), (15), (30), (45);
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (TimeKey, DateTime, Date, Year, Quarter, Month, Week, 
                             DayOfWeek, DayName, Hour, MinuteBucket, IsWeekend, FiscalPeriod)
        SELECT 
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
            @CurrentDateTime AS DateTime,
            CAST(@CurrentDateTime AS DATE) AS Date,
            YEAR(@CurrentDateTime) AS Year,
            DATEPART(QUARTER, @CurrentDateTime) AS Quarter,
            MONTH(@CurrentDateTime) AS Month,
            DATEPART(WEEK, @CurrentDateTime) AS Week,
            DATEPART(WEEKDAY, @CurrentDateTime) AS DayOfWeek,
            DATENAME(WEEKDAY, @CurrentDateTime) AS DayName,
            DATEPART(HOUR, @CurrentDateTime) AS Hour,
            Bucket AS MinuteBucket,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END AS IsWeekend,
            CASE 
                WHEN MONTH(@CurrentDateTime) <= 3 THEN 1
                WHEN MONTH(@CurrentDateTime) <= 6 THEN 2
                WHEN MONTH(@CurrentDateTime) <= 9 THEN 3
                ELSE 4
            END AS FiscalPeriod
        FROM @MinuteBuckets
        WHERE DATEPART(MINUTE, @CurrentDateTime) = Bucket;
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute to populate 2 years of time data
EXEC PopulateTimeDimension '2025-01-01', '2026-12-31';
```

### Step 3: Create Data Loading Stored Procedures

```sql
-- Example: Load warehouse operations from staging
CREATE PROCEDURE LoadWarehouseOperations
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRANSACTION;
    
    BEGIN TRY
        INSERT INTO FactWarehouseOperations (
            TimeKey, GeographyKey, ProductKey, SupplierKey, 
            OperationType, Quantity, DwellTimeMinutes, CycleTimeMinutes,
            ZoneID, ZoneGravity, OperatorID, BatchID, OrderID, 
            OperationCost, OperationTimestamp
        )
        SELECT 
            dt.TimeKey,
            dg.GeographyKey,
            dp.ProductKey,
            ds.SupplierKey,
            s.OperationType,
            s.Quantity,
            s.DwellTimeMinutes,
            s.CycleTimeMinutes,
            s.ZoneID,
            CASE 
                WHEN dp.GravityScore >= 75 THEN 'High'
                WHEN dp.GravityScore >= 40 THEN 'Medium'
                ELSE 'Low'
            END AS ZoneGravity,
            s.OperatorID,
            s.BatchID,
            s.OrderID,
            s.OperationCost,
            s.OperationTimestamp
        FROM StagingWarehouseOps s
        INNER JOIN DimTime dt ON dt.DateTime = DATEADD(MINUTE, 
            (DATEPART(MINUTE, s.OperationTimestamp) / 15) * 15, 
            DATEADD(MINUTE, -DATEPART(MINUTE, s.OperationTimestamp), s.OperationTimestamp))
        INNER JOIN DimGeography dg ON dg.LocationID = s.LocationID
        INNER JOIN DimProduct dp ON dp.SKU = s.SKU
        LEFT JOIN DimSupplier ds ON ds.SupplierID = s.SupplierID
        WHERE NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.BatchID = s.BatchID AND f.OrderID = s.OrderID
        );
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

### Step 4: Configure Power BI Connection

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter your server name and database: `LogiFleetPulse`
4. Select DirectQuery mode for real-time data or Import for scheduled refresh
5. Load the fact and dimension tables

### Step 5: Create Power BI Measures

```dax
-- Total Warehouse Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

-- Average Dwell Time
Avg Dwell Time (hrs) = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60

-- Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

-- Cross-Fact KPI: Warehouse-Fleet Efficiency
Warehouse-Fleet Efficiency = 
VAR AvgWarehouseCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeMinutes])
VAR AvgFleetLoadTime = AVERAGE(FactFleetTrips[LoadingTimeMinutes])
RETURN
DIVIDE(
    AvgWarehouseCycleTime + AvgFleetLoadTime,
    60, -- Convert to hours
    BLANK()
)

-- Gravity Zone Performance
High Gravity Zone Pick Rate = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[OperationType] = "Picking",
    FactWarehouseOperations[ZoneGravity] = "High"
) / 
CALCULATE(
    SUM(FactWarehouseOperations[CycleTimeMinutes]),
    FactWarehouseOperations[OperationType] = "Picking",
    FactWarehouseOperations[ZoneGravity] = "High"
) * 60

-- Predictive Bottleneck Index (simplified)
Bottleneck Risk Score = 
VAR CurrentDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvg = CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATEADD(DimTime[Date], -30, DAY)
)
RETURN
DIVIDE(CurrentDwellTime, HistoricalAvg, 1) * 100

-- Fuel Efficiency
Fuel Efficiency (km/L) = 
DIVIDE(
    SUM(FactFleetTrips[TripDistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)
```

## Configuration

### Data Source Integration

Create a configuration file for external data sources:

```json
{
  "dataSources": {
    "wms": {
      "type": "sql",
      "connectionString": "${WMS_CONNECTION_STRING}",
      "refreshInterval": "15min"
    },
    "telematics": {
      "type": "rest_api",
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}",
      "refreshInterval": "5min"
    },
    "weather": {
      "type": "rest_api",
      "endpoint": "https://api.weather.com/v1",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshInterval": "1hour"
    }
  },
  "staging": {
    "database": "LogiFleetPulse_Staging",
    "retentionDays": 7
  },
  "security": {
    "enableRowLevelSecurity": true,
    "roles": ["Operator", "Supervisor", "Executive"]
  }
}
```

### Row-Level Security Setup

```sql
-- Create security table
CREATE TABLE SecurityUserRole (
    UserEmail VARCHAR(200) PRIMARY KEY,
    RoleName VARCHAR(50) NOT NULL,
    GeographyFilter VARCHAR(500), -- Comma-separated location IDs
    IsActive BIT DEFAULT 1
);

-- Create RLS function
CREATE FUNCTION SecurityFilter(@UserEmail VARCHAR(200))
RETURNS TABLE
AS
RETURN (
    SELECT GeographyKey
    FROM DimGeography dg
    INNER JOIN SecurityUserRole sur ON dg.LocationID IN (
        SELECT value FROM STRING_SPLIT(sur.GeographyFilter, ',')
    )
    WHERE sur.UserEmail = @UserEmail
      AND sur.IsActive = 1
);
GO

-- Apply to fact tables (example for warehouse operations)
CREATE SECURITY POLICY WarehouseOperationsSecurity
ADD FILTER PREDICATE dbo.SecurityFilter(USER_NAME()) 
    ON FactWarehouseOperations
WITH (STATE = ON);
```

### Automated Alerting

```sql
-- Create alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(200) NOT NULL,
    MetricName VARCHAR(100) NOT NULL,
    ThresholdOperator VARCHAR(10) NOT NULL, -- >, <, >=, <=, =
    ThresholdValue DECIMAL(10, 2) NOT NULL,
    Recipients VARCHAR(MAX) NOT NULL, -- JSON array of email addresses
    IsActive BIT DEFAULT 1
);

-- Insert sample alerts
INSERT INTO AlertConfiguration (AlertName, MetricName, ThresholdOperator, ThresholdValue, Recipients)
VALUES 
    ('High Fleet Idle Time', 'IdleTimePercent', '>', 15, '["fleet.manager@company.com"]'),
    ('Warehouse Dwell Time Alert', 'AvgDwellTimeHours', '>', 72, '["warehouse.supervisor@company.com"]'),
    ('Fuel Efficiency Drop', 'FuelEfficiencyKmPerL', '<', 8.5, '["operations.director@company.com"]');

-- Stored procedure to check alerts
CREATE PROCEDURE CheckAlerts
AS
BEGIN
    DECLARE @AlertResults TABLE (
        AlertName VARCHAR(200),
        MetricValue DECIMAL(10, 2),
        ThresholdValue DECIMAL(10, 2),
        Recipients VARCHAR(MAX)
    );
    
    -- Fleet idle time check
    INSERT INTO @AlertResults
    SELECT 
        ac.AlertName,
        (SUM(ft.IdleTimeMinutes) * 100.0 / SUM(ft.TripDurationMinutes)) AS MetricValue,
        ac.ThresholdValue,
        ac.Recipients
    FROM AlertConfiguration ac
    CROSS APPLY (
        SELECT 
            SUM(IdleTimeMinutes) AS IdleTimeMinutes,
            SUM(TripDurationMinutes) AS TripDurationMinutes
        FROM FactFleetTrips ft
        INNER JOIN DimTime dt ON ft.TimeKey = dt.TimeKey
        WHERE dt.Date = CAST(GETDATE() AS DATE)
    ) ft
    WHERE ac.MetricName = 'IdleTimePercent'
      AND ac.IsActive = 1
      AND (SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(ft.TripDurationMinutes), 0)) > ac.ThresholdValue;
    
    -- Send alerts (integrate with email service)
    -- Implementation depends on your email service (SQL Server Database Mail, Azure Logic Apps, etc.)
    
    SELECT * FROM @AlertResults;
END;
GO
```

## Common Patterns

### Pattern 1: Cross-Fact Analysis (Warehouse Impact on Fleet)

```sql
-- Analyze how warehouse dwell time affects fleet departure delays
SELECT 
    dg.LocationName AS Warehouse,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMin,
    AVG(ft.DelayMinutes) AS AvgFleetDelayMin,
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    CORR(wo.DwellTimeMinutes, ft.DelayMinutes) AS Correlation
FROM FactWarehouseOperations wo
INNER JOIN DimGeography dg ON wo.GeographyKey = dg.GeographyKey
INNER JOIN FactFleetTrips ft ON ft.OriginGeographyKey = wo.GeographyKey
    AND DATEDIFF(MINUTE, wo.OperationTimestamp, ft.TripStartTimestamp) BETWEEN 0 AND 120
WHERE wo.OperationType = 'Shipping'
  AND dg.LocationType = 'Warehouse'
GROUP BY dg.LocationName
HAVING COUNT(DISTINCT ft.TripKey) > 10
ORDER BY Correlation DESC;
```

### Pattern 2: Gravity Zone Optimization Recommendation

```sql
-- Identify products that should be moved to different gravity zones
WITH ProductPerformance AS (
    SELECT 
        dp.ProductKey,
        dp.SKU,
        dp.ProductName,
        dp.GravityScore AS CurrentGravityScore,
        COUNT(*) AS PickCount,
        AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
        SUM(wo.Quantity * wo.OperationCost) AS TotalRevenue
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct dp ON wo.ProductKey = dp.ProductKey
    INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE wo.OperationType = 'Picking'
      AND dt.Date >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY dp.ProductKey, dp.SKU, dp.ProductName, dp.GravityScore
),
RecommendedGravity AS (
    SELECT 
        *,
        CASE 
            WHEN PickCount > 100 AND TotalRevenue > 10000 THEN 'High'
            WHEN PickCount > 50 OR TotalRevenue > 5000 THEN 'Medium'
            ELSE 'Low'
        END AS RecommendedZone,
        CASE 
            WHEN CurrentGravityScore >= 75 THEN 'High'
            WHEN CurrentGravityScore >= 40 THEN 'Medium'
            ELSE 'Low'
        END AS CurrentZone
    FROM ProductPerformance
)
SELECT 
    SKU,
    ProductName,
    CurrentZone,
    RecommendedZone,
    PickCount,
    AvgCycleTime,
    TotalRevenue,
    CASE 
        WHEN CurrentZone <> RecommendedZone THEN 'RELOCATE'
        ELSE 'OK'
    END AS Action
FROM RecommendedGravity
WHERE CurrentZone <> RecommendedZone
ORDER BY TotalRevenue DESC;
```

### Pattern 3: Predictive Maintenance Priority Queue

```sql
-- Fleet maintenance prioritization based on telemetry and load value
WITH FleetStatus AS (
    SELECT 
        ft.VehicleID,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.TripDistanceKm, 0)) AS AvgFuelConsumption,
        SUM(ft.LoadWeightKg) AS TotalLoadWeight,
        SUM(ft.DelayMinutes) AS TotalDelays,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN DimTime dt ON ft.TimeKey = dt.TimeKey
    WHERE dt.Date >= DATEADD(DAY, -7, GETDATE())
    GROUP BY ft.VehicleID
),
LoadValueImpact AS (
    SELECT 
        ft.VehicleID,
        SUM(wo.OperationCost) AS TotalLoadValue
    FROM FactFleetTrips ft
    INNER JOIN FactCrossDock cd ON ft.TripKey = cd.OutboundTripKey
    INNER JOIN FactWarehouseOperations wo ON cd.ProductKey = wo.ProductKey
    GROUP BY ft.VehicleID
)
SELECT 
    fs.VehicleID,
    fs.AvgFuelConsumption,
    fs.TotalDelays,
    lvi.TotalLoadValue,
    (fs.AvgFuelConsumption * 100 + fs.TotalDelays * 10 + lvi.TotalLoadValue / 1000) AS MaintenancePriority
FROM FleetStatus fs
LEFT JOIN LoadValueImpact lvi ON fs.VehicleID = lvi.VehicleID
WHERE fs.AvgFuelConsumption > (SELECT AVG(AvgFuelConsumption) * 1.15 FROM FleetStatus)
   OR fs.TotalDelays > 60
ORDER BY MaintenancePriority DESC;
```

### Pattern 4: Time-Phased Scenario Simulation

```sql
-- Simulate impact of increasing warehouse capacity from 80% to 95%
DECLARE @CurrentCapacityPct DECIMAL(5,2) = 80.0;
DECLARE @SimulatedCapacityPct DECIMAL(5,2) = 95.0;

WITH CapacityMetrics AS (
    SELECT 
        dt.Date,
        COUNT(*) AS OperationCount,
        AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.Date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY dt.Date
),
SimulatedImpact AS (
    SELECT 
        Date,
        OperationCount,
        AvgCycleTime,
        AvgDwellTime,
        -- Simulate congestion: cycle time increases non-linearly near capacity
        AvgCycleTime * POWER((@SimulatedCapacityPct / @CurrentCapacityPct), 2) AS SimulatedCycleTime,
        AvgDwellTime * (@SimulatedCapacityPct / @CurrentCapacityPct) AS SimulatedDwellTime
    FROM CapacityMetrics
)
SELECT 
    AVG(AvgCycleTime) AS Current_Avg_CycleTime,
    AVG(SimulatedCycleTime) AS Simulated_Avg_CycleTime,
    AVG(AvgDwellTime) AS Current_Avg_DwellTime,
    AVG(SimulatedDwellTime) AS Simulated_Avg_DwellTime,
    (AVG(SimulatedCycleTime) - AVG(AvgCycleTime)) / AVG(AvgCycleTime) * 100 AS CycleTime_Change_Pct,
    (AVG(SimulatedDwellTime) - AVG(AvgDwellTime)) / AVG(AvgDwellTime) * 100 AS DwellTime_Change_Pct
FROM SimulatedImpact;
```

## Power BI Dashboard Configuration

### Creating the Main Dashboard

1. **Import the template**: Use the provided `.pbit` file or create a new report
2. **Add key visuals**:
   - Card: Total Operations, Fleet Utilization %, Avg Dwell Time
   - Line Chart: Operations over time (by 15-min buckets)
   - Map: Warehouse and route nodes with size by throughput
   - Matrix: Product gravity zones vs. pick frequency
   - Gauge: Bottleneck Risk Score with threshold indicators

### DAX Time Intelligence Patterns

```dax
-- Year-over-Year comparison
YoY Operations Growth % = 
VAR CurrentPeriod = [Total Operations]
VAR PriorYear = CALCULATE(
    [Total Operations],
    SAMEPERIODLASTYEAR(DimTime[Date])
)
RETURN
DIVIDE(CurrentPeriod - PriorYear, PriorYear, 0) * 100

-- Moving Average (7-day)
MA7 Dwell Time = 
CALCULATE(
    [Avg Dwell Time (hrs)],
    DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -7, DAY)
)

-- Dynamic period selection
Selected Period Operations = 
VAR SelectedPeriodType = SELECTEDVALUE(PeriodSelector[PeriodType])
RETURN
SWITCH(
    SelectedPeriodType,
    "Daily", CALCULATE([Total Operations], DimTime[Date] = TODAY()),
    "Weekly", CALCULATE([Total Operations], DATESINPERIOD(DimTime[Date], TODAY(), -7, DAY)),
    "Monthly", CALCULATE([Total Operations], DATESMTD(DimTime[Date])),
    [Total Operations]
)
```

## Troubleshooting

### Issue: Power BI refresh fails with timeout

**Solution**: Switch from Import to DirectQuery mode or implement incremental refresh:

```powerquery
// M query for incremental refresh
let
    Source = Sql.Database("YourServer", "LogiFleetPulse"),
    FilteredRows = Table.SelectRows(Source, each [OperationTimestamp] >= RangeStart and [OperationTimestamp] < RangeEnd)
in
    FilteredRows
```

Configure incremental refresh policy: Store last 7 days, refresh daily.

### Issue: Slow cross-fact queries

**Solution**: Create indexed views for common cross-fact patterns:

```sql
CREATE VIEW vw_WarehouseFleetCorrelation
WITH SCHEMABINDING
AS
SELECT 
    wo.GeographyKey,
    dt.Date,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(ft.DelayMinutes) AS AvgFleetDelay,
    COUNT_BIG(*) AS RecordCount
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.FactFleetTrips ft ON ft.OriginGeographyKey = wo.GeographyKey
INNER JOIN dbo.DimTime dt ON wo.TimeKey = dt.TimeKey
WHERE wo.OperationType = 'Shipping'
GROUP BY wo.GeographyKey,
