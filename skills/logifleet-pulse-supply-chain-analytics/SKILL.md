---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI dashboards for logistics, fleet management, and supply chain KPI visualization
triggers:
  - how do I set up the LogiFleet Pulse data warehouse
  - configure Power BI for logistics analytics
  - create a supply chain dashboard with warehouse and fleet data
  - implement multi-fact star schema for logistics
  - connect fleet telemetry to SQL Server warehouse
  - build logistics KPI reports in Power BI
  - set up warehouse gravity zone analytics
  - query cross-dock and fleet operations data
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that combines MS SQL Server data warehousing with Power BI visualization for supply chain analytics. It provides a unified semantic layer connecting warehouse operations, fleet telemetry, inventory management, and external data sources through a custom multi-fact star schema.

**Core Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time dashboards with 15-minute refresh intervals
- Predictive bottleneck detection and fleet triage
- Warehouse Gravity Zones™ spatial optimization
- Role-based access control and multi-language support

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telemetry APIs, ERP systems

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Run the schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the complete schema script
-- (assuming schema.sql is in the repository)
```

3. **Configure data source connections:**
```json
{
  "connections": {
    "wms_api": {
      "type": "REST",
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}"
    },
    "fleet_telemetry": {
      "type": "SQL",
      "server": "${FLEET_DB_SERVER}",
      "database": "${FLEET_DB_NAME}",
      "username": "${FLEET_DB_USER}",
      "password": "${FLEET_DB_PASSWORD}"
    },
    "erp_system": {
      "type": "ODBC",
      "connection_string": "${ERP_CONNECTION_STRING}"
    }
  }
}
```

### Power BI Template Setup

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection parameters when prompted
3. Configure data refresh schedule (recommended: every 15 minutes)
4. Publish to Power BI Service for team access

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeID INT NOT NULL,
    ProductID INT NOT NULL,
    WarehouseZoneID INT NOT NULL,
    OperationType VARCHAR(50), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes DECIMAL(10,2),
    PickRate DECIMAL(10,2),
    PackingTimeSeconds INT,
    EmployeeID INT,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeID) REFERENCES DimTime(TimeID),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductID) REFERENCES DimProductGravity(ProductID)
);

CREATE NONCLUSTERED INDEX IX_WH_Time_Product ON FactWarehouseOperations(TimeID, ProductID);
CREATE NONCLUSTERED INDEX IX_WH_Zone_Type ON FactWarehouseOperations(WarehouseZoneID, OperationType);
```

**FactFleetTrips** - Fleet telemetry and route data
```sql
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeID INT NOT NULL,
    VehicleID INT NOT NULL,
    RouteID INT NOT NULL,
    GeographyID INT NOT NULL,
    FuelConsumptionLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    DistanceKM DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    DriverID INT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeID) REFERENCES DimTime(TimeID),
    CONSTRAINT FK_Fleet_Geography FOREIGN KEY (GeographyID) REFERENCES DimGeography(GeographyID)
);

CREATE NONCLUSTERED INDEX IX_Fleet_Time_Vehicle ON FactFleetTrips(TimeID, VehicleID);
CREATE NONCLUSTERED INDEX IX_Fleet_Route ON FactFleetTrips(RouteID);
```

**FactCrossDock** - Cross-dock transfer operations
```sql
CREATE TABLE FactCrossDock (
    CrossDockID INT PRIMARY KEY IDENTITY(1,1),
    TimeID INT NOT NULL,
    ProductID INT NOT NULL,
    InboundTripID INT,
    OutboundTripID INT,
    TransferTimeMinutes DECIMAL(10,2),
    TemperatureCompliant BIT,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeID) REFERENCES DimTime(TimeID),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductID) REFERENCES DimProductGravity(ProductID)
);
```

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity
```sql
CREATE TABLE DimTime (
    TimeID INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL, -- YYYYMMDD format
    TimeKey INT NOT NULL, -- HHMM format (15-min buckets: 0000, 0015, 0030, 0045)
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    WeekOfYear TINYINT,
    MonthNumber TINYINT,
    QuarterNumber TINYINT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);

CREATE UNIQUE INDEX UX_Time_DateTime ON DimTime(FullDateTime);
```

