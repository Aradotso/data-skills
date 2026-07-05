---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics intelligence with multi-fact star schema and real-time fleet optimization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logistics data warehouse with Power BI"
  - "create multi-fact star schema for warehouse and fleet data"
  - "configure LogiCore Analytics supply chain dashboard"
  - "implement warehouse gravity zones and fleet telemetry"
  - "build logistics intelligence engine with SQL Server"
  - "integrate WMS and TMS data into Power BI"
  - "set up predictive bottleneck detection for logistics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer. Built on MS SQL Server with Power BI visualization, it uses a custom multi-fact star schema with time-phased dimensions to enable cross-fact KPI analysis.

**Key capabilities:**
- Multi-fact star schema linking warehouse, fleet, and supplier data
- Real-time dashboards with 15-minute refresh intervals
- Warehouse Gravity Zones™ for spatial optimization
- Adaptive Fleet Triage Engine for predictive maintenance
- Temporal elasticity modeling for scenario analysis
- Cross-fact KPI harmonization (e.g., inventory turnover vs. fuel burn rates)

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to source systems (WMS, TMS, telemetry APIs)
- SQL Server Management Studio (SSMS) for deployment

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema deployment script
-- (assumes schema.sql exists in the repository)
:r schema.sql
GO
```

3. **Configure data source connections:**
```json
// config.json (based on config_sample.json)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "windows" // or "sql"
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}"
    },
    "weather_api": {
      "endpoint": "${WEATHER_API_ENDPOINT}",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_interval_minutes": 15
}
```

### Power BI Template Setup

1. **Open the template:**
```powershell
# Open Power BI Desktop
# File > Open > LogiFleet_Pulse_Master.pbit
```

2. **Configure data source parameters:**
- Server: Your SQL Server instance name
- Database: LogiFleetPulse
- Authentication: Windows or SQL Server credentials

## Core Data Model

### Star Schema Architecture

The data model consists of multiple fact tables connected through shared dimensions:

**Fact Tables:**
- `FactWarehouseOperations` - putaway, pick, pack, ship events
- `FactFleetTrips` - route segments, fuel, idle time
- `FactCrossDock` - cross-dock transfers
- `FactInventory` - stock levels, aging, turnover

**Dimension Tables:**
- `DimTime` - 15-minute granularity with fiscal periods
- `DimGeography` - hierarchical location data
- `DimProduct` - product hierarchy with gravity scores
- `DimSupplier` - supplier reliability metrics
- `DimVehicle` - fleet asset details
- `DimWarehouseZone` - storage zones with gravity attributes

### Creating Fact Tables

```sql
-- Example: FactWarehouseOperations
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseZoneKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'PUTAWAY','PICK','PACK','SHIP'
    Quantity INT NOT NULL,
    DwellTimeMinutes INT NULL,
    CycleTimeSeconds INT NULL,
    AssignedGravityScore DECIMAL(5,2) NULL,
    CreatedUTC DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (WarehouseZoneKey) REFERENCES DimWarehouseZone(ZoneKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Clustered columnstore index for analytical queries
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations 
ON FactWarehouseOperations;

-- Nonclustered index for time-based filtering
CREATE NONCLUSTERED INDEX IX_FactWarehouseOperations_TimeKey 
ON FactWarehouseOperations(TimeKey) 
INCLUDE (OperationType, Quantity);
```

```sql
-- Example: FactFleetTrips
CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKM DECIMAL(10,2) NOT NULL,
    FuelLiters DECIMAL(10,2) NULL,
    IdleTimeMinutes INT NULL,
    LoadWeightKG DECIMAL(10,2) NULL,
    AverageSpeedKPH DECIMAL(5,2) NULL,
    MaintenanceAlertFlag BIT DEFAULT 0,
    CreatedUTC DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON FactFleetTrips;
```

### Creating Dimension Tables

```sql
-- DimTime with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfYear INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    Hour INT NOT NULL,
    Minute15Bucket INT NOT NULL, -- 0,15,30,45
    IsWeekend BIT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL
);

