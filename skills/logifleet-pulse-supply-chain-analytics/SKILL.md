---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing engine for logistics, fleet management, and supply chain intelligence with multi-fact star schema modeling
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy SQL server data warehouse for fleet management"
  - "implement multi-fact star schema for logistics"
  - "create warehouse gravity zone analytics"
  - "build cross-modal supply chain KPI dashboard"
  - "connect fleet telemetry to warehouse operations"
  - "setup real-time logistics intelligence platform"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced data warehousing and analytics platform that unifies warehouse operations, fleet telemetry, inventory management, and supply chain metrics into a single semantic layer. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema to enable cross-domain KPI analysis (e.g., correlating warehouse dwell time with fleet fuel consumption).

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse gravity zone optimization (spatial analysis based on pick frequency and value)
- Fleet telemetry integration for predictive maintenance
- Cross-fact KPI harmonization (warehouse ↔ fleet ↔ supplier metrics)
- Real-time anomaly detection and bottleneck prediction
- Role-based dashboards with row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, TMS, telematics APIs

### Step 1: Deploy SQL Schema

1. **Clone or download the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Execute the schema creation script:**
```sql
-- Connect to your SQL Server instance
-- Run schema_master.sql in SSMS or Azure Data Studio

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeFull DATETIME NOT NULL,
    TimeSlot15Min SMALLINT,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayName NVARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod NVARCHAR(7),
    FiscalQuarter NVARCHAR(6),
    FiscalYear SMALLINT
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode NVARCHAR(20) UNIQUE NOT NULL,
    LocationName NVARCHAR(100),
    LocationType NVARCHAR(20), -- 'Warehouse', 'RouteNode', 'Depot'
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    TimeZone NVARCHAR(50)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite score: velocity * value * fragility
    VelocityClass NVARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier NVARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex TINYINT, -- 1-10 scale
    OptimalZoneType NVARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,4)
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode NVARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    LeadTimeAvg INT, -- Days
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier NVARCHAR(20) -- 'Excellent', 'Good', 'Fair', 'Poor'
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DwellTimeMinutes INT,
    CycleTimeMinutes INT,
    ZoneType NVARCHAR(50),
    EmployeeID NVARCHAR(50),
    BatchID NVARCHAR(50),
    ErrorFlag BIT DEFAULT 0
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    RouteID NVARCHAR(50),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeight DECIMAL(10,2),
    OnTimeDelivery BIT,
    WeatherCondition NVARCHAR(50),
    TrafficDelay BIT
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    QuantityTransferred INT,
    TransferTimeMinutes INT,
    DirectShipFlag BIT
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey);
```

### Step 2: Configure Data Sources

Create a configuration file for ETL connections:

```sql
-- Create configuration table for external connections
CREATE TABLE ETLConfig (
    ConfigKey NVARCHAR(100) PRIMARY KEY,
    ConfigValue NVARCHAR(500),
    Description NVARCHAR(200),
    LastUpdated DATETIME DEFAULT GETDATE()
);

-- Insert connection strings (use environment variables in production)
INSERT INTO ETLConfig (ConfigKey, ConfigValue, Description) VALUES
('WMS_API_ENDPOINT', '${WMS_API_URL}', 'Warehouse Management System API'),
('TELEMATICS_API_ENDPOINT', '${TELEMATICS_API_URL}', 'Fleet Telematics Data Feed'),
('ERP_CONNECTION_STRING', '${ERP_DB_CONNECTION}', 'ERP Database Connection'),
('WEATHER_API_KEY', '${WEATHER_API_KEY}', 'External Weather Data API Key');
```

### Step 3: Load Sample Data (for testing)

