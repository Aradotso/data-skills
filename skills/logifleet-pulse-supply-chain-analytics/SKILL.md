---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform using MS SQL Server star schema and Power BI for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "create logistics data warehouse with star schema"
  - "build Power BI dashboard for fleet management"
  - "implement warehouse gravity zone optimization"
  - "configure multi-fact logistics analytics"
  - "deploy SQL Server logistics intelligence engine"
  - "design cross-modal supply chain reporting"
  - "integrate warehouse and fleet telemetry data"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform that combines MS SQL Server data warehousing with Power BI visualization to create a unified view of warehouse operations, fleet management, and supply chain performance. It uses a multi-fact star schema architecture to harmonize cross-functional KPIs and enable real-time decision-making.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse gravity zone optimization (spatial-temporal analysis)
- Fleet telemetry integration and predictive maintenance
- Cross-fact KPI reconciliation (warehouse ↔ fleet ↔ inventory)
- Real-time anomaly detection and bottleneck prediction
- Role-based security with row-level filtering

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Database Deployment

1. **Clone and navigate to repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema creation scripts (assumed structure)
-- Run in order: dimensions first, then facts, then bridges
```

3. **Configure data source connections:**
```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "integrated",
    "connection_string": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;Trusted_Connection=True;"
  },
  "wms_api": {
    "endpoint": "${WMS_API_ENDPOINT}",
    "api_key": "${WMS_API_KEY}"
  },
  "telemetry_feed": {
    "endpoint": "${FLEET_TELEMETRY_URL}",
    "auth_token": "${TELEMETRY_AUTH_TOKEN}"
  }
}
```

### Power BI Template Setup

```bash
# Open the Power BI template file
# LogiFleet_Pulse_Master.pbit

# Update data source settings in Power BI:
# Home → Transform Data → Data Source Settings
# Update SQL Server connection string with your server details
```

## Core Schema Structure

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME(0) NOT NULL,
    Hour TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    FiscalPeriod VARCHAR(10) NOT NULL,
    IsBusinessHour BIT NOT NULL,
    CONSTRAINT UQ_DimTime_DateTime UNIQUE (FullDateTime)
);

CREATE INDEX IDX_DimTime_Date ON DimTime(Date);
CREATE INDEX IDX_DimTime_Fiscal ON DimTime(FiscalPeriod);
```

**DimProductGravity** - Products with calculated gravity scores:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Pick frequency normalized
    ValueScore DECIMAL(5,2), -- Unit price normalized
    FragilityScore DECIMAL(5,2), -- Handling risk 0-100
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    OptimalZone VARCHAR(10), -- A1, B2, etc.
    CONSTRAINT UQ_DimProductGravity_SKU UNIQUE (SKU)
);

CREATE INDEX IDX_DimProductGravity_Gravity ON DimProductGravity(GravityScore DESC);
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- Warehouse, Route Node, Distribution Center
    ParentLocationID VARCHAR(50),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    CONSTRAINT UQ_DimGeography_Location UNIQUE (LocationID)
);

CREATE INDEX IDX_DimGeography_Type ON DimGeography(LocationType);
```

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME2(0) NOT NULL,
    OperationEndTime DATETIME2(0),
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    QuantityHandled INT NOT NULL,
    DwellTimeHours DECIMAL(8,2), -- Time in current zone
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    BatchID VARCHAR(50),
    CONSTRAINT FK_FactWarehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_FactWarehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE INDEX IDX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IDX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IDX_FactWarehouse_Operation ON FactWarehouseOperations(OperationType, OperationStartTime);
```

