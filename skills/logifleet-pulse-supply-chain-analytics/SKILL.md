---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-fact data warehouse for logistics intelligence, fleet optimization, and cross-modal supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logistics data warehouse with SQL Server"
  - "configure Power BI for fleet and warehouse intelligence"
  - "implement multi-fact star schema for logistics"
  - "build real-time supply chain dashboard"
  - "create warehouse gravity zone optimization"
  - "integrate fleet telemetry with warehouse operations"
  - "set up cross-fact KPI harmonization for logistics"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence platform that combines MS SQL Server data warehousing with Power BI visualization to create a unified view across warehouse operations, fleet management, and supply chain performance. It implements a multi-fact star schema that harmonizes cross-domain KPIs, enabling queries like "show shipments delayed by weather that originated from high-dwell-time storage zones."

Key capabilities:
- **Multi-fact star schema** linking warehouse, fleet, and supplier data
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and item value
- **Adaptive Fleet Triage Engine** with predictive maintenance prioritization
- **Time-phased dimensions** at 15-minute granularity for real-time insights
- **Cross-fact KPI harmonization** (e.g., inventory turnover vs. fuel burn rates)
- **Role-based security** with Power BI row-level filtering

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telemetry feeds, ERP systems

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeValue DATETIME NOT NULL,
    Hour INT,
    DayOfWeek INT,
    IsBusinessHour BIT,
    FiscalPeriod VARCHAR(10),
    FiscalQuarter VARCHAR(10)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(50) UNIQUE,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Route Node', 'CrossDock'
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,2),
    IsFragile BIT,
    IsPerishable BIT,
    TemperatureRequirement VARCHAR(50), -- 'Frozen', 'Refrigerated', 'Ambient'
    GravityScore DECIMAL(5,2) -- Calculated: velocity * value * (1 + fragility)
);

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY,
    SupplierID VARCHAR(50) UNIQUE,
    SupplierName VARCHAR(200),
    ReliabilityScore DECIMAL(3,2), -- 0.00 to 1.00
    AvgLeadTimeDays INT,
    DefectRatePercent DECIMAL(5,2)
);

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY,
    VehicleID VARCHAR(50) UNIQUE,
    VehicleType VARCHAR(50), -- 'Van', 'Truck', 'Refrigerated'
    Capacity DECIMAL(10,2),
    FuelType VARCHAR(50),
    MaintenanceStatus VARCHAR(50) -- 'Operational', 'Scheduled', 'Critical'
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplier(SupplierKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityUnits INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    FleetKey INT FOREIGN KEY REFERENCES DimFleet(FleetKey),
    OriginKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    WeatherCondition VARCHAR(50), -- 'Clear', 'Rain', 'Snow', 'Storm'
    TrafficDelayMinutes INT,
    DriverID VARCHAR(50)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OutboundTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    QuantityUnits INT,
    TransferTimeMinutes INT,
    BypassStorage BIT
);

-- Create indexes for performance
CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE INDEX IX_FactFleet_Fleet ON FactFleetTrips(FleetKey);
CREATE INDEX IX_FactFleet_Route ON FactFleetTrips(OriginKey, DestinationKey);
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate 15-minute time buckets
CREATE PROCEDURE sp_PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    DECLARE @TimeKey INT;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        SET @TimeKey = CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT);
        
        INSERT INTO DimTime (TimeKey, DateTimeValue, Hour, DayOfWeek, IsBusinessHour, FiscalPeriod, FiscalQuarter)
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            CASE 
                WHEN DATEPART(HOUR, @CurrentDateTime) BETWEEN 8 AND 17 
                     AND DATEPART(WEEKDAY, @CurrentDateTime) BETWEEN 2 AND 6 
                THEN 1 
                ELSE 0 
            END,
            CONCAT('FY', YEAR(@CurrentDateTime), '-', FORMAT(MONTH(@CurrentDateTime), '00')),
            CONCAT('FY', YEAR(@CurrentDateTime), '-Q', DATEPART(QUARTER, @CurrentDateTime))
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Populate for 2 years
EXEC sp_PopulateTimeDimension '2025-01-01', '2026-12-31';
```

### Step 3: Calculate Warehouse Gravity Scores

```sql
-- Update product gravity scores based on velocity, value, and fragility
CREATE PROCEDURE sp_UpdateGravityScores
AS
BEGIN
    UPDATE p
    SET p.GravityScore = (
        -- Velocity (picks per day normalized)
        (SELECT COUNT(*) FROM FactWarehouseOperations wo 
         WHERE wo.ProductKey = p.ProductKey 
           AND wo.OperationType = 'Picking'
           AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd0000') AS INT)
        ) / 30.0 * 10.0
        +
        -- Value placeholder (should join to pricing table in real implementation)
        50.0
        +
        -- Fragility multiplier
        (CASE WHEN p.IsFragile = 1 THEN 20.0 ELSE 0 END)
        +
        -- Perishability urgency
        (CASE WHEN p.IsPerishable = 1 THEN 30.0 ELSE 0 END)
    )
    FROM DimProduct p;
