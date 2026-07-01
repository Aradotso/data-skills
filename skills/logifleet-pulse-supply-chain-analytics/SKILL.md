---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and cross-dock analytics
triggers:
  - set up logistics data warehouse with SQL Server
  - configure Power BI for supply chain analytics
  - implement multi-fact star schema for warehouse operations
  - build fleet tracking and warehouse management dashboard
  - create logistics KPI dashboard with Power BI
  - deploy LogiFleet Pulse analytics platform
  - integrate warehouse and fleet data for supply chain insights
  - configure real-time logistics intelligence system
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a multi-fact star schema. Built on MS SQL Server with Power BI visualization, it provides real-time cross-modal analytics for warehouse velocity, fleet optimization, and inventory management.

## What It Does

- **Multi-Fact Star Schema**: Combines FactWarehouseOperations, FactFleetTrips, and FactCrossDock with shared time-phased dimensions
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item value, and fragility
- **Fleet Triage Engine**: Proactive maintenance prioritization weighted by revenue impact
- **Cross-Fact KPI Harmonization**: Links inventory turnover with fuel burn rates, dwell time with route efficiency
- **Temporal Elasticity Modeling**: Time-phased scenario simulations for capacity planning
- **Real-Time Dashboarding**: 15-minute refresh intervals with Power BI

## Installation

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version recommended)
- SQL Server Management Studio (SSMS)
- Access to WMS, TMS, or telematics data sources

### Database Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the schema** using SSMS or sqlcmd:
```bash
sqlcmd -S YOUR_SERVER_NAME -d master -i schema/01_create_database.sql
sqlcmd -S YOUR_SERVER_NAME -d LogiFleetPulse -i schema/02_create_dimensions.sql
sqlcmd -S YOUR_SERVER_NAME -d LogiFleetPulse -i schema/03_create_facts.sql
sqlcmd -S YOUR_SERVER_NAME -d LogiFleetPulse -i schema/04_create_views.sql
sqlcmd -S YOUR_SERVER_NAME -d LogiFleetPulse -i schema/05_create_procedures.sql
```

3. **Configure data source connections**:
```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "Windows",
    "connection_string": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;Trusted_Connection=True;"
  },
  "wms_api": {
    "endpoint": "${WMS_API_ENDPOINT}",
    "api_key": "${WMS_API_KEY}"
  },
  "telematics_api": {
    "endpoint": "${TELEMATICS_API_ENDPOINT}",
    "api_key": "${TELEMATICS_API_KEY}"
  }
}
```

## Core Data Model

### Dimension Tables

**DimTime** (15-minute granularity):
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    FifteenMinuteBucket TIME,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayOfMonth TINYINT,
    WeekOfYear TINYINT,
    MonthNumber TINYINT,
    QuarterNumber TINYINT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
);

-- Create index for time-based queries
CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(FullDateTime);
```

**DimProductGravity** (warehouse gravity zones):
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite: velocity + value + fragility
    PickFrequencyScore DECIMAL(5,2),
    ValueScore DECIMAL(5,2),
    FragilityScore DECIMAL(5,2),
    OptimalZone VARCHAR(20),
    CurrentZone VARCHAR(20),
    RecommendedZone VARCHAR(20),
    LastRecalculated DATETIME
);

-- Index for gravity-based queries
CREATE NONCLUSTERED INDEX IX_Product_Gravity ON DimProductGravity(GravityScore DESC, OptimalZone);
```

**DimGeography** (hierarchical location):
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, Route Node, Customer Site
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Timezone VARCHAR(50)
);
```

### Fact Tables

**FactWarehouseOperations**:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    QuantityHandled INT,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    OperatorID VARCHAR(50),
    ZoneFrom VARCHAR(20),
    ZoneTo VARCHAR(20),
    EquipmentID VARCHAR(50),
    BatchNumber VARCHAR(50),
    CostPerUnit DECIMAL(10,4),
    TotalCost AS (QuantityHandled * CostPerUnit) PERSISTED
);

-- Composite index for cross-fact queries
CREATE NONCLUSTERED INDEX IX_Warehouse_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey, OperationType)
INCLUDE (DwellTimeMinutes, CycleTimeSeconds, TotalCost);
```

