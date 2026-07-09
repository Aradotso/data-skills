---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform combining SQL Server data warehousing with Power BI for real-time fleet and warehouse analytics
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "create logistics data warehouse with sql server"
  - "build power bi logistics dashboard"
  - "implement fleet and warehouse tracking system"
  - "configure supply chain analytics data model"
  - "deploy logifleet pulse multi-fact schema"
  - "integrate warehouse and fleet telemetry data"
  - "design cross-modal logistics kpi dashboard"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a single analytical framework. It provides:

- **Multi-fact star schema** data warehouse design for MS SQL Server
- **Power BI dashboards** for real-time logistics visualization
- **Cross-fact KPI harmonization** linking warehouse and fleet metrics
- **Predictive analytics** for bottleneck detection and fleet optimization
- **Time-phased dimensional modeling** with 15-minute granularity
- **Warehouse Gravity Zones** for optimal storage placement
- **Fleet triage engine** for maintenance prioritization

The platform integrates data from WMS, telematics, GPS feeds, supplier portals, and external APIs (weather, traffic) into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, TMS, telematics APIs

### Database Deployment

```sql
-- 1. Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- 2. Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    FifteenMinuteBucket INT,
    HourOfDay INT,
    DayOfWeek INT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, Distribution Center, Route Node
    ParentLocationKey INT,
    RegionName VARCHAR(100),
    CountryCode VARCHAR(3),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity + value + fragility
    VelocityRank INT,
    ValueTier VARCHAR(20),
    FragilityLevel VARCHAR(20),
    StorageZoneRecommendation VARCHAR(50),
    LastRecalculatedDate DATETIME
)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    LeadTimeAvgDays DECIMAL(5,2),
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier VARCHAR(20),
    LastAssessmentDate DATETIME
)
GO

-- 3. Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    QuantityHandled INT,
    CycleTimeMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(10,2),
    LaborCost DECIMAL(10,2),
    ZoneCode VARCHAR(20),
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    DistanceMiles DECIMAL(8,2),
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedGallons DECIMAL(8,2),
    LoadWeightLbs INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayReasonCode VARCHAR(50),
    OnTimeDeliveryFlag BIT,
    FuelCost DECIMAL(10,2),
    MaintenanceAlertFlag BIT,
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    QuantityTransferred INT,
    TransferTimeMinutes INT,
    TemperatureCompliance BIT,
    QualityCheckFlag BIT,
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
)
GO

-- 4. Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID)
GO
```

### Time Dimension Population

```sql
-- Populate DimTime with 15-minute granularity
DECLARE @StartDate DATETIME = '2025-01-01'
DECLARE @EndDate DATETIME = '2027-12-31'

;WITH TimeSeries AS (
    SELECT @StartDate AS DateTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, DateTime)
    FROM TimeSeries
    WHERE DateTime < @EndDate
)
INSERT INTO DimTime (TimeKey, DateTime, FifteenMinuteBucket, HourOfDay, DayOfWeek, IsWeekend, IsHoliday)
SELECT 
    CAST(FORMAT(DateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    DateTime,
    DATEPART(MINUTE, DateTime) / 15 AS FifteenMinuteBucket,
    DATEPART(HOUR, DateTime) AS HourOfDay,
    DATEPART(WEEKDAY, DateTime) AS DayOfWeek,
    CASE WHEN DATEPART(WEEKDAY, DateTime) IN (1, 7) THEN 1 ELSE 0 END AS IsWeekend,
    0 AS IsHoliday -- Update separately with holiday calendar
FROM TimeSeries
OPTION (MAXRECURSION 0)
GO
```

## Data Loading Patterns

