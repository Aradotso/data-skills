---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI supply chain analytics platform with multi-fact star schema for warehouse, fleet, and logistics intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logicore analytics warehouse schema"
  - "create Power BI logistics dashboard"
  - "configure fleet telemetry data warehouse"
  - "implement multi-fact star schema for logistics"
  - "build warehouse gravity zone analytics"
  - "troubleshoot logifleet pulse data model"
  - "optimize supply chain KPI queries"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform combining MS SQL Server data warehousing with Power BI visualization. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and supply chain data
- **Cross-modal analytics** connecting inventory, routes, and operational KPIs
- **Real-time dashboards** with 15-minute refresh cycles
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and item value

The platform harmonizes data from WMS, telematics, supplier portals, and external APIs into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to warehouse/fleet data sources (WMS, GPS, ERP)

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

Connect to your SQL Server instance and execute the schema deployment:

```sql
-- Connect to your SQL Server instance
USE master;
GO

-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema creation script
-- (Assumes schema.sql is in the repository root)
:r schema.sql
GO
```

### Step 3: Configure Data Sources

Update the configuration file with your connection strings:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}",
    "port": 1433
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telemetry_feed": "${FLEET_TELEMETRY_URL}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter SQL Server connection details when prompted

## Core Data Model Structure

### Fact Tables

The platform uses multiple fact tables for different operational areas:

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    QuantityHandled DECIMAL(18,2),
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    OperatorID INT,
    GravityZoneID INT,
    CONSTRAINT FK_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- FactFleetTrips: Vehicle routing and telemetry
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginWarehouseKey INT,
    DestinationKey INT,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    DelayMinutes INT,
    DelayReasonKey INT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);

-- FactCrossDock: Transfer operations between inbound/outbound
CREATE TABLE FactCrossDock (
    CrossDockID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripID INT,
    OutboundTripID INT,
    TransferTimeMinutes INT,
    QuantityTransferred DECIMAL(18,2),
    QualityCheckStatus VARCHAR(20)
);
```

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeFull DATETIME NOT NULL,
    DateKey INT,
    TimeOf15Min VARCHAR(5), -- '00:00', '00:15', '00:30', '00:45'
    HourOfDay INT,
    DayOfWeek VARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    FiscalYear INT
);

-- DimProductGravity: Products with calculated gravity scores
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    CategoryHierarchy VARCHAR(500),
    GravityScore DECIMAL(5,2), -- Calculated: velocity × value × (1/fragility)
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2),
    OptimalZoneID INT
);

-- DimWarehouse: Warehouse locations with zone mappings
CREATE TABLE DimWarehouse (
    WarehouseKey INT PRIMARY KEY,
    WarehouseCode VARCHAR(20),
    WarehouseName VARCHAR(100),
    GeographyKey INT,
    TotalCapacityCubicMeters DECIMAL(12,2),
    GravityZoneCount INT,
    CONSTRAINT FK_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);
```

## Key SQL Queries & Procedures

### Cross-Fact KPI Query: Dwell Time vs Fleet Idle Cost

```sql
-- Calculate correlation between warehouse dwell time and fleet idle costs
WITH WarehouseDwell AS (
    SELECT 
        wo.ProductKey,
        dt.DateKey,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.DateKey >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY wo.ProductKey, dt.DateKey
),
FleetIdle AS (
    SELECT 
        ft.TimeKey,
        dt.DateKey,
        SUM(ft.IdleTimeMinutes * v.HourlyCost / 60.0) AS IdleCostUSD
    FROM FactFleetTrips ft
    INNER JOIN DimTime dt ON ft.TimeKey = dt.TimeKey
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    WHERE dt.DateKey >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY ft.TimeKey, dt.DateKey
)
SELECT 
    wd.DateKey,
    wd.AvgDwellTime,
    fi.IdleCostUSD,
    -- Calculate correlation coefficient
    (COUNT(*) * SUM(wd.AvgDwellTime * fi.IdleCostUSD) - 
     SUM(wd.AvgDwellTime) * SUM(fi.IdleCostUSD)) / 
    (SQRT(COUNT(*) * SUM(POWER(wd.AvgDwellTime, 2)) - POWER(SUM(wd.AvgDwellTime), 2)) *
     SQRT(COUNT(*) * SUM(POWER(fi.IdleCostUSD, 2)) - POWER(SUM(fi.IdleCostUSD), 2))) 
    AS CorrelationCoefficient
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.DateKey = fi.DateKey
GROUP BY wd.DateKey, wd.AvgDwellTime, fi.IdleCostUSD
ORDER BY wd.DateKey;
```

