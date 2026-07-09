---
name: logifleet-pulse-supply-chain-analytics
description: Power BI and MS SQL Server template for multi-modal logistics intelligence with cross-fact KPI harmonization and predictive fleet optimization
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure multi-fact star schema for fleet and warehouse
  - implement warehouse gravity zones and fleet triage
  - connect logifleet pulse to telemetry data
  - create cross-modal supply chain dashboards
  - troubleshoot logifleet pulse sql schema
  - integrate logicore analytics adaptive supply chain
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced MS SQL Server and Power BI template for real-time logistics intelligence. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and inventory data
- **Cross-fact KPI harmonization** for unified supply chain metrics
- **Warehouse Gravity Zones™** for optimal storage location mapping
- **Adaptive Fleet Triage Engine** for proactive maintenance prioritization
- **Temporal elasticity modeling** for scenario simulation
- **Power BI dashboards** with 15-minute refresh cycles

The core value: unify isolated logistics data streams (WMS, telematics, supplier portals, weather APIs) into a single semantic layer for predictive decision-making.

## Installation and Setup

### Prerequisites

- MS SQL Server 2019+ (Enterprise, Standard, or Developer Edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, telematics API, ERP

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy dimension tables first
-- DimTime: 15-minute granularity time buckets
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek NVARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalQuarter INT NOT NULL,
    FiscalYear INT NOT NULL,
    INDEX IX_DimTime_DateTime NONCLUSTERED (FullDateTime)
);

-- DimGeography: Hierarchical location dimensions
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(255) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- 'Warehouse', 'RouteNode', 'CrossDock'
    AddressLine1 NVARCHAR(255),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    WarehouseCapacityCubicFeet DECIMAL(18,2),
    INDEX IX_DimGeography_Type NONCLUSTERED (LocationType)
);

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(255) NOT NULL,
    ProductCategory NVARCHAR(100),
    ProductSubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,4),
    Fragility NVARCHAR(20), -- 'Low', 'Medium', 'High'
    TemperatureRequirement NVARCHAR(50), -- 'Ambient', 'Refrigerated', 'Frozen'
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    LastGravityUpdate DATETIME2,
    INDEX IX_Product_Gravity NONCLUSTERED (GravityScore DESC)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(255) NOT NULL,
    Country NVARCHAR(100),
    LeadTimeDaysAvg DECIMAL(6,2),
    LeadTimeVariance DECIMAL(6,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    LastAssessmentDate DATE,
    INDEX IX_Supplier_Reliability NONCLUSTERED (ComplianceScore DESC)
);

-- FactWarehouseOperations: Core warehouse activity fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType NVARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2 NOT NULL,
    OperationEndTime DATETIME2,
    DurationMinutes DECIMAL(10,2),
    QuantityUnits INT,
    DwellTimeHours DECIMAL(10,2),
    ZoneAssignment NVARCHAR(50), -- Gravity zone assignment
    OperatorID NVARCHAR(50),
    BatchNumber NVARCHAR(100),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WH_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey),
    INDEX IX_Fact_WH_Time NONCLUSTERED (TimeKey),
    INDEX IX_Fact_WH_Product NONCLUSTERED (ProductKey),
    INDEX IX_Fact_WH_Zone NONCLUSTERED (ZoneAssignment)
);

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripStartTime DATETIME2 NOT NULL,
    TripEndTime DATETIME2,
    DistanceMiles DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelGallons DECIMAL(10,4),
    LoadWeightPounds DECIMAL(12,2),
    TemperatureDeviationMinutes DECIMAL(10,2), -- For refrigerated/frozen loads
    DelayReasonCode NVARCHAR(50), -- 'Weather', 'Traffic', 'MechanicalIssue', NULL
    MaintenanceFlagCount INT DEFAULT 0,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_Fact_Fleet_Time NONCLUSTERED (TimeKey),
    INDEX IX_Fact_Fleet_Vehicle NONCLUSTERED (VehicleID)
);