**FactFleetTrips** - Fleet telemetry and route performance:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DepartureTime DATETIME2(0) NOT NULL,
    ArrivalTime DATETIME2(0),
    TripDurationMinutes AS DATEDIFF(MINUTE, DepartureTime, ArrivalTime) PERSISTED,
    DistanceKM DECIMAL(8,2),
    FuelConsumedLiters DECIMAL(6,2),
    IdleTimeMinutes INT,
    LoadWeight KG INT,
    TirePressurePSI DECIMAL(4,1),
    EngineHealthScore DECIMAL(5,2), -- 0-100
    DelayMinutes INT DEFAULT 0,
    DelayReason VARCHAR(100),
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE INDEX IDX_FactFleet_Vehicle ON FactFleetTrips(VehicleID, DepartureTime);
CREATE INDEX IDX_FactFleet_Route ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey);
```

## Key SQL Queries & Patterns

### Cross-Fact KPI: Warehouse Dwell Time vs Fleet Idle Time

```sql
-- Correlate high-dwell SKUs with fleet delays
WITH HighDwellProducts AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeHours) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.SKU, p.ProductName
    HAVING AVG(w.DwellTimeHours) > 72
),
FleetDelays AS (
    SELECT
        f.VehicleID,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        AVG(f.DelayMinutes) AS AvgDelayMinutes
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY f.VehicleID
    HAVING AVG(f.IdleTimeMinutes) > 45
)
SELECT 
    hdp.*,
    fd.VehicleID,
    fd.AvgIdleTime,
    fd.AvgDelayMinutes
FROM HighDwellProducts hdp
CROSS JOIN FleetDelays fd -- Identify correlation opportunities
ORDER BY hdp.AvgDwellTime DESC, fd.AvgIdleTime DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend product relocations based on gravity score vs current zone
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone AS CurrentZone,
    CASE 
        WHEN p.GravityScore > 80 THEN 'A1' -- High gravity = near shipping
        WHEN p.GravityScore > 60 THEN 'B2'
        WHEN p.GravityScore > 40 THEN 'C3'
        ELSE 'D4' -- Low gravity = deep storage
    END AS RecommendedZone,
    COUNT(w.OperationKey) AS RecentPickCount,
    AVG(w.DurationMinutes) AS AvgPickDuration
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations w 
    ON p.ProductKey = w.ProductKey 
    AND w.OperationType = 'Picking'
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZone
HAVING p.OptimalZone <> CASE 
        WHEN p.GravityScore > 80 THEN 'A1'
        WHEN p.GravityScore > 60 THEN 'B2'
        WHEN p.GravityScore > 40 THEN 'C3'
        ELSE 'D4'
    END
ORDER BY p.GravityScore DESC;
```

### Predictive Fleet Maintenance Queue

```sql
-- Prioritize vehicles for maintenance based on telemetry and load value
WITH VehicleHealth AS (
    SELECT 
        VehicleID,
        AVG(EngineHealthScore) AS AvgHealthScore,
        MIN(TirePressurePSI) AS MinTirePressure,
        SUM(FuelConsumedLiters) AS TotalFuelBurn,
        COUNT(*) AS TripCount
    FROM FactFleetTrips
    WHERE DepartureTime >= DATEADD(DAY, -14, GETDATE())
    GROUP BY VehicleID
),
VehicleRisk AS (
    SELECT 
        vh.*,
        CASE 
            WHEN vh.AvgHealthScore < 60 THEN 5
            WHEN vh.AvgHealthScore < 75 THEN 3
            ELSE 1
        END AS HealthRiskWeight,
        CASE 
            WHEN vh.MinTirePressure < 28 THEN 4
            WHEN vh.MinTirePressure < 32 THEN 2
            ELSE 0
        END AS TireRiskWeight
    FROM VehicleHealth vh
)
SELECT 
    vr.VehicleID,
    vr.AvgHealthScore,
    vr.MinTirePressure,
    vr.TripCount,
    (vr.HealthRiskWeight + vr.TireRiskWeight) AS MaintenancePriorityScore
