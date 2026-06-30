---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics analytics platform with multi-fact star schema for warehouse operations, fleet telemetry, and cross-modal supply chain intelligence
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics warehouse and fleet dashboard
  - configure power bi supply chain intelligence
  - create multi-fact star schema for logistics
  - implement warehouse gravity zones analytics
  - build fleet optimization dashboard with sql server
  - integrate warehouse and fleet telemetry data
  - set up cross-modal supply chain reporting
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It provides a multi-fact star schema data warehouse that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for cross-modal supply chain analytics.

**Key capabilities:**
- Multi-fact dimensional modeling (warehouse operations, fleet trips, cross-dock activities)
- Real-time dashboard integration with Power BI
- Predictive bottleneck detection and fleet triage
- Warehouse Gravity Zones™ spatial optimization
- Temporal elasticity modeling for scenario planning
- Role-based access control with row-level security

## Installation & Prerequisites

### Requirements
- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, TMS, telematics APIs

### Repository Setup

```bash
# Clone the repository
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse

# Review the schema files
ls -la schema/
# Expected: create_dimensions.sql, create_facts.sql, create_views.sql, create_procedures.sql
```

### Database Deployment

```sql
-- 1. Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- 2. Deploy dimension tables first
-- Execute: schema/create_dimensions.sql

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeValue DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    TimeOf15Min TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek INT NOT NULL,
    WeekOfYear INT NOT NULL,
    MonthNumber INT NOT NULL,
    QuarterNumber INT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0
);

CREATE UNIQUE NONCLUSTERED INDEX IX_DimTime_DateTime 
ON DimTime(DateTimeValue);

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- 'Warehouse', 'RouteNode', 'Hub'
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    ParentLocationKey INT NULL,
    CONSTRAINT FK_Geography_Parent FOREIGN KEY (ParentLocationKey) 
        REFERENCES DimGeography(GeographyKey)
);

-- Create product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(300) NOT NULL,
    CategoryLevel1 NVARCHAR(100),
    CategoryLevel2 NVARCHAR(100),
    CategoryLevel3 NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT DEFAULT 0,
    RequiresRefrigeration BIT DEFAULT 0,
    GravityScore DECIMAL(5,2) DEFAULT 50.0, -- 0-100 scale
    GravityZone NVARCHAR(50), -- 'High', 'Medium', 'Low'
    LastGravityUpdate DATETIME2
);

CREATE NONCLUSTERED INDEX IX_DimProduct_GravityZone 
ON DimProduct(GravityZone);

-- Create supplier dimension
CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    Country NVARCHAR(100),
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier NVARCHAR(20) -- 'Tier1', 'Tier2', 'Tier3'
);

-- 3. Deploy fact tables
-- Execute: schema/create_facts.sql

-- Warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    SupplierKey INT NULL,
    OperationType NVARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderID NVARCHAR(100),
    BatchNumber NVARCHAR(100),
    QuantityHandled INT NOT NULL,
    CycleTimeMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    ErrorCount INT DEFAULT 0,
    StorageZone NVARCHAR(50),
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    CONSTRAINT FK_WHOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WHOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WHOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WHOps_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey)
);

CREATE NONCLUSTERED INDEX IX_FactWHOps_Time 
ON FactWarehouseOperations(TimeKey) INCLUDE (OperationType, DwellTimeHours);

-- Fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripID NVARCHAR(100) NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50) NOT NULL,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    PlannedDistanceKm DECIMAL(10,2),
    ActualDistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    OnTimeArrival BIT,
    DelayMinutes INT,
    DelayReason NVARCHAR(200),
    MaintenanceAlertFlag BIT DEFAULT 0,
    CONSTRAINT FK_FleetTrip_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrip_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrip_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrip_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Vehicle 
ON FactFleetTrips(VehicleID, StartTimeKey);

-- Cross-dock fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    HubGeographyKey INT NOT NULL,
    InboundTripKey BIGINT NULL,
    OutboundTripKey BIGINT NULL,
    QuantityTransferred INT NOT NULL,
    DwellTimeMinutes DECIMAL(10,2),
    TemperatureCompliant BIT DEFAULT 1,
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (HubGeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- 4. Create views and stored procedures
-- Execute: schema/create_views.sql, schema/create_procedures.sql
```

## Configuration

### Connection Configuration

