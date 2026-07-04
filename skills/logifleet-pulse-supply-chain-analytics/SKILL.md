---
name: logifleet-pulse-supply-chain-analytics
description: Power BI and MS SQL Server logistics intelligence engine for warehouse operations, fleet management, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "create Power BI dashboard for fleet and warehouse KPIs"
  - "configure cross-fact supply chain analytics"
  - "implement warehouse gravity zone optimization"
  - "build multi-modal logistics intelligence reporting"
  - "set up real-time fleet tracking with Power BI"
  - "create predictive bottleneck detection for logistics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Warehouse Gravity Zones** for optimal storage placement based on velocity and value
- **Adaptive Fleet Triage Engine** for prioritized maintenance based on revenue impact
- **Cross-fact KPI harmonization** (e.g., inventory dwell time vs. fleet idle costs)
- **Temporal elasticity modeling** for scenario simulation
- **Real-time dashboards** with 15-minute refresh intervals

## Installation

### Prerequisites

- MS SQL Server 2019+ (Developer, Standard, or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources (WMS, TMS, telematics APIs)

### Step 1: Deploy SQL Schema

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    FifteenMinuteBucket SMALLINT,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    FiscalPeriod VARCHAR(10),
    INDEX IX_DimTime_DateTime NONCLUSTERED (FullDateTime)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Customer Site'
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    INDEX IX_DimGeography_Type NONCLUSTERED (LocationType)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Higher = needs closer to dock
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex TINYINT, -- 1-10 scale
    INDEX IX_Product_Gravity NONCLUSTERED (GravityScore DESC)
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    LeadTimeVariance DECIMAL(5,2), -- Standard deviation in days
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    INDEX IX_Supplier_Reliability NONCLUSTERED (ComplianceScore DESC)
);

-- Deploy fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL, -- Warehouse location
    ProductKey INT NOT NULL,
    SupplierKey INT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    UnitsHandled INT,
    PickRateUnitsPerHour DECIMAL(8,2),
    StorageZone VARCHAR(50),
    ActualGravityScore DECIMAL(5,2), -- Where it was actually stored
    RecommendedGravityScore DECIMAL(5,2), -- Where it should have been
    OperatorID VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey),
    INDEX IX_Fact_Warehouse_Time NONCLUSTERED (TimeKey),
    INDEX IX_Fact_Warehouse_Product NONCLUSTERED (ProductKey)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL, -- Departure time
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TotalDistanceKM DECIMAL(10,2),
    AverageSpeedKMH DECIMAL(6,2),
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT,
    MaintenanceAlertFlag BIT DEFAULT 0,
    CargoValueUSD DECIMAL(12,2), -- For triage prioritization
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_Fact_Fleet_Time NONCLUSTERED (TimeKey),
    INDEX IX_Fact_Fleet_Vehicle NONCLUSTERED (VehicleID)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL, -- Cross-dock facility
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT NULL,
    OutboundTripKey BIGINT NULL,
    TransferTimeMinutes INT,
    UnitsTransferred INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey),
    INDEX IX_Fact_CrossDock_Time NONCLUSTERED (TimeKey)
);
```

### Step 2: Create Views for Cross-Fact KPIs

```sql
-- View: Unified logistics performance
CREATE VIEW vw_UnifiedLogisticsKPIs AS
SELECT 
    t.FullDateTime,
    t.FiscalPeriod,
    g.LocationName AS Warehouse,
    g.Region,
    p.SKU,
    p.ProductName,
    p.GravityScore AS OptimalGravityScore,
    -- Warehouse metrics
    AVG(w.DwellTimeMinutes) AS AvgDwellTimeMin,
    SUM(w.UnitsHandled) AS TotalUnitsHandled,
    AVG(w.PickRateUnitsPerHour) AS AvgPickRate,
    AVG(ABS(w.ActualGravityScore - w.RecommendedGravityScore)) AS GravityMismatch,
    -- Fleet metrics for products from this warehouse
    COUNT(DISTINCT f.TripKey) AS RelatedTrips,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleMin,
    AVG(f.FuelConsumedLiters) AS AvgFuelPerTrip,
    SUM(f.CargoValueUSD) AS TotalCargoValue
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips f ON f.OriginGeographyKey = g.GeographyKey 
    AND f.TimeKey >= w.TimeKey 
    AND f.TimeKey <= DATEADD(hour, 24, w.TimeKey)