END;
GO
```

### Step 4: Create Cross-Fact KPI Views

```sql
-- View: Fleet efficiency correlated with warehouse dwell time
CREATE VIEW vw_FleetWarehouseEfficiency AS
SELECT 
    t.DateTimeValue,
    g.Region,
    p.Category,
    -- Warehouse metrics
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    SUM(wo.QuantityUnits) AS TotalUnitsProcessed,
    -- Fleet metrics
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelEfficiency,
    -- Cross-fact KPI: Dwell impact on fleet idle
    CASE 
        WHEN AVG(wo.DwellTimeMinutes) > 180 
             AND AVG(ft.IdleTimeMinutes) > 30 
        THEN 'High Dwell + High Idle'
        ELSE 'Normal'
    END AS BottleneckStatus
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON wo.ProductKey = ft.ProductKey 
    AND CAST(wo.TimeKey / 10000 AS INT) = CAST(ft.TimeKey / 10000 AS INT) -- Same day
GROUP BY t.DateTimeValue, g.Region, p.Category;
GO
```

### Step 5: Configure Data Source Connections

Create a configuration file for your data source connections:

```json
{
  "connections": {
    "wms_api": {
      "type": "REST",
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}",
      "refresh_interval_minutes": 15
    },
    "telematics_feed": {
      "type": "MQTT",
      "broker": "${TELEMATICS_BROKER}",
      "topic": "fleet/+/telemetry",
      "username": "${TELEMATICS_USER}",
      "password": "${TELEMATICS_PASS}"
    },
    "erp_database": {
      "type": "ODBC",
      "connection_string": "Driver={SQL Server};Server=${ERP_SERVER};Database=${ERP_DB};Uid=${ERP_USER};Pwd=${ERP_PASS};"
    },
    "weather_api": {
      "type": "REST",
      "endpoint": "https://api.weather.com/v3/wx/forecast",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "incremental_load": {
    "enabled": true,
    "watermark_table": "ETL_Watermarks",
    "batch_size": 10000
  }
}
```

### Step 6: Set Up Incremental ETL

```sql
-- Watermark table for incremental loads
CREATE TABLE ETL_Watermarks (
    SourceSystem VARCHAR(50) PRIMARY KEY,
    LastLoadedTimestamp DATETIME,
    LastLoadedID BIGINT
);

-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperationsIncremental
AS
BEGIN
    DECLARE @LastTimestamp DATETIME;
    
    SELECT @LastTimestamp = LastLoadedTimestamp 
    FROM ETL_Watermarks 
    WHERE SourceSystem = 'WMS';
    
    -- Insert new records (assumes staging table populated by external ETL tool)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, QuantityUnits, DwellTimeMinutes, 
        ProcessingTimeMinutes, StorageZone, OperatorID
    )
    SELECT 
        CAST(FORMAT(s.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        ds.SupplierKey,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.ProcessingMinutes,
        s.ZoneID,
        s.OperatorID
    FROM Staging_WarehouseOps s
    INNER JOIN DimGeography dg ON s.LocationID = dg.LocationID
    INNER JOIN DimProduct dp ON s.SKU = dp.SKU
    LEFT JOIN DimSupplier ds ON s.SupplierID = ds.SupplierID
    WHERE s.OperationDateTime > @LastTimestamp;
    
    -- Update watermark
    UPDATE ETL_Watermarks 
    SET LastLoadedTimestamp = (SELECT MAX(OperationDateTime) FROM Staging_WarehouseOps)
    WHERE SourceSystem = 'WMS';
END;
GO
```

## Power BI Configuration

### Step 1: Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `${SQL_SERVER_INSTANCE}`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for performance)

### Step 2: Import Fact and Dimension Tables

Import these tables:
- `DimTime`
- `DimGeography`
- `DimProduct`
- `DimSupplier`
- `DimFleet`
- `FactWarehouseOperations`
- `FactFleetTrips`
- `FactCrossDock`
- `vw_FleetWarehouseEfficiency`

### Step 3: Create Relationships in Power BI Model

Power BI should auto-detect relationships, but verify:

```
FactWarehouseOperations[TimeKey] → DimTime[TimeKey] (many-to-one)
FactWarehouseOperations[GeographyKey] → DimGeography[GeographyKey] (many-to-one)
FactWarehouseOperations[ProductKey] → DimProduct[ProductKey] (many-to-one)
FactWarehouseOperations[SupplierKey] → DimSupplier[SupplierKey] (many-to-one)

FactFleetTrips[TimeKey] → DimTime[TimeKey] (many-to-one)
FactFleetTrips[FleetKey] → DimFleet[FleetKey] (many-to-one)
FactFleetTrips[ProductKey] → DimProduct[ProductKey] (many-to-one)
FactFleetTrips[OriginKey] → DimGeography[GeographyKey] (many-to-one, inactive)
FactFleetTrips[DestinationKey] → DimGeography[GeographyKey] (many-to-one)
```

### Step 4: Create DAX Measures

```dax
// Total Operations
TotalOperations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time
AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

// High Dwell Operations (> 3 hours)
HighDwellOperations = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[DwellTimeMinutes] > 180
)

// Fleet Utilization Rate
FleetUtilization = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
)