**FactFleetTrips**:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DriverKey INT NOT NULL FOREIGN KEY REFERENCES DimDriver(DriverKey),
    TripStartTime DATETIME,
    TripEndTime DATETIME,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AverageSpeedKmh DECIMAL(6,2),
    MaxSpeedKmh DECIMAL(6,2),
    FuelEfficiency AS (DistanceKm / NULLIF(FuelConsumedLiters, 0)) PERSISTED,
    TripCost DECIMAL(10,2),
    RevenueGenerated DECIMAL(10,2)
);

-- Index for fleet optimization queries
CREATE NONCLUSTERED INDEX IX_Fleet_Time_Vehicle 
ON FactFleetTrips(TimeKey, VehicleKey)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters, TripCost);
```

**FactCrossDock**:
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    InboundReceiveTime DATETIME,
    OutboundShipTime DATETIME,
    DockDwellMinutes AS DATEDIFF(MINUTE, InboundReceiveTime, OutboundShipTime) PERSISTED,
    QuantityTransferred INT,
    RequiresInspection BIT,
    InspectionTimeMinutes INT
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge data from WMS staging table
    MERGE FactWarehouseOperations AS target
    USING (
        SELECT 
            t.TimeKey,
            p.ProductKey,
            g.GeographyKey AS WarehouseKey,
            wms.OperationType,
            wms.QuantityHandled,
            DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) AS DwellTimeMinutes,
            DATEDIFF(SECOND, wms.StartTime, wms.EndTime) AS CycleTimeSeconds,
            wms.OperatorID,
            wms.ZoneFrom,
            wms.ZoneTo,
            wms.EquipmentID,
            wms.BatchNumber,
            wms.CostPerUnit
        FROM staging.WMSOperations wms
        JOIN DimTime t ON CAST(wms.StartTime AS DATE) = CAST(t.FullDateTime AS DATE)
            AND DATEPART(HOUR, wms.StartTime) = t.HourOfDay
            AND (DATEPART(MINUTE, wms.StartTime) / 15) * 15 = DATEPART(MINUTE, t.FifteenMinuteBucket)
        JOIN DimProductGravity p ON wms.SKU = p.SKU
        JOIN DimGeography g ON wms.WarehouseID = g.LocationID
        WHERE wms.StartTime BETWEEN @StartDate AND @EndDate
    ) AS source
    ON 1 = 0 -- Always insert (append-only fact table)
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, ProductKey, WarehouseKey, OperationType, QuantityHandled, 
                DwellTimeMinutes, CycleTimeSeconds, OperatorID, ZoneFrom, ZoneTo, 
                EquipmentID, BatchNumber, CostPerUnit)
        VALUES (source.TimeKey, source.ProductKey, source.WarehouseKey, source.OperationType,
                source.QuantityHandled, source.DwellTimeMinutes, source.CycleTimeSeconds,
                source.OperatorID, source.ZoneFrom, source.ZoneTo, source.EquipmentID,
                source.BatchNumber, source.CostPerUnit);
                
    -- Update gravity scores based on new operation data
    EXEC usp_RecalculateProductGravity;
END;
GO
```

### Product Gravity Score Calculation