GROUP BY t.FullDateTime, t.FiscalPeriod, g.LocationName, g.Region, 
    p.SKU, p.ProductName, p.GravityScore;
GO
```

### Step 3: Create Stored Procedures for Data Loading

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external WMS table (adjust to your source)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, DwellTimeMinutes, UnitsHandled,
        PickRateUnitsPerHour, StorageZone, 
        ActualGravityScore, RecommendedGravityScore, OperatorID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        wms.OperationType,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) AS DwellTimeMinutes,
        wms.UnitsHandled,
        CASE WHEN DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) > 0
            THEN (wms.UnitsHandled * 60.0) / DATEDIFF(MINUTE, wms.StartTime, wms.EndTime)
            ELSE 0 END AS PickRateUnitsPerHour,
        wms.StorageZone,
        wms.ActualZoneGravity,
        p.GravityScore AS RecommendedGravityScore,
        wms.OperatorID
    FROM EXTERNAL_WMS_TABLE wms -- Replace with your source
    INNER JOIN DimTime t ON CAST(wms.StartTime AS DATE) = CAST(t.FullDateTime AS DATE)
    INNER JOIN DimGeography g ON wms.WarehouseCode = g.LocationName
    INNER JOIN DimProductGravity p ON wms.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON wms.SupplierCode = s.SupplierCode
    WHERE wms.StartTime >= @StartDateTime 
      AND wms.StartTime < @EndDateTime
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations f
          WHERE f.TimeKey = t.TimeKey 
            AND f.ProductKey = p.ProductKey
            AND f.OperatorID = wms.OperatorID
      );
END;
GO
```

### Step 4: Configure Alert System

```sql
-- Automated alerting for critical KPIs
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    MetricName VARCHAR(100) NOT NULL,
    ThresholdValue DECIMAL(12,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    AlertMessage NVARCHAR(500),
    EmailRecipients VARCHAR(1000), -- Comma-separated
    IsActive BIT DEFAULT 1
);

-- Insert example thresholds
INSERT INTO AlertThresholds (MetricName, ThresholdValue, ComparisonOperator, AlertMessage, EmailRecipients)
VALUES 
    ('FleetIdlePercent', 15.0, '>', 'Fleet idle time exceeded 15% of trip duration', 'fleet.manager@example.com'),
    ('DwellTimeHours', 72.0, '>', 'Products in storage for more than 72 hours detected', 'warehouse.ops@example.com'),
    ('GravityMismatch', 2.0, '>', 'Significant gravity zone misalignment detected', 'logistics.director@example.com');

CREATE PROCEDURE sp_CheckAlertsAndNotify
AS
BEGIN
    -- Example: Check fleet idle time
    DECLARE @FleetIdlePercent DECIMAL(5,2);
    
    SELECT @FleetIdlePercent = AVG(IdleTimeMinutes * 100.0 / NULLIF(IdleTimeMinutes + LoadingTimeMinutes + UnloadingTimeMinutes, 0))
    FROM FactFleetTrips
    WHERE TimeKey IN (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(hour, -24, GETDATE()));
    
    IF @FleetIdlePercent > (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'FleetIdlePercent' AND IsActive = 1)
    BEGIN
        -- Send email using Database Mail (configure separately)
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = (SELECT EmailRecipients FROM AlertThresholds WHERE MetricName = 'FleetIdlePercent'),
            @subject = 'LogiFleet Pulse Alert: Fleet Idle Time Exceeded',
            @body = 'Fleet idle time is currently at ' + CAST(@FleetIdlePercent AS VARCHAR(10)) + '% which exceeds the 15% threshold.';
    END
END;
GO

-- Schedule this to run every 15 minutes using SQL Server Agent
```

## Power BI Configuration

### Import the Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter your SQL Server connection details:
   - **Server**: `your-sql-server.database.windows.net` or `localhost`
   - **Database**: `LogiFleetPulse`
   - **Authentication**: Windows or SQL Server credentials from environment variables

### Key DAX Measures