// Fuel Efficiency (km per liter)
FuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Cross-Fact: Dwell Time Impact on Fleet Idle
DwellImpactScore = 
VAR AvgDwell = [AvgDwellTime]
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    IF(
        AvgDwell > 180 && AvgIdle > 30,
        "Critical",
        IF(AvgDwell > 120 && AvgIdle > 20, "Warning", "Normal")
    )

// Warehouse Gravity Weighted Picks
GravityWeightedPicks = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[QuantityUnits] * 
    RELATED(DimProduct[GravityScore])
)

// Predictive Bottleneck Index (0-100)
BottleneckIndex = 
VAR DwellScore = MIN([AvgDwellTime] / 60 * 20, 40)
VAR IdleScore = MIN(AVERAGE(FactFleetTrips[IdleTimeMinutes]) / 10 * 30, 30)
VAR WeatherImpact = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[WeatherCondition] IN {"Rain", "Snow", "Storm"}
    ) / COUNTROWS(FactFleetTrips) * 30
RETURN
    DwellScore + IdleScore + WeatherImpact
```

### Step 5: Configure Row-Level Security

```dax
// Create role: Regional Managers
// Table: DimGeography
// Filter: [Region] = USERNAME()

// Or for more complex scenarios:
[Region] IN (
    LOOKUPVALUE(
        UserRegionMapping[Region],
        UserRegionMapping[UserEmail],
        USERNAME()
    )
)
```

### Step 6: Set Up Scheduled Refresh

1. Publish report to Power BI Service
2. Go to Dataset settings
3. Configure gateway connection (for on-premises SQL Server)
4. Set refresh schedule: Every 15 minutes (requires Power BI Premium)
5. Configure email notifications for refresh failures

## Key Queries & Patterns

### Pattern 1: Identify High-Gravity Items in Wrong Zones

```sql
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    wo.StorageZone,
    COUNT(*) AS PickFrequency,
    AVG(wo.ProcessingTimeMinutes) AS AvgProcessingTime
FROM FactWarehouseOperations wo
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
  AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd0000') AS INT)
  AND p.GravityScore > 75  -- High-gravity items
  AND wo.StorageZone NOT LIKE 'A%'  -- Not in premium zone
GROUP BY p.SKU, p.ProductName, p.GravityScore, wo.StorageZone
HAVING COUNT(*) > 50
ORDER BY p.GravityScore DESC, PickFrequency DESC;
```

### Pattern 2: Fleet Maintenance Priority Queue

```sql
SELECT 
    f.VehicleID,
    f.VehicleType,
    f.MaintenanceStatus,
    -- Revenue impact: sum of load values for trips in next 7 days
    SUM(
        ft.LoadWeightKg * 
        CASE p.Category
            WHEN 'High-Value Electronics' THEN 50
            WHEN 'Perishables' THEN 30
            WHEN 'Bulk Goods' THEN 5
            ELSE 10
        END
    ) AS RevenueImpact,
    -- Urgency: engine metrics from telematics
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS FuelEfficiencyDegradation,
    COUNT(*) AS UpcomingTrips
