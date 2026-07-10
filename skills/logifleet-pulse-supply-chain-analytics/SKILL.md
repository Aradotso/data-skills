---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for multi-modal logistics intelligence and fleet optimization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse analytics"
  - "implement cross-fact KPI harmonization"
  - "create warehouse gravity zone analytics"
  - "build logistics intelligence dashboard"
  - "integrate fleet telemetry with warehouse data"
  - "set up predictive bottleneck detection"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics template that unifies logistics operations data across warehouses, fleet management, and supply chain processes. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, cross-dock activities, and supplier data
- **Real-time dashboards** with 15-minute refresh granularity
- **Predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and item characteristics
- **Cross-modal KPI harmonization** connecting inventory metrics with fleet performance
- **Role-based access control** with row-level security

The platform is designed for 3PL operators, retail chains, distributors, and any organization managing complex logistics operations.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, ERP systems

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Execute the schema deployment script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the complete schema script
-- (Assumes schema.sql exists in repository)
:r schema.sql
GO
```

### Step 3: Configure Data Sources

Create a configuration file for your data connections:

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}",
    "port": 1433
  },
  "dataSources": {
    "wms": {
      "type": "REST",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telemetry": {
      "type": "ODBC",
      "connectionString": "${TELEMETRY_CONNECTION_STRING}"
    },
    "weather": {
      "type": "REST",
      "endpoint": "https://api.weatherapi.com/v1",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "refreshInterval": 900
}
```

### Step 4: Open Power BI Template

```powershell
# Open the Power BI template file
Start-Process "LogiFleet_Pulse_Master.pbit"
```

When prompted, enter your SQL Server connection details.

## Core Schema Components

### Fact Tables

#### FactWarehouseOperations

Captures warehouse micro-operations with time-phased granularity:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2),
    PackingTimeMinutes INT,
    AccuracyScore DECIMAL(5,2),
    StaffCount INT,
    EquipmentUtilization DECIMAL(5,2),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_FactWH_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, PickRate);
```

#### FactFleetTrips

Tracks vehicle telemetry and route performance:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginKey INT NOT NULL,
    DestinationKey INT NOT NULL,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    OnTimeDelivery BIT,
    DelayMinutes INT,
    WeatherImpact BIT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time_Vehicle 
ON FactFleetTrips(TimeKey, VehicleKey) 
INCLUDE (IdleTimeMinutes, FuelConsumedGallons);
```

#### FactCrossDock

Monitors cross-dock transfer efficiency:

```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    InboundTripKey BIGINT NOT NULL,
    OutboundTripKey BIGINT NOT NULL,
    ProductKey INT NOT NULL,
    DockKey INT NOT NULL,
    TransferTimeMinutes INT,
    QuantityUnits INT,
    TemperatureCompliance BIT,
    QualityCheckPassed BIT,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);
```

### Dimension Tables

#### DimTime (15-minute grain)

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    WeekOfYear INT,
    DayOfMonth INT,
    DayOfWeek INT,
    Hour INT,
    Minute15Block INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    IsHoliday BIT,
    FiscalPeriod INT
);

-- Populate time dimension
DECLARE @StartDate DATETIME = '2024-01-01';
DECLARE @EndDate DATETIME = '2027-12-31';

WITH TimeCTE AS (
    SELECT @StartDate AS TimeValue
    UNION ALL
    SELECT DATEADD(MINUTE, 15, TimeValue)
    FROM TimeCTE
    WHERE TimeValue < @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, WeekOfYear, DayOfMonth, DayOfWeek, Hour, Minute15Block, IsWeekend, IsHoliday)
SELECT 
    CAST(FORMAT(TimeValue, 'yyyyMMddHHmm') AS INT) / 100 AS TimeKey,
    TimeValue,
    YEAR(TimeValue),
    DATEPART(QUARTER, TimeValue),
    MONTH(TimeValue),
    DATEPART(WEEK, TimeValue),
    DAY(TimeValue),
    DATEPART(WEEKDAY, TimeValue),
    DATEPART(HOUR, TimeValue),
    (DATEPART(MINUTE, TimeValue) / 15) * 15,
    CASE WHEN DATEPART(WEEKDAY, TimeValue) IN (1,7) THEN 1 ELSE 0 END,
    0
