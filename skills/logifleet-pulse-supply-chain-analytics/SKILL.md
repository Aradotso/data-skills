---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for logistics, fleet management, and supply chain KPI analytics
triggers:
  - "set up supply chain analytics dashboard"
  - "create logistics data warehouse schema"
  - "implement fleet management KPIs in Power BI"
  - "build warehouse operations analytics"
  - "configure multi-fact star schema for logistics"
  - "deploy LogiFleet Pulse analytics platform"
  - "integrate warehouse and fleet telemetry data"
  - "create cross-modal supply chain reports"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to work with LogiFleet Pulse, an advanced MS SQL Server and Power BI data warehousing solution for supply chain analytics. The platform integrates warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer using a multi-fact star schema architecture.

## What LogiFleet Pulse Does

LogiFleet Pulse is a logistics intelligence platform that:

- **Unifies multi-source data**: Warehouse management systems, GPS/telematics, supplier portals, weather/traffic APIs, and customer orders
- **Implements multi-fact star schema**: Links warehouse operations, fleet trips, and cross-dock activities through shared dimensions
- **Provides real-time dashboards**: Power BI visualizations refreshed every 15 minutes
- **Enables cross-fact analysis**: Queries spanning warehouse dwell time, fleet efficiency, and supplier reliability
- **Supports predictive analytics**: Bottleneck detection, maintenance prioritization, and capacity simulation

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (2022 recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Access to source systems: WMS, TMS, telematics APIs

### Database Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema**:
```sql
-- Connect to your SQL Server instance
-- Execute schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_procedures.sql
```

3. **Configure data connections**:
```json
{
  "sql_server": {
    "server": "your-server.database.windows.net",
    "database": "LogiFleetPulse",
    "authentication": "SQL",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_connection": "${WMS_CONNECTION_STRING}",
    "telemetry_api": "${TELEMETRY_API_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  }
}
```

4. **Import Power BI template**:
```powershell
# Open Power BI Desktop
# File → Import → Power BI Template (.pbit)
# Navigate to: templates/LogiFleet_Pulse_Master.pbit
# Enter SQL Server connection details when prompted
```

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time tracking
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    DateKey INT,
    TimeSlot TIME,
    HourOfDay INT,
    DayOfWeek NVARCHAR(10),
    FiscalPeriod NVARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
);

-- DimGeography: Hierarchical location structure
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID NVARCHAR(50),
    LocationName NVARCHAR(100),
    LocationType NVARCHAR(20), -- Warehouse, RouteNode, CrossDock
    ParentLocationKey INT,
    Region NVARCHAR(50),
    Country NVARCHAR(50),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU NVARCHAR(50),
    ProductName NVARCHAR(200),
    Category NVARCHAR(50),
    GravityScore DECIMAL(5,2), -- Weighted score: velocity + value + fragility
    VelocityClass NVARCHAR(20), -- Fast, Medium, Slow
    ValueTier NVARCHAR(20),
    FragilityIndex INT,
    OptimalZoneType NVARCHAR(50)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierID NVARCHAR(50),
    SupplierName NVARCHAR(100),
    LeadTimeMean INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier NVARCHAR(20)
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType NVARCHAR(20), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationDuration INT, -- Minutes
    DwellTime INT, -- Minutes from receiving to shipping
    ZoneType NVARCHAR(50),
    EmployeeID NVARCHAR(50),
    OrderID NVARCHAR(50),
    UnitCount INT,
    ErrorFlag BIT
);

-- FactFleetTrips: Fleet telemetry and route data
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    TripDuration INT, -- Minutes
    DistanceMiles DECIMAL(10,2),
    FuelConsumed DECIMAL(10,2), -- Gallons
    IdleTime INT, -- Minutes
    LoadWeight DECIMAL(10,2), -- Pounds
    TemperatureCompliance BIT,
    DelayMinutes INT,
    DelayReason NVARCHAR(100)
);