### Warehouse Gravity Zone Optimization

```sql
-- Stored procedure to recalculate gravity zones based on recent velocity
CREATE PROCEDURE usp_RecalculateGravityZones
AS
BEGIN
    -- Calculate 30-day rolling velocity
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(CycleTimeSeconds) AS AvgCycleTime,
            SUM(QuantityHandled) AS TotalVolume
        FROM FactWarehouseOperations
        WHERE OperationType = 'Pick'
          AND TimeKey >= (SELECT TimeKey FROM DimTime 
                          WHERE DateTimeFull = DATEADD(DAY, -30, GETDATE()))
        GROUP BY ProductKey
    ),
    GravityCalculation AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            pv.PickFrequency,
            p.ValueTier,
            p.FragilityIndex,
            -- Gravity Score = (Frequency × Value Weight) / (Fragility × Cycle Time)
            (pv.PickFrequency * 
             CASE p.ValueTier 
                 WHEN 'High' THEN 3.0 
                 WHEN 'Medium' THEN 2.0 
                 ELSE 1.0 
             END) / 
            (p.FragilityIndex * NULLIF(pv.AvgCycleTime, 0)) AS NewGravityScore
        FROM DimProductGravity p
        LEFT JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
    )
    UPDATE DimProductGravity
    SET GravityScore = gc.NewGravityScore,
        VelocityClass = CASE 
            WHEN gc.NewGravityScore > 75 THEN 'Fast'
            WHEN gc.NewGravityScore > 25 THEN 'Medium'
            ELSE 'Slow'
        END,
        OptimalZoneID = CASE 
            WHEN gc.NewGravityScore > 75 THEN 1 -- Closest to dock
            WHEN gc.NewGravityScore > 25 THEN 2
            ELSE 3 -- Furthest from dock
        END
    FROM GravityCalculation gc
    WHERE DimProductGravity.ProductKey = gc.ProductKey;
END;
GO
```

### Predictive Bottleneck Detection

```sql
-- Identify probable bottlenecks based on trend analysis
CREATE VIEW vw_BottleneckPrediction AS
WITH HourlyTrends AS (
    SELECT 
        dt.HourOfDay,
        dt.DayOfWeek,
        wo.OperationType,
        AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
        STDEV(wo.CycleTimeSeconds) AS StdDevCycleTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.DateKey >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY dt.HourOfDay, dt.DayOfWeek, wo.OperationType
),
CurrentPerformance AS (
    SELECT 
        dt.HourOfDay,
        dt.DayOfWeek,
        wo.OperationType,
        AVG(wo.CycleTimeSeconds) AS CurrentAvgCycle
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.DateKey = CAST(GETDATE() AS DATE)
    GROUP BY dt.HourOfDay, dt.DayOfWeek, wo.OperationType
)
SELECT 
    ht.HourOfDay,
    ht.DayOfWeek,
    ht.OperationType,
    ht.AvgCycleTime AS HistoricalAvg,
    cp.CurrentAvgCycle,
    -- Flag if current is > 2 standard deviations above historical
    CASE 
        WHEN cp.CurrentAvgCycle > (ht.AvgCycleTime + 2 * ht.StdDevCycleTime) 
        THEN 'High Risk'
        WHEN cp.CurrentAvgCycle > (ht.AvgCycleTime + ht.StdDevCycleTime) 
        THEN 'Moderate Risk'
        ELSE 'Normal'
    END AS BottleneckRisk,
    ((cp.CurrentAvgCycle - ht.AvgCycleTime) / NULLIF(ht.AvgCycleTime, 0)) * 100 
    AS PercentDeviationFromNorm
FROM HourlyTrends ht
INNER JOIN CurrentPerformance cp 
    ON ht.HourOfDay = cp.HourOfDay 
    AND ht.DayOfWeek = cp.DayOfWeek 
    AND ht.OperationType = cp.OperationType;
GO
```