Create `config.json` in the project root:

```json
{
  "database": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "integrated",
    "connectionTimeout": 30
  },
  "dataSources": {
    "wms": {
      "type": "api",
      "endpoint": "${WMS_API_ENDPOINT}",
      "authHeader": "Bearer ${WMS_API_KEY}",
      "refreshIntervalMinutes": 15
    },
    "telematics": {
      "type": "api",
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "authHeader": "X-API-Key: ${TELEMATICS_API_KEY}",
      "refreshIntervalMinutes": 5
    },
    "weather": {
      "type": "api",
      "endpoint": "https://api.openweathermap.org/data/2.5/",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "powerbi": {
    "workspaceId": "${POWERBI_WORKSPACE_ID}",
    "datasetRefreshSchedule": "0 */15 * * *"
  },
  "alerts": {
    "email": {
      "smtpServer": "${SMTP_SERVER}",
      "fromAddress": "${ALERT_FROM_EMAIL}",
      "recipients": ["${LOGISTICS_MANAGER_EMAIL}"]
    },
    "thresholds": {
      "fleetIdleTimePercent": 15,
      "warehouseDwellHours": 72,
      "delayMinutesThreshold": 30
    }
  }
}
```

### Row-Level Security Setup

```sql
-- Create security table
CREATE TABLE SecurityUserMapping (
    UserID NVARCHAR(100) PRIMARY KEY,
    UserEmail NVARCHAR(200) NOT NULL,
    RoleLevel NVARCHAR(50) NOT NULL, -- 'Executive', 'Supervisor', 'Operator'
    GeographyFilter NVARCHAR(MAX), -- JSON array of allowed geography keys
    SupplierFilter NVARCHAR(MAX), -- JSON array of allowed supplier keys
    CanViewFinancials BIT DEFAULT 0
);

-- Create RLS function
CREATE FUNCTION Security.fn_GeographyPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS result
    WHERE 
        @GeographyKey IN (
            SELECT VALUE 
            FROM SecurityUserMapping 
            CROSS APPLY OPENJSON(GeographyFilter)
            WHERE UserEmail = USER_NAME()
        )
        OR EXISTS (
            SELECT 1 
            FROM SecurityUserMapping 
            WHERE UserEmail = USER_NAME() 
            AND RoleLevel = 'Executive'
        );
GO

-- Apply security policy to warehouse operations
CREATE SECURITY POLICY GeographyFilterPolicy
ADD FILTER PREDICATE Security.fn_GeographyPredicate(GeographyKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
```

## Core SQL Procedures

### Data Loading Procedures

