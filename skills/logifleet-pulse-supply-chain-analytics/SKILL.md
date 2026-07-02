---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing and fleet logistics intelligence platform for supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy SQL Server warehouse schema for fleet management"
  - "implement multi-fact star schema for logistics"
  - "create supply chain KPI dashboards"
  - "build warehouse and fleet analytics data model"
  - "integrate logistics telemetry with Power BI"
  - "optimize warehouse gravity zone analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified Power BI analytics platform. It uses a custom multi-fact star schema on MS SQL Server to enable cross-domain KPI analysis and predictive bottleneck detection.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse operations tracking (putaway, picking, packing, dwell time)
- Fleet telemetry integration (GPS, fuel, idle time, maintenance)
- Cross-dock operations monitoring
- Warehouse gravity zone optimization
- Predictive analytics and anomaly detection
- Role-based Power BI dashboards with real-time refresh

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to warehouse management system (WMS) and fleet telemetry APIs

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the main schema deployment script

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema creation script
-- This creates all fact tables, dimensions, and relationships
:r ./sql/01_create_schema.sql
:r ./sql/02_create_dimensions.sql
:r ./sql/03_create_facts.sql
:r ./sql/04_create_indexes.sql
:r ./sql/05_create_views.sql
:r ./sql/06_create_procedures.sql
```

### Step 3: Configure Data Sources

Create a configuration file for your environment:

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SQL",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "apiUrl": "${WMS_API_URL}",
      "apiKey": "${WMS_API_KEY}",
      "refreshInterval": 15
    },
    "fleetTelemetry": {
      "apiUrl": "${FLEET_API_URL}",
      "apiKey": "${FLEET_API_KEY}",
      "refreshInterval": 5
    },
    "weatherApi": {
      "apiUrl": "${WEATHER_API_URL}",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshInterval": 60
    }
  }
}
```

## Core Schema Components

### Fact Tables

#### FactWarehouseOperations

```sql
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    PickRate DECIMAL(10,2),
    AccuracyPercent DECIMAL(5,2),
    EmployeeKey INT,
    CONSTRAINT FK_FactWH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWH_Date FOREIGN KEY (DateKey) REFERENCES DimDate(DateKey),
    CONSTRAINT FK_FactWH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_FactWH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Clustered columnstore for analytical queries
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations 
ON dbo.FactWarehouseOperations;

-- Nonclustered indexes for frequent filters
CREATE NONCLUSTERED INDEX IX_FactWH_TimeDate 
ON dbo.FactWarehouseOperations(TimeKey, DateKey);

CREATE NONCLUSTERED INDEX IX_FactWH_Product 
ON dbo.FactWarehouseOperations(ProductKey, OperationType);
```

#### FactFleetTrips

```sql
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceMiles DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedGallons DECIMAL(10,2),
    LoadWeightLbs INT,
    OnTimeFlag BIT,
    DelayReasonKey INT,
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Date FOREIGN KEY (DateKey) REFERENCES DimDate(DateKey),
    CONSTRAINT FK_FactFleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON dbo.FactFleetTrips;

CREATE NONCLUSTERED INDEX IX_FactFleet_Route 
ON dbo.FactFleetTrips(RouteKey, DateKey);
```

### Dimension Tables

#### DimTime (15-minute granularity)

```sql
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    Time24 TIME NOT NULL,
    Time12 VARCHAR(10) NOT NULL,
    Hour24 INT NOT NULL,
    Hour12 INT NOT NULL,
    Minute INT NOT NULL,
    QuarterHour INT NOT NULL, -- 1-4
    ShiftCode VARCHAR(10), -- 'Morning', 'Day', 'Night'
    IsBusinessHour BIT
);

-- Populate 15-minute time intervals
DECLARE @StartTime TIME = '00:00';
DECLARE @Counter INT = 0;

WHILE @Counter < 96 -- 24 hours * 4 quarters
BEGIN
    INSERT INTO dbo.DimTime (
        TimeKey, 
        Time24, 
        Hour24, 
        Minute, 
        QuarterHour,
        IsBusinessHour
    )
    VALUES (
        @Counter,
        DATEADD(MINUTE, @Counter * 15, @StartTime),
        DATEPART(HOUR, DATEADD(MINUTE, @Counter * 15, @StartTime)),
        DATEPART(MINUTE, DATEADD(MINUTE, @Counter * 15, @StartTime)),
        (@Counter % 4) + 1,
        CASE WHEN DATEPART(HOUR, DATEADD(MINUTE, @Counter * 15, @StartTime)) BETWEEN 8 AND 17 
             THEN 1 ELSE 0 END
    );
    
    SET @Counter = @Counter + 1;
END;
```

