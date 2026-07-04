---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehousing platform for logistics, fleet management, and supply chain analytics with multi-fact star schema modeling
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure multi-fact star schema for fleet management
  - implement warehouse gravity zones in sql server
  - create fleet optimization dashboards
  - build cross-modal supply chain analytics
  - integrate telemetry data with warehouse operations
  - configure logifleet pulse kpi dashboards
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single analytical data warehouse. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema that enables cross-domain KPI analysis (e.g., correlating warehouse dwell time with fleet fuel consumption).

**Core Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Real-time warehouse and fleet operations tracking
- Predictive bottleneck detection and fleet triage
- Warehouse Gravity Zones™ for optimal spatial organization
- Cross-fact KPI harmonization (warehouse ↔ fleet ↔ supplier metrics)
- Role-based security and automated alerting

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, ERP systems

### Step 1: Deploy SQL Schema

Clone the repository and navigate to the SQL scripts directory:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse/sql
```

Execute the schema deployment script on your SQL Server instance:

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the main schema script
-- This creates all tables, views, stored procedures, and relationships
:r schema_complete.sql
GO

-- Execute indexing strategy
:r indexing_strategy.sql
GO

-- Set up row-level security
:r security_config.sql
GO
```

### Step 2: Configure Data Connections

Create a configuration file for your data source connections:

```json
{
  "connections": {
    "wms": {
      "type": "SQL",
      "server": "${WMS_SQL_SERVER}",
      "database": "${WMS_DATABASE}",
      "auth": "integrated"
    },
    "telemetry": {
      "type": "REST_API",
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "api_key_env": "FLEET_API_KEY",
      "refresh_interval_minutes": 15
    },
    "erp": {
      "type": "ODBC",
      "dsn": "${ERP_DSN}",
      "username_env": "ERP_USERNAME",
      "password_env": "ERP_PASSWORD"
    }
  },
  "refresh_schedule": {
    "warehouse_ops": "*/15 * * * *",
    "fleet_trips": "*/15 * * * *",
    "supplier_data": "0 2 * * *"
  }
}
```

### Step 3: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter SQL Server connection details when prompted
5. Configure refresh schedule in Power BI Service

## Core Data Model Structure

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    OperatorKey INT,
    GravityZoneKey INT,
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WarehouseOps_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WarehouseOps_GravityZone FOREIGN KEY (GravityZoneKey) REFERENCES DimGravityZone(GravityZoneKey)
);

CREATE NONCLUSTERED INDEX IX_WarehouseOps_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_WarehouseOps_DwellTime ON FactWarehouseOperations(DwellTimeMinutes);
```

**FactFleetTrips** - Fleet and route tracking:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginWarehouseKey INT,
    DestinationKey INT,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AverageSpeedKMH DECIMAL(5,2),
    WeatherConditionKey INT,
    OnTimeDelivery BIT,
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrips_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FleetTrips_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle ON FactFleetTrips(VehicleKey);
CREATE NONCLUSTERED INDEX IX_FleetTrips_IdleTime ON FactFleetTrips(IdleTimeMinutes);
```

### Key Dimension Tables

**DimTime** - 15-minute granularity time dimension:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot15Min INT NOT NULL, -- 0-95 (96 slots per day)
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthNumber TINYINT NOT NULL,
    MonthName VARCHAR(10) NOT NULL,
    Quarter TINYINT NOT NULL,
    Year SMALLINT NOT NULL,
    FiscalPeriod VARCHAR(20),
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);
```

**DimGravityZone** - Warehouse spatial optimization:

```sql
CREATE TABLE DimGravityZone (
    GravityZoneKey INT PRIMARY KEY,
    ZoneName VARCHAR(100) NOT NULL,
    WarehouseKey INT NOT NULL,
    GravityScore DECIMAL(5,2) NOT NULL, -- Composite score: velocity + value + fragility
    DistanceToDockMeters INT,
    PickFrequencyDaily INT,
    AverageItemValue DECIMAL(10,2),
    FragilityIndex DECIMAL(3,2), -- 0.0 to 1.0
    OptimalCapacityUnits INT,
    CurrentOccupancyUnits INT,
    LastReoptimizedDate DATETIME,
    CONSTRAINT FK_GravityZone_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);
