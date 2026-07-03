---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logistics analytics dashboard"
  - "configure supply chain data warehouse"
  - "build fleet management KPI reports"
  - "implement warehouse operations analytics"
  - "create multi-fact star schema for logistics"
  - "deploy Power BI logistics intelligence"
  - "analyze fleet telemetry and warehouse metrics"
  - "configure cross-modal supply chain reporting"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to work with **LogiFleet Pulse**, an advanced MS SQL Server and Power BI data warehousing solution for logistics and supply chain analytics. The platform provides a multi-fact star schema integrating warehouse operations, fleet telemetry, inventory management, and external data sources into unified KPI dashboards.

## What LogiFleet Pulse Does

LogiFleet Pulse is a **logistics intelligence engine** that:
- Implements a multi-fact star schema for warehouse, fleet, and cross-dock operations
- Provides time-phased dimensions (15-minute granularity) for real-time analytics
- Harmonizes cross-fact KPIs (inventory turnover vs. fuel consumption, dwell time vs. route efficiency)
- Offers Power BI dashboards with role-based access and predictive bottleneck detection
- Includes "Warehouse Gravity Zones™" for spatial optimization based on pick frequency and value
- Supports adaptive fleet triage using weighted telemetry scoring

**Primary Use Cases:**
- Real-time logistics operations monitoring
- Fleet optimization and predictive maintenance
- Warehouse efficiency and spatial planning
- Supply chain bottleneck prediction
- Cross-modal transportation analytics

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Power BI Service account (for publishing dashboards)
- Source systems: WMS, TMS, telematics API, or equivalent data feeds

### Database Setup

1. **Deploy the SQL schema:**

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    TimeStamp DATETIME2(0) NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    WeekOfYear TINYINT,
    Month TINYINT,
    Quarter TINYINT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
)

CREATE NONCLUSTERED INDEX IX_DimTime_TimeStamp ON DimTime(TimeStamp)
GO

CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- 'Warehouse', 'Route Node', 'Distribution Center'
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
)

CREATE UNIQUE INDEX IX_DimGeography_LocationCode ON DimGeography(LocationCode)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(100) NOT NULL,
    ProductName NVARCHAR(500),
    Category NVARCHAR(200),
    SubCategory NVARCHAR(200),
    GravityScore DECIMAL(5,2), -- Composite score: velocity + value + fragility
    VelocityClass NVARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier NVARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2),
    OptimalZone NVARCHAR(50) -- Recommended warehouse zone
)

CREATE UNIQUE INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU)
CREATE NONCLUSTERED INDEX IX_DimProductGravity_Gravity ON DimProductGravity(GravityScore DESC)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode NVARCHAR(100) NOT NULL,
    SupplierName NVARCHAR(500),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier NVARCHAR(20) -- 'A', 'B', 'C'
)

CREATE UNIQUE INDEX IX_DimSupplier_Code ON DimSupplierReliability(SupplierCode)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    SupplierKey INT NULL,
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    CycleTimeMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(10,2),
    ZoneAssigned NVARCHAR(50),
    OperatorID NVARCHAR(50),
    CONSTRAINT FK_FactWarehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_FactWarehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWarehouse_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
)

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    LoadingTimeMinutes DECIMAL(8,2),
    UnloadingTimeMinutes DECIMAL(8,2),
    LoadWeightLbs DECIMAL(10,2),
    TirePressureAvg DECIMAL(5,2),
    EngineHealthScore DECIMAL(5,2),
    DelayMinutes DECIMAL(8,2),
    DelayReason NVARCHAR(200),
    CONSTRAINT FK_FactFleet_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)