#### DimProductGravity (Warehouse Gravity Zones)

```sql
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    GravityScore DECIMAL(10,4), -- Composite score: velocity + value + fragility
    OptimalZone VARCHAR(50),
    ReplenishmentLeadTimeDays INT,
    UnitValue DECIMAL(10,2),
    FragilityIndex DECIMAL(5,2),
    LastReoptimizedDate DATE
);

-- Stored procedure to calculate gravity scores
CREATE PROCEDURE dbo.CalculateProductGravity
AS
BEGIN
    UPDATE dbo.DimProductGravity
    SET GravityScore = (
        -- Velocity component (40% weight)
        (CASE VelocityClass
            WHEN 'Fast' THEN 100
            WHEN 'Medium' THEN 60
            WHEN 'Slow' THEN 20
            ELSE 0
        END * 0.4) +
        
        -- Value component (40% weight)
        (CASE 
            WHEN UnitValue >= 1000 THEN 100
            WHEN UnitValue >= 500 THEN 80
            WHEN UnitValue >= 100 THEN 60
            WHEN UnitValue >= 50 THEN 40
            ELSE 20
        END * 0.4) +
        
        -- Fragility component (20% weight)
        (FragilityIndex * 20)
    ),
    OptimalZone = CASE
        WHEN GravityScore >= 80 THEN 'HighGravity_NearDock'
        WHEN GravityScore >= 60 THEN 'MediumGravity_MidZone'
        WHEN GravityScore >= 40 THEN 'LowGravity_DeepStorage'
        ELSE 'BulkZone_Overflow'
    END,
    LastReoptimizedDate = GETDATE();
END;
```

## Key Stored Procedures

### Incremental ETL for Warehouse Operations