```

## Common SQL Queries & Patterns

### Cross-Fact KPI: Dwell Time vs. Fleet Idle Correlation

```sql
-- Analyze correlation between warehouse dwell time and fleet idle time
-- for shipments originating from specific gravity zones
WITH WarehouseDwell AS (
    SELECT 
        wo.ProductKey,
        wo.GravityZoneKey,
        gz.ZoneName,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimGravityZone gz ON wo.GravityZoneKey = gz.GravityZoneKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMdd'))
    AND wo.OperationType = 'Shipping'
    GROUP BY wo.ProductKey, wo.GravityZoneKey, gz.ZoneName
),
FleetIdle AS (
    SELECT 
        ft.VehicleKey,
        ft.OriginWarehouseKey,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMdd'))
    GROUP BY ft.VehicleKey, ft.OriginWarehouseKey
)
SELECT 
    wd.ZoneName,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    CASE 
        WHEN wd.AvgDwellTime > 4320 AND fi.AvgIdleTime > 45 THEN 'High Risk'
        WHEN wd.AvgDwellTime > 2880 OR fi.AvgIdleTime > 30 THEN 'Medium Risk'
        ELSE 'Normal'
    END AS RiskCategory,
    wd.OperationCount,
    fi.TripCount
FROM WarehouseDwell wd
CROSS JOIN FleetIdle fi
WHERE fi.OriginWarehouseKey = (
    SELECT TOP 1 WarehouseKey 
    FROM DimGravityZone 
    WHERE GravityZoneKey = wd.GravityZoneKey
)
ORDER BY wd.AvgDwellTime DESC, fi.AvgIdleTime DESC;
```

### Warehouse Gravity Zone Optimization Recommendation

```sql
-- Identify products that should be moved to different gravity zones
-- based on actual pick frequency vs. current zone assignment
WITH ProductVelocity AS (
    SELECT 
        wo.ProductKey,
        p.ProductName,
        wo.GravityZoneKey,
        gz.ZoneName AS CurrentZone,
        gz.GravityScore AS CurrentGravityScore,
        COUNT(*) AS PickCount,
        AVG(wo.CycleTimeSeconds) AS AvgCycleTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGravityZone gz ON wo.GravityZoneKey = gz.GravityZoneKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.OperationType = 'Picking'
    AND t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd'))
    GROUP BY wo.ProductKey, p.ProductName, wo.GravityZoneKey, gz.ZoneName, gz.GravityScore
),
OptimalZones AS (
    SELECT 
        GravityZoneKey,
        ZoneName,
        GravityScore,
        DistanceToDockMeters,
        (CurrentOccupancyUnits * 100.0 / OptimalCapacityUnits) AS OccupancyPct
    FROM DimGravityZone
    WHERE (CurrentOccupancyUnits * 100.0 / OptimalCapacityUnits) < 85 -- Available capacity
)
SELECT 
    pv.ProductName,
    pv.CurrentZone,
    pv.CurrentGravityScore,
    pv.PickCount,
    pv.AvgCycleTime,
    oz.ZoneName AS RecommendedZone,
    oz.GravityScore AS RecommendedGravityScore,
    oz.DistanceToDockMeters,
    (pv.PickCount * (pv.AvgCycleTime - (oz.DistanceToDockMeters / 50.0))) AS EstimatedTimeSavingsSeconds
FROM ProductVelocity pv
CROSS APPLY (
    SELECT TOP 1 *
    FROM OptimalZones oz
    WHERE oz.GravityScore > pv.CurrentGravityScore
    AND oz.OccupancyPct < 85
    ORDER BY oz.GravityScore DESC, oz.DistanceToDockMeters ASC
) oz
WHERE pv.PickCount > 10 -- Minimum threshold for recommendation
ORDER BY EstimatedTimeSavingsSeconds DESC;
```

### Fleet Triage Priority Queue

```sql
-- Generate proactive maintenance queue based on telemetry and revenue impact
DECLARE @CurrentTimeKey INT = 
    (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(MINUTE, -15, GETDATE()));