```dax
// Total Dwell Time Cost (warehouse + fleet idle correlation)
TotalDwellCost = 
VAR WarehouseDwellCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] * 0.05 // $0.05 per minute storage cost
    )
VAR FleetIdleCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[IdleTimeMinutes] * 0.15 // $0.15 per minute fuel/driver cost
    )
RETURN WarehouseDwellCost + FleetIdleCost

// Gravity Zone Efficiency Score
GravityEfficiencyScore = 
AVERAGEX(
    FactWarehouseOperations,
    1 - ABS(
        FactWarehouseOperations[ActualGravityScore] - 
        FactWarehouseOperations[RecommendedGravityScore]
    ) / FactWarehouseOperations[RecommendedGravityScore]
) * 100

// Fleet Maintenance Priority Score
FleetMaintenancePriority = 
SUMX(
    FILTER(
        FactFleetTrips,
        FactFleetTrips[MaintenanceAlertFlag] = TRUE()
    ),
    FactFleetTrips[CargoValueUSD] * 
    (FactFleetTrips[IdleTimeMinutes] / 60) // Weight by potential revenue loss per hour
)

// Predictive Bottleneck Index (simplified moving average comparison)
BottleneckIndex = 
VAR CurrentAvgDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -7, DAY)
    )
VAR HistoricalAvgDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -90, DAY)
    )
RETURN 
IF(
    ISBLANK(HistoricalAvgDwell), 
    0, 
    (CurrentAvgDwell - HistoricalAvgDwell) / HistoricalAvgDwell * 100
)
```

### Row-Level Security Setup

```dax
// Create role: WarehouseManager
[LocationName] IN {
    USERPRINCIPALNAME() // Map user to specific warehouses via lookup table
}

// Create role: RegionalDirector
[Region] IN {
    LOOKUPVALUE(
        UserRegionMapping[Region],
        UserRegionMapping[UserEmail],
        USERPRINCIPALNAME()
    )
}

// Create role: Executive (no filter - sees all)
```

## Common Patterns

### Pattern 1: Gravity Zone Optimization Analysis

```sql
-- Identify SKUs stored in wrong zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore AS RecommendedGravity,
    w.StorageZone,
    AVG(w.ActualGravityScore) AS ActualAvgGravity,
    ABS(p.GravityScore - AVG(w.ActualGravityScore)) AS Mismatch,
    SUM(w.UnitsHandled) AS TotalVolume
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(day, -30, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, w.StorageZone
HAVING ABS(p.GravityScore - AVG(w.ActualGravityScore)) > 1.5
ORDER BY TotalVolume DESC;
```

### Pattern 2: Fleet Triage Priority Queue

```sql
-- Generate maintenance priority list
WITH FleetAlerts AS (
    SELECT 
        f.VehicleID,
        f.DriverID,
        t.FullDateTime AS AlertTime,
        f.CargoValueUSD,
        f.IdleTimeMinutes,
        f.FuelConsumedLiters,
        g_origin.LocationName AS Origin,
        g_dest.LocationName AS Destination,
        -- Priority scoring: cargo value * idle time ratio
        (f.CargoValueUSD * (f.IdleTimeMinutes / NULLIF(f.IdleTimeMinutes + f.LoadingTimeMinutes + f.UnloadingTimeMinutes, 0))) AS PriorityScore
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    INNER JOIN DimGeography g_origin ON f.OriginGeographyKey = g_origin.GeographyKey
    INNER JOIN DimGeography g_dest ON f.DestinationGeographyKey = g_dest.GeographyKey
    WHERE f.MaintenanceAlertFlag = 1
      AND t.FullDateTime >= DATEADD(hour, -24, GETDATE())
)
SELECT 
    VehicleID,
    Origin,
    Destination,
    CargoValueUSD,
    IdleTimeMinutes,
    PriorityScore,
    RANK() OVER (ORDER BY PriorityScore DESC) AS MaintenanceRank
FROM FleetAlerts
ORDER BY PriorityScore DESC;
```

### Pattern 3: Temporal Elasticity Simulation

```sql
-- Simulate capacity increase impact on fleet utilization
DECLARE @CapacityIncreasePct DECIMAL(5,2) = 15.0; -- 15% increase

WITH BaselineMetrics AS (
    SELECT 
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        AVG(f.FuelConsumedLiters) AS AvgFuel,
        COUNT(DISTINCT f.VehicleID) AS ActiveVehicles
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(day, -30, GETDATE())
),
WarehouseVolume AS (
    SELECT 
        SUM(w.UnitsHandled) AS TotalUnits
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(day, -30, GETDATE())
)
SELECT 
    b.AvgIdleTime AS Current_AvgIdleTime,
    b.AvgIdleTime * (1 - (@CapacityIncreasePct / 100) * 0.3) AS Projected_AvgIdleTime, -- Assume 30% correlation
    b.AvgFuel AS Current_AvgFuel,
    b.AvgFuel * (1 + (@CapacityIncreasePct / 100) * 0.5) AS Projected_AvgFuel, -- Assume 50% correlation
    v.TotalUnits AS Current_Volume,
    v.TotalUnits * (1 + @CapacityIncreasePct / 100) AS Projected_Volume
FROM BaselineMetrics b, WarehouseVolume v;
```