**DimProductGravity** - Products with gravity scoring
```sql
CREATE TABLE DimProductGravity (
    ProductID INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    CategoryLevel1 VARCHAR(100),
    CategoryLevel2 VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * (1/fragility)
    VelocityClass VARCHAR(20), -- 'FAST', 'MEDIUM', 'SLOW'
    ValueTier VARCHAR(20), -- 'HIGH', 'MEDIUM', 'LOW'
    FragilityIndex DECIMAL(3,2),
    OptimalZoneID INT
);
```

**DimGeography** - Hierarchical geography dimension
```sql
CREATE TABLE DimGeography (
    GeographyID INT PRIMARY KEY IDENTITY(1,1),
    ContinentName VARCHAR(50),
    CountryName VARCHAR(100),
    RegionName VARCHAR(100),
    WarehouseCode VARCHAR(20),
    RouteNodeCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);
```

## Common Queries and Patterns

### Cross-Fact Analysis: Dwell Time vs Fleet Idling

```sql
-- Identify correlation between warehouse dwell time and fleet idling
-- for same products on same days
SELECT 
    p.SKU,
    p.ProductName,
    t.DateKey,
    AVG(wh.DwellTimeMinutes) AS AvgDwellTime,
    AVG(fl.IdleTimeMinutes) AS AvgFleetIdleTime,
    COUNT(DISTINCT wh.OperationID) AS WarehouseOps,
    COUNT(DISTINCT fl.TripID) AS FleetTrips
FROM FactWarehouseOperations wh
INNER JOIN DimTime t ON wh.TimeID = t.TimeID
INNER JOIN DimProductGravity p ON wh.ProductID = p.ProductID
LEFT JOIN FactFleetTrips fl ON fl.TimeID = t.TimeID
    AND fl.TimeID IN (
        SELECT TimeID FROM FactWarehouseOperations 
        WHERE ProductID = wh.ProductID
    )
WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd'))
GROUP BY p.SKU, p.ProductName, t.DateKey
HAVING AVG(wh.DwellTimeMinutes) > 72 -- More than 3 days dwell time
ORDER BY AvgDwellTime DESC, AvgFleetIdleTime DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend zone reassignments based on current performance
WITH ProductPerformance AS (
    SELECT 
        wh.ProductID,
        p.SKU,
        p.GravityScore,
        p.OptimalZoneID,
        wh.WarehouseZoneID AS CurrentZoneID,
        AVG(wh.PickRate) AS AvgPickRate,
        AVG(wh.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wh
    INNER JOIN DimProductGravity p ON wh.ProductID = p.ProductID
    INNER JOIN DimTime t ON wh.TimeID = t.TimeID
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd'))
        AND wh.OperationType = 'PICK'
    GROUP BY wh.ProductID, p.SKU, p.GravityScore, p.OptimalZoneID, wh.WarehouseZoneID
)
SELECT 
    SKU,
    GravityScore,
    CurrentZoneID,
    OptimalZoneID,
    AvgPickRate,
    AvgDwellTime,
    CASE 
        WHEN CurrentZoneID <> OptimalZoneID AND AvgPickRate > 10 THEN 'HIGH_PRIORITY_MOVE'
        WHEN CurrentZoneID <> OptimalZoneID THEN 'CONSIDER_MOVE'
        ELSE 'OPTIMAL'
    END AS RecommendedAction
FROM ProductPerformance
WHERE CurrentZoneID <> OptimalZoneID
ORDER BY GravityScore DESC, AvgPickRate DESC;
```

### Predictive Bottleneck Detection