```sql
-- Populate DimTime (15-minute granularity for one year)
DECLARE @StartDate DATETIME = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2026-12-31 23:45:00';
DECLARE @CurrentDate DATETIME = @StartDate;
DECLARE @TimeKey INT = 1;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, DateTimeFull, TimeSlot15Min, HourOfDay, DayOfWeek, DayName, IsWeekend, FiscalPeriod, FiscalQuarter, FiscalYear)
    VALUES (
        @TimeKey,
        @CurrentDate,
        DATEPART(MINUTE, @CurrentDate) / 15,
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1,7) THEN 1 ELSE 0 END,
        FORMAT(@CurrentDate, 'yyyy-MM'),
        CONCAT('Q', DATEPART(QUARTER, @CurrentDate), '-', YEAR(@CurrentDate)),
        YEAR(@CurrentDate)
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
    SET @TimeKey = @TimeKey + 1;
END;

-- Sample geography data
INSERT INTO DimGeography (LocationCode, LocationName, LocationType, City, StateProvince, Country, Latitude, Longitude, TimeZone)
VALUES 
('WH-001', 'Central Distribution Center', 'Warehouse', 'Chicago', 'Illinois', 'USA', 41.8781, -87.6298, 'America/Chicago'),
('WH-002', 'West Coast Hub', 'Warehouse', 'Los Angeles', 'California', 'USA', 34.0522, -118.2437, 'America/Los_Angeles'),
('RN-101', 'Route Node - Downtown', 'RouteNode', 'Chicago', 'Illinois', 'USA', 41.8825, -87.6324, 'America/Chicago');

-- Sample product data with gravity scores
INSERT INTO DimProductGravity (SKU, ProductName, Category, SubCategory, GravityScore, VelocityClass, ValueTier, FragilityIndex, OptimalZoneType, UnitWeight, UnitVolume)
VALUES
('SKU-1001', 'Premium Electronics - Smartphone', 'Electronics', 'Mobile Devices', 85.5, 'Fast', 'High', 7, 'High-Gravity', 0.3, 0.002),
('SKU-2001', 'Bulk Paper Towels', 'Consumer Goods', 'Paper Products', 35.2, 'Medium', 'Low', 2, 'Low-Gravity', 5.0, 0.05),
('SKU-3001', 'Frozen Food - Premium Ice Cream', 'Perishables', 'Frozen', 72.8, 'Fast', 'Medium', 5, 'High-Gravity', 1.2, 0.008);
```

### Step 4: Connect Power BI

1. Open `LogiFleet_Pulse_Master.pbit` from the repository
2. When prompted, enter your SQL Server connection details:
   - Server: `YOUR_SQL_SERVER_INSTANCE`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server
3. The Power BI model will auto-detect fact and dimension tables
4. Refresh the data model to populate visualizations

## Key SQL Queries & Stored Procedures

### Cross-Fact Analysis: Warehouse Dwell Time vs Fleet Idle Time

```sql
-- Find correlations between warehouse delays and fleet inefficiencies
CREATE PROCEDURE sp_AnalyzeDwellImpactOnFleet
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    WITH WarehouseDwell AS (
        SELECT 
            wo.ProductKey,
            wo.GeographyKey,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
            SUM(wo.QuantityHandled) AS TotalVolume
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateTimeFull BETWEEN @StartDate AND @EndDate
        GROUP BY wo.ProductKey, wo.GeographyKey
    ),
    FleetPerformance AS (
        SELECT 
            ft.OriginGeographyKey AS GeographyKey,
            AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
            AVG(CAST(ft.OnTimeDelivery AS FLOAT)) AS OnTimeRate
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.DateTimeFull BETWEEN @StartDate AND @EndDate
        GROUP BY ft.OriginGeographyKey
    )
    SELECT 
        g.LocationName,
        p.ProductName,
        p.VelocityClass,
        wd.AvgDwellTime,
        fp.AvgIdleTime,
        fp.OnTimeRate,
        -- Impact score: higher dwell + higher idle = bigger problem
        (wd.AvgDwellTime * fp.AvgIdleTime) / NULLIF(fp.OnTimeRate, 0) AS ImpactScore
    FROM WarehouseDwell wd
    INNER JOIN FleetPerformance fp ON wd.GeographyKey = fp.GeographyKey
    INNER JOIN DimGeography g ON wd.GeographyKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON wd.ProductKey = p.ProductKey
    ORDER BY ImpactScore DESC;
END;
GO
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend warehouse zone reassignments based on product gravity
CREATE PROCEDURE sp_OptimizeWarehouseZones
    @WarehouseGeographyKey INT
AS
BEGIN
    WITH ProductActivity AS (
        SELECT 
            wo.ProductKey,
            wo.ZoneType AS CurrentZone,
            COUNT(*) AS PickCount,
            AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
            SUM(wo.QuantityHandled) AS TotalVolume
        FROM FactWarehouseOperations wo
        WHERE wo.GeographyKey = @WarehouseGeographyKey
          AND wo.OperationType = 'Picking'
          AND wo.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
        GROUP BY wo.ProductKey, wo.ZoneType
    )
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.OptimalZoneType AS RecommendedZone,
        pa.CurrentZone,
        pa.PickCount,
        pa.AvgCycleTime,
        CASE 
            WHEN p.OptimalZoneType <> pa.CurrentZone THEN 'REALLOCATE'
            ELSE 'OK'
        END AS Action,
        -- Estimated time savings if moved to optimal zone
        CASE 
            WHEN p.OptimalZoneType = 'High-Gravity' AND pa.CurrentZone <> 'High-Gravity' 
            THEN pa.AvgCycleTime * 0.35 * pa.PickCount -- 35% reduction
            ELSE 0
        END AS EstimatedTimeSavingsMinutes
    FROM ProductActivity pa
    INNER JOIN DimProductGravity p ON pa.ProductKey = p.ProductKey
    WHERE p.OptimalZoneType <> pa.CurrentZone
    ORDER BY EstimatedTimeSavingsMinutes DESC;
END;
GO
```