FROM TimeCTE
OPTION (MAXRECURSION 0);
```

#### DimProductGravity (with spatial optimization scoring)

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(255),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    PickFrequency INT, -- Average picks per day
    UnitValue DECIMAL(10,2),
    FragilityScore DECIMAL(3,2), -- 0 to 1
    ReplenishmentLeadTimeDays INT,
    GravityScore AS (
        (PickFrequency * 0.4) + 
        (UnitValue * 0.3) + 
        (FragilityScore * 100 * 0.2) +
        ((100.0 / NULLIF(ReplenishmentLeadTimeDays, 0)) * 0.1)
    ) PERSISTED,
    RecommendedZone VARCHAR(50)
);

-- Update recommended zones based on gravity score
UPDATE DimProductGravity
SET RecommendedZone = CASE 
    WHEN GravityScore > 75 THEN 'High-Gravity Zone A'
    WHEN GravityScore BETWEEN 50 AND 75 THEN 'Medium-Gravity Zone B'
    WHEN GravityScore BETWEEN 25 AND 50 THEN 'Low-Gravity Zone C'
    ELSE 'Bulk Storage Zone D'
END;
```

#### DimSupplierReliability

```sql
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(255),
    AverageLeadTimeDays DECIMAL(5,2),
    LeadTimeVarianceStdDev DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityRank AS (
        (OnTimeDeliveryRate * 0.4) +
        ((100 - DefectPercentage) * 0.3) +
        (ComplianceScore * 0.2) +
        ((100.0 / NULLIF(LeadTimeVarianceStdDev, 0)) * 0.1)
    ) PERSISTED
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Incremental load from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        DwellTimeMinutes, PickRate, PackingTimeMinutes,
        AccuracyScore, StaffCount, EquipmentUtilization
    )
    SELECT 
        CAST(FORMAT(s.OperationDateTime, 'yyyyMMddHHmm') AS INT) / 100 AS TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.OperationType,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.PickRate,
        s.PackingTimeMinutes,
        s.AccuracyScore,
        s.StaffCount,
        s.EquipmentUtilization
    FROM StagingWarehouseOps s
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    INNER JOIN DimGeography g ON s.WarehouseCode = g.LocationCode
    WHERE s.OperationDateTime > @LastLoadTimestamp
        AND s.IsProcessed = 0;
    
    -- Mark records as processed
    UPDATE StagingWarehouseOps
    SET IsProcessed = 1, ProcessedDateTime = GETDATE()
    WHERE OperationDateTime > @LastLoadTimestamp
        AND IsProcessed = 0;
END;
GO
```

### Cross-Fact KPI Query