## Power BI Integration

### DAX Measures for Cross-Fact KPIs

```dax
// Total Warehouse Cost Including Fleet Impact
TotalLogisticsCost = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] * 0.15 // $0.15 per minute holding cost
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        (FactFleetTrips[FuelConsumedLiters] * 1.45) + // $1.45 per liter
        (FactFleetTrips[IdleTimeMinutes] * 0.75) // $0.75 per idle minute
    )
RETURN WarehouseCost + FleetCost

// Composite Efficiency Score (0-100)
EfficiencyScore = 
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeSeconds])
VAR BenchmarkCycleTime = 180 // 3 minutes
VAR CycleEfficiency = 1 - (AvgCycleTime / BenchmarkCycleTime)

VAR AvgIdlePercent = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[IdleTimeMinutes]) + SUM(FactFleetTrips[DistanceKM]) * 0.6,
        0
    )
VAR FleetEfficiency = 1 - AvgIdlePercent

RETURN (CycleEfficiency * 0.6 + FleetEfficiency * 0.4) * 100

// Gravity Zone Compliance Rate
GravityZoneCompliance = 
DIVIDE(
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationID]),
        FactWarehouseOperations[GravityZoneID] = 
        RELATED(DimProductGravity[OptimalZoneID])
    ),
    COUNT(FactWarehouseOperations[OperationID]),
    0
) * 100
```

### Creating Linked Visuals

In Power BI, create cross-fact relationships:

1. **Model View** → Manage Relationships
2. Link `FactWarehouseOperations[TimeKey]` ↔ `FactFleetTrips[TimeKey]` (Many-to-Many with bridge table)
3. Enable bi-directional cross-filtering for interactive drill-down
4. Create hierarchy: `DimTime[FiscalYear]` → `FiscalPeriod` → `DateKey` → `HourOfDay`

## Data Ingestion Patterns

### Incremental Load from WMS API

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_IncrementalLoadWarehouseOps
    @LastLoadTimestamp DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Stage incoming data
    INSERT INTO StagingWarehouseOps (
        OperationTimestamp, SKU, OperationType, 
        Quantity, CycleTime, WarehouseCode
    )
    SELECT 
        op.timestamp,
        op.sku,
        op.operation_type,
        op.quantity,
        op.cycle_time_seconds,
        op.warehouse_code
    FROM OPENROWSET(
        BULK '${WMS_EXPORT_PATH}/*.json',
        FORMAT = 'CSV',
        FIELDTERMINATOR = '0x0b',
        FIELDQUOTE = '0x0b',
        ROWTERMINATOR = '0x0b'
    ) WITH (doc NVARCHAR(MAX)) AS rows
    CROSS APPLY OPENJSON(doc)
    WITH (
        timestamp DATETIME2 '$.timestamp',
        sku VARCHAR(50) '$.sku',
        operation_type VARCHAR(50) '$.operation_type',
        quantity DECIMAL(18,2) '$.quantity',
        cycle_time_seconds INT '$.cycle_time_seconds',
        warehouse_code VARCHAR(20) '$.warehouse_code'
    ) AS op
    WHERE op.timestamp > @LastLoadTimestamp;
    
    -- Transform and load into fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        QuantityHandled, CycleTimeSeconds, GravityZoneID
    )
    SELECT 
        dt.TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        s.OperationType,
        s.Quantity,
        s.CycleTime,
        p.OptimalZoneID
    FROM StagingWarehouseOps s
    INNER JOIN DimTime dt ON DATEPART(MINUTE, s.OperationTimestamp) / 15 = dt.TimeOf15Min
        AND CAST(s.OperationTimestamp AS DATE) = dt.DateKey
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    INNER JOIN DimWarehouse w ON s.WarehouseCode = w.WarehouseCode;
    
    -- Clear staging
    TRUNCATE TABLE StagingWarehouseOps;