### Predictive Fleet Maintenance Queue

```sql
-- Generate maintenance priority based on telemetry patterns
CREATE PROCEDURE sp_PredictiveMaintenanceQueue
AS
BEGIN
    WITH FleetRisk AS (
        SELECT 
            ft.VehicleID,
            COUNT(*) AS TripCount,
            AVG(ft.IdleTimeMinutes) AS AvgIdle,
            AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelEfficiency,
            SUM(CASE WHEN ft.OnTimeDelivery = 0 THEN 1 ELSE 0 END) AS DelayCount,
            -- Simulate telematics flags (in production, join to actual sensor data)
            CASE WHEN AVG(ft.IdleTimeMinutes) > 30 THEN 1 ELSE 0 END AS HighIdleFlag,
            CASE WHEN AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) > 0.15 THEN 1 ELSE 0 END AS LowEfficiencyFlag
        FROM FactFleetTrips ft
        WHERE ft.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days
        GROUP BY ft.VehicleID
    ),
    RevenueImpact AS (
        SELECT 
            ft.VehicleID,
            SUM(p.GravityScore * wo.QuantityHandled) AS RevenueAtRisk -- Proxy: gravity * volume
        FROM FactFleetTrips ft
        INNER JOIN FactWarehouseOperations wo ON ft.TimeKey = wo.TimeKey
        INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        WHERE ft.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
        GROUP BY ft.VehicleID
    )
    SELECT 
        fr.VehicleID,
        fr.TripCount,
        fr.AvgIdle,
        fr.AvgFuelEfficiency,
        fr.DelayCount,
        ri.RevenueAtRisk,
        -- Risk score: combine operational flags with revenue impact
        (fr.HighIdleFlag * 30 + fr.LowEfficiencyFlag * 25 + fr.DelayCount * 10) * 
        (1 + LOG(NULLIF(ri.RevenueAtRisk, 0))) AS MaintenancePriorityScore
    FROM FleetRisk fr
    LEFT JOIN RevenueImpact ri ON fr.VehicleID = ri.VehicleID
    WHERE fr.HighIdleFlag = 1 OR fr.LowEfficiencyFlag = 1 OR fr.DelayCount > 2
    ORDER BY MaintenancePriorityScore DESC;
END;
GO
```

## Power BI Configuration

### DAX Measures for Cross-Fact KPIs

```dax
// Warehouse Velocity Index
WarehouseVelocityIndex = 
VAR TotalPicks = SUM(FactWarehouseOperations[QuantityHandled])
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeMinutes])
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
RETURN
DIVIDE(TotalPicks, AvgCycleTime + AvgDwell, 0) * 100

// Fleet Efficiency Score
FleetEfficiencyScore = 
VAR OnTimeRate = DIVIDE(
    CALCULATE(COUNT(FactFleetTrips[TripKey]), FactFleetTrips[OnTimeDelivery] = TRUE),
    COUNT(FactFleetTrips[TripKey]),
    0
)
VAR IdleRatio = DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
)
VAR FuelEfficiency = DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)
RETURN
(OnTimeRate * 0.5 + (1 - IdleRatio) * 0.3 + (FuelEfficiency / 10) * 0.2) * 100

// Cross-Fact Bottleneck Indicator
BottleneckRisk = 
VAR HighDwellProducts = CALCULATE(
    DISTINCTCOUNT(FactWarehouseOperations[ProductKey]),
    FactWarehouseOperations[DwellTimeMinutes] > 72
)
VAR DelayedTrips = CALCULATE(
    COUNT(FactFleetTrips[TripKey]),
    FactFleetTrips[OnTimeDelivery] = FALSE
)
RETURN
IF(HighDwellProducts > 5 && DelayedTrips > 10, "HIGH", 
   IF(HighDwellProducts > 2 || DelayedTrips > 5, "MEDIUM", "LOW"))
```