-- Populate DimTime
WITH TimeGenerator AS (
    SELECT CAST('2024-01-01 00:00:00' AS DATETIME2) AS TimeValue
    UNION ALL
    SELECT DATEADD(MINUTE, 15, TimeValue)
    FROM TimeGenerator
    WHERE TimeValue < '2027-12-31 23:45:00'
)
INSERT INTO DimTime (
    TimeKey, FullDateTime, Year, Quarter, Month, Week,
    DayOfYear, DayOfMonth, DayOfWeek, DayName, Hour, Minute15Bucket,
    IsWeekend, FiscalYear, FiscalQuarter
)
SELECT 
    CAST(FORMAT(TimeValue, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    TimeValue,
    YEAR(TimeValue),
    DATEPART(QUARTER, TimeValue),
    MONTH(TimeValue),
    DATEPART(WEEK, TimeValue),
    DATEPART(DAYOFYEAR, TimeValue),
    DAY(TimeValue),
    DATEPART(WEEKDAY, TimeValue),
    DATENAME(WEEKDAY, TimeValue),
    DATEPART(HOUR, TimeValue),
    (DATEPART(MINUTE, TimeValue) / 15) * 15,
    CASE WHEN DATEPART(WEEKDAY, TimeValue) IN (1,7) THEN 1 ELSE 0 END,
    CASE WHEN MONTH(TimeValue) >= 7 THEN YEAR(TimeValue) + 1 ELSE YEAR(TimeValue) END,
    CASE WHEN MONTH(TimeValue) >= 7 THEN DATEPART(QUARTER, TimeValue) - 2 
         ELSE DATEPART(QUARTER, TimeValue) + 2 END
FROM TimeGenerator
OPTION (MAXRECURSION 0);
```

```sql
-- DimProduct with Gravity Scores
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100) NOT NULL,
    SubCategory NVARCHAR(100) NULL,
    UnitCost DECIMAL(10,2) NOT NULL,
    UnitWeight DECIMAL(10,3) NOT NULL,
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    GravityScore DECIMAL(5,2) NULL, -- Calculated: velocity + value + fragility
    CurrentGravityZone VARCHAR(20) NULL, -- 'HIGH','MEDIUM','LOW'
    LastGravityUpdateUTC DATETIME2 NULL
);

-- Stored procedure to calculate and update gravity scores
CREATE PROCEDURE UpdateProductGravityScores
AS
BEGIN
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(wo.OperationID) AS PickFrequency,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
            p.UnitCost,
            p.IsFragile,
            p.IsPerishable
        FROM DimProduct p
        LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        WHERE wo.OperationType = 'PICK'
          AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY p.ProductKey, p.UnitCost, p.IsFragile, p.IsPerishable
    )
    UPDATE p
    SET 
        GravityScore = 
            (CASE WHEN pm.PickFrequency > 100 THEN 40 
                  WHEN pm.PickFrequency > 50 THEN 25 
                  ELSE 10 END) + -- Velocity component
            (CASE WHEN pm.UnitCost > 500 THEN 30 
                  WHEN pm.UnitCost > 100 THEN 20 
                  ELSE 10 END) + -- Value component
            (CASE WHEN pm.IsFragile = 1 THEN 20 ELSE 0 END) + -- Fragility
            (CASE WHEN pm.IsPerishable = 1 THEN 10 ELSE 0 END), -- Perishability
        CurrentGravityZone = 
            CASE WHEN GravityScore >= 70 THEN 'HIGH'
                 WHEN GravityScore >= 40 THEN 'MEDIUM'
                 ELSE 'LOW' END,
        LastGravityUpdateUTC = GETUTCDATE()
    FROM DimProduct p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
