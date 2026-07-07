---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform using MS SQL Server star schema and Power BI for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse analytics platform
  - deploy supply chain analytics warehouse
  - configure multi-fact star schema for logistics
  - create power bi logistics dashboard
  - implement warehouse gravity zone optimization
  - build fleet telemetry analytics database
  - query cross-modal logistics KPIs
  - setup logicore supply chain data model
---

# LogiFleet Pulse — Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced MS SQL Server data warehousing solution with Power BI visualization for multi-modal logistics intelligence. It unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a semantic layer using a multi-fact star schema with time-phased dimensions.

**Key Capabilities:**
- Multi-fact star schema linking warehouse, fleet, and supplier data
- 15-minute granularity time-aware dimensions
- Cross-fact KPI harmonization (e.g., dwell time vs. fleet idling)
- Predictive bottleneck detection
- Warehouse Gravity Zones™ for spatial optimization
- Real-time Power BI dashboards with role-based access

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Access to source systems: WMS, TMS, telematics APIs

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_bridges.sql
:r schema/05_create_views.sql
:r schema/06_create_stored_procedures.sql
:r schema/07_create_indexes.sql
```

3. **Configure data sources:**
```json
// config.json (create from config_sample.json)
{
  "sqlServer": {
    "server": "YOUR_SQL_SERVER",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "apiEndpoint": "${WMS_API_URL}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telemetrics": {
      "apiEndpoint": "${FLEET_API_URL}",
      "apiKey": "${FLEET_API_KEY}"
    }
  },
  "refreshInterval": 900
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
- Enter SQL Server connection details
- Credentials will be prompted for first-time setup

## Core Data Model

### Star Schema Architecture

The platform uses a multi-fact constellation schema:

```sql
-- Fact Tables (grain defined at transaction/event level)
-- FactWarehouseOperations: one row per warehouse operation
-- FactFleetTrips: one row per route segment
-- FactCrossDock: one row per cross-dock transfer
-- FactInventorySnapshot: one row per SKU per time bucket

-- Shared Dimensions
-- DimTime: 15-minute granularity
-- DimGeography: hierarchical location data
-- DimProduct: product hierarchy with gravity scoring
-- DimSupplier: supplier reliability metrics
-- DimVehicle: fleet asset master
-- DimWarehouse: warehouse/DC master

-- Bridge Tables
-- BridgeRouteZone: many-to-many between routes and storage zones
-- BridgeProductSupplier: product-supplier relationships
```

### Key Dimension: DimTime

```sql
-- Time dimension with 15-minute buckets
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL, -- 0-3 (0=:00-:14, 1=:15-:29, etc.)
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    IsBusinessHour BIT NOT NULL,
    FiscalPeriod VARCHAR(10) NOT NULL,
    FiscalYear SMALLINT NOT NULL
);

-- Index for high-performance time-series queries
CREATE NONCLUSTERED INDEX IX_DimTime_DateTime 
    ON DimTime(DateTime) INCLUDE (DateKey, Hour, IsBusinessHour);
```

### Key Dimension: DimProductGravity

```sql
-- Product dimension with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100) NOT NULL,
    SubCategory VARCHAR(100) NOT NULL,
    UnitValue DECIMAL(12,2) NOT NULL,
    Fragility DECIMAL(3,2) NOT NULL, -- 0.0-1.0 scale
    VelocityScore DECIMAL(5,2) NOT NULL, -- picks per week
    GravityScore AS (VelocityScore * UnitValue * (1 + Fragility)) PERSISTED,
    GravityZone AS (
        CASE 
            WHEN (VelocityScore * UnitValue * (1 + Fragility)) > 1000 THEN 'High'
            WHEN (VelocityScore * UnitValue * (1 + Fragility)) > 500 THEN 'Medium'
            ELSE 'Low'
        END
    ) PERSISTED,
    OptimalZoneDistance INT NULL -- meters from shipping dock
);

-- Composite index for gravity-based queries
CREATE NONCLUSTERED INDEX IX_ProductGravity_Score 
    ON DimProductGravity(GravityZone, GravityScore DESC) 
    INCLUDE (SKU, ProductName, OptimalZoneDistance);
```

### Fact Table: FactWarehouseOperations

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimWarehouse(WarehouseKey),
    ZoneKey INT NOT NULL FOREIGN KEY REFERENCES DimZone(ZoneKey),
    OperationType VARCHAR(20) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT NULL, -- time in zone before next operation
    DistanceTraveledMeters DECIMAL(8,2) NULL,
    OperatorID VARCHAR(20) NULL,
    EquipmentID VARCHAR(20) NULL,
    BatchNumber VARCHAR(50) NULL,
    IsCompliant BIT NOT NULL DEFAULT 1,
    AnomalyFlag BIT NOT NULL DEFAULT 0
);