```sql
CREATE PROCEDURE dbo.LoadWarehouseOperations
    @LastLoadTimestamp DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operation records
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey,
        DateKey,
        WarehouseKey,
        ProductKey,
        OperationType,
        DwellTimeMinutes,
        CycleTimeSeconds,
        PickRate,
        AccuracyPercent,
        EmployeeKey
    )
    SELECT 
        t.TimeKey,
        d.DateKey,
        w.WarehouseKey,
        p.ProductKey,
        src.OperationType,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        DATEDIFF(SECOND, src.StartTime, src.EndTime) AS CycleTimeSeconds,
        src.UnitsProcessed / NULLIF(DATEDIFF(MINUTE, src.StartTime, src.EndTime), 0) AS PickRate,
        src.AccuracyPercent,
        e.EmployeeKey
    FROM StagingWarehouseOps src
    INNER JOIN DimTime t ON t.Time24 = CAST(src.StartTime AS TIME)
    INNER JOIN DimDate d ON d.Date = CAST(src.StartTime AS DATE)
    INNER JOIN DimWarehouse w ON w.WarehouseCode = src.WarehouseCode
    INNER JOIN DimProduct p ON p.SKU = src.SKU
    LEFT JOIN DimEmployee e ON e.EmployeeID = src.EmployeeID
    WHERE src.LoadTimestamp > @LastLoadTimestamp
        AND src.StartTime IS NOT NULL
        AND src.EndTime IS NOT NULL;
    
    -- Update control table
    UPDATE dbo.ETLControl
    SET LastLoadTimestamp = GETDATE(),
        RowsProcessed = @@ROWCOUNT
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Cross-Fact KPI Query

```sql
CREATE VIEW dbo.vw_IntegratedLogisticsKPI
AS
SELECT 
    d.FiscalYear,
    d.FiscalMonth,
    w.WarehouseName,
    p.Category,
    
    -- Warehouse metrics
    AVG(fwh.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    AVG(fwh.PickRate) AS AvgPickRate,
    SUM(CASE WHEN fwh.OperationType = 'Picking' THEN 1 ELSE 0 END) AS TotalPickOperations,
    
    -- Fleet metrics (for products shipped from this warehouse)
    AVG(ffl.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(ffl.FuelConsumedGallons / NULLIF(ffl.DistanceMiles, 0)) AS AvgFuelEfficiency,
    SUM(CASE WHEN ffl.OnTimeFlag = 0 THEN 1 ELSE 0 END) AS DelayedShipments,
    
    -- Cross-domain correlation
    CASE 
        WHEN AVG(fwh.DwellTimeMinutes) > 72 AND AVG(ffl.IdleTimeMinutes) > 30 THEN 'HighRisk'
        WHEN AVG(fwh.DwellTimeMinutes) > 48 OR AVG(ffl.IdleTimeMinutes) > 20 THEN 'MediumRisk'
        ELSE 'LowRisk'
    END AS BottleneckRiskLevel

FROM dbo.FactWarehouseOperations fwh
INNER JOIN dbo.DimDate d ON fwh.DateKey = d.DateKey
INNER JOIN dbo.DimWarehouse w ON fwh.WarehouseKey = w.WarehouseKey
INNER JOIN dbo.DimProduct p ON fwh.ProductKey = p.ProductKey
LEFT JOIN dbo.FactFleetTrips ffl ON 
    ffl.DateKey = fwh.DateKey 
    AND ffl.OriginGeographyKey = w.GeographyKey

GROUP BY 
    d.FiscalYear,
    d.FiscalMonth,
    w.WarehouseName,
    p.Category;
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE dbo.DetectBottleneckRisk
    @ThresholdDwellMinutes INT = 72,
    @ThresholdIdlePercent DECIMAL(5,2) = 15.0
AS
BEGIN
    SELECT 
        w.WarehouseName,
        p.SKU,
        p.ProductName,
        p.OptimalZone AS CurrentZone,
        AVG(fwh.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount,
        
        -- Calculate fleet impact
        (
            SELECT AVG(ffl.IdleTimeMinutes * 100.0 / NULLIF(ffl.DurationMinutes, 0))
            FROM dbo.FactFleetTrips ffl
            INNER JOIN dbo.DimWarehouse w2 ON ffl.OriginGeographyKey = w2.GeographyKey
            WHERE w2.WarehouseKey = w.WarehouseKey
                AND ffl.DateKey >= DATEADD(DAY, -30, CAST(GETDATE() AS DATE))
        ) AS AvgIdlePercent,
        
        CASE 
            WHEN AVG(fwh.DwellTimeMinutes) > @ThresholdDwellMinutes THEN 'IMMEDIATE_ACTION'
            WHEN AVG(fwh.DwellTimeMinutes) > @ThresholdDwellMinutes * 0.8 THEN 'MONITOR_CLOSELY'
            ELSE 'NORMAL'
        END AS AlertLevel,
        
        -- Recommendation
        CASE 
            WHEN AVG(fwh.DwellTimeMinutes) > @ThresholdDwellMinutes 
                 AND p.VelocityClass = 'Fast' 
                 AND p.OptimalZone <> 'HighGravity_NearDock'
            THEN 'Relocate to HighGravity zone near dock'
            
            WHEN AVG(fwh.DwellTimeMinutes) > @ThresholdDwellMinutes
                 AND p.ReplenishmentLeadTimeDays > 7
            THEN 'Increase safety stock or negotiate shorter lead times'
            
            ELSE 'Continue monitoring'
        END AS Recommendation
        
    FROM dbo.FactWarehouseOperations fwh
    INNER JOIN dbo.DimWarehouse w ON fwh.WarehouseKey = w.WarehouseKey
    INNER JOIN dbo.DimProduct p ON fwh.ProductKey = p.ProductKey
    INNER JOIN dbo.DimDate d ON fwh.DateKey = d.DateKey
    WHERE d.Date >= DATEADD(DAY, -90, GETDATE())
    GROUP BY 
        w.WarehouseName,
        w.WarehouseKey,
        p.SKU,
        p.ProductName,
        p.OptimalZone,
        p.VelocityClass,
        p.ReplenishmentLeadTimeDays
    HAVING AVG(fwh.DwellTimeMinutes) > @ThresholdDwellMinutes * 0.7
    ORDER BY AVG(fwh.DwellTimeMinutes) DESC;
END;
```

## Power BI Integration

### Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server details:

```
Server: ${SQL_SERVER_HOST}
Database: LogiFleetPulse
Data Connectivity mode: Import (or DirectQuery for real-time)
```

### Create Composite Model Relationships

```dax
// In Power BI, create relationships between fact and dimension tables
// This is done in the Model view

// Warehouse Operations to Product (Many-to-One)
FactWarehouseOperations[ProductKey] → DimProduct[ProductKey]

// Warehouse Operations to Time (Many-to-One)
FactWarehouseOperations[TimeKey] → DimTime[TimeKey]

// Fleet Trips to Date (Many-to-One)
FactFleetTrips[DateKey] → DimDate[DateKey]

// Create role-playing dimensions for Origin/Destination
FactFleetTrips[OriginGeographyKey] → DimGeography[GeographyKey]
FactFleetTrips[DestinationGeographyKey] → DimGeography[GeographyKey] (inactive)
```

### Key DAX Measures

```dax
// Average Dwell Time with conditional formatting
Avg Dwell Time = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    FactWarehouseOperations[OperationType] IN {"Putaway", "Picking"}
)

// Fleet Idle Time Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Cross-Fact: Dwell-to-Delivery Impact
Dwell Impact Score = 
VAR AvgDwell = [Avg Dwell Time]
VAR AvgIdle = [Fleet Idle %]
RETURN
    SWITCH(
        TRUE(),
        AvgDwell > 72 && AvgIdle > 15, "Critical",
        AvgDwell > 48 || AvgIdle > 10, "Warning",
        "Normal"
    )

// Time Intelligence: Dwell Time YoY Growth
Dwell Time YoY = 
VAR CurrentPeriod = [Avg Dwell Time]
VAR PriorPeriod = 
    CALCULATE(
        [Avg Dwell Time],
        DATEADD(DimDate[Date], -1, YEAR)
    )
RETURN
    DIVIDE(CurrentPeriod - PriorPeriod, PriorPeriod, 0)

// Warehouse Gravity Zone Efficiency
Zone Efficiency = 
DIVIDE(
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[CycleTimeSeconds] < 300
    ),
    COUNT(FactWarehouseOperations[OperationKey]),
    0
) * 100
```

### Power BI Alert Configuration

Create alerts using Power BI Service:

```dax
// Create a measure for alert thresholds
Bottleneck Alert = 
IF(
    [Avg Dwell Time] > 72 || [Fleet Idle %] > 15,
    "ALERT: Potential Bottleneck Detected",
    BLANK()
)
```

Set up data-driven alerts in Power BI Service:
1. Publish report to workspace
2. Navigate to dashboard tile
3. Select "Set alert" → Configure threshold → Choose notification channel

## Common Patterns

### Pattern 1: Incremental Data Refresh

```sql
-- Schedule this as a SQL Agent job every 15 minutes
DECLARE @LastLoad DATETIME;

SELECT @LastLoad = LastLoadTimestamp
FROM dbo.ETLControl
WHERE TableName = 'FactWarehouseOperations';

EXEC dbo.LoadWarehouseOperations @LastLoadTimestamp = @LastLoad;

-- Refresh Power BI dataset via REST API
-- Use Power Automate or Azure Logic App to trigger refresh
```

### Pattern 2: Real-Time Dashboard with DirectQuery

Configure Power BI for DirectQuery mode:

```sql
-- Create indexed view for optimal DirectQuery performance
CREATE VIEW dbo.vw_RealtimeFleetStatus
WITH SCHEMABINDING
AS
SELECT 
    ffl.VehicleKey,
    v.VehicleID,
    v.VehicleType,
    ffl.DateKey,
    MAX(ffl.TimeKey) AS LatestTimeKey,
    SUM(ffl.IdleTimeMinutes) AS TotalIdleTime,
    SUM(ffl.DurationMinutes) AS TotalDuration,
    AVG(ffl.FuelConsumedGallons) AS AvgFuelConsumed
FROM dbo.FactFleetTrips ffl
INNER JOIN dbo.DimVehicle v ON ffl.VehicleKey = v.VehicleKey
GROUP BY ffl.VehicleKey, v.VehicleID, v.VehicleType, ffl.DateKey;

-- Create clustered index for DirectQuery optimization
CREATE UNIQUE CLUSTERED INDEX UCI_RealtimeFleet 
ON dbo.vw_RealtimeFleetStatus(VehicleKey, DateKey);
```

### Pattern 3: Automated Gravity Zone Reoptimization

```sql
-- Weekly job to recalculate product gravity scores
CREATE PROCEDURE dbo.WeeklyGravityReoptimization
AS
BEGIN
    -- Step 1: Update velocity classes based on recent activity
    UPDATE p
    SET VelocityClass = CASE
        WHEN PickFrequency >= 50 THEN 'Fast'
        WHEN PickFrequency >= 20 THEN 'Medium'
        ELSE 'Slow'
    END
    FROM dbo.DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) AS PickFrequency
        FROM dbo.FactWarehouseOperations
        WHERE DateKey >= DATEADD(DAY, -30, CAST(GETDATE() AS DATE))
            AND OperationType = 'Picking'
        GROUP BY ProductKey
    ) recent ON p.ProductKey = recent.ProductKey;
    
    -- Step 2: Recalculate gravity scores
    EXEC dbo.CalculateProductGravity;
    
    -- Step 3: Generate relocation recommendations
    INSERT INTO dbo.GravityZoneRecommendations (
        ProductKey,
        CurrentZone,
        RecommendedZone,
        PriorityScore,
        RecommendationDate
    )
    SELECT 
        p.ProductKey,
        wl.CurrentZone,
        p.OptimalZone,
        p.GravityScore,
        GETDATE()
    FROM dbo.DimProductGravity p
    INNER JOIN dbo.WarehouseLocations wl ON p.ProductKey = wl.ProductKey
    WHERE p.OptimalZone <> wl.CurrentZone
        AND p.GravityScore >= 60; -- Only high-priority relocations