-- FactCrossDock: Cross-docking operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TransferDuration INT, -- Minutes
    UnitCount INT,
    BypassStorage BIT
);
```

## Key SQL Queries & Patterns

### Cross-Fact KPI Query

```sql
-- Analyze dwell time correlation with fleet idling cost per route
WITH WarehouseDwell AS (
    SELECT 
        w.GeographyKey,
        w.ProductKey,
        AVG(w.DwellTime) AS AvgDwellTimeMin,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateKey BETWEEN 20260601 AND 20260630
    GROUP BY w.GeographyKey, w.ProductKey
),
FleetIdle AS (
    SELECT 
        f.OriginGeographyKey AS GeographyKey,
        SUM(f.IdleTime) AS TotalIdleTimeMin,
        SUM(f.FuelConsumed * 3.50) AS EstimatedIdleCost -- $3.50/gallon
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateKey BETWEEN 20260601 AND 20260630
    GROUP BY f.OriginGeographyKey
)
SELECT 
    g.LocationName AS Warehouse,
    p.ProductName,
    p.GravityScore,
    wd.AvgDwellTimeMin,
    fi.TotalIdleTimeMin,
    fi.EstimatedIdleCost,
    (wd.AvgDwellTimeMin * fi.TotalIdleTimeMin) / 
        NULLIF(wd.OperationCount, 0) AS DwellIdleCorrelation
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.GeographyKey = fi.GeographyKey
INNER JOIN DimGeography g ON wd.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wd.ProductKey = p.ProductKey
WHERE wd.AvgDwellTimeMin > 72 -- Focus on items dwelling 72+ hours
ORDER BY DwellIdleCorrelation DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products misplaced in non-optimal zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneType,
    w.ZoneType AS CurrentZone,
    AVG(w.OperationDuration) AS AvgPickTimeMin,
    COUNT(*) AS PickCount
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE w.OperationType = 'Picking'
    AND w.ZoneType <> p.OptimalZoneType
    AND w.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZoneType, w.ZoneType
HAVING AVG(w.OperationDuration) > 5 -- Picks taking longer than 5 min
ORDER BY p.GravityScore DESC, PickCount DESC;
```

### Fleet Maintenance Priority Scoring

```sql
-- Generate proactive maintenance queue based on telemetry and revenue impact
WITH VehicleMetrics AS (
    SELECT 
        f.VehicleID,
        AVG(f.FuelConsumed / NULLIF(f.DistanceMiles, 0)) AS AvgFuelEfficiency,
        SUM(CASE WHEN f.IdleTime > 30 THEN 1 ELSE 0 END) AS ExcessiveIdleCount,
        SUM(CASE WHEN f.TemperatureCompliance = 0 THEN 1 ELSE 0 END) AS TempViolations,
        AVG(f.LoadWeight) AS AvgLoadWeight
    FROM FactFleetTrips f
    WHERE f.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days
    GROUP BY f.VehicleID
),
RevenueImpact AS (
    SELECT 
        f.VehicleID,
        SUM(p.GravityScore * w.UnitCount) AS TotalRevenueRisk
    FROM FactFleetTrips f
    INNER JOIN FactWarehouseOperations w ON f.TimeKey = w.TimeKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE f.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
    GROUP BY f.VehicleID
)
SELECT 
    vm.VehicleID,
    vm.AvgFuelEfficiency,
    vm.ExcessiveIdleCount,
    vm.TempViolations,
    ri.TotalRevenueRisk,
    (vm.ExcessiveIdleCount * 10 + vm.TempViolations * 50 + 
     (1 / NULLIF(vm.AvgFuelEfficiency, 0)) * 100) * 
     (ri.TotalRevenueRisk / 1000) AS MaintenancePriorityScore
FROM VehicleMetrics vm
INNER JOIN RevenueImpact ri ON vm.VehicleID = ri.VehicleID
ORDER BY MaintenancePriorityScore DESC;
```

## Stored Procedures for ETL

### Incremental Load Pattern

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimeKey INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Extract from WMS staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationDuration, DwellTime, ZoneType, EmployeeID,
        OrderID, UnitCount, ErrorFlag
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        stg.OperationType,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS OperationDuration,
        DATEDIFF(MINUTE, stg.ReceiveTime, stg.ShipTime) AS DwellTime,
        stg.ZoneType,
        stg.EmployeeID,
        stg.OrderID,
        stg.UnitCount,
        stg.ErrorFlag
    FROM WMS_Staging_Operations stg
    INNER JOIN DimTime t ON CAST(stg.StartTime AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, stg.StartTime) = t.HourOfDay
        AND DATEPART(MINUTE, stg.StartTime) / 15 = DATEPART(MINUTE, t.TimeSlot) / 15
    INNER JOIN DimGeography g ON stg.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    WHERE stg.LoadedFlag = 0
        AND t.TimeKey > @LastLoadTimeKey;
    
    -- Mark staging records as loaded
    UPDATE WMS_Staging_Operations
    SET LoadedFlag = 1
    WHERE LoadedFlag = 0;
    
    -- Return new max TimeKey
    SELECT MAX(TimeKey) AS NewLastLoadTimeKey
    FROM FactWarehouseOperations;
END;
GO
```