FROM DimFleet f
INNER JOIN FactFleetTrips ft ON f.FleetKey = ft.FleetKey
INNER JOIN DimProduct p ON ft.ProductKey = p.ProductKey
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.DateTimeValue BETWEEN GETDATE() AND DATEADD(DAY, 7, GETDATE())
  AND f.MaintenanceStatus IN ('Scheduled', 'Critical')
GROUP BY f.VehicleID, f.VehicleType, f.MaintenanceStatus
ORDER BY 
    CASE f.MaintenanceStatus WHEN 'Critical' THEN 1 ELSE 2 END,
    RevenueImpact DESC;
```

### Pattern 3: Weather-Correlated Delay Analysis

```sql
SELECT 
    g.Region,
    ft.WeatherCondition,
    COUNT(*) AS TripCount,
    AVG(ft.TrafficDelayMinutes) AS AvgDelayMinutes,
    AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(wo.QuantityUnits) AS AffectedUnits
FROM FactFleetTrips ft
INNER JOIN DimGeography g ON ft.DestinationKey = g.GeographyKey
INNER JOIN FactWarehouseOperations wo ON ft.ProductKey = wo.ProductKey
    AND CAST(ft.TimeKey / 10000 AS INT) = CAST(wo.TimeKey / 10000 AS INT)
WHERE ft.WeatherCondition IN ('Rain', 'Snow', 'Storm')
  AND ft.TrafficDelayMinutes > 15
  AND ft.TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd0000') AS INT)
GROUP BY g.Region, ft.WeatherCondition
ORDER BY AvgDelayMinutes DESC;
```

### Pattern 4: Cross-Dock Efficiency Analysis

```sql
SELECT 
    g.LocationName AS CrossDockLocation,
    p.Category,
    COUNT(*) AS TotalTransfers,
    AVG(cd.TransferTimeMinutes) AS AvgTransferTime,
    SUM(CASE WHEN cd.BypassStorage = 1 THEN 1 ELSE 0 END) AS DirectTransfers,
    SUM(cd.QuantityUnits) AS TotalUnits,
    -- Time saved vs traditional storage
    SUM(cd.TransferTimeMinutes) AS ActualMinutes,
    COUNT(*) * 180 AS EstimatedStorageMinutes,  -- 180 min avg for traditional storage
    (COUNT(*) * 180 - SUM(cd.TransferTimeMinutes)) AS MinutesSaved
FROM FactCrossDock cd
INNER JOIN DimGeography g ON cd.GeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON cd.ProductKey = p.ProductKey
INNER JOIN DimTime t ON cd.InboundTimeKey = t.TimeKey
WHERE t.DateTimeValue >= DATEADD(DAY, -30, GETDATE())
GROUP BY g.LocationName, p.Category
HAVING AVG(cd.TransferTimeMinutes) < 60  -- Successful cross-docks
ORDER BY MinutesSaved DESC;
```

## Common Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom:** Scheduled refresh fails with timeout error

**Solution:**
```sql
-- Optimize with indexed views for expensive aggregations
CREATE VIEW vw_DailyWarehouseMetrics
WITH SCHEMABINDING
AS
SELECT 
    CAST(wo.TimeKey / 10000 AS INT) AS DateKey,
    wo.GeographyKey,
    wo.ProductKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(wo.QuantityUnits) AS TotalUnits,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime
FROM dbo.FactWarehouseOperations wo
GROUP BY CAST(wo.TimeKey / 10000 AS INT), wo.GeographyKey, wo.ProductKey;
GO

CREATE UNIQUE CLUSTERED INDEX IX_DailyMetrics 
ON vw_DailyWarehouseMetrics(DateKey, GeographyKey, ProductKey);
```

### Issue: Incorrect Gravity Scores

**Symptom:** High-volume items showing low gravity scores

**Solution:** Verify velocity calculation window and add diagnostic query:

```sql
-- Diagnostic: Check pick frequency for products
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    COUNT(*) AS PicksLast30Days,
    AVG(wo.ProcessingTimeMinutes) AS AvgPickTime
FROM DimProduct p
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    AND wo.OperationType = 'Picking'
    AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd0000') AS INT)
GROUP BY p.SKU, p.ProductName, p.GravityScore
ORDER BY PicksLast30Days DESC;