END;
```

## Configuration Files

### Row-Level Security Setup

```sql
-- Create security roles for different user types
CREATE ROLE WarehouseOperator;
CREATE ROLE FleetManager;
CREATE ROLE Executive;

-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    FROM dbo.DimWarehouse w
    INNER JOIN dbo.UserWarehouseAccess ua ON w.WarehouseKey = ua.WarehouseKey
    WHERE w.WarehouseKey = @WarehouseKey
        AND ua.UserName = USER_NAME()
);

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey) 
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey) 
ON dbo.DimWarehouse
WITH (STATE = ON);
```

### Power BI Parameters

Create parameters in Power BI for dynamic filtering:

```dax
// Create parameter table
FiscalYearParameter = 
GENERATESERIES(2024, YEAR(TODAY()) + 1, 1)

// Use in measures
Filtered Operations = 
CALCULATE(
    COUNT(FactWarehouseOperations[OperationKey]),
    DimDate[FiscalYear] = SELECTEDVALUE(FiscalYearParameter[Value])
)
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom:** Dataset refresh fails with timeout error

**Solution:**
```sql
-- Partition large fact tables by date
CREATE PARTITION FUNCTION pf_DateRange (DATE)
AS RANGE RIGHT FOR VALUES 
('2025-01-01', '2025-02-01', '2025-03-01', '2025-04-01');

CREATE PARTITION SCHEME ps_DateRange
AS PARTITION pf_DateRange
ALL TO ([PRIMARY]);

-- Recreate table on partition scheme
CREATE TABLE dbo.FactWarehouseOperations_Partitioned (
    -- Same columns as original
) ON ps_DateRange(DateKey);

-- In Power BI, use incremental refresh:
-- Table Tools → Incremental refresh → Configure policy
```