WITH VehicleHealth AS (
    SELECT 
        ft.VehicleKey,
        v.VehicleNumber,
        v.VehicleType,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) AS FuelEfficiency,
        AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.IdleTimeMinutes + ft.LoadingTimeMinutes + ft.UnloadingTimeMinutes, 0)) AS IdlePercentage,
        COUNT(CASE WHEN ft.OnTimeDelivery = 0 THEN 1 END) AS DelayedTrips,
        COUNT(*) AS TotalTrips
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd'))
    GROUP BY ft.VehicleKey, v.VehicleNumber, v.VehicleType
),
RevenueImpact AS (
    SELECT 
        ft.VehicleKey,
        SUM(ISNULL(s.OrderValue, 0)) AS WeeklyRevenue
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    LEFT JOIN FactShipments s ON ft.TripKey = s.TripKey -- Assuming bridge table exists
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd'))
    GROUP BY ft.VehicleKey
)
SELECT 
    vh.VehicleNumber,
    vh.VehicleType,
    vh.FuelEfficiency,
    vh.IdlePercentage,
    vh.DelayedTrips,
    ri.WeeklyRevenue,
    -- Priority score: weighted by efficiency degradation, delays, and revenue
    (
        (CASE WHEN vh.FuelEfficiency > 0.12 THEN 30 ELSE 0 END) + -- Poor fuel efficiency
        (vh.DelayedTrips * 5) + -- Each delay adds points
        (CASE WHEN vh.IdlePercentage > 20 THEN 25 ELSE 0 END) + -- Excessive idling
        (ri.WeeklyRevenue / 1000) -- Revenue impact factor
    ) AS PriorityScore,
    CASE 
        WHEN vh.FuelEfficiency > 0.12 THEN 'Check engine/tire pressure'
        WHEN vh.IdlePercentage > 20 THEN 'Driver behavior coaching needed'
        WHEN vh.DelayedTrips > 3 THEN 'Route optimization required'
        ELSE 'Normal monitoring'
    END AS RecommendedAction
FROM VehicleHealth vh
INNER JOIN RevenueImpact ri ON vh.VehicleKey = ri.VehicleKey
ORDER BY PriorityScore DESC;
```

## Stored Procedures for ETL & Alerting

### Incremental Load from WMS

```sql
CREATE PROCEDURE usp_IncrementalLoad_WarehouseOps
    @LastLoadTimeKey INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operations since last load
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        ProductKey,
        WarehouseKey,
        OperationType,
        QuantityHandled,
        DwellTimeMinutes,
        CycleTimeSeconds,
        OperatorKey,
        GravityZoneKey
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        src.operation_type,
        src.quantity,
        DATEDIFF(MINUTE, src.start_time, src.end_time) AS DwellTimeMinutes,
        DATEDIFF(SECOND, src.start_time, src.end_time) AS CycleTimeSeconds,
        o.OperatorKey,
        gz.GravityZoneKey
    FROM OPENQUERY(WMS_LINKED_SERVER, 
        'SELECT * FROM warehouse_operations WHERE operation_timestamp > (SELECT last_load FROM sync_status)') AS src
    INNER JOIN DimTime t ON t.FullDateTime = DATEADD(MINUTE, (DATEPART(MINUTE, src.operation_timestamp) / 15) * 15, 
                                                      DATEADD(HOUR, DATEDIFF(HOUR, 0, src.operation_timestamp), 0))
    INNER JOIN DimProduct p ON p.SKU = src.product_sku
    INNER JOIN DimWarehouse w ON w.WarehouseCode = src.warehouse_code
    INNER JOIN DimOperator o ON o.OperatorID = src.operator_id
    LEFT JOIN DimGravityZone gz ON gz.WarehouseKey = w.WarehouseKey 
                                AND src.storage_zone = gz.ZoneName
    WHERE t.TimeKey > @LastLoadTimeKey;
    
    -- Update sync status
    UPDATE sync_status 
    SET last_load = GETDATE(), 
        records_loaded = @@ROWCOUNT;
        
    RETURN @@ROWCOUNT;