-- Partitioned by month for performance
CREATE PARTITION FUNCTION PF_MonthlyOperations (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, /* ... */);

CREATE PARTITION SCHEME PS_MonthlyOperations
AS PARTITION PF_MonthlyOperations ALL TO ([PRIMARY]);

-- Clustered columnstore for analytical queries
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations 
    ON FactWarehouseOperations
    ON PS_MonthlyOperations(TimeKey);
```

### Fact Table: FactFleetTrips

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripID VARCHAR(50) NOT NULL UNIQUE,
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT NOT NULL FOREIGN KEY REFERENCES DimRoute(RouteKey),
    DriverKey INT NOT NULL FOREIGN KEY REFERENCES DimDriver(DriverKey),
    DepartureTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ArrivalTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    PlannedDistanceKM DECIMAL(8,2) NOT NULL,
    ActualDistanceKM DECIMAL(8,2) NOT NULL,
    FuelConsumedLiters DECIMAL(8,2) NOT NULL,
    IdleTimeMinutes INT NOT NULL,
    LoadWeightKG DECIMAL(10,2) NOT NULL,
    OnTimeDelivery BIT NOT NULL,
    DelayReasonKey INT NULL FOREIGN KEY REFERENCES DimDelayReason(DelayReasonKey),
    WeatherCondition VARCHAR(50) NULL,
    MaintenanceAlerts INT DEFAULT 0
);

-- Clustered columnstore for fleet analytics
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
    ON FactFleetTrips;

-- Nonclustered indexes for common filters
CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle_Departure 
    ON FactFleetTrips(VehicleKey, DepartureTimeKey) 
    INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
```

## Key Stored Procedures

### Incremental Load for Warehouse Operations

```sql
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @LoadFromDateTime DATETIME2,
    @LoadToDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Stage data from external source
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, ZoneKey,
        OperationType, QuantityHandled, DwellTimeMinutes,
        DistanceTraveledMeters, OperatorID, EquipmentID,
        BatchNumber, IsCompliant, AnomalyFlag
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        z.ZoneKey,
        src.operation_type,
        src.quantity,
        DATEDIFF(MINUTE, src.start_time, src.end_time) AS DwellTimeMinutes,
        src.distance_meters,
        src.operator_id,
        src.equipment_id,
        src.batch_number,
        CASE WHEN src.exception_flag = 0 THEN 1 ELSE 0 END,
        CASE WHEN src.exception_flag = 1 THEN 1 ELSE 0 END
    FROM dbo.StagingWarehouseOperations src
    INNER JOIN DimTime t ON src.operation_datetime = t.DateTime
    INNER JOIN DimProductGravity p ON src.sku = p.SKU
    INNER JOIN DimWarehouse w ON src.warehouse_code = w.WarehouseCode
    INNER JOIN DimZone z ON src.zone_code = z.ZoneCode AND z.WarehouseKey = w.WarehouseKey
    WHERE src.operation_datetime >= @LoadFromDateTime
      AND src.operation_datetime < @LoadToDateTime
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations f
          WHERE f.TimeKey = t.TimeKey
            AND f.ProductKey = p.ProductKey
            AND f.BatchNumber = src.batch_number
      );
    
    -- Log load statistics
    INSERT INTO dbo.ETLLog (ProcedureName, RowsInserted, LoadDateTime)
    VALUES ('usp_LoadWarehouseOperations', @@ROWCOUNT, GETDATE());
END;
```