### Issue: Slow Cross-Fact Queries

**Symptom:** Dashboards load slowly when combining warehouse and fleet data

**Solution:**
```sql
-- Create materialized aggregate table
CREATE TABLE dbo.AggregateWarehouseFleetMetrics (
    DateKey INT,
    WarehouseKey INT,
    ProductKey INT,
    AvgDwellTime DECIMAL(10,2),
    AvgIdleTime DECIMAL(10,2),
    TotalOperations INT,
    TotalTrips INT,
    LastUpdated DATETIME
);

-- Scheduled refresh procedure
CREATE PROCEDURE dbo.RefreshAggregateMetrics
AS
BEGIN
    TRUNCATE TABLE dbo.AggregateWarehouseFleetMetrics;
    
    INSERT INTO dbo.AggregateWarehouseFleetMetrics
    SELECT 
        d.DateKey,
        w.WarehouseKey,
        p.ProductKey,
        AVG(fwh.DwellTimeMinutes) AS AvgDwellTime,
        AVG(ffl.IdleTimeMinutes) AS AvgIdleTime,
        COUNT(DISTINCT fwh.OperationKey) AS TotalOperations,
        COUNT(DISTINCT ffl.TripKey) AS TotalTrips,
        GETDATE() AS LastUpdated
    FROM dbo.FactWarehouseOperations fwh
    INNER JOIN dbo.DimDate d ON fwh.DateKey = d.DateKey
    INNER JOIN dbo.DimWarehouse w ON fwh.WarehouseKey = w.WarehouseKey
    INNER JOIN dbo.DimProduct p ON fwh.ProductKey = p.ProductKey
    LEFT JOIN dbo.FactFleetTrips ffl ON 
        ffl.DateKey = fwh.DateKey 
        AND ffl.OriginGeographyKey = w.GeographyKey
    GROUP BY d.DateKey, w.WarehouseKey, p.ProductKey;
END;

-- Point Power BI to aggregate table instead of joining fact tables
```

