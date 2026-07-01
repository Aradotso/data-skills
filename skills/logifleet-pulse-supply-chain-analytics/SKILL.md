---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure warehouse and fleet analytics dashboard
  - implement multi-fact star schema for logistics
  - create supply chain intelligence platform
  - build real-time fleet and warehouse monitoring
  - integrate logistics kpi tracking system
  - setup cross-modal supply chain orchestrator
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a single semantic layer using MS SQL Server and Power BI. It provides real-time visibility across multi-modal logistics operations through a custom multi-fact star schema with time-phased dimensions.

## What It Does

- **Unified Data Model**: Combines warehouse operations, fleet trips, cross-dock activities, and supplier data
- **Multi-Fact Star Schema**: Links inventory, transportation, and operational metrics through shared dimensions
- **Real-Time Dashboards**: Power BI visualizations refreshed every 15 minutes
- **Predictive Analytics**: Bottleneck detection, fleet triage, and warehouse optimization
- **Cross-Fact KPIs**: Correlates metrics like dwell time vs. fleet idling costs
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency and item value

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to source systems (WMS, TMS, telemetry feeds)
- SQL Server Management Studio (SSMS) for deployment

### Database Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema**:
```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the schema deployment script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the complete schema script
-- (Execute LogiFleet_Schema.sql from the repository)
```

3. **Configure connection strings**:
```json
{
  "sqlServer": {
    "server": "your-server.database.windows.net",
    "database": "LogiFleetPulse",
    "authentication": "SQL",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "connectionString": "${WMS_CONNECTION_STRING}",
      "refreshInterval": "15m"
    },
    "fleetTelemetry": {
      "apiEndpoint": "${FLEET_API_ENDPOINT}",
      "apiKey": "${FLEET_API_KEY}"
    }
  }
}
```

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    CycleTimeMinutes INT,
    GravityZoneID INT,
    OperationTimestamp DATETIME2,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey) INCLUDE (Quantity, DwellTimeMinutes);
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey, OperationType);
CREATE COLUMNSTORE INDEX CIX_FactWH_Analytics ON FactWarehouseOperations(TimeKey, ProductKey, Quantity, DwellTimeMinutes);
```

**FactFleetTrips** - Fleet and route tracking:
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginKey INT NOT NULL,
    DestinationKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightKG DECIMAL(12,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey) INCLUDE (FuelLiters, IdleTimeMinutes);
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(RouteKey, TripStartTime);
```

### Dimension Tables

**DimTime** - Time intelligence with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    DateKey INT,
    TimeOfDay TIME,
    Hour TINYINT,
    Minute TINYINT,
    QuarterHour TINYINT, -- 0, 1, 2, 3 (for 15-min intervals)
    DayOfWeek VARCHAR(10),
    DayOfMonth TINYINT,
    Month TINYINT,
    Quarter TINYINT,
    Year INT,
    FiscalPeriod INT,
    IsWeekend BIT,
    IsHoliday BIT,
    ShiftName VARCHAR(20) -- 'Morning', 'Afternoon', 'Night'
);
```

**DimProductGravity** - Product hierarchy with gravity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitValue DECIMAL(12,2),
    WeightKG DECIMAL(10,3),
    Fragility VARCHAR(20), -- 'Low', 'Medium', 'High'
    VelocityClass VARCHAR(20), -- 'Slow', 'Medium', 'Fast'
    GravityScore DECIMAL(5,2), -- Calculated: (velocity * value) / fragility_factor
    RecommendedZone VARCHAR(50),
    LastRecalculated DATETIME2
);
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50),
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    Address VARCHAR(500),
    City VARCHAR(100),
    State VARCHAR(100),
    Country VARCHAR(100),
    Region VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    WarehouseCapacityCBM DECIMAL(12,2),
    CurrentUtilizationPct DECIMAL(5,2)
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE [dbo].[usp_LoadWarehouseOperations]
    @LastLoadTimestamp DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last successful load if not specified
    IF @LastLoadTimestamp IS NULL
        SELECT @LastLoadTimestamp = MAX(OperationTimestamp) 
        FROM FactWarehouseOperations;
    
    -- Insert new warehouse operations from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes, CycleTimeMinutes,
        GravityZoneID, OperationTimestamp
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        st.OperationType,
        st.Quantity,
        DATEDIFF(MINUTE, st.StartTime, st.EndTime) AS DwellTimeMinutes,
        st.CycleTimeMinutes,
        p.RecommendedZone AS GravityZoneID,
        st.OperationTimestamp
    FROM Staging_WarehouseOps st
    INNER JOIN DimTime t ON st.OperationTimestamp = t.FullDateTime
    INNER JOIN DimGeography g ON st.WarehouseID = g.LocationID
    INNER JOIN DimProductGravity p ON st.SKU = p.SKU
    INNER JOIN DimSupplierReliability s ON st.SupplierID = s.SupplierID
    WHERE st.OperationTimestamp > @LastLoadTimestamp
        AND st.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE Staging_WarehouseOps
    SET IsProcessed = 1
    WHERE OperationTimestamp > @LastLoadTimestamp;
    
    -- Log the load
    INSERT INTO LoadLog (TableName, RecordsLoaded, LoadTimestamp)
    SELECT 'FactWarehouseOperations', @@ROWCOUNT, GETDATE();
END;
GO
```

