---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform with MS SQL Server data warehousing and Power BI dashboards for warehouse, fleet, and supply chain KPI harmonization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure logistics data warehouse with SQL Server"
  - "build Power BI dashboard for fleet and warehouse operations"
  - "implement cross-fact KPI supply chain analysis"
  - "create star schema for logistics intelligence"
  - "deploy warehouse gravity zone optimization"
  - "integrate fleet telemetry with warehouse data"
  - "set up real-time logistics monitoring dashboard"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time analytics
- **Warehouse Gravity Zones™** for optimal storage placement based on velocity and value
- **Adaptive Fleet Triage Engine** for predictive maintenance prioritization
- **Cross-fact KPI harmonization** (e.g., dwell time vs. fleet idling cost)
- **Real-time Power BI dashboards** with role-based access control

The platform is designed for 3PLs, retail chains, food distributors, and any organization managing complex logistics operations.

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate credentials for WMS, TMS, and telemetry data sources

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Deploy dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    FifteenMinuteBucket SMALLINT,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayOfMonth TINYINT,
    MonthOfYear TINYINT,
    FiscalQuarter TINYINT,
    FiscalYear SMALLINT,
    IsWeekend BIT,
    IsHoliday BIT
)

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) NOT NULL,
    NodeType VARCHAR(20) CHECK (NodeType IN ('Warehouse', 'Route', 'CrossDock')),
    NodeName NVARCHAR(200),
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    PostalCode VARCHAR(20),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50)
)

CREATE UNIQUE INDEX IX_DimGeography_NodeID ON DimGeography(NodeID)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName NVARCHAR(300),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    IsPerishable BIT,
    ShelfLifeDays INT,
    StandardCost DECIMAL(18,2),
    RetailPrice DECIMAL(18,2),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility factor
    RecommendedZone VARCHAR(20)
)

CREATE UNIQUE INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU)
CREATE INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(200),
    AverageLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100 scale
    LastAuditDate DATE
)

CREATE UNIQUE INDEX IX_DimSupplierReliability_SupplierID ON DimSupplierReliability(SupplierID)
GO

-- Deploy fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) CHECK (OperationType IN ('Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping')),
    OrderID VARCHAR(50),
    BatchNumber VARCHAR(50),
    QuantityHandled INT,
    CycleTimeMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(10,2),
    ZoneAssignment VARCHAR(20),
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    ErrorCount INT DEFAULT 0,
    CONSTRAINT FK_FactWarehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWarehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
)

CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
CREATE INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey)
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
CREATE INDEX IX_FactWarehouse_OperationType ON FactWarehouseOperations(OperationType)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    TripID VARCHAR(50),
    RouteSegment VARCHAR(100),
    DistanceMiles DECIMAL(10,2),
    FuelGallons DECIMAL(8,3),
    IdleTimeMinutes DECIMAL(8,2),
    LoadingTimeMinutes DECIMAL(8,2),
    UnloadingTimeMinutes DECIMAL(8,2),
    TotalTripTimeMinutes DECIMAL(10,2),
    AverageSpeedMPH DECIMAL(5,2),
    HarshBrakingEvents INT DEFAULT 0,
    HarshAccelerationEvents INT DEFAULT 0,
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes DECIMAL(8,2),
    TirePressuePSI DECIMAL(5,1),
    EngineHealthScore DECIMAL(5,2), -- 0-100 scale
    LoadWeightLbs DECIMAL(10,2),
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)

CREATE INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey)
CREATE INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID)
CREATE INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferTimeMinutes DECIMAL(8,2),
    QuantityTransferred INT,
    TemperatureCompliance BIT,
    CONSTRAINT FK_FactCrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactCrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactCrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
)