### Warehouse Operations ETL

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last hour if not specified
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -1, GETDATE())
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        QuantityHandled, CycleTimeMinutes, DwellTimeHours,
        LaborCost, ZoneCode, OperatorID, EquipmentID
    )
    SELECT 
        CAST(FORMAT(wo.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        wo.OperationType,
        wo.QuantityHandled,
        DATEDIFF(MINUTE, wo.StartTime, wo.EndTime) AS CycleTimeMinutes,
        wo.DwellTimeHours,
        wo.LaborCost,
        wo.ZoneCode,
        wo.OperatorID,
        wo.EquipmentID
    FROM 
        StagingWarehouseOps wo
        INNER JOIN DimGeography dg ON wo.LocationCode = dg.LocationCode
        INNER JOIN DimProductGravity dp ON wo.SKU = dp.SKU
    WHERE 
        wo.OperationDateTime > @LastLoadDateTime
        AND wo.ProcessedFlag = 0
    
    -- Mark as processed
    UPDATE StagingWarehouseOps
    SET ProcessedFlag = 1, ProcessedDateTime = GETDATE()
    WHERE OperationDateTime > @LastLoadDateTime
        AND ProcessedFlag = 0
END
GO
```

### Fleet Telemetry Integration

```sql
-- External table configuration for streaming telemetry (Polybase)
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = RDBMS,
    LOCATION = 'telemetry-server.company.com',
    DATABASE_NAME = 'FleetData',
    CREDENTIAL = FleetAPICredential
)
GO

-- View for real-time fleet metrics
CREATE VIEW vw_RealTimeFleetMetrics AS
SELECT 
    ft.VehicleID,
    ft.CurrentLat,
    ft.CurrentLong,
    ft.Speed,
    ft.FuelLevel,
    ft.EngineTemp,
    ft.TirePressureFrontLeft,
    ft.TirePressureFrontRight,
    ft.TirePressureRearLeft,
    ft.TirePressureRearRight,
    ft.LastUpdate,
    -- Calculate maintenance priority score
    CASE 
        WHEN ft.EngineTemp > 220 THEN 100
        WHEN ft.TirePressureFrontLeft < 30 OR ft.TirePressureFrontRight < 30 THEN 90
        WHEN ft.FuelLevel < 0.15 THEN 80
        ELSE 10
    END AS MaintenancePriorityScore
FROM 
    ExternalFleetTelemetry ft
WHERE 
    ft.LastUpdate > DATEADD(MINUTE, -15, GETDATE())
GO
```

### Gravity Score Calculation

```sql
-- Calculate and update product gravity scores
CREATE PROCEDURE usp_RecalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity rank (picks per day)
    ;WITH VelocityCalc AS (
        SELECT 
            ProductKey,
            AVG(CASE WHEN OperationType = 'Picking' THEN QuantityHandled ELSE 0 END) AS AvgDailyPicks,
            RANK() OVER (ORDER BY AVG(CASE WHEN OperationType = 'Picking' THEN QuantityHandled ELSE 0 END) DESC) AS VelocityRank
        FROM 
            FactWarehouseOperations
        WHERE 
            TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY 
            ProductKey
    ),
    -- Get value and fragility from product master
    EnrichedProducts AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            v.VelocityRank,
            pm.UnitValue,
            pm.FragilityScore,
            -- Gravity formula: 40% velocity + 30% value + 30% fragility
            (v.VelocityRank * 0.4 + 
             NTILE(100) OVER (ORDER BY pm.UnitValue DESC) * 0.3 +
             pm.FragilityScore * 0.3) AS GravityScore
        FROM 
            DimProductGravity p
            INNER JOIN VelocityCalc v ON p.ProductKey = v.ProductKey
            INNER JOIN ProductMaster pm ON p.SKU = pm.SKU
    )
    UPDATE p
    SET 
        GravityScore = ep.GravityScore,
        VelocityRank = ep.VelocityRank,
        StorageZoneRecommendation = 
            CASE 
                WHEN ep.GravityScore >= 70 THEN 'Zone A - High Gravity'
                WHEN ep.GravityScore >= 40 THEN 'Zone B - Medium Gravity'
                ELSE 'Zone C - Low Gravity'
            END,
        LastRecalculatedDate = GETDATE()
    FROM 
        DimProductGravity p
        INNER JOIN EnrichedProducts ep ON p.ProductKey = ep.ProductKey
END
GO
```

## Power BI Configuration

### Connection Setup

```powerquery
// Power Query M code for SQL Server connection
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("LOGIFLEET_SQL_SERVER"),
        Environment.GetEnvironmentVariable("LOGIFLEET_DATABASE"),
        [
            Query = null,
            CommandTimeout = #duration(0, 0, 10, 0),
            ConnectionTimeout = #duration(0, 0, 1, 30)
        ]
    ),
    FactWarehouse = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FactFleet = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
    DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
    DimGeography = Source{[Schema="dbo",Item="DimGeography"]}[Data],
    DimProduct = Source{[Schema="dbo",Item="DimProductGravity"]}[Data]
in
    Source