### Cross-Fact KPI Calculation

```sql
CREATE PROCEDURE [dbo].[usp_CalculateCrossFactKPIs]
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    -- Calculate composite KPI: Dwell Time vs Fleet Idle Cost
    SELECT 
        t.DateKey,
        g.LocationName AS Warehouse,
        p.Category AS ProductCategory,
        
        -- Warehouse metrics
        AVG(wh.DwellTimeMinutes) AS AvgDwellTimeMinutes,
        SUM(wh.Quantity) AS TotalQuantityHandled,
        
        -- Fleet metrics (for shipments from this warehouse)
        AVG(fl.IdleTimeMinutes) AS AvgFleetIdleMinutes,
        AVG(fl.FuelLiters) AS AvgFuelConsumption,
        
        -- Cross-fact composite KPI
        (AVG(wh.DwellTimeMinutes) * AVG(fl.IdleTimeMinutes)) / 
        NULLIF(SUM(wh.Quantity), 0) AS DwellIdleEfficiencyIndex,
        
        -- Cost impact (assuming $50/hour for idle fleet)
        (AVG(fl.IdleTimeMinutes) / 60.0) * 50 AS EstimatedIdleCost
        
    FROM FactWarehouseOperations wh
    INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wh.WarehouseKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
    LEFT JOIN FactFleetTrips fl ON fl.OriginKey = g.GeographyKey 
        AND fl.TimeKey = t.TimeKey
    WHERE t.DateKey BETWEEN CONVERT(INT, CONVERT(VARCHAR, @StartDate, 112))
        AND CONVERT(INT, CONVERT(VARCHAR, @EndDate, 112))
    GROUP BY t.DateKey, g.LocationName, p.Category
    ORDER BY DwellIdleEfficiencyIndex DESC;
END;
GO
```

### Gravity Zone Recalculation

```sql
CREATE PROCEDURE [dbo].[usp_RecalculateGravityZones]
AS
BEGIN
    -- Update gravity scores based on recent activity
    UPDATE p
    SET 
        GravityScore = (
            (velocity_data.PicksPerDay * 10) + 
            (p.UnitValue / 100) - 
            (CASE p.Fragility 
                WHEN 'High' THEN 5 
                WHEN 'Medium' THEN 2 
                ELSE 0 END)
        ),
        VelocityClass = CASE 
            WHEN velocity_data.PicksPerDay >= 20 THEN 'Fast'
            WHEN velocity_data.PicksPerDay >= 5 THEN 'Medium'
            ELSE 'Slow'
        END,
        RecommendedZone = CASE
            WHEN (velocity_data.PicksPerDay * p.UnitValue) > 1000 THEN 'Zone_A_HighGravity'
            WHEN (velocity_data.PicksPerDay * p.UnitValue) > 300 THEN 'Zone_B_MediumGravity'
            ELSE 'Zone_C_LowGravity'
        END,
        LastRecalculated = GETDATE()
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) / 30.0 AS PicksPerDay
        FROM FactWarehouseOperations
        WHERE OperationType = 'Pick'
            AND OperationTimestamp >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    ) velocity_data ON p.ProductKey = velocity_data.ProductKey;
    
    -- Log the recalculation
    INSERT INTO MaintenanceLog (Activity, RecordsAffected, Timestamp)
    VALUES ('Gravity Zone Recalculation', @@ROWCOUNT, GETDATE());
END;
GO
```