END;
GO
```

### Real-Time Fleet Telemetry Stream

```sql
-- Create external table for streaming GPS data
CREATE EXTERNAL TABLE ExtFleetTelemetry (
    VehicleID VARCHAR(20),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineStatus VARCHAR(20)
)
WITH (
    LOCATION = '${TELEMETRY_STREAM_URL}',
    DATA_SOURCE = FleetTelemetrySource,
    FILE_FORMAT = JSONFormat
);

-- Materialized view for latest vehicle positions
CREATE VIEW vw_CurrentFleetStatus
WITH SCHEMABINDING
AS
SELECT 
    v.VehicleKey,
    v.VehicleCode,
    t.Latitude,
    t.Longitude,
    t.Speed,
    t.FuelLevel,
    t.EngineStatus,
    t.Timestamp AS LastUpdate
FROM dbo.ExtFleetTelemetry t
INNER JOIN dbo.DimVehicle v ON t.VehicleID = v.VehicleCode
WHERE t.Timestamp = (
    SELECT MAX(Timestamp) 
    FROM dbo.ExtFleetTelemetry t2 
    WHERE t2.VehicleID = t.VehicleID
);
GO
```

## Automated Alerting System

### Configure Threshold Alerts

```sql
-- Alert configuration table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(100),
    MetricName VARCHAR(50),
    ThresholdValue DECIMAL(18,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    AlertSeverity VARCHAR(20), -- 'Critical', 'Warning', 'Info'
    NotificationChannel VARCHAR(50), -- 'Email', 'SMS', 'Teams'
    RecipientList VARCHAR(MAX)
);

-- Populate with example thresholds
INSERT INTO AlertThresholds (AlertName, MetricName, ThresholdValue, ComparisonOperator, AlertSeverity, NotificationChannel, RecipientList)
VALUES 
    ('High Fleet Idle Time', 'FleetIdlePercent', 15.0, '>', 'Warning', 'Email', '${FLEET_MANAGER_EMAIL}'),
    ('Warehouse Dwell Spike', 'AvgDwellTimeMinutes', 240.0, '>', 'Critical', 'Teams', '${OPS_TEAM_WEBHOOK}'),
    ('Gravity Zone Misalignment', 'GravityCompliance', 75.0, '<', 'Warning', 'Email', '${WAREHOUSE_SUPERVISOR_EMAIL}');

-- Automated alert execution procedure
CREATE PROCEDURE usp_EvaluateAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check each alert threshold
    DECLARE alert_cursor CURSOR FOR
        SELECT AlertID, AlertName, MetricName, ThresholdValue, ComparisonOperator
        FROM AlertThresholds
        WHERE IsActive = 1;
    
    -- Implementation would call external notification service
    -- Using SQL Server Agent Job or Azure Logic Apps
END;
GO
```

## Common Troubleshooting

### Issue: Cross-Fact Queries Running Slow

**Solution**: Ensure composite indexes exist on shared dimension keys:

```sql
-- Create composite indexes for cross-fact joins
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (QuantityHandled, CycleTimeSeconds);

CREATE NONCLUSTERED INDEX IX_FleetTrips_Time_Vehicle 
ON FactFleetTrips(TimeKey, VehicleKey) 
INCLUDE (IdleTimeMinutes, FuelConsumedLiters);