GO
```

## Data Ingestion Patterns

### Incremental Loading from WMS API

```sql
-- Stored procedure for incremental warehouse operations loading
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTimeUTC DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table (populated via API)
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseZoneKey, ProductKey, OperationType,
        Quantity, DwellTimeMinutes, CycleTimeSeconds, AssignedGravityScore
    )
    SELECT 
        CAST(FORMAT(stg.OperationTimestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        z.ZoneKey,
        p.ProductKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTimestamp, stg.EndTimestamp) AS DwellTimeMinutes,
        DATEDIFF(SECOND, stg.StartTimestamp, stg.EndTimestamp) AS CycleTimeSeconds,
        p.GravityScore
    FROM StagingWarehouseOperations stg
    INNER JOIN DimProduct p ON stg.SKU = p.SKU
    INNER JOIN DimWarehouseZone z ON stg.ZoneName = z.ZoneName
    WHERE stg.OperationTimestamp > @LastLoadTimeUTC
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations wo
          WHERE wo.ProductKey = p.ProductKey
            AND wo.TimeKey = CAST(FORMAT(stg.OperationTimestamp, 'yyyyMMddHHmm') AS INT)
            AND wo.OperationType = stg.OperationType
      );
    
    -- Update last load timestamp
    UPDATE ETLControl
    SET LastLoadTimeUTC = GETUTCDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Fleet Telemetry Integration

```sql
-- External table for streaming telemetry data (requires PolyBase)
CREATE EXTERNAL DATA SOURCE TelematicsAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${TELEMATICS_BLOB_ENDPOINT}',
    CREDENTIAL = TelematicsCredential
);

-- Process telemetry events into fleet trips
CREATE PROCEDURE ProcessFleetTelemetry
AS
BEGIN
    -- Aggregate telemetry events into trip segments
    WITH TripSegments AS (
        SELECT 
            VehicleID,
            MIN(EventTimestamp) AS TripStart,
            MAX(EventTimestamp) AS TripEnd,
            SUM(FuelConsumed) AS TotalFuel,
            SUM(CASE WHEN Speed = 0 AND EngineOn = 1 THEN DurationSeconds ELSE 0 END) / 60.0 AS IdleMinutes,
            SUM(Distance) AS TotalDistance,
            AVG(CASE WHEN Speed > 0 THEN Speed ELSE NULL END) AS AvgSpeed
        FROM StagingTelemetryEvents
        WHERE ProcessedFlag = 0
        GROUP BY VehicleID, TripID
    )
    INSERT INTO FactFleetTrips (
        StartTimeKey, EndTimeKey, VehicleKey, OriginGeographyKey,
        DestinationGeographyKey, DistanceKM, FuelLiters, IdleTimeMinutes,
        AverageSpeedKPH, MaintenanceAlertFlag
    )
    SELECT 
        CAST(FORMAT(ts.TripStart, 'yyyyMMddHHmm') AS INT),
        CAST(FORMAT(ts.TripEnd, 'yyyyMMddHHmm') AS INT),
        v.VehicleKey,
        origin.GeographyKey,
        dest.GeographyKey,
        ts.TotalDistance,
        ts.TotalFuel,
        ts.IdleMinutes,
        ts.AvgSpeed,
        CASE WHEN ts.IdleMinutes > (DATEDIFF(MINUTE, ts.TripStart, ts.TripEnd) * 0.15) THEN 1 ELSE 0 END
    FROM TripSegments ts
    INNER JOIN DimVehicle v ON ts.VehicleID = v.VehicleID
    LEFT JOIN DimGeography origin ON /* GPS to geography lookup */
    LEFT JOIN DimGeography dest ON /* GPS to geography lookup */;
    
    -- Mark events as processed
    UPDATE StagingTelemetryEvents
    SET ProcessedFlag = 1
    WHERE ProcessedFlag = 0;
END;
GO
```

## Advanced Analytics Queries

### Cross-Fact KPI: Inventory Turnover vs. Fleet Utilization