```sql
-- Detect emerging bottlenecks by comparing current vs historical patterns
WITH CurrentMetrics AS (
    SELECT 
        WarehouseZoneID,
        AVG(DwellTimeMinutes) AS CurrentAvgDwell,
        AVG(PickRate) AS CurrentAvgPickRate
    FROM FactWarehouseOperations wh
    INNER JOIN DimTime t ON wh.TimeID = t.TimeID
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd'))
    GROUP BY WarehouseZoneID
),
HistoricalBaseline AS (
    SELECT 
        WarehouseZoneID,
        AVG(DwellTimeMinutes) AS BaselineAvgDwell,
        STDEV(DwellTimeMinutes) AS StdDevDwell,
        AVG(PickRate) AS BaselineAvgPickRate,
        STDEV(PickRate) AS StdDevPickRate
    FROM FactWarehouseOperations wh
    INNER JOIN DimTime t ON wh.TimeID = t.TimeID
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd'))
        AND t.DateKey < CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd'))
    GROUP BY WarehouseZoneID
)
SELECT 
    c.WarehouseZoneID,
    c.CurrentAvgDwell,
    h.BaselineAvgDwell,
    c.CurrentAvgPickRate,
    h.BaselineAvgPickRate,
    (c.CurrentAvgDwell - h.BaselineAvgDwell) / NULLIF(h.StdDevDwell, 0) AS DwellZScore,
    (h.BaselineAvgPickRate - c.CurrentAvgPickRate) / NULLIF(h.StdDevPickRate, 0) AS PickRateZScore,
    CASE 
        WHEN (c.CurrentAvgDwell - h.BaselineAvgDwell) / NULLIF(h.StdDevDwell, 0) > 2 THEN 'CRITICAL_BOTTLENECK'
        WHEN (c.CurrentAvgDwell - h.BaselineAvgDwell) / NULLIF(h.StdDevDwell, 0) > 1.5 THEN 'EMERGING_BOTTLENECK'
        ELSE 'NORMAL'
    END AS BottleneckStatus
FROM CurrentMetrics c
INNER JOIN HistoricalBaseline h ON c.WarehouseZoneID = h.WarehouseZoneID
WHERE (c.CurrentAvgDwell - h.BaselineAvgDwell) / NULLIF(h.StdDevDwell, 0) > 1.5
ORDER BY DwellZScore DESC;
```

### Fleet Maintenance Triage with Revenue Impact

```sql
-- Prioritize fleet maintenance based on load value and issue severity
WITH FleetIssues AS (
    SELECT 
        ft.VehicleID,
        AVG(ft.FuelConsumptionLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelPerKM,
        AVG(ft.IdleTimeMinutes / NULLIF((ft.IdleTimeMinutes + 60), 0)) AS IdleTimeRatio,
        SUM(ft.LoadWeightKG) AS TotalLoadKG,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeID = t.TimeID
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd'))
    GROUP BY ft.VehicleID
),
VehicleRevenue AS (
    SELECT 
        ft.VehicleID,
        SUM(p.ValueTier) AS EstimatedRevenueImpact -- Simplified; would use actual revenue data
    FROM FactFleetTrips ft
    INNER JOIN FactCrossDock cd ON ft.TripID = cd.OutboundTripID
    INNER JOIN DimProductGravity p ON cd.ProductID = p.ProductID
    GROUP BY ft.VehicleID
)
SELECT 
    fi.VehicleID,
    fi.AvgFuelPerKM,
    fi.IdleTimeRatio,
    fi.TotalLoadKG,
    fi.TripCount,
    COALESCE(vr.EstimatedRevenueImpact, 0) AS RevenueAtRisk,
    CASE 
        WHEN fi.AvgFuelPerKM > 0.15 THEN 'FUEL_EFFICIENCY_ISSUE'
        WHEN fi.IdleTimeRatio > 0.20 THEN 'EXCESSIVE_IDLING'
        ELSE 'MONITOR'
    END AS IssueType,
    (COALESCE(vr.EstimatedRevenueImpact, 0) * 
     (fi.AvgFuelPerKM / 0.1) * 
     (fi.IdleTimeRatio / 0.1)) AS PriorityScore
FROM FleetIssues fi
LEFT JOIN VehicleRevenue vr ON fi.VehicleID = vr.VehicleID
WHERE fi.AvgFuelPerKM > 0.12 OR fi.IdleTimeRatio > 0.15
ORDER BY PriorityScore DESC;
```