CREATE INDEX IX_FactCrossDock_Time ON FactCrossDock(TimeKey)
GO
```

### Step 2: Create ETL Stored Procedures

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example integration with WMS (adjust to your data source)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, OrderID,
        QuantityHandled, CycleTimeMinutes, DwellTimeHours, ZoneAssignment
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        wms.OperationType,
        wms.OrderID,
        wms.Quantity,
        wms.CycleTime,
        wms.DwellTime,
        wms.ZoneAssignment
    FROM ExternalSource.WMS_Operations wms
    INNER JOIN DimTime t ON DATEADD(MINUTE, (DATEPART(MINUTE, wms.OperationDateTime) / 15) * 15, 
        DATEADD(HOUR, DATEDIFF(HOUR, 0, wms.OperationDateTime), 0)) = t.DateTime
    INNER JOIN DimGeography g ON wms.WarehouseID = g.NodeID
    INNER JOIN DimProductGravity p ON wms.SKU = p.SKU
    WHERE wms.LastModified > @LastLoadTime
    
    RETURN @@ROWCOUNT
END
GO

-- Calculate and update Gravity Scores
CREATE PROCEDURE sp_UpdateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickFrequency,
            SUM(QuantityHandled) AS TotalVolume
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE DateTime >= DATEADD(DAY, -30, GETDATE()))
        GROUP BY ProductKey
    )
    UPDATE p
    SET GravityScore = 
        (ISNULL(pv.PickFrequency, 0) * 0.4) +  -- Velocity weight
        (ISNULL(p.RetailPrice / NULLIF(p.StandardCost, 0), 1) * 0.3) +  -- Value weight
        (CASE WHEN p.IsFragile = 1 THEN 15 ELSE 0 END) +  -- Fragility penalty
        (CASE WHEN p.IsPerishable = 1 THEN 20 ELSE 0 END),  -- Perishability penalty
        RecommendedZone = 
        CASE 
            WHEN GravityScore > 50 THEN 'High-Gravity'
            WHEN GravityScore > 25 THEN 'Medium-Gravity'
            ELSE 'Low-Gravity'
        END
    FROM DimProductGravity p
    LEFT JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
END
GO

-- Fleet maintenance triage scoring
CREATE PROCEDURE sp_GenerateFleetTriageQueue
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        VehicleID,
        MAX(EngineHealthScore) AS CurrentHealthScore,
        AVG(TirePressuePSI) AS AvgTirePressure,
        SUM(HarshBrakingEvents + HarshAccelerationEvents) AS TotalHarshEvents,
        SUM(IdleTimeMinutes) / NULLIF(SUM(TotalTripTimeMinutes), 0) AS IdlePercentage,
        -- Calculate revenue impact
        (SELECT SUM(p.RetailPrice * wo.QuantityHandled)
         FROM FactWarehouseOperations wo
         INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
         WHERE wo.OrderID IN (
             SELECT DISTINCT OrderID 
             FROM FactWarehouseOperations 
             WHERE OperationType = 'Shipping'
         )) AS TypicalCargoValue,
        -- Priority score
        (100 - MAX(EngineHealthScore)) * 0.4 +
        (40 - AVG(TirePressuePSI)) * 2 +
        (SUM(HarshBrakingEvents + HarshAccelerationEvents) * 0.5) AS TriageScore
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE DateTime >= DATEADD(DAY, -7, GETDATE()))
    GROUP BY VehicleID
    ORDER BY TriageScore DESC
END
GO
```

### Step 3: Configure Power BI Connection

Create a `config.json` (not tracked in git):

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SqlPassword",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "refreshSchedule": {
    "intervalMinutes": 15,
    "enableRealTime": true
  },
  "alerting": {
    "enabled": true,
    "smtpServer": "${SMTP_SERVER}",
    "smtpPort": 587,
    "fromEmail": "${ALERT_FROM_EMAIL}",
    "alertRecipients": ["${ALERT_RECIPIENT_EMAIL}"]
  }
}
```

## Power BI Dashboard Setup

### Import Template

1. Open Power BI Desktop
2. Go to File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit` from the repository
4. Enter SQL Server connection details when prompted
5. Refresh the data model

### Key Visualizations Included

**Warehouse Gravity Zone Heatmap**
```dax
// DAX Measure: Gravity Score by Zone
GravityZoneScore = 
CALCULATE(
    AVERAGE(DimProductGravity[GravityScore]),
    FILTER(
        FactWarehouseOperations,
        FactWarehouseOperations[OperationType] = "Picking"
    )
)

// DAX Measure: Optimal vs Current Placement
PlacementEfficiency = 
VAR CurrentZone = MAX(FactWarehouseOperations[ZoneAssignment])
VAR RecommendedZone = RELATED(DimProductGravity[RecommendedZone])
RETURN
IF(CurrentZone = RecommendedZone, 1, 0)
```

