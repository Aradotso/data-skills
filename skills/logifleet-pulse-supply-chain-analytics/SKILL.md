---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-fact data warehouse for real-time logistics, fleet tracking, and warehouse optimization
triggers:
  - set up logistics analytics warehouse
  - implement supply chain dashboard with Power BI
  - create multi-fact star schema for fleet management
  - build warehouse gravity zone analysis
  - configure cross-modal logistics KPI tracking
  - deploy LogiFleet Pulse analytics engine
  - integrate warehouse and fleet data models
  - analyze dwell time and fleet utilization metrics
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics platform that unifies warehouse operations, fleet telemetry, and supply chain metrics into a single semantic layer. It uses a multi-fact star schema in MS SQL Server with Power BI dashboards to provide:

- Cross-fact KPI harmonization (warehouse + fleet metrics combined)
- Warehouse Gravity Zones™ (spatial optimization based on pick frequency and value)
- Adaptive fleet triage engine (maintenance prioritization by revenue impact)
- Temporal elasticity modeling (scenario simulation across time)
- Real-time bottleneck prediction and anomaly detection

The platform integrates data from WMS, telematics/GPS, supplier portals, weather APIs, and customer orders into unified analytics.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
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
-- (Assumes schema.sql or similar file in repository)
```

3. **Configure data source connections:**
```json
// config.json (based on config_sample.json)
{
  "connections": {
    "sqlServer": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;User Id=${SQL_USER};Password=${SQL_PASSWORD};",
    "wmsApi": "${WMS_API_ENDPOINT}",
    "telematicsApi": "${TELEMATICS_API_ENDPOINT}",
    "weatherApi": "${WEATHER_API_KEY}"
  },
  "refreshInterval": "15m",
  "alertThresholds": {
    "fleetIdlePercent": 15,
    "dwellTimeHours": 72,
    "temperatureToleranceMinutes": 20
  }
}
```

4. **Import Power BI template:**
   - Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
   - Enter SQL Server connection string when prompted
   - Refresh data model

## Core Data Model

### Multi-Fact Star Schema

The platform uses these core fact tables:

```sql
-- FactWarehouseOperations
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    WarehouseZoneKey INT FOREIGN KEY REFERENCES DimWarehouseZone(ZoneKey),
    OperationType VARCHAR(20), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    QuantityHandled INT,
    AssignedGravityScore DECIMAL(5,2),
    OperatorID INT
);

-- FactFleetTrips
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimRoute(RouteKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripSegmentType VARCHAR(20), -- 'LOADING', 'TRANSIT', 'UNLOADING', 'IDLE'
    DurationMinutes INT,
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(8,2),
    LoadWeightKG DECIMAL(10,2),
    IdleTimeMinutes INT,
    AvgSpeedKMH DECIMAL(5,2)
);

-- FactCrossDock
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripID),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripID),
    TransferDurationMinutes INT,
    QuantityTransferred INT
);
```

### Key Dimensions

```sql
-- DimTime (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfWeek INT,
    Hour INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    IsBusinessHours BIT,
    FiscalPeriod VARCHAR(10)
);

-- DimProductGravity (warehouse gravity zones)
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50),
    ProductName VARCHAR(200),
    CategoryHierarchy VARCHAR(500),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * (1/fragility)
    PickFrequencyRank INT,
    UnitValue DECIMAL(10,2),
    FragilityIndex DECIMAL(3,2),
    OptimalZoneID INT,
    LastReclassificationDate DATETIME
);