### Row-Level Security (RLS) Setup

```dax
// Create role: WarehouseManager
[GeographyKey] IN VALUES(
    UserGeographyAccess[GeographyKey]
)

// Create role: FleetSupervisor
[VehicleID] IN VALUES(
    UserFleetAccess[VehicleID]
)
```

Then create mapping tables in SQL:
```sql
CREATE TABLE UserGeographyAccess (
    UserEmail NVARCHAR(200),
    GeographyKey INT
);

CREATE TABLE UserFleetAccess (
    UserEmail NVARCHAR(200),
    VehicleID NVARCHAR(50)
);
```

## Common Patterns & Use Cases

### Pattern 1: Real-Time Alerting

```sql
-- Create alert procedure for high dwell time
CREATE PROCEDURE sp_AlertHighDwellTime
    @ThresholdHours INT = 48,
    @EmailRecipients NVARCHAR(500)
AS
BEGIN
    DECLARE @AlertBody NVARCHAR(MAX);
    
    WITH HighDwellItems AS (
        SELECT 
            p.SKU,
            p.ProductName,
            g.LocationName,
            MAX(wo.DwellTimeMinutes) / 60.0 AS DwellHours,
            SUM(wo.QuantityHandled) AS Quantity
        FROM FactWarehouseOperations wo
        INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
        WHERE wo.DwellTimeMinutes > (@ThresholdHours * 60)
          AND wo.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
        GROUP BY p.SKU, p.ProductName, g.LocationName
    )
    SELECT @AlertBody = STRING_AGG(
        CONCAT(SKU, ' - ', ProductName, ' at ', LocationName, ': ', 
               CAST(DwellHours AS VARCHAR), ' hours (', Quantity, ' units)'),
        CHAR(13) + CHAR(10)
    )
    FROM HighDwellItems;
    
    IF @AlertBody IS NOT NULL
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @EmailRecipients,
            @subject = 'ALERT: High Dwell Time Detected',
            @body = @AlertBody;
    END;
END;
GO

-- Schedule via SQL Agent Job (runs every 15 minutes)
EXEC sp_AlertHighDwellTime @ThresholdHours = 48, @EmailRecipients = '${ALERT_EMAIL}';
```

### Pattern 2: Incremental Data Loading

```sql
-- ETL procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimeKey INT
AS
BEGIN
    -- Assuming external staging table populated by SSIS or Azure Data Factory
    INSERT INTO FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, SupplierKey, OperationType, QuantityHandled, DwellTimeMinutes, CycleTimeMinutes, ZoneType, EmployeeID, BatchID)
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.QuantityHandled,
        stg.DwellTimeMinutes,
        stg.CycleTimeMinutes,
        stg.ZoneType,
        stg.EmployeeID,
        stg.BatchID
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime t ON stg.OperationDateTime = t.DateTimeFull
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON stg.SupplierCode = s.SupplierCode
    WHERE t.TimeKey > @LastLoadTimeKey;
    
    -- Update last load watermark
    UPDATE ETLConfig SET ConfigValue = CAST((SELECT MAX(TimeKey) FROM FactWarehouseOperations) AS NVARCHAR(50))
    WHERE ConfigKey = 'LastWarehouseLoadTimeKey';
END;
GO
```

### Pattern 3: Temporal Elasticity Simulation