```sql
-- Incremental warehouse operations load
CREATE OR ALTER PROCEDURE dbo.usp_LoadWarehouseOperations
    @LastLoadTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if not specified
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(HOUR, -24, GETDATE());
    
    -- Staging table for external data
    CREATE TABLE #StagingWHOps (
        OperationDateTime DATETIME2,
        SKU NVARCHAR(100),
        LocationID NVARCHAR(50),
        SupplierID NVARCHAR(50),
        OperationType NVARCHAR(50),
        OrderID NVARCHAR(100),
        QuantityHandled INT,
        CycleTimeMinutes DECIMAL(10,2),
        StorageZone NVARCHAR(50)
    );
    
    -- Load from external WMS (example using OPENROWSET)
    -- In production, use linked server or external table
    INSERT INTO #StagingWHOps
    SELECT * FROM OPENROWSET(
        BULK '${WMS_EXPORT_PATH}',
        FORMAT = 'CSV',
        FIRSTROW = 2
    ) AS wms;
    
    -- Transform and load into fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, SupplierKey,
        OperationType, OrderID, QuantityHandled, 
        CycleTimeMinutes, StorageZone
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.SupplierKey,
        stg.OperationType,
        stg.OrderID,
        stg.QuantityHandled,
        stg.CycleTimeMinutes,
        stg.StorageZone
    FROM #StagingWHOps stg
    INNER JOIN DimTime t ON CAST(stg.OperationDateTime AS DATETIME2) = t.DateTimeValue
    INNER JOIN DimProduct p ON stg.SKU = p.SKU
    INNER JOIN DimGeography g ON stg.LocationID = g.LocationID
    LEFT JOIN DimSupplier s ON stg.SupplierID = s.SupplierID
    WHERE stg.OperationDateTime > @LastLoadTime;
    
    DROP TABLE #StagingWHOps;
END;
GO

-- Calculate warehouse gravity scores
CREATE OR ALTER PROCEDURE dbo.usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate gravity based on velocity, value, and operational priority
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            COUNT(DISTINCT wo.OperationKey) AS PickFrequency,
            AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
            AVG(wo.DwellTimeHours) AS AvgDwellTime,
            p.IsFragile,
            p.RequiresRefrigeration
        FROM DimProduct p
        LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
        GROUP BY p.ProductKey, p.SKU, p.IsFragile, p.RequiresRefrigeration
    ),
    ScoredProducts AS (
        SELECT 
            ProductKey,
            SKU,
            -- Normalized scoring 0-100
            CASE 
                WHEN PickFrequency IS NULL THEN 25.0
                ELSE (
                    (PickFrequency / NULLIF(MAX(PickFrequency) OVER(), 0) * 40.0) + -- 40% velocity
                    ((1.0 / NULLIF(AvgDwellTime, 0)) / (1.0 / NULLIF(MAX(AvgDwellTime) OVER(), 0)) * 30.0) + -- 30% turnover
                    (IsFragile * 15.0) + -- 15% fragility bonus
                    (RequiresRefrigeration * 15.0) -- 15% cold chain bonus
                )
            END AS GravityScore
        FROM ProductMetrics
    )
    UPDATE p
    SET 
        p.GravityScore = sp.GravityScore,
        p.GravityZone = CASE 
            WHEN sp.GravityScore >= 70 THEN 'High'
            WHEN sp.GravityScore >= 40 THEN 'Medium'
            ELSE 'Low'
        END,
        p.LastGravityUpdate = GETDATE()
    FROM DimProduct p
    INNER JOIN ScoredProducts sp ON p.ProductKey = sp.ProductKey;
END;
GO
```

### Cross-Fact KPI Queries

```sql
-- Dwell time vs fleet idling correlation
CREATE OR ALTER VIEW vw_DwellTimeFleetImpact AS
SELECT 
    t.Date,
    t.HourOfDay,
    g.LocationName AS Warehouse,
    p.GravityZone,
    AVG(wo.DwellTimeHours) AS AvgDwellHours,
    SUM(ft.IdleTimeMinutes) / NULLIF(COUNT(DISTINCT ft.TripKey), 0) AS AvgFleetIdleMinutes,
    COUNT(DISTINCT wo.OrderID) AS OrderCount,
    COUNT(DISTINCT ft.TripKey) AS TripCount
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
LEFT JOIN FactFleetTrips ft 
    ON ft.OriginGeographyKey = g.GeographyKey
    AND ft.StartTimeKey BETWEEN t.TimeKey - 96 AND t.TimeKey + 96 -- +/- 24 hour window
WHERE wo.OperationType IN ('Picking', 'Shipping')
GROUP BY t.Date, t.HourOfDay, g.LocationName, p.GravityZone;
GO

-- Predictive bottleneck detection
CREATE OR ALTER PROCEDURE dbo.usp_DetectBottlenecks
    @LookbackDays INT = 7,
    @ThresholdPercentile DECIMAL(5,2) = 90.0
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @LookbackTimeKey INT = (
        SELECT MIN(TimeKey) 
        FROM DimTime 
        WHERE Date >= DATEADD(DAY, -@LookbackDays, CAST(GETDATE() AS DATE))
    );
    
    -- Warehouse bottlenecks
    SELECT 
        'Warehouse' AS BottleneckType,
        g.LocationName AS Location,
        wo.StorageZone,
        p.CategoryLevel1 AS Category,
        AVG(wo.DwellTimeHours) AS AvgDwellHours,
        PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wo.DwellTimeHours) 
            OVER (PARTITION BY g.LocationName) AS P95DwellHours,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    WHERE wo.TimeKey >= @LookbackTimeKey
    GROUP BY g.LocationName, wo.StorageZone, p.CategoryLevel1
    HAVING AVG(wo.DwellTimeHours) > (
        SELECT PERCENTILE_CONT(@ThresholdPercentile / 100.0) WITHIN GROUP (ORDER BY DwellTimeHours)
        FROM FactWarehouseOperations
        WHERE TimeKey >= @LookbackTimeKey
    )
    
    UNION ALL
    
    -- Fleet bottlenecks
    SELECT 
        'Fleet' AS BottleneckType,
        g.LocationName AS Location,
        ft.DelayReason AS StorageZone,
        NULL AS Category,
        AVG(ft.IdleTimeMinutes) AS AvgDwellHours,
        PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY ft.IdleTimeMinutes) 
            OVER (PARTITION BY g.LocationName) AS P95DwellHours,
        COUNT(*) AS OperationCount
    FROM FactFleetTrips ft
    INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
    WHERE ft.StartTimeKey >= @LookbackTimeKey
    AND ft.IdleTimeMinutes > 0
    GROUP BY g.LocationName, ft.DelayReason
    HAVING AVG(ft.IdleTimeMinutes) > (
        SELECT PERCENTILE_CONT(@ThresholdPercentile / 100.0) WITHIN GROUP (ORDER BY IdleTimeMinutes)
        FROM FactFleetTrips
        WHERE StartTimeKey >= @LookbackTimeKey
    )
    ORDER BY AvgDwellHours DESC;
END;
GO
```