## Power BI Integration

### Connecting to the Data Model

```dax
// In Power BI, create connection to SQL Server
// Data -> Get Data -> SQL Server

// Connection settings (stored in Power BI template):
Server = #"your-server.database.windows.net"
Database = "LogiFleetPulse"

// Load fact and dimension tables
Source = Sql.Database(Server, Database),
FactWarehouseOperations = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
FactFleetTrips = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
DimProductGravity = Source{[Schema="dbo",Item="DimProductGravity"]}[Data],
DimGeography = Source{[Schema="dbo",Item="DimGeography"]}[Data]
```

### Key DAX Measures

```dax
// Average Dwell Time
Avg Dwell Time = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

// Fleet Utilization Rate
Fleet Utilization % = 
VAR TotalTripTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
DIVIDE(TotalTripTime - TotalIdleTime, TotalTripTime, 0) * 100

// Cross-Fact: Dwell Impact on Delivery
Dwell Delivery Impact = 
VAR WarehouseDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        USERELATIONSHIP(FactFleetTrips[OriginKey], DimGeography[GeographyKey])
    )
VAR FleetDelay = AVERAGE(FactFleetTrips[DelayMinutes])
RETURN
WarehouseDwell * FleetDelay / 100

// Warehouse Capacity Utilization
Warehouse Utilization % = 
DIVIDE(
    SUM(DimGeography[CurrentUtilizationCBM]),
    SUM(DimGeography[WarehouseCapacityCBM]),
    0
) * 100

// Gravity Zone Efficiency Score
Gravity Zone Efficiency = 
VAR FastMovers = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[VelocityClass] = "Fast",
        DimProductGravity[RecommendedZone] = "Zone_A_HighGravity"
    )
VAR TotalFastMovers = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[VelocityClass] = "Fast"
    )
RETURN
DIVIDE(FastMovers, TotalFastMovers, 0) * 100

// Predictive Bottleneck Index (time-phased)
Bottleneck Risk Score = 
VAR CurrentUtil = [Warehouse Utilization %]
VAR DwellTrend = 
    CALCULATE(
        [Avg Dwell Time],
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -7, DAY)
    ) / 
    CALCULATE(
        [Avg Dwell Time],
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -30, DAY)
    )
VAR FleetDelayTrend = 
    CALCULATE(
        AVERAGE(FactFleetTrips[DelayMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -7, DAY)
    )
RETURN
(CurrentUtil * 0.4) + (DwellTrend * 100 * 0.3) + (FleetDelayTrend * 0.3)
```

### Time Intelligence Patterns

```dax
// YTD Warehouse Operations
YTD Operations = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    DATESYTD(DimTime[FullDateTime])
)

// Rolling 30-Day Fleet Fuel Consumption
Rolling 30D Fuel = 
CALCULATE(
    SUM(FactFleetTrips[FuelLiters]),
    DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -30, DAY)
)

// Hour-of-Day Analysis (15-minute granularity)
Peak Hour Operations = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FILTER(
        ALL(DimTime[Hour]),
        DimTime[Hour] = HOUR(NOW()) - 1
    )
)
```

## Automated Alerting

```sql
CREATE PROCEDURE [dbo].[usp_CheckAlertsAndNotify]
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @RecipientEmail VARCHAR(500) = '${ALERT_EMAIL}';
    
    -- Check for high dwell time
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations
        WHERE OperationTimestamp >= DATEADD(HOUR, -1, GETDATE())
            AND DwellTimeMinutes > 120
        GROUP BY WarehouseKey
        HAVING AVG(DwellTimeMinutes) > 90
    )
    BEGIN
        SET @AlertMessage = 'ALERT: High warehouse dwell time detected. Average exceeds 90 minutes in the last hour.';
        
        -- Log alert
        INSERT INTO AlertLog (AlertType, Message, Timestamp)
        VALUES ('HighDwellTime', @AlertMessage, GETDATE());
        
        -- Send notification (requires Database Mail configuration)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @RecipientEmail,
            @subject = 'LogiFleet Pulse: High Dwell Time Alert',
            @body = @AlertMessage;
    END
    
    -- Check for fleet idle time exceeding threshold
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips
        WHERE TripStartTime >= DATEADD(HOUR, -2, GETDATE())
        GROUP BY VehicleKey
        HAVING SUM(IdleTimeMinutes) > 30
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeds 30 minutes for one or more vehicles in the last 2 hours.';
        
        INSERT INTO AlertLog (AlertType, Message, Timestamp)
        VALUES ('HighFleetIdle', @AlertMessage, GETDATE());
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @RecipientEmail,
            @subject = 'LogiFleet Pulse: High Fleet Idle Alert',
            @body = @AlertMessage;
    END
END;
GO

-- Schedule this procedure to run every 15 minutes via SQL Server Agent
```