-- Recalculate if needed
EXEC sp_UpdateGravityScores;
```

### Issue: Cross-Fact KPIs Not Matching

**Symptom:** Warehouse and fleet metrics don't align in cross-fact view

**Solution:** Check time grain alignment and relationship cardinality:

```sql
-- Verify time key alignment between facts
SELECT 
    'Warehouse' AS Source,
    MIN(TimeKey) AS MinTimeKey,
    MAX(TimeKey) AS MaxTimeKey,
    COUNT(DISTINCT TimeKey) AS UniqueTimeKeys
FROM FactWarehouseOperations
UNION ALL
SELECT 
    'Fleet',
    MIN(TimeKey),
    MAX(TimeKey),
    COUNT(DISTINCT TimeKey)
FROM FactFleetTrips;

-- Check for orphaned records
SELECT COUNT(*) AS OrphanedWarehouseOps
FROM FactWarehouseOperations wo
WHERE NOT EXISTS (
    SELECT 1 FROM DimTime t WHERE t.TimeKey = wo.TimeKey
);
```

### Issue: Role-Level Security Not Filtering

**Symptom:** Users see data outside their region

**Solution:** Verify RLS implementation and test:

```sql
-- Create test user mapping table
CREATE TABLE UserRegionMapping (
    UserEmail VARCHAR(200) PRIMARY KEY,
    Region VARCHAR(100)
);

INSERT INTO UserRegionMapping VALUES 
('user@company.com', 'North America'),
('manager@company.com', 'Europe');

-- Test RLS logic
DECLARE @TestUser VARCHAR(200) = 'user@company.com';

SELECT DISTINCT g.Region
FROM DimGeography g
WHERE g.Region = (
    SELECT Region 
    FROM UserRegionMapping 
    WHERE UserEmail = @TestUser
);
```

## Advanced Configuration

### Automated Alerting Setup

```sql
-- Create alert configuration table
CREATE TABLE AlertRules (
    AlertID INT PRIMARY KEY IDENTITY,
    AlertName VARCHAR(200),
    MetricType VARCHAR(50),
    Threshold DECIMAL(10,2),
    ComparisonOperator VARCHAR(10),
    NotificationEmail VARCHAR(200),
    IsActive BIT
);

-- Example alert rules
INSERT INTO AlertRules VALUES
('High Dwell Time Alert', 'AvgDwellTime', 180, '>', 'logistics@company.com', 1),
('Fleet Idle Threshold', 'FleetIdlePercent', 15, '>', 'fleet@company.com', 1),
('Low Fuel Efficiency', 'FuelEfficiency', 6.0, '<', 'operations@company.com', 1);

-- Stored procedure to check alerts and send notifications
CREATE PROCEDURE sp_CheckAlerts
AS
BEGIN
    DECLARE @AlertName VARCHAR(200), @Threshold DECIMAL(10,2), @Email VARCHAR(200);
    DECLARE @CurrentValue DECIMAL(10,2);
    
    -- Check dwell time alert
    SELECT @CurrentValue = AVG(DwellTimeMinutes)
    FROM FactWarehouseOperations
    WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -1, GETDATE()), 'yyyyMMddHHmm') AS INT);
    
    IF @CurrentValue > (SELECT Threshold FROM AlertRules WHERE MetricType = 'AvgDwellTime' AND IsActive = 1)
    BEGIN
        -- Log alert (integrate with email service)
        INSERT INTO AlertLog (AlertTime, MetricType, Value, EmailSent)
        VALUES (GETDATE(), 'AvgDwellTime', @CurrentValue, 1);
    END
    
    -- Additional alert checks...
END;
GO

-- Schedule via SQL Server Agent (every 15 minutes)
```

### External API Integration Pattern

```sql
-- Stored procedure to fetch and store weather data
CREATE PROCEDURE sp_UpdateWeatherForActiveRoutes
AS
BEGIN
    -- Assumes external tool (Python, Logic App) populates staging table
    MERGE FactFleetTrips AS target
    USING Staging_WeatherUpdates AS source
    ON target.TripKey = source.TripKey
    WHEN MATCHED THEN
        UPDATE SET 
            target.WeatherCondition = source.WeatherCondition,
            target.TrafficDelayMinutes = 
                target.TrafficDelayMinutes + 
                CASE source.WeatherCondition
                    WHEN 'Storm' THEN 20
                    WHEN 'Snow' THEN 15
                    WHEN 'Rain' THEN 5
                    ELSE 0
                END;
END;
GO
```

## Best Practices
