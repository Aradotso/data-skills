---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for logistics and supply chain analytics with multi-fact star schema
triggers:
  - set up logifleet pulse supply chain analytics
  - configure power bi logistics dashboard
  - deploy sql server warehouse schema for fleet management
  - implement multi-fact star schema for logistics
  - create supply chain kpi dashboard
  - integrate warehouse and fleet data analytics
  - build logistics intelligence platform
  - setup logicore analytics adaptive supply chain
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer. Built on MS SQL Server with Power BI visualization, it uses a custom multi-fact star schema to enable cross-domain analytics like correlating warehouse dwell time with fleet idle costs.

**Key Features:**
- Multi-fact star schema with time-phased dimensions
- Real-time dashboards (15-minute refresh intervals)
- Cross-fact KPI harmonization (warehouse + fleet + supplier data)
- Predictive bottleneck detection
- Warehouse Gravity Zones™ for spatial optimization
- Role-based access control and row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- Database permissions: CREATE TABLE, VIEW, PROCEDURE
- Optional: Azure Synapse Analytics for external data enrichment

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
USE master;
GO

-- Create the logistics database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName NVARCHAR(10) NOT NULL,
    HourOfDay INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    IsWeekend BIT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalPeriod INT NOT NULL
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    ContinentCode NVARCHAR(2),
    ContinentName NVARCHAR(50),
    CountryCode NVARCHAR(3),
    CountryName NVARCHAR(100),
    Region NVARCHAR(100),
    City NVARCHAR(100),
    PostalCode NVARCHAR(20),
    WarehouseID NVARCHAR(50),
    WarehouseName NVARCHAR(200),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitValue DECIMAL(10,2),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility_factor
    OptimalZone NVARCHAR(50), -- High-gravity, Medium-gravity, Low-gravity
    LastUpdated DATETIME DEFAULT GETDATE()
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200),
    AvgLeadTimeDays DECIMAL(5,2),
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    OnTimeDeliveryRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    LastEvaluated DATETIME DEFAULT GETDATE()
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    QuantityHandled INT,
    CycleTimeMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(10,2),
    StorageZone NVARCHAR(50),
    ErrorOccurred BIT,
    ErrorType NVARCHAR(100),
    LoadTimestamp DATETIME DEFAULT GETDATE()
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,3),
    LoadWeightKG DECIMAL(10,2),
    DeliveryOnTime BIT,
    DelayMinutes DECIMAL(10,2),
    DelayReason NVARCHAR(200),
    LoadTimestamp DATETIME DEFAULT GETDATE()
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    QuantityTransferred INT,
    DwellMinutes DECIMAL(8,2),
    LoadTimestamp DATETIME DEFAULT GETDATE()
);

-- Create indexes for performance
CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey);
CREATE INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey);
CREATE INDEX IX_FactFleet_Destination ON FactFleetTrips(DestinationGeographyKey);
```

### Step 2: Create Stored Procedures for Data Loading

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartDateTime DATETIME,
    @EndDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external WMS table or staging area
    -- Replace with your actual source connection
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, QuantityHandled, CycleTimeMinutes,
        DwellTimeHours, StorageZone, ErrorOccurred, ErrorType
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        src.OperationType,
        src.QuantityHandled,
        src.CycleTimeMinutes,
        src.DwellTimeHours,
        src.StorageZone,
        src.ErrorOccurred,
        src.ErrorType
    FROM StagingWarehouseOps src
    INNER JOIN DimTime t ON CAST(src.OperationDateTime AS DATE) = t.Date 
        AND DATEPART(HOUR, src.OperationDateTime) = t.HourOfDay
        AND (DATEPART(MINUTE, src.OperationDateTime) / 15) * 15 = t.MinuteBucket
    INNER JOIN DimGeography g ON src.WarehouseID = g.WarehouseID
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON src.SupplierID = s.SupplierID
    WHERE src.OperationDateTime BETWEEN @StartDateTime AND @EndDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.TimeKey = t.TimeKey 
                AND f.ProductKey = p.ProductKey
                AND f.GeographyKey = g.GeographyKey
        );
END;
GO

-- Stored procedure for calculating gravity scores
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(w.CycleTimeMinutes) AS AvgCycleTime,
            SUM(w.QuantityHandled) AS TotalVolume
        FROM DimProductGravity p
        LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
        WHERE w.TimeKey >= (SELECT MAX(TimeKey) - 43200 FROM DimTime) -- Last 30 days
        GROUP BY p.ProductKey
    )
    UPDATE p
    SET 
        GravityScore = (
            (ISNULL(m.PickFrequency, 0) * 0.4) + 
            (p.UnitValue * 0.3) + 
            (CASE WHEN p.IsFragile = 1 THEN 50 ELSE 0 END * 0.2) +
            (ISNULL(m.TotalVolume, 0) / 1000.0 * 0.1)
        ),
        OptimalZone = CASE 
            WHEN (ISNULL(m.PickFrequency, 0) * 0.4 + p.UnitValue * 0.3) > 75 THEN 'High-Gravity'
            WHEN (ISNULL(m.PickFrequency, 0) * 0.4 + p.UnitValue * 0.3) > 40 THEN 'Medium-Gravity'
            ELSE 'Low-Gravity'
        END,
        LastUpdated = GETDATE()
    FROM DimProductGravity p
    LEFT JOIN ProductMetrics m ON p.ProductKey = m.ProductKey;
END;
GO
```