### Cross-Fact KPI Query: Dwell Time vs Fleet Idling

```sql
CREATE PROCEDURE dbo.usp_GetDwellTimeVsFleetIdling
    @StartDate DATE,
    @EndDate DATE,
    @WarehouseKey INT = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH WarehouseDwell AS (
        SELECT 
            wo.ProductKey,
            p.SKU,
            p.ProductName,
            p.GravityZone,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
            SUM(wo.QuantityHandled) AS TotalQuantity
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        WHERE t.DateKey >= CONVERT(INT, REPLACE(CONVERT(VARCHAR, @StartDate, 112), '-', ''))
          AND t.DateKey <= CONVERT(INT, REPLACE(CONVERT(VARCHAR, @EndDate, 112), '-', ''))
          AND (@WarehouseKey IS NULL OR wo.WarehouseKey = @WarehouseKey)
          AND wo.OperationType = 'Picking'
        GROUP BY wo.ProductKey, p.SKU, p.ProductName, p.GravityZone
    ),
    FleetIdling AS (
        SELECT 
            ft.VehicleKey,
            v.VehicleID,
            AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
            SUM(ft.FuelConsumedLiters) AS TotalFuelLiters
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.DepartureTimeKey = t.TimeKey
        INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
        WHERE t.DateKey >= CONVERT(INT, REPLACE(CONVERT(VARCHAR, @StartDate, 112), '-', ''))
          AND t.DateKey <= CONVERT(INT, REPLACE(CONVERT(VARCHAR, @EndDate, 112), '-', ''))
        GROUP BY ft.VehicleKey, v.VehicleID
    )
    SELECT 
        wd.GravityZone,
        wd.AvgDwellTimeMinutes,
        wd.TotalQuantity,
        fi.AvgIdleTimeMinutes,
        fi.TotalFuelLiters,
        -- Correlation metric: higher dwell + higher idling = inefficiency
        (wd.AvgDwellTimeMinutes / NULLIF(wd.TotalQuantity, 0)) * 
        (fi.AvgIdleTimeMinutes / NULLIF(fi.TotalFuelLiters, 0)) AS InefficiencyIndex
    FROM WarehouseDwell wd
    CROSS JOIN FleetIdling fi
    ORDER BY InefficiencyIndex DESC;
END;
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE dbo.usp_DetectPredictiveBottlenecks
    @ThresholdPercentile DECIMAL(3,2) = 0.90
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Identify zones where dwell time exceeds 90th percentile
    WITH DwellPercentiles AS (
        SELECT 
            ZoneKey,
            PERCENTILE_CONT(@ThresholdPercentile) WITHIN GROUP (ORDER BY DwellTimeMinutes) 
                OVER (PARTITION BY ZoneKey) AS P90DwellTime
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days (15min * 96 * 7)
    ),
    PotentialBottlenecks AS (
        SELECT 
            wo.ZoneKey,
            z.ZoneName,
            w.WarehouseName,
            AVG(wo.DwellTimeMinutes) AS CurrentAvgDwell,
            dp.P90DwellTime,
            COUNT(*) AS OperationCount,
            SUM(CASE WHEN wo.AnomalyFlag = 1 THEN 1 ELSE 0 END) AS AnomalyCount
        FROM FactWarehouseOperations wo
        INNER JOIN DimZone z ON wo.ZoneKey = z.ZoneKey
        INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
        INNER JOIN DwellPercentiles dp ON wo.ZoneKey = dp.ZoneKey
        WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
        GROUP BY wo.ZoneKey, z.ZoneName, w.WarehouseName, dp.P90DwellTime
        HAVING AVG(wo.DwellTimeMinutes) > dp.P90DwellTime * 1.2 -- 20% above P90
    )
    SELECT 
        pb.*,
        'High Risk' AS RiskLevel,
        CASE 
            WHEN pb.AnomalyCount > 5 THEN 'Immediate Investigation Required'
            ELSE 'Monitor Closely'
        END AS Recommendation
    FROM PotentialBottlenecks pb
    ORDER BY pb.CurrentAvgDwell DESC;
END;
```