-- DimGeography (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    NodeType VARCHAR(20), -- 'WAREHOUSE', 'ROUTE_POINT', 'CUSTOMER'
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    FullHierarchy VARCHAR(500)
);
```

## Key SQL Operations

### Cross-Fact KPI Query

```sql
-- Calculate combined warehouse efficiency + fleet utilization
WITH WarehouseMetrics AS (
    SELECT 
        t.Year,
        t.Month,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        AVG(w.CycleTimeSeconds) AS AvgCycleTime,
        SUM(w.QuantityHandled) AS TotalVolume
    FROM FactWarehouseOperations w
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.Year = YEAR(GETDATE())
    GROUP BY t.Year, t.Month
),
FleetMetrics AS (
    SELECT 
        t.Year,
        t.Month,
        AVG(f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) AS AvgIdlePercent,
        SUM(f.FuelLiters) AS TotalFuelUsed,
        AVG(f.DistanceKM / NULLIF(f.DurationMinutes, 0) * 60) AS AvgSpeedKMH
    FROM FactFleetTrips f
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.Year = YEAR(GETDATE())
    GROUP BY t.Year, t.Month
)
SELECT 
    w.Year,
    w.Month,
    w.AvgDwellTime,
    w.AvgCycleTime,
    w.TotalVolume,
    f.AvgIdlePercent,
    f.TotalFuelUsed,
    f.AvgSpeedKMH,
    -- Combined efficiency score
    (100 - w.AvgDwellTime/10) * (100 - f.AvgIdlePercent) / 100 AS CombinedEfficiencyScore
FROM WarehouseMetrics w
JOIN FleetMetrics f ON w.Year = f.Year AND w.Month = f.Month
ORDER BY w.Year, w.Month;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be moved to different gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneID AS CurrentZone,
    CASE 
        WHEN p.GravityScore >= 80 THEN 1 -- High gravity: near shipping dock
        WHEN p.GravityScore >= 50 THEN 2 -- Medium gravity: middle zones
        ELSE 3 -- Low gravity: back storage
    END AS RecommendedZone,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS PickOperations
FROM DimProductGravity p
JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
WHERE w.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZoneID
HAVING CASE 
    WHEN p.GravityScore >= 80 THEN 1
    WHEN p.GravityScore >= 50 THEN 2
    ELSE 3
END <> p.OptimalZoneID
ORDER BY ABS(p.GravityScore - p.OptimalZoneID * 30) DESC;
```

### Predictive Bottleneck Detection

```sql
-- Stored procedure for real-time bottleneck alerts
CREATE PROCEDURE sp_DetectBottlenecks
AS
BEGIN
    -- Detect warehouse congestion
    INSERT INTO AlertQueue (AlertType, Severity, Message, GeneratedAt)
    SELECT 
        'WAREHOUSE_CONGESTION',
        'HIGH',
        'Zone ' + CAST(ZoneKey AS VARCHAR) + ' dwell time exceeds threshold: ' + 
        CAST(AVG(DwellTimeMinutes) AS VARCHAR) + ' minutes',
        GETDATE()
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 4 FROM DimTime) -- Last hour
    GROUP BY ZoneKey
    HAVING AVG(DwellTimeMinutes) > 120;

    -- Detect fleet idle time anomalies
    INSERT INTO AlertQueue (AlertType, Severity, Message, GeneratedAt)
    SELECT 
        'FLEET_IDLE_ANOMALY',
        'MEDIUM',
        'Vehicle ' + CAST(VehicleKey AS VARCHAR) + ' idle time: ' + 
        CAST(SUM(IdleTimeMinutes) * 100.0 / SUM(DurationMinutes) AS VARCHAR) + '%',
        GETDATE()
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 4 FROM DimTime)
    GROUP BY VehicleKey
    HAVING SUM(IdleTimeMinutes) * 100.0 / NULLIF(SUM(DurationMinutes), 0) > 15;

    -- Send alerts via configured channels
    EXEC sp_SendAlerts;
END;
GO
```

## Power BI DAX Patterns

### Cross-Fact Measures

```dax
// Total Logistics Cost (combines warehouse labor + fleet fuel)
Total Logistics Cost = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        [CycleTimeSeconds] / 3600 * [HourlyLaborRate]
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        [FuelLiters] * [FuelPricePerLiter]
    )