```sql
-- Correlate slow-moving inventory with underutilized delivery routes
SELECT 
    p.Category,
    p.SubCategory,
    g.Region,
    -- Inventory metrics
    AVG(fi.StockLevel) AS AvgStockLevel,
    AVG(fi.DaysSinceLastMovement) AS AvgAgeDays,
    SUM(wo.Quantity) AS TotalPicks,
    -- Fleet metrics
    COUNT(DISTINCT ft.TripID) AS TotalTrips,
    AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
    AVG(ft.LoadWeightKG) AS AvgLoadWeight,
    -- Cross-fact insight
    CASE 
        WHEN AVG(fi.DaysSinceLastMovement) > 30 AND AVG(ft.LoadWeightKG) < 500 
        THEN 'Opportunity: Consolidate slow SKUs into shared routes'
        WHEN AVG(fi.DaysSinceLastMovement) < 7 AND AVG(ft.IdleTimeMinutes) > 30
        THEN 'Opportunity: Increase delivery frequency to reduce warehouse dwell'
        ELSE 'Optimized'
    END AS RecommendedAction
FROM DimProduct p
INNER JOIN FactInventory fi ON p.ProductKey = fi.ProductKey
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
INNER JOIN FactFleetTrips ft ON wo.WarehouseZoneKey = ft.OriginGeographyKey -- Simplified join
INNER JOIN DimGeography g ON ft.DestinationGeographyKey = g.GeographyKey
WHERE fi.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY p.Category, p.SubCategory, g.Region
HAVING AVG(fi.DaysSinceLastMovement) > 14 OR AVG(ft.IdleTimeMinutes) > 20
ORDER BY AvgAgeDays DESC;
```

### Warehouse Gravity Zone Effectiveness

```sql
-- Analyze whether products are in optimal gravity zones
SELECT 
    z.ZoneName,
    z.DistanceFromDockMeters,
    p.CurrentGravityZone,
    COUNT(DISTINCT p.ProductKey) AS SKUCount,
    AVG(wo.CycleTimeSeconds) AS AvgPickSeconds,
    SUM(wo.Quantity) AS TotalPicks,
    -- Misalignment detection
    CASE 
        WHEN p.CurrentGravityZone = 'HIGH' AND z.DistanceFromDockMeters > 50 
        THEN 'MISALIGNED: High-gravity SKU too far from dock'
        WHEN p.CurrentGravityZone = 'LOW' AND z.DistanceFromDockMeters < 20
        THEN 'MISALIGNED: Low-gravity SKU too close to dock'
        ELSE 'ALIGNED'
    END AS ZoneAlignment
FROM DimWarehouseZone z
INNER JOIN FactWarehouseOperations wo ON z.ZoneKey = wo.WarehouseZoneKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'PICK'
  AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY z.ZoneName, z.DistanceFromDockMeters, p.CurrentGravityZone
ORDER BY TotalPicks DESC;
```

### Predictive Bottleneck Index