## Power BI Integration

### DAX Measures for Cross-Fact KPIs

**Average Dwell Time per Gravity Zone:**
```dax
Avg Dwell Time by Gravity = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    FactWarehouseOperations[OperationType] = "Picking"
)
```

**Fleet Efficiency Ratio:**
```dax
Fleet Efficiency % = 
VAR TotalTripTime = SUM(FactFleetTrips[ActualDistanceKM]) / AVERAGE(FactFleetTrips[PlannedDistanceKM])
VAR IdlePercentage = DIVIDE(SUM(FactFleetTrips[IdleTimeMinutes]), SUM(FactFleetTrips[ActualDistanceKM]) * 60, 0)
RETURN
    (1 - IdlePercentage) * TotalTripTime
```

**Cross-Fact: Cost per Delayed Shipment:**
```dax
Cost Per Delayed Shipment = 
VAR DelayedShipments = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[OnTimeDelivery] = FALSE
    )
VAR TotalDwellCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] * 0.05  -- $0.05 per minute storage cost
    )
VAR TotalFuelCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelConsumedLiters] * 1.30  -- $1.30 per liter
    )
RETURN
    DIVIDE(TotalDwellCost + TotalFuelCost, DelayedShipments, 0)
```

**Warehouse Gravity Zone Utilization:**
```dax
Gravity Zone Utilization = 
VAR HighGravityOps = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        RELATED(DimProductGravity[GravityZone]) = "High"
    )
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(HighGravityOps, TotalOps, 0)
```

### Connecting Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit`
2. Enter connection parameters:
   - Server: `YOUR_SQL_SERVER`
   - Database: `LogiFleetPulse`
3. Choose authentication method (Windows or SQL Server)
4. Select tables to load (all fact and dimension tables)
5. Relationships are auto-detected from the schema

### Row-Level Security (RLS)

```dax
-- Create security table in SQL first
CREATE TABLE dbo.SecurityRoles (
    UserEmail VARCHAR(100) NOT NULL,
    WarehouseKey INT NULL,
    GeographyKey INT NULL,
    RoleLevel VARCHAR(20) NOT NULL  -- 'User', 'Supervisor', 'Executive'
);

-- DAX filter in Power BI
[WarehouseRLS] = 
VAR CurrentUser = USERPRINCIPALNAME()
VAR AllowedWarehouses = 
    CALCULATETABLE(
        VALUES(SecurityRoles[WarehouseKey]),
        SecurityRoles[UserEmail] = CurrentUser
    )
RETURN
    FactWarehouseOperations[WarehouseKey] IN AllowedWarehouses
    || ISEMPTY(AllowedWarehouses)  -- Executives see all
```

Apply filter to FactWarehouseOperations and FactFleetTrips tables.

## Common Patterns and Queries

### Pattern 1: Time-Series Analysis with 15-Minute Buckets