## Stored Procedures for Incremental Loading

### Warehouse Operations ETL

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if no parameter provided
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -24, GETDATE());
    
    -- Insert new warehouse operations
    INSERT INTO FactWarehouseOperations (
        TimeID, ProductID, WarehouseZoneID, OperationType,
        DwellTimeMinutes, PickRate, PackingTimeSeconds, EmployeeID
    )
    SELECT 
        t.TimeID,
        p.ProductID,
        wz.ZoneID,
        src.OperationType,
        src.DwellTimeMinutes,
        src.PickRate,
        src.PackingTimeSeconds,
        src.EmployeeID
    FROM ExternalWarehouseData src -- External table or staging
    INNER JOIN DimTime t ON t.FullDateTime = DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, '2000-01-01', src.OperationDateTime) / 15) * 15, 
        '2000-01-01') -- Round to 15-minute bucket
    INNER JOIN DimProductGravity p ON p.SKU = src.SKU
    INNER JOIN WarehouseZones wz ON wz.ZoneCode = src.ZoneCode
    WHERE src.OperationDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations existing
            WHERE existing.TimeID = t.TimeID
                AND existing.ProductID = p.ProductID
                AND existing.OperationType = src.OperationType
                AND existing.EmployeeID = src.EmployeeID
        );
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' warehouse operations.';
END;
GO
```

### Fleet Telemetry ETL

```sql
CREATE PROCEDURE sp_LoadFleetTelemetry
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -24, GETDATE());
    
    INSERT INTO FactFleetTrips (
        TimeID, VehicleID, RouteID, GeographyID,
        FuelConsumptionLiters, IdleTimeMinutes, DistanceKM, LoadWeightKG, DriverID
    )
    SELECT 
        t.TimeID,
        v.VehicleID,
        r.RouteID,
        g.GeographyID,
        tel.FuelConsumption,
        tel.IdleTime,
        tel.Distance,
        tel.LoadWeight,
        d.DriverID
    FROM ExternalFleetTelemetry tel
    INNER JOIN DimTime t ON t.FullDateTime = DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, '2000-01-01', tel.TripDateTime) / 15) * 15, 
        '2000-01-01')
    INNER JOIN Vehicles v ON v.VehicleCode = tel.VehicleCode
    INNER JOIN Routes r ON r.RouteCode = tel.RouteCode
    INNER JOIN DimGeography g ON g.RouteNodeCode = tel.EndLocationCode
    LEFT JOIN Drivers d ON d.DriverCode = tel.DriverCode
    WHERE tel.TripDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactFleetTrips existing
            WHERE existing.TimeID = t.TimeID
                AND existing.VehicleID = v.VehicleID
                AND existing.RouteID = r.RouteID
        );
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' fleet trips.';
END;
GO
```

## Automated Alerting System

```sql
CREATE PROCEDURE sp_CheckAndSendAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessages TABLE (
        AlertType VARCHAR(50),
        Severity VARCHAR(20),
        Message VARCHAR(500),
        MetricValue DECIMAL(10,2)
    );
    
    -- Alert 1: Excessive fleet idling
    INSERT INTO @AlertMessages
    SELECT 
        'FLEET_IDLING',
        'HIGH',
        'Vehicle ' + CAST(VehicleID AS VARCHAR) + ' idle time exceeded 15% threshold',
        (IdleTimeMinutes * 100.0 / NULLIF((IdleTimeMinutes + TripActiveMinutes), 0))
    FROM (
        SELECT 
            VehicleID,
            SUM(IdleTimeMinutes) AS IdleTimeMinutes,
            SUM(DistanceKM / 60.0) AS TripActiveMinutes -- Simplified calculation
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeID = t.TimeID
        WHERE t.DateKey = CONVERT(INT, FORMAT(GETDATE(), 'yyyyMMdd'))
        GROUP BY VehicleID
    ) x
    WHERE (IdleTimeMinutes * 100.0 / NULLIF((IdleTimeMinutes + TripActiveMinutes), 0)) > 15;
    
    -- Alert 2: Cold storage dwell time exceeded
    INSERT INTO @AlertMessages
    SELECT 
        'DWELL_TIME_CRITICAL',
        'CRITICAL',
        'SKU ' + p.SKU + ' in cold storage exceeds 72 hours',
        AVG(wh.DwellTimeMinutes)
    FROM FactWarehouseOperations wh
    INNER JOIN DimProductGravity p ON wh.ProductID = p.ProductID
    INNER JOIN DimTime t ON wh.TimeID = t.TimeID
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -3, GETDATE()), 'yyyyMMdd'))
        AND wh.WarehouseZoneID IN (SELECT ZoneID FROM WarehouseZones WHERE ZoneType = 'COLD_STORAGE')
        AND wh.DwellTimeMinutes > 72 * 60
    GROUP BY p.SKU
    HAVING AVG(wh.DwellTimeMinutes) > 72 * 60;
    
    -- Send alerts (integrate with email/Teams/SMS service)
    IF EXISTS (SELECT 1 FROM @AlertMessages)
    BEGIN
        -- Example: Log to alerts table for dashboard display
        INSERT INTO AlertLog (AlertDateTime, AlertType, Severity, Message, MetricValue)
        SELECT GETDATE(), AlertType, Severity, Message, MetricValue
        FROM @AlertMessages;
        
        -- TODO: Integrate with external notification service
        -- EXEC msdb.dbo.sp_send_dbmail for email alerts
    END;