### Step 3: Create Alerting Stored Procedure

```sql
CREATE PROCEDURE sp_CheckFleetAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Alert: Fleet idle time exceeds 15% of trip duration
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
            AND (IdleTimeMinutes / NULLIF(DurationMinutes, 0)) > 0.15
    )
    BEGIN
        -- Insert into alert table or send notification
        INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
        VALUES (
            'Fleet Idle Excess',
            'High',
            'Multiple trips exceeded 15% idle time threshold in last 24 hours',
            GETDATE()
        );
    END;
    
    -- Alert: Warehouse dwell time > 72 hours for high-gravity products
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations w
        INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
        WHERE w.DwellTimeHours > 72
            AND p.OptimalZone = 'High-Gravity'
            AND w.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime)
    )
    BEGIN
        INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
        VALUES (
            'Excessive Dwell Time',
            'Critical',
            'High-gravity products have dwell time > 72 hours',
            GETDATE()
        );
    END;
END;
GO

-- Schedule this procedure with SQL Server Agent
```

### Step 4: Configure Power BI Connection

```powerquery
// Power Query M code for connecting to SQL Server
let
    Source = Sql.Database(
        #"SQL_SERVER_HOST", 
        "LogiFleetPulse",
        [
            Query = null,
            CommandTimeout = #duration(0, 0, 10, 0),
            ConnectionTimeout = #duration(0, 0, 5, 0)
        ]
    ),
    FactWarehouseOps = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FactFleet = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
    DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
    DimProduct = Source{[Schema="dbo",Item="DimProductGravity"]}[Data],
    DimGeography = Source{[Schema="dbo",Item="DimGeography"]}[Data]
in
    Source
```

## Key Configuration

### Environment Variables

Set these environment variables for secure configuration:

```bash
# SQL Server Connection
export LOGIFLEET_SQL_SERVER="your-server.database.windows.net"
export LOGIFLEET_SQL_DATABASE="LogiFleetPulse"
export LOGIFLEET_SQL_USER="your_username"
export LOGIFLEET_SQL_PASSWORD="your_password"

# External API Keys (optional)
export LOGIFLEET_WEATHER_API_KEY="your_weather_api_key"
export LOGIFLEET_TRAFFIC_API_KEY="your_traffic_api_key"

# Power BI Service (for publishing)
export POWERBI_WORKSPACE_ID="your_workspace_id"
```

### Connection String Template

```json
{
  "connectionStrings": {
    "LogiFleetPulse": "Server=${LOGIFLEET_SQL_SERVER};Database=${LOGIFLEET_SQL_DATABASE};User Id=${LOGIFLEET_SQL_USER};Password=${LOGIFLEET_SQL_PASSWORD};Encrypt=True;TrustServerCertificate=False;"
  },
  "refreshSchedule": {
    "intervalMinutes": 15,
    "peakHoursOnly": false
  },
  "alerting": {
    "enabled": true,
    "checkIntervalMinutes": 30,
    "emailNotifications": true
  }
}
```

## Common Usage Patterns

### Cross-Fact KPI Analysis

```sql
-- Query: Correlate warehouse dwell time with fleet delay minutes
SELECT 
    t.Date,
    t.DayName,
    g.Region,
    p.Category,
    AVG(w.DwellTimeHours) AS AvgWarehouseDwell,
    AVG(f.DelayMinutes) AS AvgFleetDelay,
    COUNT(DISTINCT f.VehicleID) AS ActiveVehicles,
    SUM(w.QuantityHandled) AS TotalVolume
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips f 
    ON f.OriginGeographyKey = g.GeographyKey
    AND f.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY t.Date, t.DayName, g.Region, p.Category
HAVING AVG(w.DwellTimeHours) > 48
ORDER BY AvgWarehouseDwell DESC;
```