```sql
-- Warehouse activity heatmap by hour and quarter-hour
SELECT 
    t.DayName,
    t.Hour,
    t.QuarterHour,
    COUNT(*) AS OperationCount,
    AVG(wo.DwellTimeMinutes) AS AvgDwell
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateKey >= 20260601
  AND t.DateKey < 20260701
GROUP BY t.DayName, t.Hour, t.QuarterHour
ORDER BY 
    CASE t.DayName 
        WHEN 'Monday' THEN 1
        WHEN 'Tuesday' THEN 2
        WHEN 'Wednesday' THEN 3
        WHEN 'Thursday' THEN 4
        WHEN 'Friday' THEN 5
        WHEN 'Saturday' THEN 6
        WHEN 'Sunday' THEN 7
    END,
    t.Hour,
    t.QuarterHour;
```

### Pattern 2: Gravity Zone Rebalancing Recommendation

```sql
-- Identify products that should move to different zones
WITH CurrentPlacement AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.GravityZone AS CurrentZone,
        p.GravityScore,
        z.ZoneName,
        z.DistanceFromDockMeters AS CurrentDistance,
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        COUNT(*) AS PickCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimZone z ON wo.ZoneKey = z.ZoneKey
    WHERE wo.OperationType = 'Picking'
      AND wo.TimeKey >= (SELECT MAX(TimeKey) - 2016 FROM DimTime)  -- Last 3 weeks
    GROUP BY p.ProductKey, p.SKU, p.GravityZone, p.GravityScore, z.ZoneName, z.DistanceFromDockMeters
),
OptimalPlacement AS (
    SELECT 
        ProductKey,
        SKU,
        CASE 
            WHEN GravityScore > 1000 AND CurrentDistance > 50 THEN 'Move to High Gravity Zone'
            WHEN GravityScore < 500 AND CurrentDistance < 100 THEN 'Move to Low Gravity Zone'
            ELSE 'Current Placement OK'
        END AS Recommendation,
        GravityScore,
        CurrentDistance,
        AvgDwell,
        PickCount
    FROM CurrentPlacement
)
SELECT * 
FROM OptimalPlacement
WHERE Recommendation LIKE 'Move%'
ORDER BY GravityScore DESC, PickCount DESC;
```

### Pattern 3: Fleet Maintenance Priority Queue

```sql
-- Adaptive fleet triage based on telemetry and revenue impact
WITH VehicleHealth AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.MaintenanceStatus,
        AVG(ft.MaintenanceAlerts) AS AvgAlerts,
        SUM(ft.LoadWeightKG) AS TotalLoadKG,
        SUM(ft.IdleTimeMinutes) AS TotalIdleTime,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    WHERE ft.DepartureTimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)  -- Last 7 days
    GROUP BY v.VehicleKey, v.VehicleID, v.MaintenanceStatus
),
RevenueImpact AS (
    SELECT 
        ft.VehicleKey,
        SUM(p.UnitValue * wo.QuantityHandled) AS EstimatedCargoValue
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wo ON ft.TripID = wo.BatchNumber
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE ft.DepartureTimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
    GROUP BY ft.VehicleKey
)
SELECT 
    vh.VehicleID,
    vh.MaintenanceStatus,
    vh.AvgAlerts,
    vh.TotalIdleTime,
    ISNULL(ri.EstimatedCargoValue, 0) AS RevenueAtRisk,
    -- Priority score: alerts * revenue * (1 + idle ratio)
    vh.AvgAlerts * ISNULL(ri.EstimatedCargoValue, 0) * 
        (1 + CAST(vh.TotalIdleTime AS DECIMAL) / NULLIF(vh.TripCount * 60, 0)) AS PriorityScore,
    CASE 
        WHEN vh.AvgAlerts > 3 AND ISNULL(ri.EstimatedCargoValue, 0) > 50000 THEN 'Critical - Immediate Action'
        WHEN vh.AvgAlerts > 2 THEN 'High - Schedule This Week'
        WHEN vh.AvgAlerts > 1 THEN 'Medium - Monitor'
        ELSE 'Low - Routine Check'
    END AS MaintenancePriority
FROM VehicleHealth vh
LEFT JOIN RevenueImpact ri ON vh.VehicleKey = ri.VehicleKey
WHERE vh.MaintenanceStatus != 'In Service'
ORDER BY PriorityScore DESC;
```