-- Enable columnstore for analytical queries
CREATE COLUMNSTORE INDEX CCI_WarehouseOps 
ON FactWarehouseOperations (TimeKey, ProductKey, OperationType, QuantityHandled, DwellTimeMinutes);
```

### Issue: Power BI Refresh Timeout

**Solution**: Implement incremental refresh policy:

1. In Power BI Desktop → Model → Table → Incremental Refresh
2. Set parameters: `RangeStart` and `RangeEnd`
3. Configure: Archive data older than 12 months, refresh last 30 days

```powerquery
// Power Query M for incremental refresh
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    FilteredRows = Table.SelectRows(
        Source, 
        each [OperationTimestamp] >= RangeStart and [OperationTimestamp] < RangeEnd
    )
in
    FilteredRows
```

### Issue: Gravity Zone Calculations Not Updating

**Solution**: Schedule the recalculation procedure via SQL Agent:

```sql
-- Create SQL Agent Job (T-SQL)
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'Recalculate Gravity Zones',
    @enabled = 1,
    @description = N'Daily recalculation of product gravity scores';

EXEC sp_add_jobstep
    @job_name = N'Recalculate Gravity Zones',
    @step_name = N'Execute Gravity Calculation',
    @subsystem = N'TSQL',
    @command = N'EXEC usp_RecalculateGravityZones;',
    @database_name = N'LogiFleetPulse';

EXEC sp_add_schedule
    @schedule_name = N'Daily at 2 AM',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @active_start_time = 020000;

EXEC sp_attach_schedule
    @job_name = N'Recalculate Gravity Zones',
    @schedule_name = N'Daily at 2 AM';

EXEC sp_add_jobserver
    @job_name = N'Recalculate Gravity Zones',
    @server_name = @@SERVERNAME;
GO
```

## Performance Optimization

### Partitioning Strategy for Large Fact Tables

```sql
-- Create partition function for monthly partitions
CREATE PARTITION FUNCTION pf_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601,
    20260701, 20260801, 20260901, 20261001, 20261101, 20261201
);

-- Create partition scheme
CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition
ALL TO ([PRIMARY]);

-- Apply to fact table (requires recreating table)
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationID INT IDENTITY(1,1),
    DateKey INT NOT NULL,
    -- ... other columns
    CONSTRAINT PK_WarehouseOps_Part PRIMARY KEY (DateKey, OperationID)
) ON ps_MonthlyPartition(DateKey);
```

## Configuration Best Practices

1. **Row-Level Security**: Implement RLS for multi-tenant environments:

```sql
-- Create security predicate function
CREATE FUNCTION fn_SecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessGranted
WHERE @WarehouseKey IN (
    SELECT WarehouseKey FROM UserWarehouseAccess 
    WHERE UserName = USER_NAME()
);

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
```

2. **Environment-Specific Configs**: Use JSON config per environment:

```json
{
  "environments": {
    "production": {
      "sql_server": "${PROD_SQL_HOST}",
      "refresh_minutes": 15,
      "alert_enabled": true
    },
    "staging": {
      "sql_server": "${STAGING_SQL_HOST}",
      "refresh_minutes": 60,
      "alert_enabled": false
    }
  }
}
```

3. **Logging & Auditing**: Track all data modifications:

```sql
-- Create audit table
CREATE TABLE AuditLog (
    AuditID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TableName VARCHAR(100),
    Operation VARCHAR(20),
    ModifiedBy VARCHAR(100) DEFAULT SYSTEM_USER,
    ModifiedDate DATETIME DEFAULT GETDATE(),
    RowsAffected INT
);

-- Add triggers to fact tables
CREATE TRIGGER tr_AuditWarehouseOps
ON FactWarehouseOperations
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    INSERT INTO AuditLog (TableName, Operation, RowsAffected)
    VALUES (
        'FactWarehouseOperations',
        CASE 
            WHEN EXISTS(SELECT * FROM inserted) AND EXISTS(SELECT * FROM deleted) THEN 'UPDATE'
            WHEN EXISTS(SELECT * FROM inserted) THEN 'INSERT'
            ELSE 'DELETE'
        END,
        @@ROWCOUNT
    );
END;
GO
```

This skill provides comprehensive coverage for deploying and operating LogiFleet Pulse as a production-grade supply chain analytics platform.