```sql
CREATE PROCEDURE usp_GetCrossDockEfficiencyByGravityZone
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SELECT 
        p.RecommendedZone,
        COUNT(DISTINCT cd.CrossDockKey) AS TotalTransfers,
        AVG(cd.TransferTimeMinutes) AS AvgTransferMinutes,
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
        SUM(ft.FuelConsumedGallons) AS TotalFuelGallons,
        AVG(p.GravityScore) AS AvgGravityScore,
        -- Calculate efficiency score
        (100.0 - AVG(cd.TransferTimeMinutes)) * 
        (100.0 - AVG(ft.IdleTimeMinutes)) / 100.0 AS EfficiencyScore
    FROM FactCrossDock cd
    INNER JOIN DimTime t ON cd.TimeKey = t.TimeKey
    INNER JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    INNER JOIN FactFleetTrips ft ON cd.OutboundTripKey = ft.TripKey
    WHERE CAST(t.FullDateTime AS DATE) BETWEEN @StartDate AND @EndDate
    GROUP BY p.RecommendedZone
    ORDER BY EfficiencyScore DESC;
END;
GO
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE usp_PredictBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    WITH RecentTrends AS (
        SELECT 
            WarehouseKey,
            ProductKey,
            AVG(DwellTimeMinutes) AS AvgDwell,
            STDEV(DwellTimeMinutes) AS StdDevDwell,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -168, GETDATE()) -- Last 7 days
        GROUP BY WarehouseKey, ProductKey
    ),
    CurrentState AS (
        SELECT 
            WarehouseKey,
            ProductKey,
            AVG(DwellTimeMinutes) AS CurrentAvgDwell
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
        GROUP BY WarehouseKey, ProductKey
    )
    SELECT 
        g.LocationName AS Warehouse,
        p.ProductName,
        p.RecommendedZone,
        rt.AvgDwell AS HistoricalAvgDwell,
        cs.CurrentAvgDwell,
        rt.StdDevDwell,
        -- Bottleneck probability
        CASE 
            WHEN cs.CurrentAvgDwell > rt.AvgDwell + (2 * rt.StdDevDwell) THEN 'HIGH'
            WHEN cs.CurrentAvgDwell > rt.AvgDwell + rt.StdDevDwell THEN 'MEDIUM'
            ELSE 'LOW'
        END AS BottleneckRisk,
        -- Estimated impact in next period
        (cs.CurrentAvgDwell - rt.AvgDwell) * rt.OperationCount AS EstimatedDelayMinutes
    FROM RecentTrends rt
    INNER JOIN CurrentState cs ON rt.WarehouseKey = cs.WarehouseKey 
        AND rt.ProductKey = cs.ProductKey
    INNER JOIN DimGeography g ON rt.WarehouseKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON rt.ProductKey = p.ProductKey
    WHERE cs.CurrentAvgDwell > rt.AvgDwell
    ORDER BY EstimatedDelayMinutes DESC;
END;
GO
```

### Automated Alerting

```sql
CREATE PROCEDURE usp_CheckAlertsAndNotify
AS
BEGIN
    DECLARE @AlertLog TABLE (
        AlertType VARCHAR(50),
        Severity VARCHAR(20),
        Message NVARCHAR(500),
        AffectedEntity VARCHAR(255),
        MetricValue DECIMAL(10,2),
        Threshold DECIMAL(10,2)
    );
    
    -- Fleet idle time threshold
    INSERT INTO @AlertLog
    SELECT 
        'Fleet Idle Time Exceeded',
        'HIGH',
        'Vehicle ' + v.VehicleID + ' idle time is ' + 
            CAST(AVG(ft.IdleTimeMinutes) AS VARCHAR) + ' minutes',
        v.VehicleID,
        AVG(ft.IdleTimeMinutes),
        15.0
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
    GROUP BY v.VehicleID
    HAVING AVG(ft.IdleTimeMinutes) > 15;
    
    -- Warehouse dwell time alert
    INSERT INTO @AlertLog
    SELECT 
        'Warehouse Dwell Time Alert',
        'MEDIUM',
        'SKU ' + p.SKU + ' at ' + g.LocationName + 
            ' has dwell time of ' + CAST(AVG(wo.DwellTimeMinutes) AS VARCHAR) + ' minutes',
        p.SKU,
        AVG(wo.DwellTimeMinutes),
        72.0 * 60
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
    GROUP BY p.SKU, g.LocationName
    HAVING AVG(wo.DwellTimeMinutes) > (72 * 60);
    
    -- Insert alerts into logging table
    INSERT INTO AlertHistory (AlertDateTime, AlertType, Severity, Message, AffectedEntity, MetricValue, Threshold)
    SELECT GETDATE(), AlertType, Severity, Message, AffectedEntity, MetricValue, Threshold
    FROM @AlertLog;
    
    -- Return alerts for email/SMS processing
    SELECT * FROM @AlertLog;
END;
GO
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

```dax
// Warehouse Efficiency Score
Warehouse Efficiency Score = 
VAR AvgPickRate = AVERAGE(FactWarehouseOperations[PickRate])
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR TargetPickRate = 120 // picks per hour
VAR TargetDwellTime = 60 // minutes
RETURN
    ((AvgPickRate / TargetPickRate) * 0.6) + 
    ((TargetDwellTime / AVERAGEX(FactWarehouseOperations, [DwellTimeMinutes] + 1)) * 0.4) * 100