### Issue: Gravity Zone Recommendations Not Accurate

**Symptom:** Products assigned to wrong zones, not matching operational patterns

**Solution:**
```sql
-- Add moving average to smooth velocity calculations
CREATE PROCEDURE dbo.CalculateProductGravity_Enhanced
AS
BEGIN
    WITH VelocityTrend AS (
        SELECT 
            ProductKey,
            AVG(DailyPickCount) AS AvgDailyPicks,
            STDEV(DailyPickCount) AS PickVariability
        FROM (
            SELECT 
                ProductKey,
                DateKey,
                COUNT(*) AS DailyPickCount
            FROM dbo.FactWarehouseOperations
            WHERE OperationType = 'Picking'
                AND DateKey >= DATEADD(DAY, -90, CAST(GETDATE() AS DATE))
            GROUP BY ProductKey, DateKey
        ) daily
        GROUP BY ProductKey
    )
    UPDATE p
    SET 
        VelocityClass = CASE
            WHEN vt.AvgDailyPicks >= 50 THEN 'Fast'
            WHEN vt.AvgDailyPicks >= 20 THEN 'Medium'
            WHEN vt.AvgDailyPicks >= 5 THEN 'Slow'
            ELSE 'VeryLow'
        END,
        GravityScore = (
            -- Enhanced scoring with volatility penalty
            (CASE 
                WHEN vt.AvgDailyPicks >= 50 THEN 100
                WHEN vt.AvgDailyPicks >= 20 THEN 70
                WHEN vt.AvgDailyPicks >= 5 THEN 40
                ELSE 10
            END * 0.4) +
            (CASE 
                WHEN p.UnitValue >= 1000 THEN 100
                WHEN p.UnitValue >= 500 THEN 80
                WHEN p.UnitValue >= 100 THEN 60
                ELSE 30
            END * 