FROM VehicleRisk vr
WHERE (vr.HealthRiskWeight + vr.TireRiskWeight) >= 4
ORDER BY MaintenancePriorityScore DESC, vr.TripCount DESC;
```

### Time-Phased Dwell Analysis (15-minute buckets)

```sql
-- Identify dwell time patterns by time of day
SELECT 
    t.Hour,
    t.MinuteBucket,
    t.DayName,
    AVG(w.DwellTimeHours) AS AvgDwellHours,
    COUNT(*) AS OperationCount,
    STRING_AGG(p.SKU, ', ') WITHIN GROUP (ORDER BY w.DwellTimeHours DESC) AS TopDwellSKUs
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
    AND w.OperationType = 'Picking'
GROUP BY t.Hour, t.MinuteBucket, t.DayName
HAVING AVG(w.DwellTimeHours) > 48
ORDER BY t.DayName, t.Hour, t.MinuteBucket;
```

## Data Ingestion Patterns

### Incremental Load Stored Procedure

```sql
CREATE OR ALTER PROCEDURE usp_IncrementalLoadWarehouseOps
    @LastLoadTime DATETIME2(0)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations since last load
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        OperationStartTime, OperationEndTime, QuantityHandled,
        DwellTimeHours, OperatorID, EquipmentID, BatchID
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        src.OperationType,
        src.OperationStartTime,
        src.OperationEndTime,
        src.QuantityHandled,
        src.DwellTimeHours,
        src.OperatorID,
        src.EquipmentID,
        src.BatchID
    FROM StagingWarehouseOps src
    INNER JOIN DimTime t 
        ON CAST(src.OperationStartTime AS DATE) = t.Date
        AND DATEPART(HOUR, src.OperationStartTime) = t.Hour
        AND (DATEPART(MINUTE, src.OperationStartTime) / 15) * 15 = t.MinuteBucket
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    INNER JOIN DimGeography g ON src.LocationID = g.LocationID
    WHERE src.OperationStartTime > @LastLoadTime;
    
    -- Update gravity scores based on new activity
    UPDATE p
    SET VelocityScore = (
        SELECT COUNT(*) * 1.0 / NULLIF(DATEDIFF(DAY, MIN(w.OperationStartTime), MAX(w.OperationStartTime)), 0)
        FROM FactWarehouseOperations w
        WHERE w.ProductKey = p.ProductKey
            AND w.OperationType = 'Picking'
            AND w.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    )
    FROM DimProductGravity p
    WHERE EXISTS (
        SELECT 1 FROM FactWarehouseOperations w
        WHERE w.ProductKey = p.ProductKey
            AND w.OperationStartTime > @LastLoadTime
    );
END;
GO
```

### External API Integration (Telemetry Feed)

```sql
-- Create external table for streaming telemetry (requires PolyBase)
CREATE EXTERNAL DATA SOURCE TelemetryFeed
WITH (
    TYPE = GENERIC,
    LOCATION = '${FLEET_TELEMETRY_URL}',
    CREDENTIAL = TelemetryAPICredential
);

CREATE EXTERNAL FILE FORMAT TelemetryJSON
WITH (
    FORMAT_TYPE = JSON
);

CREATE EXTERNAL TABLE ExtTelemetryStream (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2(0),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,1),
    TirePressure DECIMAL(4,1)
)
WITH (
    DATA_SOURCE = TelemetryFeed,
    FILE_FORMAT = TelemetryJSON,
    LOCATION = '/api/v2/stream'
);
```

## Power BI DAX Measures

### Cross-Fact Composite KPI

```dax
// Measure: Logistics Efficiency Index
Logistics Efficiency Index = 
VAR WarehouseThroughput = 
    DIVIDE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        SUM(FactWarehouseOperations[DurationMinutes]),
        0
    )
VAR FleetUtilization = 
    DIVIDE(
        SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[TripDurationMinutes]),
        0
    )
VAR InventoryTurnover = 
    DIVIDE(
        COUNTROWS(FactWarehouseOperations),
        DISTINCTCOUNT(FactWarehouseOperations[ProductKey]),
        0
    )