## Common Patterns

### Pattern 1: Cross-Fact Analysis Query

```sql
-- Correlate warehouse performance with delivery delays
SELECT 
    wh_summary.Warehouse,
    wh_summary.AvgDwellTime,
    fleet_summary.AvgDelay,
    CASE 
        WHEN wh_summary.AvgDwellTime > 60 AND fleet_summary.AvgDelay > 15 
        THEN 'High Risk'
        WHEN wh_summary.AvgDwellTime > 40 OR fleet_summary.AvgDelay > 10 
        THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS RiskLevel
FROM (
    SELECT 
        g.LocationName AS Warehouse,
        AVG(wh.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations wh
    INNER JOIN DimGeography g ON wh.WarehouseKey = g.GeographyKey
    WHERE wh.OperationTimestamp >= DATEADD(DAY, -7, GETDATE())
    GROUP BY g.LocationName
) wh_summary
INNER JOIN (
    SELECT 
        g.LocationName AS Warehouse,
        AVG(fl.DelayMinutes) AS AvgDelay
    FROM FactFleetTrips fl
    INNER JOIN DimGeography g ON fl.OriginKey = g.GeographyKey
    WHERE fl.TripStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY g.LocationName
) fleet_summary ON wh_summary.Warehouse = fleet_summary.Warehouse
ORDER BY RiskLevel DESC, wh_summary.AvgDwellTime DESC;
```

### Pattern 2: Gravity Zone Optimization Analysis

```sql
-- Identify products in wrong gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.VelocityClass,
    p.RecommendedZone,
    wh.CurrentZone,
    COUNT(*) AS PickOperations,
    AVG(wh.CycleTimeMinutes) AS AvgCycleTime,
    CASE 
        WHEN p.RecommendedZone <> wh.CurrentZone THEN 'Requires Relocation'
        ELSE 'Optimized'
    END AS Status
FROM DimProductGravity p
INNER JOIN (
    SELECT 
        ProductKey,
        GravityZoneID AS CurrentZone,
        AVG(CycleTimeMinutes) AS CycleTimeMinutes
    FROM FactWarehouseOperations
    WHERE OperationType = 'Pick'
        AND OperationTimestamp >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ProductKey, GravityZoneID
) wh ON p.ProductKey = wh.ProductKey
WHERE p.VelocityClass = 'Fast'
ORDER BY PickOperations DESC;
```

### Pattern 3: Time-Phased Fleet Analysis

```sql
-- Analyze fleet performance by time of day and day of week
SELECT 
    t.DayOfWeek,
    t.Hour,
    t.ShiftName,
    COUNT(fl.TripID) AS TotalTrips,
    AVG(fl.IdleTimeMinutes) AS AvgIdleTime,
    AVG(fl.FuelLiters) AS AvgFuelConsumption,
    SUM(fl.DelayMinutes) AS TotalDelayMinutes,
    AVG(fl.LoadWeightKG) AS AvgLoadWeight
FROM FactFleetTrips fl
INNER JOIN DimTime t ON fl.TimeKey = t.TimeKey
WHERE fl.TripStartTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY t.DayOfWeek, t.Hour, t.ShiftName
ORDER BY AvgIdleTime DESC;
```

## Configuration

### Row-Level Security Setup