END;
GO

-- Schedule this to run every 15 minutes via SQL Server Agent
```

## Power BI DAX Measures

### Cross-Fact KPI: Dwell Time per Fleet Trip

```dax
// In Power BI Data Model
AvgDwellTimePerTrip = 
VAR CurrentTimeContext = SELECTEDVALUE(DimTime[TimeID])
VAR WarehouseDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FILTER(
            ALL(FactWarehouseOperations),
            FactWarehouseOperations[TimeID] = CurrentTimeContext
        )
    )
VAR FleetTripCount = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FILTER(
            ALL(FactFleetTrips),
            FactFleetTrips[TimeID] = CurrentTimeContext
        )
    )
RETURN
    DIVIDE(WarehouseDwell, FleetTripCount, 0)
```

### Gravity Zone Performance Index

```dax
GravityZoneEfficiency = 
VAR ExpectedPickRate = 
    CALCULATE(
        AVERAGE(DimProductGravity[GravityScore]) * 10
    )
VAR ActualPickRate = 
    AVERAGE(FactWarehouseOperations[PickRate])
RETURN
    DIVIDE(ActualPickRate, ExpectedPickRate, 0) * 100
```

### Fleet Cost per Delivery

```dax
CostPerDelivery = 
VAR TotalFuelCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelConsumptionLiters] * 1.50 // Assume $1.50/liter
    )
VAR TotalDeliveries = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] = "SHIP"
    )
RETURN
    DIVIDE(TotalFuelCost, TotalDeliveries, 0)
```

## Configuration Best Practices

### Row-Level Security (RLS)

```sql
-- Create security roles table
CREATE TABLE UserRoles (
    UserEmail VARCHAR(200),
    RoleType VARCHAR(50), -- 'WAREHOUSE_MANAGER', 'FLEET_SUPERVISOR', 'EXECUTIVE'
    GeographyFilter INT, -- NULL = all geographies
    WarehouseZoneFilter INT -- NULL = all zones
);