**Fleet Idle Time vs Cargo Value**
```dax
// DAX Measure: Idle Cost Impact
IdleCostImpact = 
VAR IdleMinutes = SUM(FactFleetTrips[IdleTimeMinutes])
VAR FuelCostPerIdleHour = 2.5 // $/hour average
VAR CargoValue = 
    CALCULATE(
        SUMX(
            FactWarehouseOperations,
            FactWarehouseOperations[QuantityHandled] * RELATED(DimProductGravity[RetailPrice])
        ),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
RETURN
(IdleMinutes / 60) * FuelCostPerIdleHour * (CargoValue / 10000)
```

**Cross-Fact Dwell Time Analysis**
```dax
// DAX Measure: Dwell Time Impact on Delivery
DwellTimeDelayCorrelation = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgTripDelay = AVERAGE(FactFleetTrips[TrafficDelayMinutes])
RETURN
DIVIDE(AvgDwellTime, AvgTripDelay, 0)
```

### Row-Level Security Setup

```dax
// Create role: RegionalManager
[Region] = USERNAME()

// Create role: WarehouseOperator
[GeographyKey] IN (
    LOOKUPVALUE(
        UserWarehouseAccess[GeographyKey],
        UserWarehouseAccess[UserEmail],
        USERNAME()
    )
)
```

## Key SQL Queries & Patterns

### Cross-Fact Analysis: Dwell Time vs Fleet Idle

```sql
-- Find shipments with high dwell time that caused fleet delays
SELECT 
    wo.OrderID,
    p.SKU,
    p.ProductName,
    wo.DwellTimeHours,
    ft.VehicleID,
    ft.IdleTimeMinutes,
    ft.TrafficDelayMinutes,
    g_origin.NodeName AS WarehouseName,
    g_dest.NodeName AS DeliveryLocation,
    t.DateTime AS ShipmentDateTime
FROM FactWarehouseOperations wo
INNER JOIN FactFleetTrips ft ON wo.OrderID = ft.TripID
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimGeography g_origin ON wo.GeographyKey = g_origin.GeographyKey
INNER JOIN DimGeography g_dest ON ft.DestinationGeographyKey = g_dest.GeographyKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE wo.DwellTimeHours > 72
    AND ft.IdleTimeMinutes > (ft.TotalTripTimeMinutes * 0.15)
    AND t.DateTime >= DATEADD(QUARTER, -1, GETDATE())
ORDER BY wo.DwellTimeHours DESC, ft.IdleTimeMinutes DESC
```

### Warehouse Zone Optimization Recommendation

```sql
-- Identify products in wrong gravity zones
WITH ZoneMismatch AS (
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.RecommendedZone,
        wo.ZoneAssignment AS CurrentZone,
        COUNT(*) AS PickEvents,
        AVG(wo.CycleTimeMinutes) AS AvgCycleTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationType = 'Picking'
        AND wo.ZoneAssignment != p.RecommendedZone
        AND wo.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE DateTime >= DATEADD(MONTH, -1, GETDATE()))
    GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone, wo.ZoneAssignment
)
SELECT 
    SKU,
    ProductName,
    CurrentZone,
    RecommendedZone,
    PickEvents,
    AvgCycleTime,
    -- Estimated time savings
    PickEvents * (AvgCycleTime * 0.25) AS EstimatedMinutesSaved
FROM ZoneMismatch
WHERE PickEvents > 10
ORDER BY EstimatedMinutesSaved DESC
```

### Predictive Bottleneck Detection