END;
GO
```

### Automated KPI Alert System

```sql
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertTable TABLE (
        AlertType VARCHAR(100),
        Severity VARCHAR(20),
        Message VARCHAR(500),
        Value DECIMAL(10,2),
        Threshold DECIMAL(10,2)
    );
    
    -- Check 1: Fleet idle time threshold (>15% of trip duration)
    INSERT INTO @AlertTable
    SELECT 
        'Fleet Idle Time Excessive' AS AlertType,
        'High' AS Severity,
        'Vehicle ' + v.VehicleNumber + ' idle time ' + CAST(AVG(IdleTimeMinutes) AS VARCHAR) + ' mins' AS Message,
        AVG(IdleTimeMinutes * 100.0 / NULLIF(IdleTimeMinutes + LoadingTimeMinutes + UnloadingTimeMinutes, 0)) AS Value,
        15.0 AS Threshold
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -1, GETDATE()), 'yyyyMMdd'))
    GROUP BY ft.VehicleKey, v.VehicleNumber
    HAVING AVG(IdleTimeMinutes * 100.0 / NULLIF(IdleTimeMinutes + LoadingTimeMinutes + UnloadingTimeMinutes, 0)) > 15;
    
    -- Check 2: Warehouse dwell time (>72 hours)
    INSERT INTO @AlertTable
    SELECT 
        'Warehouse Dwell Time Alert' AS AlertType,
        'Medium' AS Severity,
        'SKU ' + p.SKU + ' in zone ' + gz.ZoneName + ' dwell: ' + CAST(AVG(DwellTimeMinutes)/60 AS VARCHAR) + ' hours' AS Message,
        AVG(DwellTimeMinutes) AS Value,
        4320.0 AS Threshold -- 72 hours in minutes
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGravityZone gz ON wo.GravityZoneKey = gz.GravityZoneKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -1, GETDATE()), 'yyyyMMdd'))
    AND wo.OperationType = 'Putaway'
    GROUP BY wo.ProductKey, p.SKU, wo.GravityZoneKey, gz.ZoneName
    HAVING AVG(DwellTimeMinutes) > 4320;
    
    -- Send alerts (integrate with email/Teams)
    IF EXISTS (SELECT 1 FROM @AlertTable)
    BEGIN
        -- Log alerts to table
        INSERT INTO AlertLog (AlertDateTime, AlertType, Severity, Message, Value, Threshold)
        SELECT GETDATE(), AlertType, Severity, Message, Value, Threshold
        FROM @AlertTable;
        
        -- TODO: Integrate with email/Teams notification system
        -- Example: sp_send_dbmail or REST API call to Teams webhook
    END;
    
    SELECT * FROM @AlertTable;
END;
GO
```

## Power BI DAX Measures

### Cross-Fact Composite KPI

```dax
Fleet Efficiency Index = 
VAR AvgIdlePct = 
    AVERAGE(FactFleetTrips[IdleTimeMinutes]) / 
    (AVERAGE(FactFleetTrips[IdleTimeMinutes]) + AVERAGE(FactFleetTrips[LoadingTimeMinutes]) + AVERAGE(FactFleetTrips[UnloadingTimeMinutes]))
VAR AvgFuelEfficiency = 
    AVERAGE(FactFleetTrips[FuelConsumedLiters]) / AVERAGE(FactFleetTrips[DistanceKM])
VAR OnTimeRate = 
    CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeDelivery] = TRUE()) / COUNTROWS(FactFleetTrips)
RETURN
    (1 - AvgIdlePct) * 0.3 + 
    (1 - (AvgFuelEfficiency / 0.15)) * 0.3 + -- Normalize against 0.15 L/km baseline
    OnTimeRate * 0.4
```

### Warehouse Gravity Zone Performance

```dax
Gravity Zone Efficiency Score = 
VAR CurrentZone = SELECTEDVALUE(DimGravityZone[ZoneName])
VAR AvgPickTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[CycleTimeSeconds]),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR ExpectedPickTime = 
    (SELECTEDVALUE(DimGravityZone[DistanceToDockMeters]) / 50) + 30 -- 50m per second walk + 30s base
VAR OccupancyRate = 
    SELECTEDVALUE(DimGravityZone[CurrentOccupancyUnits]) / 
    SELECTEDVALUE(DimGravityZone[OptimalCapacityUnits])
RETURN
    IF(
        ISBLANK(AvgPickTime),
        BLANK(),
        (ExpectedPickTime / AvgPickTime) * 0.6 + (1 - OccupancyRate) * 0.4
    )
```

### Predictive Bottleneck Index

```dax
Bottleneck Risk Index = 
VAR WarehouseCapacity = 
    CALCULATE(
        SUM(DimGravityZone[CurrentOccupancyUnits]) / SUM(DimGravityZone[OptimalCapacityUnits])
    )
VAR FleetUtilization = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FILTER(DimTime, DimTime[HourOfDay] >= 8 && DimTime[HourOfDay] <= 18)
    ) / CALCULATE(COUNTROWS(FactFleetTrips))
VAR DelayRate = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[OnTimeDelivery] = FALSE()
    ) / COUNTROWS(FactFleetTrips)
RETURN
    (WarehouseCapacity * 0.4) + (FleetUtilization * 0.3) + (DelayRate * 0.3)