```sql
-- Calculate bottleneck risk score combining multiple factors
CREATE VIEW vw_BottleneckRiskIndex AS
WITH WarehouseLoad AS (
    SELECT 
        TimeKey,
        WarehouseZoneKey,
        COUNT(*) AS ActiveOperations,
        AVG(CycleTimeSeconds) AS AvgCycleTime
    FROM FactWarehouseOperations
    WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -4, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY TimeKey, WarehouseZoneKey
),
FleetCongestion AS (
    SELECT 
        StartTimeKey,
        DestinationGeographyKey,
        COUNT(*) AS ActiveDeliveries,
        AVG(IdleTimeMinutes) AS AvgIdleTime
    FROM FactFleetTrips
    WHERE StartTimeKey >= CAST(FORMAT(DATEADD(HOUR, -4, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY StartTimeKey, DestinationGeographyKey
)
SELECT 
    t.FullDateTime,
    z.ZoneName,
    g.City,
    wl.ActiveOperations,
    fc.ActiveDeliveries,
    wl.AvgCycleTime,
    fc.AvgIdleTime,
    -- Bottleneck score (0-100)
    (CASE WHEN wl.ActiveOperations > 50 THEN 30 ELSE wl.ActiveOperations * 0.6 END) +
    (CASE WHEN fc.ActiveDeliveries > 20 THEN 30 ELSE fc.ActiveDeliveries * 1.5 END) +
    (CASE WHEN wl.AvgCycleTime > 180 THEN 20 ELSE wl.AvgCycleTime / 9 END) +
    (CASE WHEN fc.AvgIdleTime > 30 THEN 20 ELSE fc.AvgIdleTime * 0.67 END) AS BottleneckScore,
    -- Risk level
    CASE 
        WHEN BottleneckScore >= 70 THEN 'CRITICAL'
        WHEN BottleneckScore >= 50 THEN 'HIGH'
        WHEN BottleneckScore >= 30 THEN 'MODERATE'
        ELSE 'LOW'
    END AS RiskLevel
FROM DimTime t
LEFT JOIN WarehouseLoad wl ON t.TimeKey = wl.TimeKey
LEFT JOIN DimWarehouseZone z ON wl.WarehouseZoneKey = z.ZoneKey
LEFT JOIN FleetCongestion fc ON t.TimeKey = fc.StartTimeKey
LEFT JOIN DimGeography g ON fc.DestinationGeographyKey = g.GeographyKey
WHERE BottleneckScore > 30;
GO
```

## Power BI DAX Measures

### Cross-Fact Measures

```dax
// Inventory Turnover Rate
Inventory Turnover = 
VAR TotalPicks = SUM(FactWarehouseOperations[Quantity])
VAR AvgInventory = AVERAGE(FactInventory[StockLevel])
RETURN
DIVIDE(TotalPicks, AvgInventory, 0)

// Fleet Utilization %
Fleet Utilization % = 
VAR TotalTripTime = SUM(FactFleetTrips[DurationMinutes])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
DIVIDE(TotalTripTime - TotalIdleTime, TotalTripTime, 0) * 100

// Warehouse Efficiency Score
Warehouse Efficiency = 
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeSeconds])
VAR BenchmarkCycleTime = 120 // seconds
VAR DwellPenalty = AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60
RETURN
(BenchmarkCycleTime / AvgCycleTime) * 100 - (DwellPenalty * 5)

// Cross-Fact: Cost per Pick
Cost Per Pick = 
VAR TotalPicks = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[OperationType] = "PICK"
)
VAR TotalFleetCost = SUM(FactFleetTrips[FuelLiters]) * 1.5 // $1.50/liter
VAR TotalWarehouseCost = CALCULATE(
    SUM(FactWarehouseOperations[Quantity]) * 0.25, // $0.25 per unit handling
    FactWarehouseOperations[OperationType] IN {"PUTAWAY", "PICK", "PACK"}
)
RETURN
DIVIDE(TotalFleetCost + TotalWarehouseCost, TotalPicks, 0)
```

### Time Intelligence Measures

```dax
// YoY Growth in Pick Volume
Pick Volume YoY Growth % = 
VAR CurrentPicks = SUM(FactWarehouseOperations[Quantity])
VAR PriorYearPicks = CALCULATE(
    SUM(FactWarehouseOperations[Quantity]),
    DATEADD(DimTime[FullDateTime], -1, YEAR)
)
RETURN
DIVIDE(CurrentPicks - PriorYearPicks, PriorYearPicks, 0) * 100

// Rolling 7-Day Average Fleet Idle Time
Fleet Idle 7D Avg = 
AVERAGEX(
    DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -7, DAY),
    CALCULATE(AVERAGE(FactFleetTrips[IdleTimeMinutes]))
)
```

### Gravity Zone Optimization Measure