### Pattern 4: Temporal Elasticity Simulation

```sql
-- Simulate impact of increased warehouse capacity on fleet utilization
DECLARE @CapacityIncreasePercent DECIMAL(5,2) = 0.15;  -- 15% increase

WITH BaselineMetrics AS (
    SELECT 
        AVG(wo.DwellTimeMinutes) AS BaselineDwell,
        COUNT(DISTINCT ft.TripKey) AS BaselineTrips,
        AVG(ft.IdleTimeMinutes) AS BaselineIdleTime
    FROM FactWarehouseOperations wo
    CROSS APPLY (
        SELECT TOP 1 * FROM FactFleetTrips ft
        WHERE ft.DepartureTimeKey >= wo.TimeKey
          AND ft.DepartureTimeKey <= wo.TimeKey + 96  -- Within 24 hours
    ) ft
    WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 2016 FROM DimTime)
),
SimulatedMetrics AS (
    SELECT 
        AVG(wo.DwellTimeMinutes * (1 - @CapacityIncreasePercent * 0.6)) AS SimulatedDwell,  -- Assume 60% efficiency gain
        COUNT(DISTINCT ft.TripKey) AS SimulatedTrips,
        AVG(ft.IdleTimeMinutes * (1 - @CapacityIncreasePercent * 0.4)) AS SimulatedIdleTime  -- 40% reduction
    FROM FactWarehouseOperations wo
    CROSS APPLY (
        SELECT TOP 1 * FROM FactFleetTrips ft
        WHERE ft.DepartureTimeKey >= wo.TimeKey
          AND ft.DepartureTimeKey <= wo.TimeKey + 96
    ) ft
    WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 2016 FROM DimTime)
)
SELECT 
    bm.BaselineDwell,
    sm.SimulatedDwell,
    (bm.BaselineDwell - sm.SimulatedDwell) AS DwellReduction,
    bm.BaselineIdleTime,
    sm.SimulatedIdleTime,
    (bm.BaselineIdleTime - sm.SimulatedIdleTime) AS IdleTimeReduction,
    bm.BaselineTrips,
    sm.SimulatedTrips,
    CAST((sm.SimulatedTrips - bm.BaselineTrips) AS DECIMAL) / NULLIF(bm.BaselineTrips, 0) * 100 AS TripVolumeChangePercent
FROM BaselineMetrics bm
CROSS JOIN SimulatedMetrics sm;
```

## Configuration and Environment Variables

### SQL Server Connection String

```bash
# .env file
SQL_SERVER=your-server.database.windows.net
SQL_DATABASE=LogiFleetPulse
SQL_USER=logifleet_admin
SQL_PASSWORD=${SECURE_SQL_PASSWORD}
SQL_DRIVER=ODBC Driver 17 for SQL Server

# Python connection example
import pyodbc
import os

conn_str = (
    f"DRIVER={{{os.getenv('SQL_DRIVER')}}};"
    f"SERVER={os.getenv('SQL_SERVER')};"
    f"DATABASE={os.getenv('SQL_DATABASE')};"
    f"UID={os.getenv('SQL_USER')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)
conn = pyodbc.connect(conn_str)
```

### External API Configuration

```bash
# Fleet telemetry API
FLEET_API_URL=https://api.fleet-provider.com/v2
FLEET_API_KEY=${FLEET_TELEMETRY_KEY}

# Weather API for delay correlation
WEATHER_API_URL=https://api.weatherprovider.com
WEATHER_API_KEY=${WEATHER_SERVICE_KEY}

# WMS webhook endpoint
WMS_WEBHOOK_SECRET=${WMS_SECRET_TOKEN}
```

### Scheduled Refresh Configuration

```sql
-- SQL Server Agent job for incremental loads (every 15 minutes)
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleet_IncrementalLoad',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Load Warehouse Operations',