RETURN WarehouseCost + FleetCost

// Inventory Velocity vs Fleet Utilization
Velocity-Utilization Ratio = 
DIVIDE(
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "PICK"
    ),
    CALCULATE(
        AVERAGE(FactFleetTrips[DurationMinutes]) - AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        FactFleetTrips[TripSegmentType] = "TRANSIT"
    ),
    0
)
```

### Time Intelligence

```dax
// Dwell Time Moving Average (30-day)
Dwell Time MA30 = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -30,
        DAY
    )
)

// Fleet Idle Time YoY Comparison
Idle Time YoY % = 
VAR CurrentPeriod = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR PreviousYear = 
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        SAMEPERIODLASTYEAR(DimTime[FullDateTime])
    )
RETURN
DIVIDE(CurrentPeriod - PreviousYear, PreviousYear, 0)
```

## Data Loading Patterns

### Incremental ETL

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Stage data from WMS
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseZoneKey, OperationType,
        DwellTimeMinutes, CycleTimeSeconds, QuantityHandled,
        AssignedGravityScore, OperatorID
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        z.ZoneKey,
        wms.operation_type,
        DATEDIFF(MINUTE, wms.start_time, wms.end_time) AS DwellTimeMinutes,
        DATEDIFF(SECOND, wms.cycle_start, wms.cycle_end) AS CycleTimeSeconds,
        wms.quantity,
        p.GravityScore,
        wms.operator_id
    FROM WMS_ExternalTable wms
    JOIN DimTime t ON CAST(wms.timestamp AS DATETIME) >= t.FullDateTime 
        AND CAST(wms.timestamp AS DATETIME) < DATEADD(MINUTE, 15, t.FullDateTime)
    JOIN DimProductGravity p ON wms.sku = p.SKU
    JOIN DimWarehouseZone z ON wms.zone_code = z.ZoneCode
    WHERE wms.timestamp > @LastLoadDateTime;
    
    -- Update last load timestamp
    UPDATE ETL_Control
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### External Table Configuration (Polybase)

```sql
-- Create external data source for telemetry API
CREATE EXTERNAL DATA SOURCE TelematicsAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${TELEMATICS_BLOB_URL}',
    CREDENTIAL = TelematicsCredential
);