```sql
-- Time-phased congestion prediction
WITH HourlyVolume AS (
    SELECT 
        t.HourOfDay,
        t.DayOfWeek,
        COUNT(*) AS OperationCount,
        AVG(CycleTimeMinutes) AS AvgCycleTime,
        STDEV(CycleTimeMinutes) AS CycleTimeStdDev
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.HourOfDay, t.DayOfWeek
),
CurrentLoad AS (
    SELECT 
        t.HourOfDay,
        t.DayOfWeek,
        COUNT(*) AS CurrentOps
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
    GROUP BY t.HourOfDay, t.DayOfWeek
)
SELECT 
    hv.HourOfDay,
    hv.DayOfWeek,
    hv.OperationCount AS HistoricalAvg,
    cl.CurrentOps,
    -- Congestion probability
    CASE 
        WHEN cl.CurrentOps > (hv.OperationCount + (2 * hv.CycleTimeStdDev)) THEN 'High Risk'
        WHEN cl.CurrentOps > (hv.OperationCount + hv.CycleTimeStdDev) THEN 'Medium Risk'
        ELSE 'Normal'
    END AS CongestionRisk
FROM HourlyVolume hv
LEFT JOIN CurrentLoad cl ON hv.HourOfDay = cl.HourOfDay AND hv.DayOfWeek = cl.DayOfWeek
WHERE cl.CurrentOps IS NOT NULL
ORDER BY 
    CASE 
        WHEN cl.CurrentOps > (hv.OperationCount + (2 * hv.CycleTimeStdDev)) THEN 1
        WHEN cl.CurrentOps > (hv.OperationCount + hv.CycleTimeStdDev) THEN 2
        ELSE 3
    END
```

## Alerting Configuration

### Create Alert Stored Procedure

```sql
CREATE PROCEDURE sp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertTable TABLE (
        AlertType VARCHAR(50),
        Severity VARCHAR(20),
        Message NVARCHAR(500),
        MetricValue DECIMAL(18,2)
    )
    
    -- Check fleet idle time threshold
    INSERT INTO @AlertTable
    SELECT 
        'Fleet Idle Time Exceeded' AS AlertType,
        'High' AS Severity,
        'Vehicle ' + VehicleID + ' idle time: ' + CAST(IdlePercentage AS VARCHAR(10)) + '%' AS Message,
        IdlePercentage AS MetricValue
    FROM (
        SELECT 
            VehicleID,
            (SUM(IdleTimeMinutes) / NULLIF(SUM(TotalTripTimeMinutes), 0)) * 100 AS IdlePercentage
        FROM FactFleetTrips
        WHERE TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE DateTime >= DATEADD(DAY, -1, GETDATE()))
        GROUP BY VehicleID
    ) AS FleetIdle
    WHERE IdlePercentage > 15
    
    -- Check warehouse dwell time threshold
    INSERT INTO @AlertTable
    SELECT 
        'Excessive Dwell Time' AS AlertType,
        'Medium' AS Severity,
        'SKU ' + p.SKU + ' dwell time: ' + CAST(wo.DwellTimeHours AS VARCHAR(10)) + ' hours' AS Message,
        wo.DwellTimeHours AS MetricValue
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.DwellTimeHours > 72
        AND wo.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE DateTime >= DATEADD(HOUR, -1, GETDATE()))
    
    -- Check perishable product temperature compliance
    INSERT INTO @AlertTable
    SELECT 
        'Temperature Non-Compliance' AS AlertType,
        'Critical' AS Severity,
        'Cross-dock transfer at ' + g.NodeName + ' failed temperature check' AS Message,
        NULL AS MetricValue
    FROM FactCrossDock cd
    INNER JOIN DimGeography g ON cd.GeographyKey = g.GeographyKey
    WHERE cd.TemperatureCompliance = 0
        AND cd.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE DateTime >= DATEADD(MINUTE, -20, GETDATE()))
    
    -- Return alerts
    SELECT * FROM @AlertTable
    ORDER BY 
        CASE Severity
            WHEN 'Critical' THEN 1
            WHEN 'High' THEN 2
            WHEN 'Medium' THEN 3
            ELSE 4
        END
END
GO
```

### Schedule Alert Job (SQL Agent)