```sql
CREATE PROCEDURE usp_RecalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on 90-day rolling window
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(p.ValueScore) AS AvgValue,
            p.FragilityScore,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTime
        FROM DimProductGravity p
        JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
            AND wo.OperationType = 'Picking'
        GROUP BY p.ProductKey, p.FragilityScore
    ),
    NormalizedScores AS (
        SELECT 
            ProductKey,
            -- Normalize to 0-100 scale
            (PickFrequency - MIN(PickFrequency) OVER()) * 100.0 / 
                NULLIF(MAX(PickFrequency) OVER() - MIN(PickFrequency) OVER(), 0) AS PickScore,
            AvgValue AS ValueScore,
            FragilityScore,
            (AvgDwellTime - MIN(AvgDwellTime) OVER()) * 100.0 / 
                NULLIF(MAX(AvgDwellTime) OVER() - MIN(AvgDwellTime) OVER(), 0) AS DwellScore
        FROM ProductMetrics
    )
    UPDATE p
    SET 
        p.PickFrequencyScore = ns.PickScore,
        p.ValueScore = ns.ValueScore,
        p.FragilityScore = ns.FragilityScore,
        p.GravityScore = (ns.PickScore * 0.4 + ns.ValueScore * 0.3 + ns.FragilityScore * 0.2 - ns.DwellScore * 0.1),
        p.OptimalZone = CASE 
            WHEN (ns.PickScore * 0.4 + ns.ValueScore * 0.3 + ns.FragilityScore * 0.2 - ns.DwellScore * 0.1) >= 75 THEN 'Zone-A-High'
            WHEN (ns.PickScore * 0.4 + ns.ValueScore * 0.3 + ns.FragilityScore * 0.2 - ns.DwellScore * 0.1) >= 50 THEN 'Zone-B-Medium'
            ELSE 'Zone-C-Low'
        END,
        p.LastRecalculated = GETDATE()
    FROM DimProductGravity p
    JOIN NormalizedScores ns ON p.ProductKey = ns.ProductKey;
END;
GO
```

### Cross-Fact KPI Queries

```sql
-- Create view for cross-fact analytics
CREATE VIEW vw_CrossFactKPIs AS
SELECT 
    t.FullDateTime,
    t.DayOfWeek,
    t.FiscalPeriod,
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.CurrentZone,
    p.OptimalZone,
    -- Warehouse metrics
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.QuantityHandled ELSE 0 END) AS TotalPicked,
    AVG(CASE WHEN wo.OperationType = 'Picking' THEN wo.DwellTimeMinutes ELSE NULL END) AS AvgPickDwellTime,
    SUM(wo.TotalCost) AS TotalWarehouseCost,
    -- Fleet metrics (joined via cross-dock or delivery)
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleTime,
    AVG(ft.FuelEfficiency) AS AvgFuelEfficiency,
    SUM(ft.TripCost) AS TotalFleetCost,
    -- Cross-dock metrics
    AVG(cd.DockDwellMinutes) AS AvgCrossDockDwell,
    -- Composite KPIs
    (SUM(wo.TotalCost) + SUM(ft.TripCost)) / NULLIF(SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.QuantityHandled ELSE 0 END), 0) AS CostPerUnitShipped,
    AVG(wo.DwellTimeMinutes) * AVG(ft.IdleTimeMinutes) / 60.0 AS CompositeFrictionIndex
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactCrossDock cd ON wo.ProductKey = cd.ProductKey AND t.TimeKey = cd.TimeKey
LEFT JOIN FactFleetTrips ft ON cd.OutboundTripKey = ft.TripKey
GROUP BY 
    t.FullDateTime, t.DayOfWeek, t.FiscalPeriod,
    p.SKU, p.ProductName, p.GravityScore, p.CurrentZone, p.OptimalZone;
GO
```

## Power BI Configuration

### Connect to SQL Server

1. **Open Power BI Desktop**
2. **Get Data** > **SQL Server**
3. **Server**: `${SQL_SERVER_HOST}`
4. **Database**: `LogiFleetPulse`
5. **Data Connectivity mode**: `DirectQuery` (for real-time) or `Import` (for performance)

### Import Data Model

```powerquery
// M Query for time intelligence
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
    #"Changed Type" = Table.TransformColumnTypes(DimTime,{
        {"FullDateTime", type datetime},
        {"TimeKey", Int64.Type}
    }),
    #"Added Rolling90Days" = Table.AddColumn(#"Changed Type", "Rolling90Days", 
        each [FullDateTime] >= Date.AddDays(DateTime.LocalNow(), -90))
in
    #"Added Rolling90Days"
```

### DAX Measures

**Composite Friction Index**:
```dax
CompositeFrictionIndex = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    (AvgDwellTime * AvgIdleTime) / 60
```