RETURN
    (WarehouseThroughput * 0.4) + 
    (FleetUtilization * 100 * 0.35) + 
    (InventoryTurnover * 0.25)
```

### Gravity Zone Heat Map

```dax
// Measure: Zone Optimization Score
Zone Optimization Score = 
VAR CurrentZonePicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR OptimalZonePicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] = "Picking",
        FILTER(
            DimProductGravity,
            DimProductGravity[OptimalZone] = DimGeography[LocationID]
        )
    )
RETURN
    DIVIDE(OptimalZonePicks, CurrentZonePicks, 0) * 100
```

### Time Intelligence for Dwell Time Trend

```dax
// Measure: Dwell Time Month-over-Month Change
Dwell Time MoM % = 
VAR CurrentMonthDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        DATESMTD(DimTime[Date])
    )
VAR PriorMonthDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        DATEADD(DimTime[Date], -1, MONTH)
    )
RETURN
    DIVIDE(
        CurrentMonthDwell - PriorMonthDwell,
        PriorMonthDwell,
        0
    )
```

## Configuration & Security

### Row-Level Security Implementation

```sql
-- Create security roles table
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY(1,1),
    RoleName VARCHAR(50) NOT NULL,
    UserEmail VARCHAR(200) NOT NULL,
    AllowedRegions VARCHAR(MAX), -- JSON array
    AllowedWarehouseIDs VARCHAR(MAX) -- JSON array
);

-- Power BI RLS filter (apply to DimGeography)
-- DAX expression in Power BI role definition:
[Region] IN USERGEOGRAPHIES()
```

### Automated Alerting Stored Procedure

```sql
CREATE OR ALTER PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    -- Check fleet idle time threshold
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips
        WHERE DepartureTime >= DATEADD(HOUR, -1, GETDATE())
            AND IdleTimeMinutes > 15
        GROUP BY VehicleID
        HAVING AVG(IdleTimeMinutes * 1.0 / NULLIF(TripDurationMinutes, 0)) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 15% threshold in last hour.';
        -- Send via SQL Server Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet KPI Alert',
            @body = @AlertMessage;
    END;
    
    -- Check warehouse dwell time anomaly
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
        GROUP BY w.ProductKey
        HAVING AVG(w.DwellTimeHours) > 96 -- 4 days
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Excessive dwell time detected (>96 hours) for multiple SKUs today.';
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet Dwell Time Alert',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule via SQL Agent job (run every 15 minutes)
-- EXEC usp_CheckKPIThresholds;
```

## Common Troubleshooting

### Issue: Power BI refresh fails with timeout

**Solution:** Partition large fact tables by date:
```sql
-- Partition FactWarehouseOperations by month
ALTER TABLE FactWarehouseOperations
SWITCH PARTITION $PARTITION.ByMonth(GETDATE()) 
TO FactWarehouseOperations_Current;

-- Use incremental refresh in Power BI:
-- Set RangeStart and RangeEnd parameters in Power Query
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    FilteredData = Table.SelectRows(
        Source, 
        each [OperationStartTime] >= RangeStart and [OperationStartTime] < RangeEnd
    )
in
    FilteredData
```

### Issue: Slow cross-fact queries

**Solution:** Create indexed views for common joins:
```sql
CREATE VIEW vw_WarehouseFleetBridge
WITH SCHEMABINDING
AS
SELECT 
    w.TimeKey,
    w.ProductKey,
    w.GeographyKey AS WarehouseGeographyKey,
    f.VehicleID,
    f.OriginGeographyKey,
    f.DestinationGeographyKey,
    w.DwellTimeHours,
    f.IdleTimeMinutes
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.FactFleetTrips f 
    ON w.GeographyKey = f.OriginGeographyKey
    AND w.TimeKey = f.TimeKey;