// Fleet Fuel Efficiency per Gravity Zone
Fuel Per Gravity Zone = 
CALCULATE(
    SUM(FactFleetTrips[FuelConsumedGallons]) / SUM(FactFleetTrips[DistanceMiles]),
    USERELATIONSHIP(FactFleetTrips[RouteKey], DimRoute[RouteKey])
)

// Predictive Bottleneck Index
Bottleneck Index = 
VAR CurrentDwell = CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -4, HOUR)
)
VAR HistoricalDwell = CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -168, HOUR)
)
VAR StdDev = STDEV.P(FactWarehouseOperations[DwellTimeMinutes])
RETURN
    IF(
        CurrentDwell > HistoricalDwell + (2 * StdDev),
        100,
        ((CurrentDwell - HistoricalDwell) / StdDev) * 50
    )

// Cross-Dock Transfer Velocity
Transfer Velocity = 
DIVIDE(
    COUNTROWS(FactCrossDock),
    AVERAGE(FactCrossDock[TransferTimeMinutes]) / 60,
    0
) // Transfers per hour

// Gravity Zone Compliance
Gravity Zone Compliance % = 
VAR TotalProducts = COUNTROWS(DimProductGravity)
VAR ProperlyZoned = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[ActualZone] = DimProductGravity[RecommendedZone]
)
RETURN
    DIVIDE(ProperlyZoned, TotalProducts, 0) * 100
```

### Power Query M for Data Refresh

```m
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse"
    ),
    
    // Get last refresh timestamp
    LastRefresh = List.Max(
        Table.Column(
            Sql.Database(
                Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
                "LogiFleetPulse"
            ){[Schema="dbo",Item="RefreshLog"]}[Data], 
            "LastRefreshTime"
        )
    ),
    
    // Incremental load
    WarehouseOps = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FilteredOps = Table.SelectRows(
        WarehouseOps, 
        each [CreatedDateTime] > LastRefresh
    ),
    
    // Transform and clean
    CleanedData = Table.TransformColumnTypes(
        FilteredOps,
        {
            {"DwellTimeMinutes", Int64.Type},
            {"PickRate", type number},
            {"AccuracyScore", type number}
        }
    )
in
    CleanedData
```

## Common Patterns

### Pattern 1: Multi-Dimensional Slicing by Time and Geography

```sql
-- Analyze fleet performance by time of day and route
SELECT 
    t.Hour,
    t.DayOfWeek,
    g_origin.Region AS OriginRegion,
    g_dest.Region AS DestinationRegion,
    COUNT(*) AS TripCount,
    AVG(ft.FuelConsumedGallons / NULLIF(ft.DistanceMiles, 0)) AS AvgFuelEfficiency,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    SUM(CASE WHEN ft.OnTimeDelivery = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS OnTimeRate
FROM FactFleetTrips ft
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
INNER JOIN DimGeography g_origin ON ft.OriginKey = g_origin.GeographyKey
INNER JOIN DimGeography g_dest ON ft.DestinationKey = g_dest.GeographyKey
WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY t.Hour, t.DayOfWeek, g_origin.Region, g_dest.Region
ORDER BY t.DayOfWeek, t.Hour;
```

### Pattern 2: Temporal Elasticity Simulation

```sql
-- Simulate impact of 95% warehouse capacity vs 80%
WITH CapacityScenarios AS (
    SELECT 
        'Current (80%)' AS Scenario,
        WarehouseKey,
        AVG(DwellTimeMinutes) AS AvgDwell,
        AVG(PickRate) AS AvgPickRate
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 100 FROM DimTime)
    GROUP BY WarehouseKey
    
    UNION ALL
    
    SELECT 
        'Projected (95%)' AS Scenario,
        WarehouseKey,
        AVG(DwellTimeMinutes) * 1.35 AS AvgDwell, -- 35% increase based on historical
        AVG(PickRate) * 0.82 AS AvgPickRate -- 18% decrease
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 100 FROM DimTime)
    GROUP BY WarehouseKey
)
SELECT 
    cs.Scenario,
    g.LocationName,
    cs.AvgDwell,
    cs.AvgPickRate,
    -- Calculate downstream fleet impact
    (cs.AvgDwell * ft_avg.AvgTripsPerDay) / 60.0 AS EstimatedFleetDelayHours,
    (cs.AvgDwell * ft_avg.AvgTripsPerDay * 0.5) AS EstimatedFuelWasteGallons