```dax
// Gravity Misalignment Count
Gravity Misalignment Count = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FILTER(
        DimProduct,
        (DimProduct[CurrentGravityZone] = "HIGH" && 
         RELATED(DimWarehouseZone[DistanceFromDockMeters]) > 50) ||
        (DimProduct[CurrentGravityZone] = "LOW" && 
         RELATED(DimWarehouseZone[DistanceFromDockMeters]) < 20)
    )
)
```

## Automated Alerting

### SQL Server Agent Job for Threshold Alerts

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName VARCHAR(100) NOT NULL,
    MetricQuery NVARCHAR(MAX) NOT NULL,
    ThresholdValue DECIMAL(10,2) NOT NULL,
    ComparisonOperator VARCHAR(10) NOT NULL, -- '>','<','>=','<='
    RecipientEmails NVARCHAR(500) NOT NULL,
    IsActive BIT DEFAULT 1
);

-- Insert sample thresholds
INSERT INTO AlertThresholds (AlertName, MetricQuery, ThresholdValue, ComparisonOperator, RecipientEmails)
VALUES 
('Fleet Idle Time Excessive', 
 'SELECT AVG(IdleTimeMinutes) FROM FactFleetTrips WHERE StartTimeKey >= CAST(FORMAT(DATEADD(HOUR, -4, GETDATE()), ''yyyyMMddHHmm'') AS INT)',
 20.0, '>', '${LOGISTICS_MANAGER_EMAIL}'),
('Warehouse Dwell Time High',
 'SELECT AVG(DwellTimeMinutes) FROM FactWarehouseOperations WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -2, GETDATE()), ''yyyyMMddHHmm'') AS INT)',
 90.0, '>', '${WAREHOUSE_MANAGER_EMAIL}');

-- Stored procedure to check and send alerts
CREATE PROCEDURE CheckAndSendAlerts
AS
BEGIN
    DECLARE @AlertName VARCHAR(100),
            @MetricQuery NVARCHAR(MAX),
            @ThresholdValue DECIMAL(10,2),
            @ComparisonOperator VARCHAR(10),
            @RecipientEmails NVARCHAR(500),
            @CurrentValue DECIMAL(10,2),
            @AlertTriggered BIT;
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertName, MetricQuery, ThresholdValue, ComparisonOperator, RecipientEmails
    FROM AlertThresholds
    WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @AlertName, @MetricQuery, @ThresholdValue, @ComparisonOperator, @RecipientEmails;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Execute metric query
        EXEC sp_executesql @MetricQuery, N'@Result DECIMAL(10,2) OUTPUT', @CurrentValue OUTPUT;
        
        -- Check threshold
        SET @AlertTriggered = CASE 
            WHEN @ComparisonOperator = '>' AND @CurrentValue > @ThresholdValue THEN 1
            WHEN @ComparisonOperator = '<' AND @CurrentValue < @ThresholdValue THEN 1
            WHEN @ComparisonOperator = '>=' AND @CurrentValue >= @ThresholdValue THEN 1
            WHEN @ComparisonOperator = '<=' AND @CurrentValue <= @ThresholdValue THEN 1
            ELSE 0
        END;
        
        IF @AlertTriggered = 1
        BEGIN
            -- Send email via Database Mail
            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = 'LogiFleetPulse',
                @recipients = @RecipientEmails,
                @subject = @AlertName,
                @body = CONCAT('Alert triggered: ', @AlertName, CHAR(13), CHAR(10),
                              'Current Value: ', @CurrentValue, CHAR(13), CHAR(10),
                              'Threshold: ', @ComparisonOperator, ' ', @ThresholdValue);
        END;
        
        FETCH NEXT FROM alert_cursor INTO @AlertName, @MetricQuery, @ThresholdValue, @ComparisonOperator, @RecipientEmails;
    END;
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END;
GO
```

## Row-Level Security

### Implementing Role-Based Access

```sql
-- Create security roles table
CREATE TABLE SecurityRoles (
    RoleID INT IDENTITY(1,1) PRIMARY KEY,
    RoleName VARCHAR(50) NOT NULL