```sql
-- Create security roles for different user groups
CREATE ROLE WarehouseManagers;
CREATE ROLE FleetOperators;
CREATE ROLE Executives;

-- Create security predicate function
CREATE FUNCTION [dbo].[fn_SecurityPredicate](@UserRole AS VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessGranted
WHERE 
    @UserRole = USER_NAME() 
    OR IS_ROLEMEMBER('Executives') = 1
    OR (
        IS_ROLEMEMBER('WarehouseManagers') = 1 
        AND @UserRole LIKE 'Warehouse%'
    )
    OR (
        IS_ROLEMEMBER('FleetOperators') = 1 
        AND @UserRole LIKE 'Fleet%'
    );
GO

-- Apply security policy to fact tables
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE [dbo].[fn_SecurityPredicate](WarehouseKey)
ON FactWarehouseOperations
WITH (STATE = ON);
```

### External Data Source Configuration

```sql
-- Configure external data source for streaming telemetry
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${AZURE_STORAGE_ENDPOINT}',
    CREDENTIAL = FleetAPICredential
);

-- Create external table for real-time fleet data
CREATE EXTERNAL TABLE Ext_FleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = FleetTelemetryAPI,
    LOCATION = '/telemetry/feed',
    FILE_FORMAT = JSONFormat
);
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom**: Queries joining FactWarehouseOperations and FactFleetTrips take > 10 seconds

**Solution**:
```sql
-- Add columnstore indexes for analytical queries
CREATE COLUMNSTORE INDEX CIX_FactWH_Analytics 
ON FactWarehouseOperations(TimeKey, ProductKey, WarehouseKey, Quantity, DwellTimeMinutes);

CREATE COLUMNSTORE INDEX CIX_FactFleet_Analytics 
ON FactFleetTrips(TimeKey, OriginKey, DestinationKey, FuelLiters, IdleTimeMinutes);

-- Partition large fact tables by date
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT PK_FactWH PRIMARY KEY (OperationID, OperationTimestamp);

CREATE PARTITION FUNCTION PF_Monthly (DATETIME2)
AS RANGE RIGHT FOR VALUES ('2025-01-01', '2025-02-01', '2025-03-01');

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme (requires recreate)
```

### Issue: Power BI Refresh Failures

**Symptom**: Scheduled refresh fails with timeout errors

**Solution**:
```sql
-- Create indexed views for complex aggregations
CREATE VIEW vw_DailyWarehouseSummary
WITH SCHEMABINDING
AS
SELECT 
    t.DateKey,
    wh.WarehouseKey,
    p.Category,
    COUNT_BIG(*) AS OperationCount,
    SUM(wh.Quantity) AS TotalQuantity,
    AVG(wh.DwellTimeMinutes) AS AvgDwellTime
FROM dbo.FactWarehouseOperations wh
INNER JOIN dbo.DimTime t ON wh.TimeKey = t.TimeKey
INNER JOIN dbo.DimProductGravity p ON wh.ProductKey = p.ProductKey
GROUP BY t.DateKey, wh.WarehouseKey, p.Category;
GO

CREATE UNIQUE CLUSTERED INDEX IX_DailyWHSummary 
ON vw_DailyWarehouseSummary(DateKey, WarehouseKey, Category);

-- In Power BI, point to view instead of fact table for summary visuals
```

### Issue: Incorrect Gravity Zone Recommendations

**Symptom**: Products with high pick frequency not assigned to high-gravity zones

**Solution**:
```sql
-- Verify velocity calculation includes recent data
SELECT 
    p.SKU,
    p.VelocityClass,
    p.GravityScore,
    p.LastRecalculated,
    recent_picks.PickCount
FROM DimProductGravity p
LEFT JOIN (
    SELECT ProductKey, COUNT(*) AS PickCount
    FROM FactWarehouseOperations
    WHERE OperationType = 'Pick'
        AND OperationTimestamp >= DATEADD(DAY, -7, GETDATE())
    GROUP BY ProductKey
) recent_picks ON p.ProductKey = recent_picks.ProductKey
WHERE p.LastRecalculated < DATEADD(DAY, -7, GETDATE())
    OR recent_picks.PickCount > 50;

-- Force recalculation
EXEC usp_RecalculateGravityZones;
```

### Issue: Missing Time Dimension Records

**Symptom**: Fact table inserts fail due to missing TimeKey

**Solution**:
```sql
-- Populate DimTime with future dates (run quarterly)
DECLARE @StartDate DATETIME2 = GETDATE();
DECLARE @EndDate DATETIME2 = DATEADD(MONTH, 3, GETDATE());
DECLARE @CurrentDateTime DATETIME2 = @StartDate;

WHILE @CurrentDateTime < @EndDate