CREATE NONCLUSTERED INDEX IX_FactFleet_StartTime ON FactFleetTrips(StartTimeKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTimeKey INT NOT NULL,
    OutboundTimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    InboundVehicleID NVARCHAR(50),
    OutboundVehicleID NVARCHAR(50),
    QuantityTransferred INT,
    TransferTimeMinutes DECIMAL(8,2),
    CONSTRAINT FK_FactCrossDock_InboundTime FOREIGN KEY (InboundTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactCrossDock_OutboundTime FOREIGN KEY (OutboundTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactCrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_FactCrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
)

CREATE NONCLUSTERED INDEX IX_FactCrossDock_InboundTime ON FactCrossDock(InboundTimeKey)
GO
```

2. **Create stored procedures for data loading:**

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimeKey INT
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, SupplierKey,
        OperationType, QuantityHandled, CycleTimeMinutes,
        DwellTimeHours, ZoneAssigned, OperatorID
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.SupplierKey,
        src.OperationType,
        src.QuantityHandled,
        src.CycleTimeMinutes,
        src.DwellTimeHours,
        src.ZoneAssigned,
        src.OperatorID
    FROM ExternalWarehouseData src
    INNER JOIN DimTime t ON CAST(src.OperationTimestamp AS DATETIME2(0)) = t.TimeStamp
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    INNER JOIN DimGeography g ON src.LocationCode = g.LocationCode
    LEFT JOIN DimSupplierReliability s ON src.SupplierCode = s.SupplierCode
    WHERE t.TimeKey > @LastLoadTimeKey
END
GO

-- Calculate Warehouse Gravity Scores
CREATE PROCEDURE usp_CalculateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE p
    SET GravityScore = (
        (CASE VelocityClass 
            WHEN 'Fast' THEN 50 
            WHEN 'Medium' THEN 30 
            WHEN 'Slow' THEN 10 
        END) +
        (CASE ValueTier 
            WHEN 'High' THEN 30 
            WHEN 'Medium' THEN 20 
            WHEN 'Low' THEN 10 
        END) +
        (FragilityIndex * 20)
    ),
    OptimalZone = CASE 
        WHEN VelocityClass = 'Fast' AND ValueTier = 'High' THEN 'Zone-A-Priority'
        WHEN VelocityClass = 'Fast' THEN 'Zone-A-Standard'
        WHEN VelocityClass = 'Medium' THEN 'Zone-B'
        ELSE 'Zone-C'
    END
    FROM DimProductGravity p
END
GO
```

3. **Create views for common analytics queries:**

```sql
-- Cross-fact KPI view: Dwell time vs Fleet idle cost
CREATE VIEW vw_DwellTimeVsFleetCost
AS
SELECT 
    t.FiscalYear,
    t.Quarter,
    t.Month,
    g.Region,
    p.Category,
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(ft.FuelConsumedGallons) AS AvgFuelConsumption,
    COUNT(DISTINCT wo.OperationKey) AS WarehouseOperations,
    COUNT(DISTINCT ft.TripKey) AS FleetTrips
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON ft.StartTimeKey = t.TimeKey AND ft.OriginGeographyKey = g.GeographyKey
GROUP BY t.FiscalYear, t.Quarter, t.Month, g.Region, p.Category
GO

-- Fleet maintenance priority queue
CREATE VIEW vw_FleetMaintenancePriority
AS
SELECT 
    ft.VehicleID,
    AVG(ft.EngineHealthScore) AS AvgEngineHealth,
    AVG(ft.TirePressureAvg) AS AvgTirePressure,
    SUM(ft.DistanceMiles) AS TotalMiles,
    SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
    COUNT(*) AS TotalTrips,
    -- Priority score: lower engine health + higher mileage + longer idle = higher priority
    (100 - AVG(ft.EngineHealthScore)) * 0.5 + 
    (SUM(ft.DistanceMiles) / 1000) * 0.3 + 
    (SUM(ft.IdleTimeMinutes) / 60) * 0.2 AS MaintenancePriorityScore
FROM FactFleetTrips ft
WHERE ft.StartTimeKey >= (SELECT MAX(TimeKey) - 720 FROM DimTime) -- Last 30 days (48 intervals/day)
GROUP BY ft.VehicleID
GO
```

### Power BI Configuration

1. **Connect to SQL Server:**
   - Open `LogiFleet_Pulse_Master.pbit` template
   - Enter your SQL Server connection: `Server=${SQL_SERVER};Database=LogiFleetPulse`
   - Use Windows Authentication or SQL credentials stored in environment variables

2. **Refresh Data Model:**

Power BI automatically detects relationships based on foreign keys. Verify the star schema structure:

```
DimTime (1) → (*) FactWarehouseOperations
DimTime (1) → (*) FactFleetTrips
DimTime (1) → (*) FactCrossDock
DimProductGravity (1) → (*) FactWarehouseOperations
DimProductGravity (1) → (*) FactCrossDock
DimGeography (1) → (*) FactWarehouseOperations
DimGeography (1) → (*) FactFleetTrips (Origin & Destination)
```

3. **Configure Row-Level Security (RLS):**

```dax
-- In Power BI Desktop, create role "RegionManagers"
[Region] = USERNAME()

-- For warehouse operators, restrict to specific zones
[ZoneAssigned] IN {"Zone-A-Priority", "Zone-A-Standard"}
```

## Key DAX Measures

### Cross-Fact KPI: Inventory Turnover vs Fuel Efficiency

```dax
InventoryTurnoverRate = 
VAR TotalShipped = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
VAR AvgInventory = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Putaway"
    )
RETURN
    DIVIDE(TotalShipped, AvgInventory, 0)

FuelEfficiencyMPG = 
    DIVIDE(
        SUM(FactFleetTrips[DistanceMiles]),
        SUM(FactFleetTrips[FuelConsumedGallons]),
        0
    )

OperationalSynergy = 
    [InventoryTurnoverRate] * [FuelEfficiencyMPG] / 10
```

### Warehouse Gravity Zone Performance

```dax
GravityZoneEfficiency = 
VAR ZonePickTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[CycleTimeMinutes]),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR ExpectedPickTime = 
    CALCULATE(
        AVERAGEX(
            DimProductGravity,
            SWITCH(
                DimProductGravity[OptimalZone],
                "Zone-A-Priority", 3,
                "Zone-A-Standard", 5,
                "Zone-B", 8,
                12
            )
        )
    )
RETURN
    DIVIDE(ExpectedPickTime, ZonePickTime, 0) * 100
```

### Predictive Bottleneck Index

```dax
BottleneckIndex = 
VAR DwellSpike = 
    DIVIDE(
        CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeHours]), DATESINPERIOD(DimTime[TimeStamp], MAX(DimTime[TimeStamp]), -2, HOUR)),
        CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeHours]), DATESINPERIOD(DimTime[TimeStamp], MAX(DimTime[TimeStamp]), -24, HOUR)),
        1
    )
VAR FleetDelay = 
    DIVIDE(
        SUM(FactFleetTrips[DelayMinutes]),
        SUM(FactFleetTrips[DistanceMiles]),
        0
    )
RETURN
    (DwellSpike - 1) * 50 + FleetDelay * 10
```

## Configuration Files

### Data Source Configuration (JSON)

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER}",
    "database": "LogiFleetPulse",
    "authType": "WindowsIntegrated",
    "timeout": 30
  },
  "externalSources": {
    "wmsApi": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}",
      "refreshIntervalMinutes": 15
    },
    "telematicsApi": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}",
      "refreshIntervalMinutes": 5
    },
    "weatherApi": {
      "endpoint": "https://api.weather.com/v3/wx/conditions/current",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshIntervalMinutes": 60
    }
  },
  "alerts": {
    "dwellTimeThresholdHours": 72,
    "idleTimeThresholdPercent": 15,
    "engineHealthThreshold": 60,
    "emailRecipients": ["${ALERT_EMAIL_OPS}", "${ALERT_EMAIL_MANAGEMENT}"]
  }
}
```

### Automated Alert Configuration (SQL)

```sql
-- Create alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName NVARCHAR(200),
    AlertType NVARCHAR(50), -- 'DwellTime', 'FleetIdle', 'EngineHealth', 'BottleneckIndex'
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator NVARCHAR(10), -- '>', '<', '='
    RecipientEmails NVARCHAR(MAX),
    IsActive BIT DEFAULT 1
)