## Power BI Integration

### Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Connection settings:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)

### DAX Measures for Cross-Fact Analysis

```dax
-- Fleet efficiency score combining multiple metrics
Fleet Efficiency Score = 
VAR AvgIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR AvgOnTime = AVERAGE(FactFleetTrips[OnTimeArrival])
VAR AvgFuelEfficiency = DIVIDE(
    AVERAGE(FactFleetTrips[ActualDistanceKm]),
    AVERAGE(FactFleetTrips[FuelConsumedLiters])
)
VAR MaxFuelEfficiency = 15.0  -- Benchmark: 15 km/liter
VAR NormalizedIdle = 1 - DIVIDE(AvgIdleTime, 60, 1)  -- Penalize > 60 min
VAR NormalizedFuel = DIVIDE(AvgFuelEfficiency, MaxFuelEfficiency, 1)
RETURN
    (AvgOnTime * 0.4 + NormalizedIdle * 0.3 + NormalizedFuel * 0.3) * 100

-- Warehouse gravity zone compliance
Gravity Zone Compliance % = 
VAR HighGravityPicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProduct[GravityZone] = "High",
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR HighGravityOptimalZone = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProduct[GravityZone] = "High",
        FactWarehouseOperations[OperationType] = "Picking",
        FactWarehouseOperations[StorageZone] IN {"A1", "A2", "B1"}  -- Optimal zones
    )
RETURN
    DIVIDE(HighGravityOptimalZone, HighGravityPicks, 0)

-- Cross-fact KPI: Cost per delivery mile
Cost Per Delivery Mile = 
VAR TotalFuelCost = SUMX(
    FactFleetTrips,
    FactFleetTrips[FuelConsumedLiters] * 1.5  -- Fuel cost per liter
)
VAR TotalLaborCost = SUMX(
    FactFleetTrips,
    (DATEDIFF(
        RELATED(DimTime[StartTime]),
        RELATED(DimTime[EndTime]),
        HOUR
    ) * 25)  -- Hourly labor rate
)
VAR TotalMiles = SUM(FactFleetTrips[ActualDistanceKm]) * 0.621371  -- km to miles
RETURN
    DIVIDE(TotalFuelCost + TotalLaborCost, TotalMiles, 0)

-- Temporal elasticity: Capacity utilization trend
Capacity Utilization Trend = 
VAR CurrentPeriod = MAX(DimTime[Date])
VAR PriorPeriod = DATEADD(DimTime[Date], -30, DAY)
VAR CurrentUtil = 
    DIVIDE(
        CALCULATE(SUM(FactWarehouseOperations[QuantityHandled]), DimTime[Date] = CurrentPeriod),
        10000  -- Max warehouse capacity
    )
VAR PriorUtil = 
    DIVIDE(
        CALCULATE(SUM(FactWarehouseOperations[QuantityHandled]), DimTime[Date] = PriorPeriod),
        10000
    )
RETURN
    (CurrentUtil - PriorUtil) / NULLIF(PriorUtil, 0)
```

### Key Visualizations

**Warehouse Operations Dashboard:**
- Dwell time heatmap by storage zone (matrix visual)
- Gravity zone distribution (donut chart)
- Pick cycle time trend (line chart with anomaly detection)
- Top bottleneck locations (bar chart)