CREATE UNIQUE CLUSTERED INDEX IDX_WHFleetBridge 
ON vw_WarehouseFleetBridge(TimeKey, ProductKey, VehicleID);
```

### Issue: Gravity scores not updating

**Solution:** Verify calculation triggers:
```sql
-- Force recalculation
UPDATE DimProductGravity
SET VelocityScore = (
    SELECT COUNT(*) * 1.0 / 30.0 -- Picks per day over 30 days
    FROM FactWarehouseOperations w
    WHERE w.ProductKey = DimProductGravity.ProductKey
        AND w.OperationType = 'Picking'
        AND w.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
);

-- Check for missing foreign keys
SELECT DISTINCT w.ProductKey
FROM FactWarehouseOperations w
LEFT JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL;
```

## Advanced Patterns

### Temporal Elasticity Simulation

```sql
-- Scenario: What if warehouse capacity increases to 95%?
WITH CurrentState AS (
    SELECT 
        AVG(w.DwellTimeHours) AS BaselineDwell,
        COUNT(*) AS BaselineOps
    FROM FactWarehouseOperations w
    WHERE w.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
),
SimulatedState AS (
    SELECT 
        AVG(w.DwellTimeHours * 1.25) AS ProjectedDwell, -- 25% increase assumption
        COUNT(*) * 1.15 AS ProjectedOps -- 15% more operations
    FROM FactWarehouseOperations w
    WHERE w.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
)
SELECT 
    cs.BaselineDwell,
    ss.ProjectedDwell,
    cs.BaselineOps,
    ss.ProjectedOps,
    ((ss.ProjectedDwell - cs.BaselineDwell) / cs.BaselineDwell * 100) AS DwellImpactPct
FROM CurrentState cs, SimulatedState ss;
```

### Collaborative Anomaly Tagging

```sql
CREATE TABLE AnomalyAnnotations (
    AnnotationID INT PRIMARY KEY IDENTITY(1,1),
    FactTable VARCHAR(50) NOT NULL, -- 'FactWarehouseOperations', 'FactFleetTrips'
    RecordKey BIGINT NOT NULL,
    AnnotatedBy VARCHAR(200) NOT NULL,
    AnnotationDate DATETIME2(0) DEFAULT GETDATE(),
    AnomalyType VARCHAR(50), -- 'False Positive', 'Confirmed Delay', 'Data Error'
    Notes NVARCHAR(500)
);

-- Tag an anomaly
INSERT INTO AnomalyAnnotations (FactTable, RecordKey, AnnotatedBy, AnomalyType, Notes)
VALUES ('FactFleetTrips', 123456, 'user@example.com', 'Confirmed Delay', 'Supplier strike caused 2-hour delay');
```

## Best Practices

1. **Index Strategy:** Always index foreign keys and date columns in fact tables
2. **Partitioning:** Partition fact tables by month for tables >10M rows
3. **Incremental Refresh:** Use Power BI incremental refresh for datasets >1GB
4. **Security:** Implement RLS at the geography/region level for multi-tenant scenarios
5. **Monitoring:** Schedule the alert procedure every 15 minutes via SQL Agent
6. **Data Quality:** Validate staging data before loading into fact tables
7. **Documentation:** Maintain data dictionary with business logic for calculated columns

## Environment Variables Reference

```bash
# SQL Server Configuration
SQL_SERVER_HOST=your-sql-server.database.windows.net
SQL_SERVER_DATABASE=LogiFleetPulse
SQL_SERVER_USER=your-username
SQL_SERVER_PASSWORD=your-password

# External API Integrations
WMS_API_ENDPOINT=https://api.wms-provider.com/v2
WMS_API_KEY=your-wms-api-key
FLEET_TELEMETRY_URL=https://telemetry.fleet-provider.com/stream
TELEMETRY_AUTH_TOKEN=your-telemetry-token

# Alerting
ALERT_EMAIL=logistics-team@yourcompany.com
SMTP_SERVER=smtp.office365.com
SMTP_PORT=587
SMTP_USER=alerts@yourcompany.com
SMTP_PASSWORD=your-smtp-password
```