### Gravity Zone Optimization Query

```sql
-- Identify products in wrong gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone,
    w.StorageZone AS CurrentZone,
    COUNT(*) AS PickEvents,
    AVG(w.CycleTimeMinutes) AS AvgCycleTime,
    SUM(w.QuantityHandled) AS TotalVolume
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
WHERE w.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days
    AND w.StorageZone != p.OptimalZone
    AND p.GravityScore > 50 -- Focus on high-gravity items
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZone, w.StorageZone
HAVING COUNT(*) > 10
ORDER BY p.GravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify congestion patterns by hour of day
WITH HourlyMetrics AS (
    SELECT 
        t.HourOfDay,
        t.DayName,
        g.WarehouseName,
        COUNT(*) AS Operations,
        AVG(w.CycleTimeMinutes) AS AvgCycleTime,
        STDEV(w.CycleTimeMinutes) AS StdDevCycleTime
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    WHERE t.TimeKey >= (SELECT MAX(TimeKey) - 2016 FROM DimTime) -- Last 3 weeks
    GROUP BY t.HourOfDay, t.DayName, g.WarehouseName
)
SELECT 
    HourOfDay,
    DayName,
    WarehouseName,
    Operations,
    AvgCycleTime,
    StdDevCycleTime,
    CASE 
        WHEN AvgCycleTime > (SELECT AVG(AvgCycleTime) * 1.5 FROM HourlyMetrics) 
            THEN 'High Bottleneck Risk'
        WHEN StdDevCycleTime > (SELECT AVG(StdDevCycleTime) * 1.5 FROM HourlyMetrics)
            THEN 'High Variability Risk'
        ELSE 'Normal'
    END AS BottleneckRisk
FROM HourlyMetrics
WHERE AvgCycleTime > (SELECT AVG(AvgCycleTime) FROM HourlyMetrics)
ORDER BY AvgCycleTime DESC;
```

### Fleet Performance by Route

```sql
-- Analyze fleet efficiency by route segment
SELECT 
    origin.City + ' → ' + dest.City AS RouteSegment,
    COUNT(DISTINCT f.VehicleID) AS UniqueVehicles,
    COUNT(*) AS TotalTrips,
    AVG(f.DistanceKM) AS AvgDistanceKM,
    AVG(f.DurationMinutes) AS AvgDurationMin,
    AVG(f.IdleTimeMinutes) AS AvgIdleMin,
    AVG(f.FuelConsumedLiters) AS AvgFuelLiters,
    AVG(f.FuelConsumedLiters / NULLIF(f.DistanceKM, 0)) AS FuelEfficiencyLPer100KM,
    SUM(CASE WHEN f.DeliveryOnTime = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS OnTimeRate
FROM FactFleetTrips f
INNER JOIN DimGeography origin ON f.OriginGeographyKey = origin.GeographyKey
INNER JOIN DimGeography dest ON f.DestinationGeographyKey = dest.GeographyKey
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY origin.City, dest.City
HAVING COUNT(*) >= 5
ORDER BY FuelEfficiencyLPer100KM DESC;
```

## Power BI DAX Measures

```dax
// Measure: Warehouse Efficiency Score
Warehouse Efficiency = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR ErrorRate = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactWarehouseOperations), FactWarehouseOperations[ErrorOccurred] = TRUE()),
        TotalOps,
        0
    )
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeMinutes])
VAR BenchmarkCycleTime = 15 // minutes
RETURN
    (1 - ErrorRate) * 0.6 + 
    (1 - DIVIDE(AvgCycleTime, BenchmarkCycleTime, 1)) * 0.4

// Measure: Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Measure: Gravity Zone Compliance
Gravity Zone Compliance % = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR CompliantOps = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[StorageZone] = RELATED(DimProductGravity[OptimalZone])
    )
RETURN
    DIVIDE(CompliantOps, TotalOps, 0) * 100

// Measure: Cross-Fact Delay Impact
Delay Impact Score = 
VAR WarehouseDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR FleetDelay = AVERAGE(FactFleetTrips[DelayMinutes])
VAR DwellWeight = 0.6
VAR DelayWeight = 0.4
RETURN
    (WarehouseDwell / 24 * DwellWeight) + (FleetDelay / 60 * DelayWeight)
```