```sql
-- Create SQL Server Agent job to run every 15 minutes
USE msdb
GO

EXEC sp_add_job 
    @job_name = 'LogiFleet_KPI_Monitor',
    @enabled = 1,
    @description = 'Monitor logistics KPIs and send alerts'
GO

EXEC sp_add_jobstep
    @job_name = 'LogiFleet_KPI_Monitor',
    @step_name = 'Check Thresholds',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC sp_CheckKPIThresholds',
    @retry_attempts = 3,
    @retry_interval = 5
GO

EXEC sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15
GO

EXEC sp_attach_schedule
    @job_name = 'LogiFleet_KPI_Monitor',
    @schedule_name = 'Every15Minutes'
GO
```

## Integration Patterns

### External API Integration (Weather Data)

```sql
-- Create external table for weather API (requires polybase or linked server)
-- Example using OPENROWSET with REST API

CREATE PROCEDURE sp_EnrichFleetDataWithWeather
AS
BEGIN
    -- Update trips with weather conditions
    UPDATE ft
    SET WeatherCondition = w.Condition
    FROM FactFleetTrips ft
    INNER JOIN (
        -- Call weather API (pseudo-code - implement via SSIS, Azure Data Factory, or stored proc)
        SELECT 
            Latitude,
            Longitude,
            DateTime,
            Condition
        FROM OPENROWSET(
            'ExternalWeatherAPI',
            'Endpoint=${WEATHER_API_ENDPOINT};ApiKey=${WEATHER_API_KEY}'
        ) AS WeatherData
    ) w ON ft.OriginGeographyKey = w.GeographyKey
    WHERE ft.WeatherCondition IS NULL
        AND ft.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE DateTime >= DATEADD(DAY, -7, GETDATE()))
END
GO
```

### Real-Time Streaming Integration

```sql
-- Example: Azure Stream Analytics query to push telemetry into SQL
-- (Configure in Azure Portal, not in SQL directly)

-- Stream Analytics Query:
/*
SELECT
    VehicleID,
    DriverID,
    System.Timestamp AS EventTime,
    Latitude,
    Longitude,
    Speed,
    FuelLevel,
    EngineTemp,
    TirePressure
INTO
    [LogiFleetSQLOutput]
FROM
    [FleetTelemetryInput]
*/

-- Create staging table for streaming data
CREATE TABLE StagingFleetTelemetry (
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    EventTime DATETIME2,
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,1),
    TirePressure DECIMAL(5,1)
)
GO

-- Merge staging data into fact table
CREATE PROCEDURE sp_MergeStreamingTelemetry
AS
BEGIN
    MERGE FactFleetTrips AS target
    USING (
        SELECT 
            t.TimeKey,
            g.GeographyKey AS OriginGeographyKey,
            s.VehicleID,
            s.DriverID,
            AVG(s.Speed) AS AvgSpeed,
            AVG(s.TirePressure) AS AvgTirePressure,
            MAX(s.EngineTemp) AS MaxEngineTemp
        FROM StagingFleetTelemetry s
        INNER JOIN DimTime t ON DATEADD(MINUTE, (DATEPART(MINUTE, s.EventTime) / 15) * 15, 
            DATEADD(HOUR, DATEDIFF(HOUR, 0, s.EventTime), 0)) = t.DateTime
        CROSS APPLY (
            SELECT TOP 1 GeographyKey 
            FROM DimGeography 
            ORDER BY 
                ABS(Latitude - s.Latitude) + ABS(Longitude - s.Longitude)
        ) g
        GROUP BY t.TimeKey, g.GeographyKey, s.VehicleID, s.DriverID
    ) AS source
    ON target.VehicleID = source.VehicleID 
        AND target.TimeKey = source.TimeKey
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, OriginGeographyKey, VehicleID, DriverID, AverageSpeedMPH, TirePressuePSI)
        VALUES (source.TimeKey, source.OriginGeographyKey, source.VehicleID, source.DriverID, 
                source.AvgSpeed, source.AvgTirePressure);
    
    -- Clear staging table
    TRUNCATE TABLE StagingFleetTelemetry;
END
GO
```

## Common Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom:** Dashboard refresh fails with timeout error

**Solution:**
```sql
-- Add table partitioning for large fact tables
ALTER DATABASE LogiFleetPulse SET COMPATIBILITY_LEVEL = 150;
GO

-- Create partition function (monthly partitions)
CREATE