FROM CapacityScenarios cs
INNER JOIN DimGeography g ON cs.WarehouseKey = g.GeographyKey
CROSS APPLY (
    SELECT AVG(TripCount) AS AvgTripsPerDay
    FROM (
        SELECT COUNT(*) AS TripCount
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE ft.OriginKey = cs.WarehouseKey
        GROUP BY CAST(t.FullDateTime AS DATE)
    ) sub
) ft_avg
ORDER BY cs.Scenario, EstimatedFleetDelayHours DESC;
```

### Pattern 3: Gravity Zone Optimization Recommendation

```sql
-- Identify products in wrong zones and calculate relocation ROI
WITH ProductPerformance AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.RecommendedZone,
        wo.ActualZone,
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        AVG(wo.PickRate) AS AvgPickRate,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 500 FROM DimTime)
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.RecommendedZone, wo.ActualZone
),
RelocationImpact AS (
    SELECT 
        pp.*,
        -- Estimated improvement if moved to recommended zone
        pp.AvgDwell * 0.25 AS EstimatedDwellReduction,
        pp.AvgPickRate * 0.15 AS EstimatedPickRateIncrease,
        pp.OperationCount * (pp.AvgDwell * 0.25) AS TotalMinutesSaved
    FROM ProductPerformance pp
    WHERE pp.ActualZone <> pp.RecommendedZone
)
SELECT 
    SKU,
    ProductName,
    ActualZone AS CurrentZone,
    RecommendedZone AS TargetZone,
    AvgDwell AS CurrentAvgDwell,
    AvgDwell - EstimatedDwellReduction AS ProjectedAvgDwell,
    TotalMinutesSaved / 60.0 AS EstimatedHoursSavedPerMonth,
    (TotalMinutesSaved / 60.0) * 25.0 AS EstimatedCostSavings -- $25/hour labor cost
FROM RelocationImpact
WHERE TotalMinutesSaved > 60 -- At least 1 hour saved
ORDER BY EstimatedCostSavings DESC;
```

## Troubleshooting

### Issue: Power BI Refresh Takes Too Long

**Solution:** Implement incremental refresh with partitioning:

```sql
-- Partition fact tables by month
CREATE PARTITION SCHEME PS_FactByMonth
AS PARTITION PF_Monthly
TO ([FG_Archive], [FG_2024_Q1], [FG_2024_Q2], [FG_2024_Q3], [FG_2024_Q4], [PRIMARY]);

-- Recreate fact table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey BIGINT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OperationMonth AS DATEFROMPARTS(TimeKey / 1000000, (TimeKey / 10000) % 100, 1) PERSISTED,
    -- ... other columns
) ON PS_FactByMonth(OperationMonth);
```

In Power BI, enable incremental refresh:
- Range: Last 2 years
- Refresh: Last 7 days
- Detect data changes: Yes, using TimeKey column

### Issue: Cross-Fact Queries Return Inconsistent Results

**Solution:** Verify relationship cardinality and use bridge tables:

```sql
-- Create bridge table for many-to-many relationships
CREATE TABLE BridgeRouteToZone (
    BridgeKey INT IDENTITY(1,1) PRIMARY KEY,
    RouteKey INT NOT NULL,
    ZoneKey INT NOT NULL,
    AllocationPercentage DECIMAL(5,2),
    CONSTRAINT FK_Bridge_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey),
    CONSTRAINT FK_Bridge_Zone FOREIGN KEY (ZoneKey) REFERENCES DimWarehouseZone(ZoneKey)
);

-- Query using bridge
SELECT 
    r.RouteName,
    z.ZoneName,
    SUM(ft.FuelConsumedGallons * b.AllocationPercentage / 100.0) AS AllocatedFuel
FROM FactFleetTrips ft
INNER JOIN BridgeRouteToZone b ON ft.RouteKey = b.RouteKey
INNER JOIN D