-- In Power BI, create RLS roles:
-- Role: WarehouseManager
[UserEmail] = USERPRINCIPALNAME() 
&& DimGeography[GeographyID] IN (
    SELECT GeographyFilter 
    FROM UserRoles 
    WHERE UserEmail = USERPRINCIPALNAME() 
        AND RoleType = 'WAREHOUSE_MANAGER'
)
```

### Performance Tuning

```sql
-- Partition large fact tables by date
CREATE PARTITION FUNCTION PF_DateKey (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601
    -- Add monthly partitions
);

CREATE PARTITION SCHEME PS_DateKey
AS PARTITION PF_DateKey
ALL TO ([PRIMARY]);

-- Apply to fact tables
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- Same structure as above
) ON PS_DateKey(DateKey);

-- Columnstore indexes for analytical queries
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOps
ON FactWarehouseOperations;

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips
ON FactFleetTrips;
```

## Troubleshooting

### Issue: Power BI refresh fails with timeout

**Solution:** Implement incremental refresh in Power BI:
```powerquery
// In Power Query, create RangeStart and RangeEnd parameters
let
    Source = Sql.Database("${SQL_SERVER}", "${DATABASE}"),
    FilteredData = Table.SelectRows(Source, 
        each [OperationDateTime] >= RangeStart and [OperationDateTime] < RangeEnd
    )
in
    FilteredData
```

### Issue: Slow cross-fact queries

**Solution:** Create indexed views for common joins:
```sql
CREATE VIEW vw_WarehouseFleetJoin
WITH SCHEMABINDING
AS
SELECT 
    wh.TimeID,
    wh.ProductID,
    wh.DwellTimeMinutes,
    fl.VehicleID,
    fl.IdleTimeMinutes,
    COUNT_BIG(*) AS RecordCount
FROM dbo.FactWarehouseOperations wh
INNER JOIN dbo.FactFleetTrips fl ON wh.TimeID = fl.TimeID
GROUP BY wh.TimeID, wh.ProductID, wh.DwellTimeMinutes, fl.VehicleID, fl.IdleTimeMinutes;
GO

CREATE UNIQUE CLUSTERED INDEX IX_WHFleet ON vw_WarehouseFleetJoin(TimeID, ProductID, VehicleID);
```

### Issue: Gravity score calculation outdated

**Solution:** Schedule recalculation job:
```sql
CREATE PROCEDURE sp_RecalculateGravityScores
AS
BEGIN
    UPDATE p
    SET GravityScore = (
        (PickFrequency / 100.0) * 
        (ValueTier / 10.0) * 
        (1.0 / NULLIF(FragilityIndex, 0))
    )
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductID,
            COUNT(*) AS PickFrequency
        FROM FactWarehouseOperations
        WHERE OperationType = 'PICK'
            AND TimeID >= (SELECT MAX(TimeID) - 4320 FROM DimTime) -- Last 30 days
        GROUP BY ProductID
    ) freq ON p.ProductID = freq.ProductID;
END;
GO

-- Schedule via SQL Server Agent to run daily
```

## Environment Variables Reference

- `${SQL_SERVER}` - SQL Server hostname or IP
- `${DATABASE}` - Database name (default: LogiFleetPulse)
- `${WMS_API_ENDPOINT}` - Warehouse Management System API URL
- `${WMS_API_TOKEN}` - API authentication token
- `${FLEET_DB_SERVER}` - Fleet telemetry database server
- `${FLEET_DB_NAME}` - Fleet database name
- `${FLEET_DB_USER}` - Fleet database username
- `${FLEET_DB_PASSWORD}` - Fleet database password
- `${ERP_CONNECTION_STRING}` - ERP system ODBC connection string
- `${POWER_BI_WORKSPACE}` - Power BI workspace ID for publishing