```

## Configuration & Environment Variables

### SQL Server Connection String Template

```plaintext
# Environment variables for SQL Server connection
LOGIFLEET_SQL_SERVER=your-server.database.windows.net
LOGIFLEET_SQL_DATABASE=LogiFleetPulse
LOGIFLEET_SQL_USERNAME=logifleet_admin
LOGIFLEET_SQL_PASSWORD=<use secure secret management>
LOGIFLEET_SQL_ENCRYPT=true
LOGIFLEET_SQL_TRUST_CERT=false
```

### Data Source API Configuration

```plaintext
# Fleet telemetry API
FLEET_TELEMETRY_ENDPOINT=https://api.fleetprovider.com/v2/telemetry
FLEET_API_KEY=<stored in Azure Key Vault or similar>
FLEET_REFRESH_INTERVAL=15

# Weather data for delay correlation
WEATHER_API_ENDPOINT=https://api.weather.com/v1/conditions
WEATHER_API_KEY=<stored in secure vault>

# WMS integration
WMS_SQL_SERVER=wms-prod.internal
WMS_DATABASE=WarehouseManagement
WMS_USERNAME=readonly_user
WMS_PASSWORD=<use integrated auth or secret manager>
```

### Power BI Service Configuration

```json
{
  "workspace": "LogiFleet Analytics",
  "refresh_schedule": {
    "enabled": true,
    "frequency": "every_15_minutes",
    "timezone": "UTC",
    "retry_on_failure": true,
    "max_retries": 3
  },
  "gateway": {
    "name": "OnPrem-Gateway-01",
    "cluster": "production"
  },
  "row_level_security": {
    "enabled": true,
    "roles": [
      {
        "role": "warehouse_manager",
        "filter": "[DimWarehouse][WarehouseKey] = LOOKUPVALUE([DimUser][AssignedWarehouseKey], [DimUser][Username], USERNAME())"
      },
      {
        "role": "fleet_supervisor",
        "filter": "[DimVehicle][FleetRegion] = LOOKUPVALUE([DimUser][AssignedRegion], [DimUser][Username], USERNAME())"
      },
      {
        "role": "executive",
        "filter": "1=1"
      }
    ]
  }
}
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom:** Queries joining FactWarehouseOperations and FactFleetTrips take >10 seconds

**Solution:**
1. Verify columnstore indexes exist on fact tables for large datasets (>10M rows):
```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps
ON FactWarehouseOperations (TimeKey, ProductKey, GravityZoneKey, DwellTimeMinutes);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON FactFleetTrips (TimeKey, VehicleKey, RouteKey, IdleTimeMinutes, FuelConsumedLiters);
```

2. Check query plan for missing statistics:
```sql
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

3. Partition large fact tables by DateKey:
```sql
-- Example partitioning scheme (monthly partitions)
CREATE PARTITION FUNCTION PF_DateKey (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401);

CREATE PARTITION SCHEME PS_DateKey
AS PARTITION PF_DateKey ALL TO ([PRIMARY]);
```

### Issue: Power BI Refresh Failures

**Symptom:** Scheduled refresh fails with "timeout" or "credential" errors

**Solution:**
1. Verify gateway connectivity:
```powershell
# Test SQL Server connection from gateway machine
Test-NetConnection -ComputerName $env:LOGIFLEET_SQL_SERVER -Port 1433
```

2. Increase command timeout in Power Query Advanced Options:
```powerquery
// In Power Query M code, add timeout parameter
Source = Sql.Databases(ServerName, [CommandTimeout=#duration(0, 1, 0, 0)]) // 1 hour timeout
```

3. Check gateway service account permissions:
```sql
-- Grant necessary permissions to gateway service account
GRANT SELECT ON SCHEMA::dbo TO [DOMAIN\GatewayServiceAccount];
GRANT EXECUTE ON SCHEMA::dbo TO [DOMAIN\GatewayServiceAccount];
```

### Issue: Gravity Zone Recommendations Not Updating

**Symptom:** DimGravityZone.GravityScore remains static despite changing pick patterns

**Solution:**
1. Verify gravity score recalculation job is running:
```sql
-- Check last execution of recalculation procedure
SELECT 
    job_name,
    last_run_date,
    last_run_time,
    last_run_outcome
FROM msdb.dbo.sysjobs j
INNER JOIN msdb.dbo.sysjobhistory h ON j.job_id = h.job_id
WHERE job_name = '