-- FactCrossDock: Cross-docking operations (inbound-to-outbound without storage)
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    CrossDockStartTime DATETIME2 NOT NULL,
    CrossDockEndTime DATETIME2,
    TransferDurationMinutes DECIMAL(10,2),
    QuantityUnits INT,
    CONSTRAINT FK_XD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_XD_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_XD_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    INDEX IX_Fact_XD_Time NONCLUSTERED (TimeKey)
);
```

### Step 2: Create Stored Procedures for Data Loading

```sql
-- Stored procedure for calculating product gravity scores
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE p
    SET 
        p.GravityScore = (
            -- Velocity: picks per week
            (SELECT COUNT(*) FROM FactWarehouseOperations w 
             WHERE w.ProductKey = p.ProductKey 
             AND w.OperationType = 'Picking'
             AND w.OperationStartTime >= DATEADD(WEEK, -4, GETDATE())
            ) / 4.0
        ) * 
        -- Value multiplier (placeholder - integrate with ERP pricing)
        1.0 *
        -- Fragility penalty
        CASE p.Fragility
            WHEN 'Low' THEN 1.0
            WHEN 'Medium' THEN 0.7
            WHEN 'High' THEN 0.4
            ELSE 1.0
        END,
        p.LastGravityUpdate = GETDATE()
    FROM DimProductGravity p;
    
    PRINT 'Gravity scores updated for ' + CAST(@@ROWCOUNT AS NVARCHAR) + ' products.';
END;
GO

-- Stored procedure for generating maintenance alerts
CREATE PROCEDURE sp_GenerateFleetMaintenanceAlerts
    @ThresholdIdlePercent DECIMAL(5,2) = 15.0,
    @LookbackHours INT = 24
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        f.VehicleID,
        COUNT(*) AS TripCount,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        AVG(f.DurationMinutes) AS AvgTripMinutes,
        (AVG(f.IdleTimeMinutes) / NULLIF(AVG(f.DurationMinutes), 0)) * 100 AS IdlePercentage,
        SUM(f.MaintenanceFlagCount) AS TotalMaintenanceFlags,
        MAX(f.TripEndTime) AS LastTripEnd
    FROM FactFleetTrips f
    WHERE f.TripStartTime >= DATEADD(HOUR, -@LookbackHours, GETDATE())
    GROUP BY f.VehicleID
    HAVING (AVG(f.IdleTimeMinutes) / NULLIF(AVG(f.DurationMinutes), 0)) * 100 > @ThresholdIdlePercent
    ORDER BY IdlePercentage DESC;
END;
GO

-- Stored procedure for warehouse zone optimization recommendations
CREATE PROCEDURE sp_RecommendWarehouseZoneRealignment
    @WarehouseLocationID NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductVelocity AS (
        SELECT 
            w.ProductKey,
            p.SKU,
            p.ProductName,
            p.GravityScore,
            w.ZoneAssignment AS CurrentZone,
            COUNT(*) AS PicksLast30Days,
            AVG(w.DurationMinutes) AS AvgPickDuration
        FROM FactWarehouseOperations w
        JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
        JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
        WHERE g.LocationID = @WarehouseLocationID
        AND w.OperationType = 'Picking'
        AND w.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY w.ProductKey, p.SKU, p.ProductName, p.GravityScore, w.ZoneAssignment
    )
    SELECT 
        SKU,
        ProductName,
        CurrentZone,
        GravityScore,
        PicksLast30Days,
        AvgPickDuration,
        CASE 
            WHEN GravityScore >= 80 AND CurrentZone NOT LIKE 'A%' THEN 'Move to Zone A (High Gravity)'
            WHEN GravityScore BETWEEN 50 AND 79 AND CurrentZone NOT LIKE 'B%' THEN 'Move to Zone B (Medium Gravity)'
            WHEN GravityScore < 50 AND CurrentZone NOT LIKE 'C%' THEN 'Move to Zone C (Low Gravity)'
            ELSE 'Optimal Placement'
        END AS Recommendation
    FROM ProductVelocity
    WHERE GravityScore IS NOT NULL
    ORDER BY GravityScore DESC;