## Configuration

### Environment Variables

Store sensitive configuration in environment variables:

```bash
# SQL Server connection
export LOGIFLEET_SQL_SERVER="your-server.database.windows.net"
export LOGIFLEET_SQL_DATABASE="LogiFleetPulse"
export LOGIFLEET_SQL_USER="your-username"
export LOGIFLEET_SQL_PASSWORD="your-password"

# Email alerts (Database Mail)
export LOGIFLEET_SMTP_SERVER="smtp.office365.com"
export LOGIFLEET_SMTP_PORT="587"
export LOGIFLEET_SMTP_USER="alerts@yourcompany.com"
export LOGIFLEET_SMTP_PASSWORD="your-smtp-password"

# External API keys (weather, traffic)
export WEATHER_API_KEY="your-weather-api-key"
export TRAFFIC_API_KEY="your-traffic-api-key"
```

### Power BI Connection String Template

```m
// In Power BI Advanced Editor
let
    Server = Environment.GetEnvironmentVariable("LOGIFLEET_SQL_SERVER"),
    Database = Environment.GetEnvironmentVariable("LOGIFLEET_SQL_DATABASE"),
    Source = Sql.Database(Server, Database, [Query="SELECT * FROM vw_UnifiedLogisticsKPIs"])
in
    Source
```

### Refresh Schedule Configuration

```sql
-- SQL Agent Job for incremental loads (run every 15 minutes)
EXEC sp_LoadWarehouseOperations 
    @StartDateTime = DATEADD(minute, -15, GETDATE()),
    @EndDateTime = GETDATE();

-- SQL Agent Job for alerts (run every 15 minutes)
EXEC sp_CheckAlertsAndNotify;
```

## Troubleshooting

### Issue: Power BI Template Won't Connect

**Symptoms**: Connection errors when opening `.pbit` file

**Solutions**:
1. Verify SQL Server firewall rules allow your IP
2. Check credentials have `db_datareader` role minimum
3. Test connection with SQL Server Management Studio first
4. Ensure all referenced views exist in the database

```sql
-- Grant minimum permissions
USE LogiFleetPulse;
CREATE USER [PowerBIService] FOR LOGIN [YourLogin];
ALTER ROLE db_datareader ADD MEMBER [PowerBIService];
GRANT EXECUTE ON sp_LoadWarehouseOperations TO [PowerBIService];
```

### Issue: Slow Query Performance

**Symptoms**: Dashboards take >30 seconds to refresh

**Solutions**:
1. Ensure columnstore indexes on fact tables:

```sql
-- Add columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Analytics
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, DwellTimeMinutes, UnitsHandled);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_Analytics
ON FactFleetTrips (TimeKey, OriginGeographyKey, DestinationGeographyKey, IdleTimeMinutes, CargoValueUSD);
```

2. Partition large fact tables by month:

```sql
-- Create partition scheme
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, 202604); -- Extend as needed

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly ALL TO ([PRIMARY]);

-- Recreate fact table on partition scheme (requires migration)
```

3. Use Power BI aggregations for summary tables:

```sql
CREATE TABLE AggWarehouseDaily (
    DateKey INT,
    GeographyKey INT,
    ProductKey INT,
    TotalDwellTime BIGINT,
    TotalUnits BIGINT,
    AvgPickRate DECIMAL(8,2),
    PRIMARY KEY (DateKey, GeographyKey, ProductKey)
);

-- Populate via scheduled job
INSERT INTO AggWarehouseDaily
SELECT 
    CAST(CONVERT(VARCHAR(8), t.FullDateTime, 112) AS INT) AS DateKey,
    GeographyKey,
    ProductKey,
    SUM(DwellTimeMinutes) AS TotalDwellTime,
    SUM(UnitsHandled) AS TotalUnits,
    AVG(PickRateUnitsPerHour) AS AvgPickRate
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
GROUP BY CAST(CONVERT(VARCHAR(8), t.FullDateTime, 112) AS INT), GeographyKey, ProductKey;
```