### Alert Generation

```sql
CREATE PROCEDURE usp_GenerateFleetAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check for excessive idle time threshold breach
    INSERT INTO AlertLog (AlertTime, AlertType, VehicleID, Message, Severity)
    SELECT 
        GETDATE(),
        'ExcessiveIdle',
        f.VehicleID,
        'Vehicle ' + f.VehicleID + ' exceeded 15% idle time: ' + 
            CAST(ROUND((SUM(f.IdleTime) * 100.0) / SUM(f.TripDuration), 2) AS NVARCHAR) + '%',
        'High'
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateKey = CAST(FORMAT(GETDATE(), 'yyyyMMdd') AS INT)
    GROUP BY f.VehicleID
    HAVING (SUM(f.IdleTime) * 100.0) / NULLIF(SUM(f.TripDuration), 0) > 15;
    
    -- Check for temperature compliance violations on perishable loads
    INSERT INTO AlertLog (AlertTime, AlertType, VehicleID, Message, Severity)
    SELECT 
        GETDATE(),
        'TemperatureViolation',
        f.VehicleID,
        'Temperature violation on vehicle ' + f.VehicleID + 
            ' carrying high-value perishables (Trip ' + CAST(f.TripKey AS NVARCHAR) + ')',
        'Critical'
    FROM FactFleetTrips f
    INNER JOIN FactWarehouseOperations w ON f.TimeKey = w.TimeKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE f.TemperatureCompliance = 0
        AND p.Category IN ('Perishables', 'Pharmaceuticals')
        AND f.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime); -- Last 24 hours
END;
GO
```

## Power BI DAX Measures

### Cross-Fact KPIs

```dax
-- Dwell Time per SKU vs Fleet Idling Cost per Route
DwellIdleCorrelation = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTime])
VAR TotalIdleCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[IdleTime] * (FactFleetTrips[FuelConsumed] / FactFleetTrips[TripDuration]) * 3.50
    )
VAR OperationCount = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(AvgDwell * TotalIdleCost, OperationCount, 0)

-- Warehouse Gravity Zone Efficiency
GravityZoneEfficiency = 
VAR OptimalPlacements = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[ZoneType] = RELATED(DimProductGravity[OptimalZoneType])
    )
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(OptimalPlacements, TotalOperations, 0)

-- Fleet Maintenance Priority Score
MaintenancePriorityScore = 
VAR IdleScore = SUMX(FactFleetTrips, IF(FactFleetTrips[IdleTime] > 30, 10, 0))
VAR TempScore = SUMX(FactFleetTrips, IF(FactFleetTrips[TemperatureCompliance] = 0, 50, 0))
VAR EfficiencyScore = 
    DIVIDE(
        100,
        AVERAGE(FactFleetTrips[FuelConsumed] / FactFleetTrips[DistanceMiles]),
        0
    )
VAR RevenueRisk = 
    SUMX(
        FactWarehouseOperations,
        RELATED(DimProductGravity[GravityScore]) * FactWarehouseOperations[UnitCount]
    ) / 1000
RETURN
    (IdleScore + TempScore + EfficiencyScore) * RevenueRisk

-- Predictive Bottleneck Index
BottleneckIndex = 
VAR CapacityUtilization = 
    DIVIDE(
        COUNTROWS(FactWarehouseOperations),
        [MaxDailyCapacity],
        0
    )
VAR AvgDelayMinutes = AVERAGE(FactFleetTrips[DelayMinutes])
VAR SupplierReliability = AVERAGE(DimSupplierReliability[ComplianceScore])
RETURN
    (CapacityUtilization * 0.4) + 
    (AvgDelayMinutes / 60 * 0.3) + 
    ((100 - SupplierReliability) / 100 * 0.3)
```

## Configuration Files

### Data Source Configuration