**Product Gravity Drift**:
```dax
GravityDrift = 
CALCULATE(
    COUNTROWS(DimProductGravity),
    DimProductGravity[CurrentZone] <> DimProductGravity[OptimalZone]
) / COUNTROWS(DimProductGravity)
```

**Fleet Utilization Rate**:
```dax
FleetUtilizationRate = 
VAR TotalTripTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(TotalTripTime - TotalIdleTime, TotalTripTime, 0)
```

**Cross-Dock Efficiency**:
```dax
CrossDockEfficiency = 
VAR IdealDwellTime = 45 -- Minutes
VAR ActualAvgDwell = AVERAGE(FactCrossDock[DockDwellMinutes])
RETURN
    DIVIDE(IdealDwellTime, ActualAvgDwell, 1)
```

### Import Power BI Template

```bash
# Download and configure
curl -L https://raw.githubusercontent.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse/main/LogiFleet_Pulse_Master.pbit -o LogiFleet_Pulse.pbit

# Open in Power BI Desktop
# Configure data source connection string when prompted
```

## Automated Alerting

### Create Alert Stored Procedure

```sql
CREATE PROCEDURE usp_CheckLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @AlertSubject NVARCHAR(200);
    
    -- Check 1: Fleet idle time exceeds 15%
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips ft
        JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -24, GETDATE())
        GROUP BY ft.VehicleKey
        HAVING SUM(ft.IdleTimeMinutes) * 1.0 / SUM(ft.TripDurationMinutes) > 0.15
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Fleet Idle Time Exceeds 15%';
        SET @AlertMessage = 'One or more vehicles exceeded 15% idle time in the last 24 hours. Review route optimization.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END;
    
    -- Check 2: Product gravity drift exceeds 20%
    IF (
        SELECT COUNT(*) * 1.0 / (SELECT COUNT(*) FROM DimProductGravity)
        FROM DimProductGravity
        WHERE CurrentZone <> OptimalZone
    ) > 0.20
    BEGIN
        SET @AlertSubject = 'ALERT: Product Gravity Drift Exceeds 20%';
        SET @AlertMessage = 'More than 20% of products are in sub-optimal warehouse zones. Consider rearrangement.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END;
    
    -- Check 3: Cross-dock dwell time exceeds 60 minutes
    IF EXISTS (
        SELECT 1
        FROM FactCrossDock cd
        JOIN DimTime t ON cd.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
            AND cd.DockDwellMinutes > 60
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Cross-Dock Dwell Time Exceeds 60 Minutes';
        SET @AlertMessage = 'Recent cross-dock operations are taking longer than expected. Check loading bay congestion.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule alert check every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_Alert_Monitor';

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_Alert_Monitor',
    @step_name = 'Check Alerts',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC usp_CheckLogisticsAlerts;';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_Alert_Monitor',
    @schedule_name = 'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = 'LogiFleet_Alert_Monitor';
```

## Common Patterns

### Pattern 1: Analyze Warehouse Gravity Zone Optimization

```sql
-- Identify products in sub-optimal zones with high impact
SELECT 
    p.SKU,
    p.ProductName,
    p.CurrentZone,
    p.OptimalZone,
    p.GravityScore,
    COUNT(wo.OperationKey) AS TotalOperations,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    SUM(wo.TotalCost) AS TotalCost,
    -- Estimated savings if moved to optimal zone
    SUM(wo.TotalCost) * 0.15 AS EstimatedSavings
FROM DimProductGravity p
JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    AND p.CurrentZone <> p.OptimalZone
    AND p.GravityScore > 60
GROUP BY p.SKU, p.ProductName, p.CurrentZone, p.OptimalZone, p.GravityScore
ORDER BY EstimatedSavings DESC;
```

### Pattern 2: Fleet Route Optimization Analysis