```sql
-- Simulate capacity changes
CREATE PROCEDURE sp_SimulateCapacityImpact
    @CapacityIncreasePct DECIMAL(5,2),
    @SimulationDays INT = 30
AS
BEGIN
    WITH BaselineMetrics AS (
        SELECT 
            AVG(wo.DwellTimeMinutes) AS BaselineDwell,
            AVG(wo.CycleTimeMinutes) AS BaselineCycle,
            AVG(CAST(ft.OnTimeDelivery AS FLOAT)) AS BaselineOTD
        FROM FactWarehouseOperations wo
        CROSS JOIN FactFleetTrips ft
        WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - (@SimulationDays * 96) FROM DimTime)
    ),
    SimulatedMetrics AS (
        SELECT 
            -- Assume capacity increase reduces dwell by logarithmic factor
            AVG(wo.DwellTimeMinutes) * (1 - (LOG(@CapacityIncreasePct + 100) / LOG(200))) AS SimulatedDwell,
            AVG(wo.CycleTimeMinutes) * 0.95 AS SimulatedCycle, -- 5% improvement
            AVG(CAST(ft.OnTimeDelivery AS FLOAT)) + 0.05 AS SimulatedOTD -- +5% OTD
        FROM FactWarehouseOperations wo
        CROSS JOIN FactFleetTrips ft
        WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - (@SimulationDays * 96) FROM DimTime)
    )
    SELECT 
        b.BaselineDwell,
        s.SimulatedDwell,
        b.BaselineCycle,
        s.SimulatedCycle,
        b.BaselineOTD,
        s.SimulatedOTD,
        (b.BaselineDwell - s.SimulatedDwell) / NULLIF(b.BaselineDwell, 0) * 100 AS DwellReductionPct,
        (s.SimulatedOTD - b.BaselineOTD) * 100 AS OTDImprovementPct
    FROM BaselineMetrics b
    CROSS JOIN SimulatedMetrics s;
END;
GO

-- Execute simulation
EXEC sp_SimulateCapacityImpact @CapacityIncreasePct = 15.0, @SimulationDays = 30;
```

## Troubleshooting

### Issue: Power BI Refresh Fails

**Symptom:** "Unable to connect to data source" or timeout errors

**Solutions:**
1. Verify SQL Server allows remote connections:
   ```sql
   EXEC sp_configure 'remote access', 1;
   RECONFIGURE;
   ```

2. Check firewall rules for port 1433 (default SQL Server port)

3. Increase Power BI Gateway timeout:
   - Open Gateway settings → Connectors → SQL Server
   - Set query timeout to 600 seconds

4. Partition large fact tables:
   ```sql
   -- Create partition function for monthly partitions
   CREATE PARTITION FUNCTION PF_MonthlyPartition (INT)
   AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, ...);
   
   -- Apply to fact table (rebuild required)
   CREATE PARTITION SCHEME PS_Monthly AS PARTITION PF_MonthlyPartition ALL TO ([PRIMARY]);
   ```

### Issue: Slow Cross-Fact Queries

**Symptom:** Queries joining multiple fact tables take > 30 seconds

**Solutions:**
1. Add composite indexes on join columns:
   ```sql
   CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeGeo 
   ON FactWarehouseOperations(TimeKey, GeographyKey) 
   INCLUDE (ProductKey, QuantityHandled);
   ```

2. Use indexed views for common aggregations:
   ```sql
   CREATE VIEW vw_DailyWarehouseSummary WITH SCHEMABINDING
   AS
   SELECT 
       wo.TimeKey,
       wo.GeographyKey,
       wo.ProductKey,
       COUNT_BIG(*) AS OpCount,
       SUM(wo.QuantityHandled) AS TotalQuantity,
       AVG(wo.DwellTimeMinutes) AS AvgDwell
   FROM dbo.FactWarehouseOperations wo
   GROUP BY wo.TimeKey, wo.GeographyKey, wo.ProductKey;
   GO
   
   CREATE UNIQUE CLUSTERED INDEX IX_DailyWarehouseSummary 
   ON vw_DailyWarehouseSummary(TimeKey, GeographyKey, ProductKey);
   ```

3. Enable query store for performance tracking:
   ```sql
   ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;
   ```

### Issue: Gravity Zone Recommendations Not Updating

**Symptom:** `sp_OptimizeWarehouseZones` returns stale data

**Solutions:**
1. Verify data freshness:
   ```sql
   SELECT MAX(t.DateTimeFull) AS LastDataTimestamp
   FROM FactWarehouseOperations wo
   INNER JOIN DimTime