## Troubleshooting

### Issue: Power BI Connection Timeout

**Solution:**
```powerquery
// Increase timeout in Power Query
let
    Source = Sql.Database(
        #"SQL_SERVER_HOST", 
        "LogiFleetPulse",
        [CommandTimeout = #duration(0, 0, 30, 0)] // 30 minutes
    )
in
    Source
```

### Issue: Slow Cross-Fact Queries

**Solution:**
```sql
-- Add covering indexes
CREATE INDEX IX_FactWarehouse_Covering 
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey)
INCLUDE (DwellTimeHours, CycleTimeMinutes);

CREATE INDEX IX_FactFleet_Covering
ON FactFleetTrips (TimeKey, OriginGeographyKey, DestinationGeographyKey)
INCLUDE (DelayMinutes, IdleTimeMinutes, FuelConsumedLiters);

-- Use columnstore for large fact tables (> 10M rows)
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, DwellTimeHours);
```

### Issue: Dimension Key Lookup Failures

**Solution:**
```sql
-- Create staging validation procedure
CREATE PROCEDURE sp_ValidateStagingData
AS
BEGIN
    -- Check for missing time keys
    SELECT DISTINCT OperationDateTime
    FROM StagingWarehouseOps
    WHERE NOT EXISTS (
        SELECT 1 FROM DimTime 
        WHERE Date = CAST(OperationDateTime AS DATE)
    );
    
    -- Check for missing geography keys
    SELECT DISTINCT WarehouseID
    FROM StagingWarehouseOps
    WHERE NOT EXISTS (
        SELECT 1 FROM DimGeography 
        WHERE WarehouseID = StagingWarehouseOps.WarehouseID
    );
END;
GO
```

### Issue: Power BI Row-Level Security Not Working

**Solution:**
```dax
// Create RLS role in Power BI (Modeling > Manage Roles)
[Region Access] = 
    DimGeography[Region] = USERPRINCIPALNAME() 
    || USERNAME() = "admin@company.com"

// Or use a security table
[Dynamic Security] = 
VAR UserEmail = USERPRINCIPALNAME()
VAR UserRegions = 
    CALCULATETABLE(
        VALUES(SecurityTable[Region]),
        SecurityTable[UserEmail] = UserEmail
    )
RETURN
    DimGeography[Region] IN UserRegions
```

## Advanced Patterns

### Real-Time Data Refresh with Change Tracking

```sql
-- Enable change tracking on fact tables
ALTER DATABASE LogiFleetPulse
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 7 DAYS, AUTO_CLEANUP = ON);

ALTER TABLE FactWarehouseOperations
ENABLE CHANGE_TRACKING
WITH (TRACK_COLUMNS_UPDATED = OFF);

-- Query for incremental refresh
SELECT 
    f.*,
    ct.SYS_CHANGE_OPERATION,
    ct.SYS_CHANGE_VERSION
FROM FactWarehouseOperations f
RIGHT OUTER JOIN CHANGETABLE(
    CHANGES FactWarehouseOperations, @last_sync_version
) AS ct ON f.OperationKey = ct.OperationKey;
```

### External Data Integration (Weather Impact)

```sql
-- Create external table for weather data
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST,
    LOCATION = 'https://api.weatherservice.com'
);

-- Correlate delays with weather conditions
SELECT 
    t.Date,
    g.Region,
    AVG(f.DelayMinutes) AS AvgDelay,
    w.Condition,
    w.PrecipitationMM
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON f.DestinationGeographyKey = g.GeographyKey
LEFT JOIN ExternalWeatherData w 
    ON w.Date = t.Date 
    AND w.Region = g.Region
WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())
GROUP BY t.Date, g.Region, w.Condition, w.PrecipitationMM
HAVING AVG(f.DelayMinutes) > 30;
```

## Best Practices

1. **Partition large fact tables** by date for query performance
2. **Schedule gravity score updates** daily during off-peak hours
3. **Implement incremental refresh** in Power BI for datasets > 1GB
4. **Use DirectQuery** for real-time dashboards, Import mode for historical analysis
5. **Archive fact data** older than 2 years to separate tables
6. **Monitor query performance** with SQL Server Query Store
7. **Validate dimension integrity** before loading new facts
8. **Document custom DAX measures** with inline comments
9. **Test RLS policies** thoroughly before production deployment
10. **Backup Power BI reports** to version control (export as .pbix)