-- Sample alert configurations
INSERT INTO AlertConfiguration (AlertName, AlertType, ThresholdValue, ComparisonOperator, RecipientEmails)
VALUES 
    ('Excessive Dwell Time', 'DwellTime', 72, '>', '${ALERT_EMAIL_OPS}'),
    ('Fleet Idle Warning', 'FleetIdle', 15, '>', '${ALERT_EMAIL_FLEET}'),
    ('Low Engine Health', 'EngineHealth', 60, '<', '${ALERT_EMAIL_MAINTENANCE}'),
    ('Bottleneck Risk', 'BottleneckIndex', 50, '>', '${ALERT_EMAIL_MANAGEMENT}')

-- Alert execution stored procedure
CREATE PROCEDURE usp_CheckAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX)
    
    -- Check dwell time alerts
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.TimeStamp >= DATEADD(HOUR, -24, GETDATE())
        AND wo.DwellTimeHours > (SELECT ThresholdValue FROM AlertConfiguration WHERE AlertType = 'DwellTime')
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Dwell time threshold exceeded. Check warehouse operations.'
        -- Send email using sp_send_dbmail (configure Database Mail first)
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = (SELECT RecipientEmails FROM AlertConfiguration WHERE AlertType = 'DwellTime'),
            @subject = 'LogiFleet Alert: Excessive Dwell Time',
            @body = @AlertMessage
    END
    
    -- Check fleet idle alerts
    IF EXISTS (
        SELECT 1
        FROM (
            SELECT 
                VehicleID,
                SUM(IdleTimeMinutes) * 100.0 / SUM(DATEDIFF(MINUTE, 0, DATEADD(MINUTE, IdleTimeMinutes + LoadingTimeMinutes + UnloadingTimeMinutes, 0))) AS IdlePercentage
            FROM FactFleetTrips ft
            INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
            WHERE t.TimeStamp >= DATEADD(HOUR, -24, GETDATE())
            GROUP BY VehicleID
        ) AS FleetIdle
        WHERE IdlePercentage > (SELECT ThresholdValue FROM AlertConfiguration WHERE AlertType = 'FleetIdle')
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeds threshold. Review route optimization.'
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = (SELECT RecipientEmails FROM AlertConfiguration WHERE AlertType = 'FleetIdle'),
            @subject = 'LogiFleet Alert: Excessive Fleet Idle Time',
            @body = @AlertMessage
    END
END
GO

-- Schedule alert checks every 15 minutes using SQL Server Agent
```

## Common Analytical Patterns

### Pattern 1: Cross-Modal Correlation Analysis

Analyze how warehouse operations impact fleet performance:

```sql
-- Find correlation between warehouse dwell time and fleet delays
WITH WarehouseDwell AS (
    SELECT 
        g.GeographyKey,
        g.LocationName,
        t.TimeKey,
        AVG(wo.DwellTimeHours) AS AvgDwellTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.TimeStamp >= DATEADD(DAY, -30, GETDATE())
    GROUP BY g.GeographyKey, g.LocationName, t.TimeKey
),
FleetDelays AS (
    SELECT 
        ft.OriginGeographyKey,
        ft.StartTimeKey,
        AVG(ft.DelayMinutes) AS AvgDelay
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    WHERE t.TimeStamp >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ft.OriginGeographyKey, ft.StartTimeKey
)
SELECT 
    wd.LocationName,
    AVG(wd.AvgDwellTime) AS AvgWarehouseDwellHours,
    AVG(fd.AvgDelay) AS AvgFleetDelayMinutes,
    -- Simple correlation coefficient approximation
    (AVG(wd.AvgDwellTime * fd.AvgDelay) - AVG(wd.AvgDwellTime) * AVG(fd.AvgDelay)) / 
    (STDEV(wd.AvgDwellTime) * STDEV(fd.AvgDelay)) AS CorrelationCoefficient
FROM WarehouseDwell wd
INNER JOIN FleetDelays fd ON wd.GeographyKey = fd.OriginGeographyKey AND wd.TimeKey = fd.StartTimeKey
GROUP BY wd.LocationName
ORDER BY CorrelationCoefficient DESC
```

### Pattern 2: Temporal Elasticity Simulation

Model impact of capacity changes:

```sql
-- Simulate fleet utilization under different warehouse capacity scenarios
DECLARE @CurrentCapacityPercent DECIMAL(5,2) = 80
DECLARE @ProposedCapacityPercent DECIMAL(5,2) = 95
DECLARE @CapacityImpactFactor DECIMAL(5,2) = (@ProposedCapacityPercent - @CurrentCapacityPercent) / 100.0

SELECT 
    ft.VehicleID,
    SUM(ft.DistanceMiles) AS CurrentTotalMiles,
    SUM(ft.DistanceMiles) * (1 + @CapacityImpactFactor * 0.3) AS ProjectedTotalMiles,
    AVG(ft.IdleTimeMinutes) AS CurrentAvgIdleTime,
    AVG(ft.IdleTimeMinutes) * (1 - @CapacityImpactFactor * 0.2) AS ProjectedAvgIdleTime,
    SUM(ft.FuelConsumedGallons) AS CurrentFuelUsage,
    SUM(ft.FuelConsumedGallons) * (1 + @CapacityImpactFactor * 0.25) AS ProjectedFuelUsage
FROM FactFleetTrips ft
INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
WHERE t.TimeStamp >= DATEADD(DAY, -90, GETDATE())
GROUP BY ft.VehicleID
ORDER BY CurrentFuelUsage DESC
```

### Pattern 3: Gravity Zone Optimization

Identify products that should be reassigned to different zones:

```sql
-- Products not in optimal zones based on actual pick performance
SELECT 
    p.SKU,
    p.ProductName,
    p.OptimalZone AS RecommendedZone,
    wo.ZoneAssigned AS CurrentZone,
    AVG(wo.CycleTimeMinutes) AS AvgPickTime,
    COUNT(*) AS PickFrequency,
    p.GravityScore,
    CASE 
        WHEN wo.ZoneAssigned <> p.OptimalZone THEN 'Reassign'
        ELSE 'OK'
    END AS RecommendedAction
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
  AND wo.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
GROUP BY p.SKU, p.ProductName, p.OptimalZone, wo.ZoneAssigned, p.GravityScore
HAVING COUNT(*) > 10 -- Minimum pick frequency threshold
ORDER BY p.GravityScore DESC, PickFrequency DESC
```

## Troubleshooting

### Issue: Slow Dashboard Refresh

**Symptoms:** Power BI dashboards take >30 seconds to refresh

**Solutions:**
1. **Add aggregation tables:**
```sql
-- Pre-aggregate daily summaries
CREATE TABLE FactWarehouseDaily (
    SummaryDate DATE,
    GeographyKey INT,
    ProductKey INT,
    TotalOperations INT,
    AvgDwellTime DECIMAL(10,2),
    TotalQuantity INT
)

CREATE CLUSTERED INDEX IX_FactWarehouseDaily_Date ON FactWarehouseDaily(SummaryDate)

-- Populate nightly via scheduled job
INSERT INTO FactWarehouseDaily
SELECT 
    CAST(t.TimeStamp AS DATE),
    wo.GeographyKey,
    wo.ProductKey,
    COUNT(*),
    AVG(wo.DwellTimeHours),
    SUM(wo.QuantityHandled)
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE CAST(t.TimeStamp AS DATE) = CAST(GETDATE() - 1 AS DATE)
GROUP BY CAST(t.TimeStamp AS DATE), wo.GeographyKey, wo.ProductKey
```

2. **Partition fact tables by date:**
```sql
-- Create partition function and scheme
CREATE PARTITION FUNCTION pfDailyPartition (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301) -- Monthly partitions

CREATE PARTITION SCHEME psDailyPartition
AS PARTITION pfDailyPartition ALL TO ([PRIMARY])

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations (
    -- columns as before
) ON psDailyPartition(TimeKey)
```

### Issue: Missing Data from External Sources

**Symptoms:** Gaps in telemetry or WMS data

**Solutions:**
1. **Add data quality checks:**
```sql
CREATE PROCEDURE usp_ValidateDataLoad
    @SourceTable NVARCHAR(100),
    @ExpectedMinRows INT
AS
BEGIN
    DECLARE @ActualRows INT
    DECLARE @LatestTimestamp DATETIME2
    
    IF @SourceTable = 'FactFleetTrips'
    BEGIN
        SELECT @ActualRows = COUNT(*), @LatestTimestamp = MAX(t.TimeStamp)
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
        WHERE t.TimeStamp >= DATEADD(HOUR, -1, GETDATE())
        
        IF @ActualRows < @ExpectedMinRows
        BEGIN
            RAISERROR('Data quality issue: Insufficient fleet trips loaded in last hour', 16, 1)
        END
    END
END
GO
```

2. **Implement retry logic for API calls (in data pipeline):**
```sql
-- Example using SQLCLR or external ETL tool
-- Log failed API calls to retry queue
CREATE TABLE APIRetryQueue (
    RetryID INT IDENTITY(1,1),
    APIEndpoint NVARCHAR(500),
    RequestPayload NVARCHAR(MAX),
    FailureTimestamp DATETIME2 DEFAULT GETDATE(),
    RetryCount INT DEFAULT 0,
    LastError NVARCHAR(MAX)