```json
{
  "refresh_schedule": {
    "interval_minutes": 15,
    "enabled": true,
    "time_ranges": [
      {"start": "00:00", "end": "23:59"}
    ]
  },
  "data_sources": {
    "wms": {
      "type": "SQL",
      "connection_string": "${WMS_CONNECTION_STRING}",
      "tables": ["Operations", "Inventory", "Zones"]
    },
    "telemetry": {
      "type": "REST_API",
      "endpoint": "${TELEMETRY_API_ENDPOINT}",
      "auth_header": "Bearer ${TELEMETRY_API_KEY}",
      "poll_interval_seconds": 60
    },
    "weather": {
      "type": "REST_API",
      "endpoint": "https://api.weatherapi.com/v1/current.json",
      "api_key": "${WEATHER_API_KEY}",
      "locations": ["warehouse_lat_lon"]
    }
  },
  "alerts": {
    "channels": ["email", "teams"],
    "email_recipients": ["${ALERT_EMAIL}"],
    "teams_webhook": "${TEAMS_WEBHOOK_URL}"
  }
}
```

### Power BI Row-Level Security

```dax
// Table: DimGeography
[Region] = USERNAME() || USERNAME() = "admin@company.com"

// Table: FactWarehouseOperations
[GeographyKey] IN 
    CALCULATETABLE(
        VALUES(DimGeography[GeographyKey]),
        DimGeography[Region] = USERNAME()
    )
```

## Common Patterns

### Pattern 1: Time-Phased Simulation

```sql
-- Simulate warehouse capacity increase from 80% to 95%
DECLARE @CurrentCapacity DECIMAL(5,2) = 0.80;
DECLARE @TargetCapacity DECIMAL(5,2) = 0.95;
DECLARE @SimulationDays INT = 30;

WITH HistoricalLoad AS (
    SELECT 
        t.DateKey,
        COUNT(*) AS OperationCount,
        AVG(w.OperationDuration) AS AvgDuration
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateKey BETWEEN 
        CAST(FORMAT(DATEADD(DAY, -@SimulationDays, GETDATE()), 'yyyyMMdd') AS INT)
        AND CAST(FORMAT(GETDATE(), 'yyyyMMdd') AS INT)
    GROUP BY t.DateKey
)
SELECT 
    DateKey,
    OperationCount AS ActualOperations,
    OperationCount * (@TargetCapacity / @CurrentCapacity) AS ProjectedOperations,
    AvgDuration AS CurrentAvgDuration,
    AvgDuration * (1 + (@TargetCapacity - @CurrentCapacity) * 0.3) AS ProjectedAvgDuration,
    (OperationCount * (@TargetCapacity / @CurrentCapacity) * 
     AvgDuration * (1 + (@TargetCapacity - @CurrentCapacity) * 0.3)) / 60 AS ProjectedTotalHours
FROM HistoricalLoad
ORDER BY DateKey;
```

### Pattern 2: Anomaly Detection with Human-in-the-Loop

```sql
-- Create anomaly tracking table
CREATE TABLE AnomalyLog (
    AnomalyID INT PRIMARY KEY IDENTITY,
    DetectionTime DATETIME2,
    FactTable NVARCHAR(50),
    RecordKey BIGINT,
    AnomalyType NVARCHAR(50),
    ExpectedValue DECIMAL(18,2),
    ActualValue DECIMAL(18,2),
    DeviationPercentage DECIMAL(5,2),
    UserValidated BIT DEFAULT 0,
    ValidationNotes NVARCHAR(500),
    RootCause NVARCHAR(200)
);

-- Detect dwell time anomalies
INSERT INTO AnomalyLog (DetectionTime, FactTable, RecordKey, AnomalyType, ExpectedValue, ActualValue, DeviationPercentage)
SELECT 
    GETDATE(),
    'FactWarehouseOperations',
    w.OperationKey,
    'DwellTimeSpike',
    (SELECT AVG(DwellTime) FROM FactWarehouseOperations WHERE ProductKey = w.ProductKey),
    w.DwellTime,
    ((w.DwellTime - (SELECT AVG(DwellTime) FROM FactWarehouseOperations WHERE ProductKey = w.ProductKey)) * 100.0) /
        (SELECT AVG(DwellTime) FROM FactWarehouseOperations WHERE ProductKey = w.ProductKey)
FROM FactWarehouseOperations w
WHERE w.DwellTime > (SELECT AVG(DwellTime) * 2 FROM FactWarehouseOperations WHERE ProductKey = w.ProductKey)
    AND w.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime);
```

### Pattern 3: Dynamic Dimension Updates