**Fleet Performance Dashboard:**
- Real-time vehicle map (Azure Maps visual)
- Idle time by route segment (decomposition tree)
- Fuel efficiency vs. load weight (scatter chart)
- On-time delivery rate (KPI card with trend)

**Executive Summary Dashboard:**
- Cost per delivery mile (KPI card)
- Cross-fact efficiency score (gauge)
- Predictive bottleneck alerts (table with conditional formatting)
- Capacity utilization forecast (area chart)

## Common Patterns

### Pattern 1: Time-Phased Dimension Queries

```sql
-- Query operations by specific time buckets
SELECT 
    t.Date,
    t.HourOfDay,
    t.IsWeekend,
    COUNT(*) AS OperationCount,
    AVG(wo.CycleTimeMinutes) AS AvgCycleTime
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.Date BETWEEN '2026-06-01' AND '2026-06-30'
    AND t.HourOfDay BETWEEN 8 AND 17  -- Business hours
GROUP BY t.Date, t.HourOfDay, t.IsWeekend
ORDER BY t.Date, t.HourOfDay;
```

### Pattern 2: Hierarchical Geography Rollup

```sql
-- Aggregate metrics by geography hierarchy
WITH GeographyHierarchy AS (
    SELECT 
        g1.GeographyKey,
        g1.LocationName AS Location,
        g2.LocationName AS Region,
        g3.LocationName AS Country
    FROM DimGeography g1
    LEFT JOIN DimGeography g2 ON g1.ParentLocationKey = g2.GeographyKey
    LEFT JOIN DimGeography g3 ON g2.ParentLocationKey = g3.GeographyKey
)
SELECT 
    gh.Country,
    gh.Region,
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    SUM(ft.ActualDistanceKm) AS TotalDistanceKm,
    AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes
FROM FactFleetTrips ft
INNER JOIN GeographyHierarchy gh ON ft.OriginGeographyKey = gh.GeographyKey
GROUP BY ROLLUP(gh.Country, gh.Region);
```

### Pattern 3: Supplier Performance Scoring

```sql
-- Calculate supplier reliability impact on operations
SELECT 
    s.SupplierName,
    s.ReliabilityTier,
    COUNT(DISTINCT wo.OrderID) AS OrderCount,
    AVG(wo.DwellTimeHours) AS AvgDwellTime,
    AVG(s.LeadTimeDaysAvg) AS AvgLeadTime,
    s.DefectRatePercent,
    -- Composite supplier score
    (
        (1 - s.DefectRatePercent / 100.0) * 0.4 +
        (s.ComplianceScore / 100.0) * 0.3 +
        (1 - (s.LeadTimeVariance / NULLIF(s.LeadTimeDaysAvg, 0))) * 0.3
    ) * 100 AS SupplierQualityScore
FROM DimSupplier s
LEFT JOIN FactWarehouseOperations wo ON s.SupplierKey = wo.SupplierKey
WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)  -- Last 30 days
GROUP BY 
    s.SupplierName, s.ReliabilityTier, s.LeadTimeDaysAvg, 
    s.LeadTimeVariance, s.DefectRatePercent, s.ComplianceScore
ORDER BY SupplierQualityScore DESC;
```

### Pattern 4: Automated Alert Generation

```sql
-- Create alert stored procedure
CREATE OR ALTER PROCEDURE dbo.usp_GenerateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertTable TABLE (
        AlertType NVARCHAR(50),
        Severity NVARCHAR(20),
        Message NVARCHAR(500),
        AffectedEntity NVARCHAR(200),
        MetricValue DECIMAL(10,2)
    );
    
    -- Fleet idle time alerts
    INSERT INTO @AlertTable
    SELECT 
        'Fleet Idle Time Exceeded',
        CASE 
            WHEN (ft.IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, 
                t1.DateTimeValue, t2.DateTimeValue), 0) * 100) > 25 THEN 'Critical'
            ELSE 'Warning'
        END,
        'Vehicle ' + ft.VehicleID + ' idle time exceeds threshold',
        ft.VehicleID,
        ft.IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, 
            t1.DateTimeValue, t2.DateTimeValue), 0) * 100
    FROM FactFleetTrips ft
    INNER JOIN DimTime t1 ON ft.StartTimeKey = t1.TimeKey
    LEFT JOIN DimTime t2 ON ft