```

### DAX Measures for Cross-Fact KPIs

```dax
// Measure: Average Dwell Time Per SKU
Avg Dwell Time = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    FactWarehouseOperations[OperationType] = "Picking"
)

// Measure: Fleet Idle Time Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// Measure: Cross-Fact Efficiency Ratio
// Links dwell time with fleet utilization
Logistics Efficiency Index = 
VAR WarehouseEff = 1 / (1 + [Avg Dwell Time] / 24)
VAR FleetEff = 1 - ([Fleet Idle %] / 100)
RETURN
    (WarehouseEff * 0.5 + FleetEff * 0.5) * 100

// Measure: On-Time Delivery Rate
OTD Rate = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[OnTimeDeliveryFlag] = 1
    ),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Measure: High-Gravity Products in Wrong Zone
Misplaced High Gravity SKUs = 
CALCULATE(
    DISTINCTCOUNT(FactWarehouseOperations[ProductKey]),
    FILTER(
        DimProductGravity,
        DimProductGravity[GravityScore] >= 70 &&
        DimProductGravity[StorageZoneRecommendation] <> 
            RELATED(FactWarehouseOperations[ZoneCode])
    )
)

// Measure: Predictive Bottleneck Score
Bottleneck Risk Score = 
VAR HighDwell = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[DwellTimeHours] > 72
)
VAR HighIdle = CALCULATE(
    COUNTROWS(FactFleetTrips),
    FactFleetTrips[IdleTimeMinutes] > FactFleetTrips[TripDurationMinutes] * 0.15
)
VAR TotalOps = COUNTROWS(FactWarehouseOperations) + COUNTROWS(FactFleetTrips)
RETURN
    IF(
        TotalOps > 0,
        ((HighDwell + HighIdle) / TotalOps) * 100,
        0
    )
```

### Time Intelligence Patterns

```dax
// Measure: Same Period Last Year Comparison
Dwell Time SPLY = 
CALCULATE(
    [Avg Dwell Time],
    SAMEPERIODLASTYEAR(DimTime[DateTime])
)

// Measure: Rolling 7-Day Average
Dwell Time 7D MA = 
CALCULATE(
    [Avg Dwell Time],
    DATESINPERIOD(
        DimTime[DateTime],
        LASTDATE(DimTime[DateTime]),
        -7,
        DAY
    )
)

// Measure: Period-over-Period Change
Fleet Efficiency Change = 
VAR CurrentPeriod = [Logistics Efficiency Index]
VAR PriorPeriod = CALCULATE(
    [Logistics Efficiency Index],
    DATEADD(DimTime[DateTime], -1, MONTH)
)
RETURN
    CurrentPeriod - PriorPeriod
```

## Automated Alerting System

### SQL Server Agent Job for Threshold Monitoring

```sql
-- Create alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(200),
    MetricType VARCHAR(100),
    ThresholdValue DECIMAL(10,2),
    ThresholdOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    AlertSeverity VARCHAR(20),
    RecipientEmails VARCHAR(MAX),
    IsActive BIT DEFAULT 1
)
GO

-- Insert sample alert rules
INSERT INTO AlertConfiguration (AlertName, MetricType, ThresholdValue, ThresholdOperator, AlertSeverity, RecipientEmails)
VALUES 
    ('High Fleet Idle Time', 'Fleet_Idle_Percentage', 15.0, '>', 'High', '${ALERT_EMAIL_FLEET}'),
    ('Excessive Dwell Time', 'Warehouse_Dwell_Hours', 72.0, '>', 'Medium', '${ALERT_EMAIL_WAREHOUSE}'),
    ('Low OTD Rate', 'OnTime_Delivery_Rate', 90.0, '<', 'High', '${ALERT_EMAIL_OPERATIONS}'),
    ('Maintenance Alert', 'Fleet_Maintenance_Flag', 1.0, '=', 'Critical', '${ALERT_EMAIL_MAINTENANCE}')
GO