-- Create external table for fleet GPS data
CREATE EXTERNAL TABLE Fleet_GPS_External (
    vehicle_id VARCHAR(50),
    timestamp DATETIME2,
    latitude DECIMAL(10,6),
    longitude DECIMAL(10,6),
    speed_kmh DECIMAL(5,2),
    fuel_level_percent DECIMAL(5,2),
    engine_status VARCHAR(20)
)
WITH (
    LOCATION = '/telemetry/gps/',
    DATA_SOURCE = TelematicsAPI,
    FILE_FORMAT = JSONFormat
);
```

## Configuration

### Row-Level Security

```sql
-- Create security function for warehouse zone access
CREATE FUNCTION fn_SecurityPredicateWarehouse(@ZoneKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS result
WHERE 
    @ZoneKey IN (
        SELECT ZoneKey 
        FROM dbo.UserZoneAccess 
        WHERE UserName = USER_NAME()
    )
    OR IS_MEMBER('LogisticsAdmin') = 1;
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseZoneSecurity
ADD FILTER PREDICATE dbo.fn_SecurityPredicateWarehouse(WarehouseZoneKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
GO
```

### Alert Configuration

```sql
-- Configure automated alerts
INSERT INTO AlertThresholds (MetricName, ThresholdValue, AlertSeverity, NotificationChannels)
VALUES 
    ('FleetIdlePercent', 15, 'MEDIUM', 'EMAIL,TEAMS'),
    ('DwellTimeHours', 72, 'HIGH', 'EMAIL,SMS,TEAMS'),
    ('TemperatureVarianceMinutes', 20, 'CRITICAL', 'EMAIL,SMS,PHONE,TEAMS'),
    ('GravityScoreDrift', 25, 'LOW', 'EMAIL');

-- Schedule alert job (SQL Server Agent)
EXEC msdb.dbo.sp_add_job 
    @job_name = 'LogiFleet_Bottleneck_Detection';

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_Bottleneck_Detection',
    @step_name = 'Run_Bottleneck_Detection',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_DetectBottlenecks;',
    @database_name = 'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every_15_Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;
```

## Common Patterns

### Warehouse Gravity Score Calculation

```sql
-- Recalculate gravity scores based on recent activity
UPDATE p
SET 
    p.GravityScore = (
        -- Velocity component (40% weight)
        (pick_freq.PicksPerDay / NULLIF(max_picks.MaxPicks, 0)) * 40 +
        -- Value component (40% weight)
        (p.UnitValue / NULLIF(max_value.MaxValue, 0)) * 40 +
        -- Fragility component (20% weight, inverted)
        ((1 - p.FragilityIndex) * 20)
    ),
    p.PickFrequencyRank = pick_freq.Rank,
    p.LastReclassificationDate = GETDATE()
FROM DimProductGravity p
CROSS APPLY (
    SELECT COUNT(*) AS PicksPerDay,
           DENSE_RANK() OVER (ORDER BY COUNT(*) DESC) AS Rank
    FROM FactWarehouseOperations
    WHERE ProductKey = p.ProductKey
      AND OperationType = 'PICK'
      AND TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
) pick_freq
CROSS APPLY (SELECT MAX(PicksPerDay) AS MaxPicks FROM (
    SELECT COUNT(*) AS PicksPerDay
    FROM FactWarehouseOperations
    WHERE OperationType = 'PICK'
      AND TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
    GROUP BY ProductKey
) x) max_picks
CROSS APPLY (SELECT MAX(UnitValue) AS MaxValue FROM DimProductGravity) max_value;
```

### Fleet Route Optimization Query

```sql
-- Identify inefficient routes (high idle time, low load utilization)
WITH RoutePerformance AS (
    SELECT 
        r.RouteKey,
        r.RouteCode,
        r.OriginNode,
        r.DestinationNode,
        AVG(f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) AS AvgIdlePercent,
        AVG(f.LoadWeightKG / NULLIF(v.MaxCapacityKG, 1)) * 100 AS AvgLoadUtilization,
        AVG(f.FuelLiters / NULLIF(f.DistanceKM, 0)) AS AvgFuelPerKM,
        COUNT(*) AS TotalTrips
    FROM FactFleetTrips f
    JOIN DimRoute r ON f.RouteKey = r.RouteKey
    JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    WHERE f.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
      AND f.TripSegmentType = 'TRANSIT'
    GROUP BY r.RouteKey, r.RouteCode, r.OriginNode, r.DestinationNode
)
SELECT 
    RouteCode,
    OriginNode,
    DestinationNode,
    AvgIdlePercent,
    AvgLoadUtilization,
    AvgFuelPerKM,
    TotalTrips,
    CASE 
        WHEN AvgIdlePercent > 15 AND AvgLoadUtilization < 60 THEN 'CONSOLIDATE_TRIPS'
        WHEN AvgIdlePercent > 20 THEN 'ROUTE_OPTIMIZATION_NEEDED'
        WHEN AvgLoadUtilization < 50 THEN 'INCREASE_LOAD_DENSITY'
        ELSE 'ACCEPTABLE'
    END AS RecommendedAction
FROM RoutePerformance
WHERE AvgIdlePercent > 10 OR AvgLoadUtilization < 70
ORDER BY (AvgIdlePercent * 0.6 + (100 - AvgLoadUtilization) * 0.4) DESC;
```

## Troubleshooting

### Power BI Connection Issues

```powershell
# Test SQL Server connectivity
Test-NetConnection -ComputerName $env:SQL_SERVER_HOST -Port 1433

# Verify credentials
sqlcmd -S $env:SQL_SERVER_HOST -U $env:SQL_USER -P $env:SQL_PASSWORD -Q "SELECT @@VERSION"
```

**Common fixes:**
- Ensure SQL Server allows remote connections
- Enable TCP/IP protocol in SQL Server Configuration Manager
- Add firewall rule for port 1433
- Verify user has db_datareader permissions on LogiFleetPulse database

### Data Refresh Failures

```sql
-- Check ETL control table for last successful load
SELECT 
    TableName,
    LastLoadDateTime,
    RowsProcessed,
    ErrorMessage
FROM ETL_Control
ORDER BY LastLoadDateTime DESC;

-- Verify external table connectivity
SELECT TOP 10 * FROM Fleet_GPS_External;

-- Check for orphaned records
SELECT COUNT(*) AS OrphanCount
FROM FactWarehouseOperations w
LEFT JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL;
```

### Performance Optimization

```sql
-- Create filtered index for recent operations
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Recent
ON FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (DwellTimeMinutes, CycleTimeSeconds)
WHERE TimeKey >= (SELECT MAX(TimeKey) - 8640 FROM DimTime); -- Last 90 days

-- Partition large fact tables
ALTER PARTITION SCHEME PS_FactFleetTrips
NEXT USED [PRIMARY];

ALTER PARTITION FUNCTION PF_TimeKey()
SPLIT RANGE (20260701); -- Add new partition for July 2026

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Gravity Score Anomalies

```sql
-- Validate gravity score calculation
SELECT 
    ProductKey,
    SKU,
    GravityScore,
    PickFrequencyRank,
    UnitValue,
    FragilityIndex,
    CASE 
        WHEN GravityScore < 0 OR GravityScore > 100 THEN 'OUT_OF_RANGE'
        WHEN GravityScore IS NULL THEN 'NULL_VALUE'
        WHEN PickFrequencyRank IS NULL THEN 'MISSING_PICKS'
        ELSE 'VALID'
    END AS ValidationStatus
FROM DimProductGravity
WHERE GravityScore < 0 OR GravityScore > 100 OR GravityScore IS NULL;

-- Recalculate if needed
EXEC sp_RecalculateGravityScores;
```

### Alert Queue Backlog

```sql
-- Check alert processing status
SELECT 
    AlertType,
    COUNT(*) AS PendingCount,
    MIN(GeneratedAt) AS OldestAlert
FROM AlertQueue
WHERE ProcessedAt IS NULL
GROUP BY AlertType;

-- Clear stale alerts (older than 7 days)
DELETE FROM AlertQueue
WHERE GeneratedAt < DATEADD(DAY, -7, GETDATE())
  AND ProcessedAt IS NULL;

-- Manually trigger alert processing
EXEC sp_SendAlerts;
```

## Environment Variables Reference

Required environment variables for deployment:

```bash
# SQL Server connection
export SQL_SERVER_HOST="your-server.database.windows.net"
export SQL_USER="logifleet_user"
export SQL_PASSWORD="your-secure-password"

# External data sources
export WMS_API_ENDPOINT="https://api.yourwms.com/v2"
export WMS_API_KEY="your-wms-api-key"
export TELEMATICS_API_ENDPOINT="https://telematics.provider.com/api"
export TELEMATICS_API_KEY="your-telematics-key"
export WEATHER_API_KEY="your-weather-api-key"

# Blob storage for polybase
export TELEMATICS_BLOB_URL="https://youraccount.blob.core.windows.net/telemetry"
export AZURE_STORAGE_ACCOUNT_KEY="your-storage-key"

# Alert configuration
export SMTP_SERVER="smtp.office365.com"
export SMTP_PORT="587"
export SMTP_USER="alerts@yourcompany.com"
export SMTP_PASSWORD="smtp-password"
export TEAMS_WEBHOOK_URL="https://outlook.office.com/webhook/..."
```