END;
GO
```

### Step 3: Configure Data Sources

Create a configuration file for external data ingestion (not included in SQL):

```json
{
  "dataSources": {
    "wms": {
      "type": "SQL",
      "connectionString": "${WMS_CONNECTION_STRING}",
      "refreshIntervalMinutes": 15,
      "tables": ["Receiving", "Putaway", "Picking", "Packing", "Shipping"]
    },
    "telematics": {
      "type": "REST_API",
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}",
      "refreshIntervalMinutes": 15,
      "batchSize": 500
    },
    "weather": {
      "type": "REST_API",
      "endpoint": "https://api.weatherapi.com/v1/",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshIntervalMinutes": 60
    }
  },
  "alerting": {
    "smtpServer": "${SMTP_SERVER}",
    "smtpPort": 587,
    "smtpUsername": "${SMTP_USERNAME}",
    "smtpPassword": "${SMTP_PASSWORD}",
    "alertRecipients": ["logistics.manager@company.com", "operations@company.com"]
  }
}
```

### Step 4: Import Power BI Template

1. Download `LogiFleet_Pulse_Master.pbit` from the repository
2. Open in Power BI Desktop
3. When prompted, enter your SQL Server connection details:
   - **Server**: your-sql-server.database.windows.net (or localhost)
   - **Database**: LogiFleetPulse
   - **Authentication**: Windows or SQL Server authentication

The template includes pre-built measures and visualizations:

```dax
-- Example DAX measure: Cross-Fact Fleet Efficiency Score
Fleet Efficiency Score = 
VAR TotalTrips = COUNTROWS(FactFleetTrips)
VAR AvgIdlePercent = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[DurationMinutes])
    ) * 100
VAR AvgFuelEfficiency = 
    DIVIDE(
        SUM(FactFleetTrips[DistanceMiles]),
        SUM(FactFleetTrips[FuelGallons])
    )
VAR DelayRate = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[DelayReasonCode] <> BLANK()),
        TotalTrips
    ) * 100

RETURN
    (100 - AvgIdlePercent) * 0.4 + 
    (AvgFuelEfficiency / 10) * 0.3 + 
    (100 - DelayRate) * 0.3
```

```dax
-- Example DAX measure: Warehouse Gravity Zone Compliance
Gravity Zone Compliance % = 
VAR HighGravityInZoneA = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        RELATED(DimProductGravity[GravityScore]) >= 80,
        FactWarehouseOperations[ZoneAssignment] = "A"
    )
VAR TotalHighGravity = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        RELATED(DimProductGravity[GravityScore]) >= 80
    )

RETURN
    DIVIDE(HighGravityInZoneA, TotalHighGravity) * 100
```

## Key Commands and Operations

### SQL Server Management

```sql
-- Check data freshness
SELECT 
    'FactWarehouseOperations' AS TableName,
    MAX(OperationStartTime) AS LatestRecord,
    COUNT(*) AS TotalRecords
FROM FactWarehouseOperations
UNION ALL
SELECT 
    'FactFleetTrips',
    MAX(TripStartTime),
    COUNT(*)
FROM FactFleetTrips;

-- Run gravity score update (should be scheduled daily)
EXEC sp_UpdateProductGravityScores;

-- Generate maintenance alerts
EXEC sp_GenerateFleetMaintenanceAlerts 
    @ThresholdIdlePercent = 15.0, 
    @LookbackHours = 24;

-- Get zone realignment recommendations for specific warehouse
EXEC sp_RecommendWarehouseZoneRealignment 
    @WarehouseLocationID = 'WH-EAST-01';