```sql
-- Recalculate product gravity scores based on 30-day trends
UPDATE p
SET 
    GravityScore = 
        (VelocityScore * 0.5) + 
        (ValueScore * 0.3) + 
        (FragilityScore * 0.2),
    VelocityClass = 
        CASE 
            WHEN PicksPerDay > 10 THEN 'Fast'
            WHEN PicksPerDay > 3 THEN 'Medium'
            ELSE 'Slow'
        END,
    OptimalZoneType = 
        CASE 
            WHEN (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) > 75 THEN 'HighGravity-NearDock'
            WHEN (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) > 50 THEN 'MediumGravity-MidWarehouse'
            ELSE 'LowGravity-DeepStorage'
        END
FROM DimProductGravity p
INNER JOIN (
    SELECT 
        w.ProductKey,
        COUNT(*) * 1.0 / 30 AS PicksPerDay,
        (COUNT(*) * 1.0 / 30) * 10 AS VelocityScore, -- Normalize to 0-100
        AVG(w.UnitCount) * p2.ValueTier_Numeric AS ValueScore,
        p2.FragilityIndex * 10 AS FragilityScore
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimProductGravity p2 ON w.ProductKey = p2.ProductKey
    WHERE t.DateKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT)
        AND w.OperationType = 'Picking'
    GROUP BY w.ProductKey, p2.ValueTier_Numeric, p2.FragilityIndex
) calc ON p.ProductKey = calc.ProductKey;
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom**: Dataset refresh fails with timeout error after 2+ hours

**Solution**:
```sql
-- Implement incremental refresh by creating partitioned views
CREATE VIEW vw_WarehouseOperations_Last30Days
AS
SELECT * FROM FactWarehouseOperations
WHERE TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime);

CREATE VIEW vw_WarehouseOperations_Historical
AS
SELECT * FROM FactWarehouseOperations
WHERE TimeKey < (SELECT MAX(TimeKey) - 2880 FROM DimTime);
```

In Power BI: Configure incremental refresh to only refresh last 30 days, archive historical data

### Issue: Slow Cross-Fact Queries

**Symptom**: Queries joining multiple fact tables take 30+ seconds

**Solution**:
```sql
-- Create covering indexes on foreign keys
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeGeoProduct
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey)
INCLUDE (DwellTime, OperationDuration, UnitCount);

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeGeoVehicle
ON FactFleetTrips (TimeKey, OriginGeographyKey, VehicleID)
INCLUDE (IdleTime, FuelConsumed, DelayMinutes);

-- Enable columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Columnstore
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, DwellTime, OperationDuration);
```

### Issue: Dimension Table Bloat

**Symptom**: DimTime table growing too large (1M+ rows)

**Solution**:
```sql
-- Partition DimTime by year
ALTER PARTITION SCHEME ps_DimTime
NEXT USED [PRIMARY];

ALTER TABLE DimTime
PARTITION BY RANGE (DateKey)
(
    PARTITION p2025 VALUES LESS THAN (20260101),
    PARTITION p2026 VALUES LESS THAN (20270101),
    PARTITION p2027 VALUES LESS THAN (20280101)
);

-- Archive old partitions
ALTER TABLE DimTime SWITCH PARTITION 1 
TO DimTime_Archive PARTITION 1;
```

### Issue: Alert Storms

**Symptom**: Thousands of duplicate alerts generated in short period

**Solution**:
```sql
-- Add alert throttling logic
ALTER PROCEDURE usp_GenerateFleetAlerts
AS
BEGIN
    -- Only generate alert if not already sent in last 4 hours
    INSERT INTO AlertLog (AlertTime, AlertType, VehicleID, Message, Severity)
    SELECT 
        GETDATE(),
        'ExcessiveIdle',
        f.VehicleID,
        'Vehicle ' + f.VehicleID + ' exceeded 15% idle time...',
        'High'
    FROM FactFleetTrips f
    WHERE NOT EXISTS (
        SELECT 1 FROM AlertLog a
        WHERE a.VehicleID = f.VehicleID
            AND a.AlertType = 'ExcessiveIdle'
            AND a.AlertTime >= DATEADD(HOUR, -4, GETDATE())
    )
    GROUP BY f.VehicleID
    HAVING (SUM(f.IdleTime) * 100.0) / NULLIF(SUM(f.TripDuration), 0) > 15;
END;
```

## Performance Optimization

### Indexing Strategy

```sql
-- Fact table indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