-- Alert evaluation stored procedure
CREATE PROCEDURE usp_EvaluateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertResults TABLE (
        AlertName VARCHAR(200),
        CurrentValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2),
        Severity VARCHAR(20),
        Recipients VARCHAR(MAX)
    )
    
    -- Check fleet idle time
    INSERT INTO @AlertResults
    SELECT 
        ac.AlertName,
        (SUM(CAST(ft.IdleTimeMinutes AS DECIMAL)) / NULLIF(SUM(ft.TripDurationMinutes), 0)) * 100 AS CurrentValue,
        ac.ThresholdValue,
        ac.AlertSeverity,
        ac.RecipientEmails
    FROM 
        AlertConfiguration ac
        CROSS APPLY (
            SELECT IdleTimeMinutes, TripDurationMinutes
            FROM FactFleetTrips
            WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -1, GETDATE()), 'yyyyMMddHHmm') AS INT)
        ) ft
    WHERE 
        ac.MetricType = 'Fleet_Idle_Percentage'
        AND ac.IsActive = 1
    HAVING 
        (SUM(CAST(ft.IdleTimeMinutes AS DECIMAL)) / NULLIF(SUM(ft.TripDurationMinutes), 0)) * 100 > ac.ThresholdValue
    
    -- Check warehouse dwell time
    INSERT INTO @AlertResults
    SELECT 
        ac.AlertName,
        AVG(wo.DwellTimeHours) AS CurrentValue,
        ac.ThresholdValue,
        ac.AlertSeverity,
        ac.RecipientEmails
    FROM 
        AlertConfiguration ac
        CROSS APPLY (
            SELECT DwellTimeHours
            FROM FactWarehouseOperations
            WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -24, GETDATE()), 'yyyyMMddHHmm') AS INT)
        ) wo
    WHERE 
        ac.MetricType = 'Warehouse_Dwell_Hours'
        AND ac.IsActive = 1
    HAVING 
        AVG(wo.DwellTimeHours) > ac.ThresholdValue
    
    -- Send alerts via Database Mail
    IF EXISTS (SELECT 1 FROM @AlertResults)
    BEGIN
        DECLARE @EmailBody NVARCHAR(MAX)
        
        SELECT @EmailBody = STRING_AGG(
            CONCAT(
                'Alert: ', AlertName, CHAR(13), CHAR(10),
                'Current Value: ', CAST(CurrentValue AS VARCHAR(20)), CHAR(13), CHAR(10),
                'Threshold: ', CAST(ThresholdValue AS VARCHAR(20)), CHAR(13), CHAR(10),
                'Severity: ', Severity, CHAR(13), CHAR(10),
                CHAR(13), CHAR(10)
            ),
            '---' + CHAR(13) + CHAR(10)
        )
        FROM @AlertResults
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_ADMIN}',
            @subject = 'LogiFleet Pulse - Threshold Alerts Triggered',
            @body = @EmailBody
    END
END
GO
```

## Advanced Analytics Queries

### Identify Optimal Storage Zone Reassignments

```sql
-- Find products that should be moved to different zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.StorageZoneRecommendation AS RecommendedZone,
    wo.ZoneCode AS CurrentZone,
    COUNT(*) AS PickOperations,
    AVG(wo.CycleTimeMinutes) AS AvgPickTime,
    -- Calculate potential time savings if moved to recommended zone
    (AVG(wo.CycleTimeMinutes) - 
     AVG(CASE 
         WHEN wo.ZoneCode = p.StorageZoneRecommendation 
         THEN wo.CycleTimeMinutes 
         ELSE NULL 
     END)) AS PotentialTimeSavingsMinutes
FROM 
    DimProductGravity p
    INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
WHERE 
    wo.OperationType = 'Picking'
    AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
    AND p.StorageZoneRecommendation <> wo.ZoneCode
GROUP BY 
    p.SKU, p.ProductName, p.GravityScore, 
    p.StorageZoneRecommendation, wo.ZoneCode
HAVING 
    COUNT(*) > 50 -- Only products with significant pick volume
ORDER BY 
    PotentialTimeSavingsMinutes DESC