```

### Power BI Refresh Schedule

Set up automatic refresh in Power BI Service:

1. Publish the report to Power BI Service
2. Go to Dataset Settings
3. Configure scheduled refresh:
   - **Frequency**: Every 15 minutes (requires Power BI Premium)
   - **Time zone**: Match your operational timezone
   - **Failure notifications**: Enable email alerts

### Row-Level Security Implementation

```sql
-- Create security table
CREATE TABLE SecurityUserWarehouse (
    UserEmail NVARCHAR(255) NOT NULL,
    LocationID NVARCHAR(50) NOT NULL,
    RoleLevel NVARCHAR(50) NOT NULL, -- 'User', 'Supervisor', 'Executive'
    PRIMARY KEY (UserEmail, LocationID)
);

-- Insert security mappings
INSERT INTO SecurityUserWarehouse VALUES
    ('john.smith@company.com', 'WH-EAST-01', 'Supervisor'),
    ('jane.doe@company.com', 'WH-WEST-02', 'User'),
    ('cfo@company.com', '%', 'Executive'); -- % grants access to all locations
```

In Power BI, create RLS roles:

```dax
-- RLS role: Warehouse User
[LocationID] IN 
    (SELECT LocationID FROM SecurityUserWarehouse 
     WHERE UserEmail = USERPRINCIPALNAME())

-- RLS role: Executive (no filter - sees all data)
1 = 1
```

## Common Patterns and Use Cases

### Pattern 1: Cross-Fact Analysis (Warehouse + Fleet)

```sql
-- Identify products with high warehouse dwell time AND high fleet delay correlation
WITH HighDwellProducts AS (
    SELECT 
        w.ProductKey,
        AVG(w.DwellTimeHours) AS AvgDwellHours
    FROM FactWarehouseOperations w
    WHERE w.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY w.ProductKey
    HAVING AVG(w.DwellTimeHours) > 72
),
DelayedShipments AS (
    SELECT 
        xd.ProductKey,
        COUNT(*) AS DelayedShipmentCount
    FROM FactCrossDock xd
    JOIN FactFleetTrips f ON xd.OutboundTripKey = f.TripKey
    WHERE f.DelayReasonCode IS NOT NULL
    AND xd.CrossDockStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY xd.ProductKey
)
SELECT 
    p.SKU,
    p.ProductName,
    hdp.AvgDwellHours,
    ISNULL(ds.DelayedShipmentCount, 0) AS DelayedShipments,
    s.SupplierName,
    s.LeadTimeDaysAvg,
    s.ComplianceScore
FROM HighDwellProducts hdp
JOIN DimProductGravity p ON hdp.ProductKey = p.ProductKey
LEFT JOIN DelayedShipments ds ON hdp.ProductKey = ds.ProductKey
LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
LEFT JOIN DimSupplierReliability s ON w.SupplierKey = s.SupplierKey
ORDER BY hdp.AvgDwellHours DESC, DelayedShipments DESC;
```

### Pattern 2: Temporal Elasticity Simulation

```sql
-- Simulate fleet utilization under 95% warehouse capacity vs. current 80%
DECLARE @CurrentCapacityPercent DECIMAL(5,2) = 80.0;
DECLARE @TargetCapacityPercent DECIMAL(5,2) = 95.0;