### Issue: DAX Measures Return Blank

**Symptoms**: Calculated measures show no values in visuals

**Solutions**:
1. Check relationship cardinality and cross-filter direction
2. Verify filter context with `CALCULATETABLE` debugging:

```dax
// Debug measure
DebugFilterContext = 
CONCATENATEX(
    FILTERS(DimTime[FullDateTime]),
    DimTime[FullDateTime],
    ", "
)
```

3. Ensure date tables have continuous date ranges:

```sql
-- Populate DimTime with 15-minute intervals
DECLARE @StartDate DATETIME2 = '2025-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2027-12-31 23:45:00';

WHILE @StartDate <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, FullDateTime, FifteenMinuteBucket, HourOfDay, DayOfWeek, FiscalPeriod)
    VALUES (
        CONVERT(INT, FORMAT(@StartDate, 'yyyyMMddHHmm')),
        @StartDate,
        DATEPART(MINUTE, @StartDate) / 15,
        DATEPART(HOUR, @StartDate),
        DATEPART(WEEKDAY, @StartDate),
        FORMAT(@StartDate, 'yyyy-MM')
    );
    
    SET @StartDate = DATEADD(MINUTE, 15, @StartDate);
END;
```

### Issue: Gravity Zone Recommendations Not Appearing

**Symptoms**: `RecommendedGravityScore` column is NULL

**Solutions**:
1. Ensure all products have calculated gravity scores:

```sql
-- Recalculate gravity scores based on historical velocity
UPDATE DimProductGravity
SET GravityScore = (
    SELECT 
        CASE 
            WHEN AVG(w.UnitsHandled) > 1000 THEN 9.0 -- High gravity
            WHEN AVG(w.UnitsHandled) > 500 THEN 6.0  -- Medium gravity
            ELSE 3.0                                 -- Low gravity
        END
    FROM FactWarehouseOperations w
    WHERE w.ProductKey = DimProductGravity.ProductKey
      AND w.TimeKey IN (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(day, -90, GETDATE()))
)
WHERE GravityScore IS NULL;
```

2. Validate data quality in source systems
3. Check for products without transaction history

## Advanced Usage

### Integrating External Weather API

```sql
-- Create external table for weather data (requires Polybase or linked server)
CREATE EXTERNAL TABLE ExtWeatherData (
    LocationID INT,
    Timestamp DATETIME2,
    Condition VARCHAR(50),
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2)
)
WITH (
    LOCATION = 'weather_api_endpoint', -- Configure based on your API
    DATA_SOURCE = WeatherAPISource,
    FILE_FORMAT = JSONFormat
);

-- Update fleet trips with weather correlation
UPDATE f
SET f.WeatherCondition = w.Condition
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON f.OriginGeographyKey = g.GeographyKey
INNER JOIN ExtWeatherData w ON g.GeographyKey = w.LocationID 
    AND ABS(DATEDIFF(MINUTE, t.FullDateTime, w.Timestamp)) < 30;
```

### Natural Language Query Setup in Power BI

1. Go to Modeling → Q&A Setup in Power BI Desktop
2. Add synonyms:
   - "idle" → "IdleTimeMinutes"
   - "fuel consumption" → "FuelConsumedLiters"
   - "warehouse" → "LocationName where LocationType = 'Warehouse'"
3. Test queries: "show average idle time by vehicle last month"

### Export to Excel for Ad-Hoc Analysis

```sql
-- Create view optimized for Excel pivot tables
CREATE VIEW vw_ExcelExport AS
SELECT 
    CONVERT(VARCHAR(10), t.FullDateTime, 120) AS Date,
    g.LocationName AS Location,
    p.SKU,
    p.ProductName,
    p.VelocityClass,
    w.DwellTimeMinutes,
    w.UnitsHandled,
    w.StorageZone,
    f.VehicleID,
    f.IdleTimeMinutes AS FleetIdleMin,
    f.FuelConsumedLiters
FROM FactWarehouseOperations w
LEFT JOIN FactFleetTrips f ON f.OriginGeographyKey = w.GeographyKey 
    AND ABS(DATEDIFF(hour, (SELECT