```

### Fleet Route Optimization Analysis

```sql
-- Identify routes with high idle time and their contributing factors
WITH RouteAnalysis AS (
    SELECT 
        ft.OriginGeographyKey,
        ft.DestinationGeographyKey,
        og.LocationName AS Origin,
        dg.LocationName AS Destination,
        COUNT(*) AS TripCount,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(ft.TripDurationMinutes) AS AvgTripDuration,
        AVG(CAST(ft.IdleTimeMinutes AS DECIMAL) / NULLIF(ft.TripDurationMinutes, 0)) * 100 AS IdlePercentage,
        AVG(ft.FuelConsumedGallons) AS AvgFuelConsumption,
        SUM(CASE WHEN ft.OnTimeDeliveryFlag = 0 THEN 1 ELSE 0 END) AS DelayedTrips
    FROM 
        FactFleetTrips ft
        INNER JOIN DimGeography og ON ft.OriginGeographyKey = og.GeographyKey
        INNER JOIN DimGeography dg ON ft.DestinationGeographyKey = dg.GeographyKey
    WHERE 
        ft.TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY 
        ft.OriginGeographyKey, ft.DestinationGeographyKey,
        og.LocationName, dg.LocationName
)
SELECT 
    Origin,
    Destination,
    TripCount,
    AvgIdleTime,
    IdlePercentage,
    AvgFuelConsumption,
    DelayedTrips,
    CAST((DelayedTrips * 100.0 / TripCount) AS DECIMAL(5,2)) AS DelayPercentage,
    -- Priority score for route optimization
    (IdlePercentage * 0.4 + 
     (DelayedTrips * 100.0 / TripCount) * 0.4 + 
     (AvgFuelConsumption / NULLIF((SELECT AVG(AvgFuelConsumption) FROM RouteAnalysis), 0) * 100 - 100) * 0.2
    ) AS OptimizationPriorityScore
FROM 
    RouteAnalysis
WHERE 
    TripCount >= 10
ORDER BY 
    OptimizationPriorityScore DESC
```

### Supplier Performance Impact on Operations

```sql
-- Correlate supplier reliability with warehouse dwell time
SELECT 
    s.SupplierCode,
    s.SupplierName,
    s.LeadTimeAvgDays,
    s.LeadTimeVariance,
    s.DefectPercentage,
    s.ReliabilityTier,
    COUNT(DISTINCT wo.ProductKey) AS ProductCount,
    AVG(wo.DwellTimeHours) AS AvgDwellTime,
    AVG(p.GravityScore) AS AvgProductGravity,
    -- Calculate impact score
    (s.LeadTimeVariance * 0.3 + 
     s.DefectPercentage * 1000 * 0.3 +
     AVG(wo.DwellTimeHours) * 0.4
    ) AS SupplierImpactScore
FROM 
    DimSupplierReliability s
    INNER JOIN ProductMaster pm ON s.SupplierCode = pm.SupplierCode
    INNER JOIN DimProductGravity p ON pm.SKU = p.SKU
    INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
WHERE 
    wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -60, GETDATE()), 'yyyyMMddHHmm') AS INT)
    AND wo.OperationType IN ('Receiving', 'Putaway')
GROUP BY 
    s.SupplierCode, s.SupplierName, s.LeadTimeAvgDays,
    s.LeadTimeVariance, s.DefectPercentage, s.ReliabilityTier
ORDER BY 
    SupplierImpactScore DESC
```

## Configuration Management

### Environment Variables Setup

```bash
# SQL Server connection
export LOGIFLEET_SQL_SERVER="sql-server.company.com"
export LOGIFLEET_DATABASE="LogiFleetPulse"
export LOGIFLEET_SQL_USER="logifleet_app"
export LOGIFLEET_SQL_PASSWORD="${DB_PASSWORD}"

# Alert email recipients
export ALERT_EMAIL_ADMIN="logistics-admin@company.com"
export ALERT_EMAIL_FLEET="fleet-manager@company.com"
export ALERT_EMAIL_WAREHOUSE="warehouse-ops@company.com"
export ALERT_EMAIL_OPERATIONS="operations-team@company.com"
export ALERT_EMAIL_MAINTENANCE="maintenance@company.com"

# External API integrations
export WEATHER_API_KEY="${WEATHER_API_KEY}"
export TRAFFIC_API_KEY="${TRAFFIC_API_KEY}"
export TELEMETRY_API_ENDPOINT="https://telemetry.company.com/api"
export TELEMETRY_API_KEY="${TELEMETRY_API_KEY}"

# Power BI service
export POWERBI_WORKSPACE_ID="${POWERBI_WORKSPACE}"
export POWERBI_REFRESH_SCHEDULE="0 */15 * * *" # Every 15 minutes
```

### Connection String Template

```xml
<!-- App.config or Web.config -->
<connectionStrings>
  <add name="LogiFleetPulse" 
       connectionString="Server=${LOGIFLEET_SQL_SERVER};Database=${LOGIFLEET_DATABASE};User Id=${LOGIFLEET_SQL_USER};Password=${LOGIFLEET_SQL_PASSWORD};Encrypt=True;TrustServer