WITH CurrentState AS (
    SELECT 
        COUNT(DISTINCT f.VehicleID) AS ActiveVehicles,
        SUM(f.DistanceMiles) AS TotalMiles,
        AVG(f.LoadWeightPounds) AS AvgLoadWeight
    FROM FactFleetTrips f
    WHERE f.TripStartTime >= DATEADD(DAY, -7, GETDATE())
),
WarehouseVolume AS (
    SELECT 
        SUM(w.QuantityUnits) AS CurrentUnits,
        (SUM(w.QuantityUnits) / @CurrentCapacityPercent) * @TargetCapacityPercent AS ProjectedUnits
    FROM FactWarehouseOperations w
    WHERE w.OperationStartTime >= DATEADD(DAY, -7, GETDATE())
)
SELECT 
    cs.ActiveVehicles,
    cs.TotalMiles,
    cs.AvgLoadWeight,
    wv.CurrentUnits,
    wv.ProjectedUnits,
    wv.ProjectedUnits - wv.CurrentUnits AS AdditionalUnits,
    -- Simple linear projection (replace with ML model in production)
    cs.ActiveVehicles * (wv.ProjectedUnits / NULLIF(wv.CurrentUnits, 0)) AS ProjectedVehiclesNeeded,
    cs.TotalMiles * (wv.ProjectedUnits / NULLIF(wv.CurrentUnits, 0)) AS ProjectedMiles
FROM CurrentState cs, WarehouseVolume wv;
```

### Pattern 3: Predictive Bottleneck Detection

```sql
-- Detect probable bottlenecks in next 48 hours based on historical patterns
WITH HourlyLoad AS (
    SELECT 
        DATEPART(HOUR, w.OperationStartTime) AS HourOfDay,
        DATEPART(WEEKDAY, w.OperationStartTime) AS DayOfWeek,
        COUNT(*) AS OperationCount,
        AVG(w.DurationMinutes) AS AvgDuration
    FROM FactWarehouseOperations w
    WHERE w.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY DATEPART(HOUR, w.OperationStartTime), DATEPART(WEEKDAY, w.OperationStartTime)
),
UpcomingHours AS (
    SELECT 
        DATEADD(HOUR, number, GETDATE()) AS FutureHour
    FROM master..spt_values
    WHERE type = 'P' AND number BETWEEN 0 AND 47
)
SELECT 
    uh.FutureHour,
    DATEPART(HOUR, uh.FutureHour) AS HourOfDay,
    DATEPART(WEEKDAY, uh.FutureHour) AS DayOfWeek,
    hl.OperationCount AS HistoricalAvgCount,
    hl.AvgDuration AS HistoricalAvgDuration,
    CASE 
        WHEN hl.OperationCount > 100 AND hl.AvgDuration > 15 THEN 'High Risk'
        WHEN hl.OperationCount > 75 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM UpcomingHours uh
LEFT JOIN HourlyLoad hl 
    ON DATEPART(HOUR, uh.FutureHour) = hl.HourOfDay
    AND DATEPART(WEEKDAY, uh.FutureHour) = hl.DayOfWeek
WHERE hl.OperationCount IS NOT NULL
ORDER BY uh.FutureHour;
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Columnstore index for large fact tables (millions of rows)
CREATE COLUMNSTORE INDEX CSI_FactWarehouseOperations 
ON FactWarehouseOperations;

CREATE COLUMNSTORE INDEX CSI_FactFleetTrips 
ON FactFleetTrips;

-- Filtered index for recent operations (query hot path)
CREATE NONCLUSTERED INDEX IX_WH_Recent 
ON FactWarehouseOperations (OperationStartTime, ProductKey)
WHERE OperationStartTime >= DATEADD(DAY, -90, GETDATE());
```

### Partitioning for Performance

```sql
-- Partition FactFleetTrips by month for efficient historical archiving
CREATE PARTITION FUNCTION PF_TripsByMonth (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01',
    '2026-05-01', '2026-06-01', '2026-07-01', '2026-08-01'
);

CREATE PARTITION SCHEME PS_TripsByMonth
AS PARTITION PF_TripsByMonth
ALL TO ([PRIMARY]);