```sql
-- Identify routes with high idle time and low efficiency
SELECT 
    orig.LocationName AS Origin,
    dest.LocationName AS Destination,
    COUNT(ft.TripKey) AS TripCount,
    AVG(ft.DistanceKm) AS AvgDistance,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    AVG(ft.FuelEfficiency) AS AvgFuelEfficiency,
    AVG(ft.IdleTimeMinutes * 1.0 / ft.TripDurationMinutes) AS IdleTimePercent,
    SUM(ft.TripCost) AS TotalCost,
    -- Flag for optimization
    CASE 
        WHEN AVG(ft.IdleTimeMinutes * 1.0 / ft.TripDurationMinutes) > 0.15 THEN 'HIGH_IDLE'
        WHEN AVG(ft.FuelEfficiency) < 8.0 THEN 'LOW_EFFICIENCY'
        ELSE 'NORMAL'
    END AS OptimizationFlag
FROM FactFleetTrips ft
JOIN DimGeography orig ON ft.OriginKey = orig.GeographyKey
JOIN DimGeography dest ON ft.DestinationKey = dest.GeographyKey
JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY orig.LocationName, dest.LocationName
HAVING COUNT(ft.TripKey) >= 10
ORDER BY IdleTimePercent DESC, AvgFuelEfficiency ASC;
```

### Pattern 3: Cross-Dock Performance Monitoring

```sql
-- Analyze cross-dock throughput and bottlenecks
SELECT 
    t.FullDateTime,
    t.HourOfDay,
    COUNT(cd.CrossDockKey) AS CrossDockVolume,
    AVG(cd.DockDwellMinutes) AS AvgDwellMinutes,
    MAX(cd.DockDwellMinutes) AS MaxDwellMinutes,
    SUM(CASE WHEN cd.RequiresInspection = 1 THEN 1 ELSE 0 END) AS InspectionCount,
    AVG(cd.InspectionTimeMinutes) AS AvgInspectionTime,
    -- Bottleneck indicator
    CASE 
        WHEN AVG(cd.DockDwellMinutes) > 60 THEN 'BOTTLENECK'
        WHEN MAX(cd.DockDwellMinutes) > 90 THEN 'OUTLIER_DETECTED'
        ELSE 'NORMAL'
    END AS Status
FROM FactCrossDock cd
JOIN DimTime t ON cd.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -7, GETDATE())
GROUP BY t.FullDateTime, t.HourOfDay
ORDER BY t.FullDateTime DESC;
```

### Pattern 4: Predictive Bottleneck Detection

```sql
-- Predict potential bottlenecks using historical patterns
WITH HourlyMetrics AS (
    SELECT 
        t.HourOfDay,
        t.DayOfWeek,
        AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwell,
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdle,
        COUNT(wo.OperationKey) AS OperationVolume
    FROM DimTime t
    LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
    LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY t.HourOfDay, t.DayOfWeek
),
CurrentMetrics AS (
    SELECT 
        t.HourOfDay,
        t.DayOfWeek,
        AVG(wo.DwellTimeMinutes) AS CurrentWarehouseDwell,
        AVG(ft.IdleTimeMinutes) AS CurrentFleetIdle,
        COUNT(wo.OperationKey) AS CurrentVolume
    FROM DimTime t
    LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
    LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -2, GETDATE())
    GROUP BY t.HourOfDay, t.DayOfWeek
)
SELECT 
    cm.HourOfDay,
    cm.DayOfWeek,
    cm.CurrentWarehouseDwell,
    hm.AvgWarehouseDwell AS HistoricalAvg,
    (cm.CurrentWarehouseDwell - hm.AvgWarehouseDwell) / NULLIF(hm.AvgWarehouseDwell, 0) AS DwellDeviation,
    cm.CurrentFleetIdle,
    hm.AvgFleetIdle AS HistoricalFleetIdle,
    (cm.CurrentFleetIdle - hm.AvgFleetIdle) / NULLIF(hm.AvgFleetIdle, 0) AS IdleDeviation,
    CASE 
        WHEN (cm.CurrentWarehouseDwell - hm.AvgWarehouseDwell) / NULLIF(hm.AvgWarehouseDwell, 0) > 0.25 THEN 'WAREHOUSE_BOTTLENECK_RISK'
        WHEN (cm.CurrentFleetIdle - hm.AvgFleetIdle)