-- Rebuild FactFleetTrips on partition scheme
-- (Requires data migration - do during maintenance window)
```

## Troubleshooting

### Issue: Power BI Report Refresh Fails

**Symptom**: "Data source error" or timeout in Power BI Service

**Solution**:
1. Check SQL Server firewall rules allow Power BI Service IPs
2. Verify gateway is online (for on-premises SQL Server)
3. Test query performance in SSMS:

```sql
-- Check long-running queries
SELECT 
    session_id,
    start_time,
    status,
    command,
    wait_type,
    wait_time,
    blocking_session_id,
    SUBSTRING(text, 1, 500) AS query_text
FROM sys.dm_exec_requests
CROSS APPLY sys.dm_exec_sql_text(sql_handle)
WHERE database_id = DB_ID('LogiFleetPulse')
AND session_id > 50;
```

### Issue: Gravity Scores Not Updating

**Symptom**: Products remain in wrong zones despite changing velocity

**Solution**:
1. Verify stored procedure runs successfully:

```sql
-- Manually execute and check output
EXEC sp_UpdateProductGravityScores;

-- Check last update times
SELECT 
    SKU, 
    ProductName, 
    GravityScore, 
    LastGravityUpdate,
    DATEDIFF(HOUR, LastGravityUpdate, GETDATE()) AS HoursSinceUpdate
FROM DimProductGravity
WHERE LastGravityUpdate IS NOT NULL
ORDER BY HoursSinceUpdate DESC;
```

2. Schedule SQL Server Agent job:

```sql
-- Create automated job (requires SQL Server Agent)
USE msdb;
GO

EXEC sp_add_job @job_name = 'LogiFleet_UpdateGravityScores';

EXEC sp_add_jobstep
    @job_name = 'LogiFleet_UpdateGravityScores',
    @step_name = 'Execute Gravity Update',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC sp_UpdateProductGravityScores;';

EXEC sp_add_schedule
    @schedule_name = 'DailyAt2AM',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @active_start_time = 020000; -- 2:00 AM

EXEC sp_attach_schedule
    @job_name = 'LogiFleet_UpdateGravityScores',
    @schedule_name = 'DailyAt2AM';

EXEC sp_add_jobserver @job_name = 'LogiFleet_UpdateGravityScores';
```

### Issue: Cross-Fact Queries Too Slow

**Symptom**: Dashboard takes >30 seconds to load

**Solution**:
1. Create aggregated views for common cross-fact patterns:

```sql
-- Pre-aggregated view for fleet-warehouse correlation
CREATE VIEW vw_FleetWarehouseDaily
WITH SCHEMABINDING
AS
SELECT 
    CAST(w.OperationStartTime AS DATE) AS OperationDate,
    w.GeographyKey,
    w.ProductKey,
    COUNT_BIG(*) AS OperationCount,
    AVG(w.DurationMinutes) AS AvgWarehouseDuration,
    AVG(w.DwellTimeHours) AS AvgDwellTime,
    (SELECT AVG(f.IdleTimeMinutes) 
     FROM dbo.FactFleetTrips f 
     WHERE f.OriginGeographyKey = w.GeographyKey 
     AND CAST(f.TripStartTime AS DATE) = CAST(w.OperationStartTime AS DATE)
    ) AS AvgFleetIdleTime
FROM dbo.FactWarehouseOperations w
GROUP BY CAST(w.OperationStartTime AS DATE), w.GeographyKey, w.ProductKey;
GO

-- Create indexed view for performance
CREATE UNIQUE CLUSTERED INDEX UCI_FleetWarehouseDaily 
ON vw_FleetWarehouseDaily (OperationDate, GeographyKey, ProductKey);
```

2. Use Power BI query folding verification:

In Power BI Desktop, go to View → Query Dependencies to ensure queries are pushed to SQL Server rather than processed in Power BI.

### Issue: External API Data Not Integrating

**Symptom**: Weather or telematics data missing in dashboards

**Solution**:
1. Create staging tables for external data:

```sql
-- Staging table for weather data
CREATE TABLE StagingWeatherData (
    WeatherID INT IDENTITY(1,1) PRIMARY
