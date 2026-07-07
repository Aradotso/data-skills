---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine with MS SQL Server data warehousing and Power BI dashboards for supply chain orchestration
triggers:
  - "set up LogiFleet Pulse logistics analytics"
  - "configure supply chain data warehouse with Power BI"
  - "implement multi-fact star schema for logistics"
  - "create warehouse and fleet analytics dashboards"
  - "deploy LogiCore Analytics supply chain solution"
  - "build logistics intelligence platform with SQL Server"
  - "integrate warehouse and fleet telemetry data"
  - "set up cross-modal supply chain KPI tracking"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** for cross-domain KPI harmonization
- **Real-time dashboards** refreshed every 15 minutes
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Warehouse gravity zones** for spatial optimization
- **Fleet triage engine** with proactive maintenance prioritization

The platform is designed for 3PL operators, retail chains, food distributors, and any organization managing complex supply chains.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telemetry APIs, ERP systems
- Optional: Azure Synapse Analytics for big data enrichment

### Step 1: Deploy SQL Schema

```sql
-- Execute the main schema deployment script
-- Assumes you have a database named LogiFleetDB created

USE LogiFleetDB;
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    UnitsProcessed INT,
    ProcessingDuration DECIMAL(10,2), -- in minutes
    StaffHours DECIMAL(10,2),
    ZoneGravityScore DECIMAL(5,2),
    CONSTRAINT FK_Warehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Warehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_Warehouse_Location FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey)
);

CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    DistanceKm DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    AvgSpeed DECIMAL(5,2),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE TABLE FactCrossDock (
    CrossDockID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferDurationMinutes INT,
    UnitsTransferred INT,
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Create time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    Day INT,
    HourOfDay INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    DayOfWeek VARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10)
);

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Based on pick frequency
    ValueScore DECIMAL(5,2), -- Based on unit price
    FragilityScore DECIMAL(5,2), -- 1-10 scale
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(20)
);

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200),
    LocationType VARCHAR(20), -- 'WAREHOUSE', 'ROUTE_NODE', 'DOCK'
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6)
);

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    AvgLeadTimeDays DECIMAL(5,2),
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityIndex AS (100 - (LeadTimeVariance * 0.3 + DefectRatePercent * 0.4 + (100 - ComplianceScore) * 0.3)) PERSISTED
);
```

### Step 2: Create Indexes and Partitioning

```sql
-- Optimize fact tables with columnstore indexes for analytics
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps 
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, OperationType, DwellTimeMinutes);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips 
ON FactFleetTrips (TimeKey, VehicleKey, RouteKey, DistanceKm, FuelLiters, IdleTimeMinutes);

-- Create filtered indexes for common queries
CREATE INDEX IX_WarehouseOps_HighDwell 
ON FactWarehouseOperations (DwellTimeMinutes, TimeKey) 
WHERE DwellTimeMinutes > 72;

CREATE INDEX IX_FleetTrips_DelayedRoutes 
ON FactFleetTrips (DelayMinutes, RouteKey, TimeKey) 
WHERE DelayMinutes > 30;

-- Partition strategy for large fact tables (by month)
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, 202604, 202605, 202606);

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly ALL TO ([PRIMARY]);

-- Apply partitioning to FactWarehouseOperations
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- Same columns as above
    -- ...
) ON PS_Monthly(TimeKey);
```

### Step 3: Load Sample Data (ETL Stored Procedure)

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME,
    @CurrentLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Extract from source WMS (external table or linked server)
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType, 
        DwellTimeMinutes, UnitsProcessed, ProcessingDuration, 
        StaffHours, ZoneGravityScore
    )
    SELECT 
        dt.TimeKey,
        dp.ProductKey,
        dg.GeographyKey,
        src.OperationType,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        src.UnitsProcessed,
        src.ProcessingDuration,
        src.StaffHours,
        dp.GravityScore
    FROM ExternalWMS.dbo.Operations src
    INNER JOIN DimTime dt ON dt.FullDateTime = DATEADD(MINUTE, 
        (DATEPART(MINUTE, src.OperationDateTime) / 15) * 15, 
        DATEADD(HOUR, DATEDIFF(HOUR, 0, src.OperationDateTime), 0))
    INNER JOIN DimProductGravity dp ON dp.SKU = src.SKU
    INNER JOIN DimGeography dg ON dg.LocationID = src.WarehouseID
    WHERE src.OperationDateTime >= @LastLoadDateTime
      AND src.OperationDateTime < @CurrentLoadDateTime;
      
    -- Update metadata table
    UPDATE ETL_Metadata 
    SET LastLoadDateTime = @CurrentLoadDateTime 
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Step 4: Configure Power BI Connection

```sql
-- Create a view for Power BI to simplify cross-fact queries
CREATE VIEW vw_UnifiedLogisticsKPI AS
SELECT 
    t.FullDateTime,
    t.Year,
    t.Quarter,
    t.Month,
    t.DayOfWeek,
    g.LocationName AS WarehouseName,
    g.Region,
    p.SKU,
    p.ProductName,
    p.Category,
    p.GravityScore,
    w.OperationType,
    SUM(w.UnitsProcessed) AS TotalUnitsProcessed,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    SUM(w.ProcessingDuration) AS TotalProcessingMinutes,
    SUM(w.StaffHours) AS TotalStaffHours
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
GROUP BY 
    t.FullDateTime, t.Year, t.Quarter, t.Month, t.DayOfWeek,
    g.LocationName, g.Region, p.SKU, p.ProductName, p.Category, 
    p.GravityScore, w.OperationType;
GO

-- Create view for fleet analytics
CREATE VIEW vw_FleetPerformanceKPI AS
SELECT 
    t.FullDateTime,
    t.Year,
    t.Month,
    t.DayOfWeek,
    r.RouteName,
    r.RouteType,
    v.VehicleID,
    v.VehicleType,
    COUNT(f.TripID) AS TotalTrips,
    SUM(f.DistanceKm) AS TotalDistanceKm,
    SUM(f.FuelLiters) AS TotalFuelLiters,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    SUM(f.DelayMinutes) AS TotalDelayMinutes,
    AVG(f.AvgSpeed) AS AvgSpeed
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
INNER JOIN DimRoute r ON f.RouteKey = r.RouteKey
INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
GROUP BY 
    t.FullDateTime, t.Year, t.Month, t.DayOfWeek,
    r.RouteName, r.RouteType, v.VehicleID, v.VehicleType;
GO
```

### Step 5: Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Update connection string parameters:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetDB`
   - Authentication: Use environment variables `${SQL_USERNAME}` and `${SQL_PASSWORD}`
3. The template auto-detects views and establishes relationships
4. Publish to Power BI Service for 15-minute refresh cycles

## Configuration

### Data Source Configuration

Create a `config.json` file for external data source integration:

```json
{
  "dataSources": {
    "wms": {
      "type": "REST_API",
      "endpoint": "${WMS_API_ENDPOINT}",
      "authentication": {
        "type": "bearer",
        "token": "${WMS_API_TOKEN}"
      },
      "refreshIntervalMinutes": 15
    },
    "telematics": {
      "type": "MQTT",
      "broker": "${MQTT_BROKER_HOST}",
      "port": 8883,
      "topics": ["fleet/telemetry/+/location", "fleet/telemetry/+/diagnostics"],
      "credentials": {
        "username": "${MQTT_USERNAME}",
        "password": "${MQTT_PASSWORD}"
      }
    },
    "weather": {
      "type": "REST_API",
      "endpoint": "https://api.weatherapi.com/v1/forecast.json",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshIntervalMinutes": 60
    }
  },
  "alerting": {
    "email": {
      "smtpServer": "${SMTP_SERVER}",
      "port": 587,
      "from": "alerts@logifleet.example.com",
      "recipients": ["ops@example.com", "manager@example.com"]
    },
    "teams": {
      "webhookUrl": "${TEAMS_WEBHOOK_URL}"
    }
  }
}
```

### Automated Alerting Setup

```sql
-- Create alert configuration table
CREATE TABLE AlertThresholds (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName VARCHAR(100),
    MetricName VARCHAR(50),
    ThresholdValue DECIMAL(10,2),
    ThresholdOperator VARCHAR(10), -- '>', '<', '=', '>=', '<='
    AlertLevel VARCHAR(20), -- 'INFO', 'WARNING', 'CRITICAL'
    RecipientEmail VARCHAR(200),
    IsActive BIT DEFAULT 1
);

-- Insert sample thresholds
INSERT INTO AlertThresholds (AlertName, MetricName, ThresholdValue, ThresholdOperator, AlertLevel, RecipientEmail)
VALUES 
    ('High Fleet Idling', 'IdleTimePercent', 15, '>', 'WARNING', '${OPS_EMAIL}'),
    ('Excessive Dwell Time', 'DwellTimeHours', 72, '>', 'CRITICAL', '${WAREHOUSE_MANAGER_EMAIL}'),
    ('Low Gravity Zone Compliance', 'GravityCompliancePercent', 80, '<', 'WARNING', '${WAREHOUSE_MANAGER_EMAIL}'),
    ('Fuel Efficiency Drop', 'KmPerLiter', 8, '<', 'WARNING', '${FLEET_MANAGER_EMAIL}');

-- Stored procedure to check alerts and send notifications
CREATE PROCEDURE sp_CheckAndSendAlerts
AS
BEGIN
    DECLARE @AlertID INT, @AlertName VARCHAR(100), @MetricValue DECIMAL(10,2);
    
    -- Check fleet idling threshold
    DECLARE @IdlePercent DECIMAL(5,2);
    SELECT @IdlePercent = AVG(CAST(IdleTimeMinutes AS DECIMAL) / NULLIF(IdleTimeMinutes + LoadingTimeMinutes + UnloadingTimeMinutes, 0) * 100)
    FROM FactFleetTrips
    WHERE TimeKey >= CAST(CONVERT(VARCHAR(8), DATEADD(DAY, -1, GETDATE()), 112) AS INT);
    
    IF @IdlePercent > (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'IdleTimePercent' AND IsActive = 1)
    BEGIN
        -- Send email via SQL Server Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${OPS_EMAIL}',
            @subject = 'ALERT: High Fleet Idling Detected',
            @body = 'Fleet idling time has exceeded 15% threshold. Current: ' + CAST(@IdlePercent AS VARCHAR(10)) + '%';
    END;
    
    -- Check warehouse dwell time
    DECLARE @MaxDwell INT;
    SELECT @MaxDwell = MAX(DwellTimeMinutes) / 60
    FROM FactWarehouseOperations
    WHERE TimeKey >= CAST(CONVERT(VARCHAR(8), DATEADD(DAY, -1, GETDATE()), 112) AS INT);
    
    IF @MaxDwell > (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'DwellTimeHours' AND IsActive = 1)
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${WAREHOUSE_MANAGER_EMAIL}',
            @subject = 'CRITICAL: Excessive Warehouse Dwell Time',
            @body = 'SKUs with dwell time exceeding 72 hours detected. Max dwell: ' + CAST(@MaxDwell AS VARCHAR(10)) + ' hours';
    END;
END;
GO

-- Schedule this to run every 15 minutes via SQL Agent Job
```

## Common Patterns and Use Cases

### Pattern 1: Cross-Fact Analysis (Dwell Time vs. Fleet Idling)

```sql
-- Find correlation between high-dwell SKUs and fleet idle time
WITH HighDwellProducts AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.TimeKey >= CAST(CONVERT(VARCHAR(8), DATEADD(MONTH, -3, GETDATE()), 112) AS INT)
    GROUP BY p.SKU, p.ProductName
    HAVING AVG(w.DwellTimeMinutes) > 72 * 60
),
FleetIdlingBySKU AS (
    SELECT 
        p.SKU,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        COUNT(f.TripID) AS TripCount
    FROM FactFleetTrips f
    INNER JOIN FactCrossDock cd ON f.TripID = cd.OutboundTripKey
    INNER JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    WHERE f.TimeKey >= CAST(CONVERT(VARCHAR(8), DATEADD(MONTH, -3, GETDATE()), 112) AS INT)
    GROUP BY p.SKU
)
SELECT 
    hdp.SKU,
    hdp.ProductName,
    hdp.AvgDwellTime / 60 AS AvgDwellHours,
    fis.AvgIdleTime,
    (hdp.AvgDwellTime / 60) * fis.AvgIdleTime AS ImpactScore
FROM HighDwellProducts hdp
LEFT JOIN FleetIdlingBySKU fis ON hdp.SKU = fis.SKU
ORDER BY ImpactScore DESC;
```

### Pattern 2: Warehouse Gravity Zone Optimization

```sql
-- Identify SKUs that should be reassigned to different gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.Category,
    p.GravityScore,
    p.RecommendedZone AS CurrentZone,
    CASE 
        WHEN p.GravityScore >= 8 THEN 'HIGH_GRAVITY'
        WHEN p.GravityScore >= 5 THEN 'MEDIUM_GRAVITY'
        ELSE 'LOW_GRAVITY'
    END AS OptimalZone,
    AVG(w.DwellTimeMinutes) / 60 AS AvgDwellHours,
    SUM(w.UnitsProcessed) AS TotalUnitsProcessed
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
WHERE w.TimeKey >= CAST(CONVERT(VARCHAR(8), DATEADD(MONTH, -1, GETDATE()), 112) AS INT)
GROUP BY p.SKU, p.ProductName, p.Category, p.GravityScore, p.RecommendedZone
HAVING AVG(w.DwellTimeMinutes) / 60 > 24
ORDER BY p.GravityScore DESC;

-- Update recommended zones based on analysis
UPDATE DimProductGravity
SET RecommendedZone = CASE 
    WHEN GravityScore >= 8 THEN 'HIGH_GRAVITY'
    WHEN GravityScore >= 5 THEN 'MEDIUM_GRAVITY'
    ELSE 'LOW_GRAVITY'
END
WHERE GravityScore IS NOT NULL;
```

### Pattern 3: Predictive Fleet Maintenance Prioritization

```sql
-- Create composite maintenance score
CREATE VIEW vw_FleetMaintenancePriority AS
SELECT 
    v.VehicleID,
    v.VehicleType,
    v.LastMaintenanceDate,
    DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
    AVG(f.DistanceKm) AS AvgDailyDistance,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    COUNT(CASE WHEN f.DelayMinutes > 30 THEN 1 END) AS DelayIncidents,
    -- Weighted maintenance priority score
    (
        (DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) / 30.0) * 0.3 +
        (AVG(f.DistanceKm) / 500.0) * 0.3 +
        (AVG(f.IdleTimeMinutes) / 60.0) * 0.2 +
        (COUNT(CASE WHEN f.DelayMinutes > 30 THEN 1 END) / 10.0) * 0.2
    ) * 100 AS MaintenancePriorityScore
FROM DimVehicle v
LEFT JOIN FactFleetTrips f ON v.VehicleKey = f.VehicleKey
WHERE f.TimeKey >= CAST(CONVERT(VARCHAR(8), DATEADD(MONTH, -1, GETDATE()), 112) AS INT)
GROUP BY v.VehicleID, v.VehicleType, v.LastMaintenanceDate;

-- Query top priority vehicles
SELECT TOP 10 *
FROM vw_FleetMaintenancePriority
ORDER BY MaintenancePriorityScore DESC;
```

### Pattern 4: Temporal Elasticity Simulation

```sql
-- Simulate impact of warehouse capacity increase on fleet utilization
DECLARE @CapacityIncreasePct DECIMAL(5,2) = 0.15; -- 15% increase

WITH BaselineMetrics AS (
    SELECT 
        AVG(CAST(UnitsProcessed AS DECIMAL) / NULLIF(ProcessingDuration, 0)) AS AvgThroughputRate,
        AVG(DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations
    WHERE TimeKey >= CAST(CONVERT(VARCHAR(8), DATEADD(MONTH, -3, GETDATE()), 112) AS INT)
),
SimulatedMetrics AS (
    SELECT 
        AvgThroughputRate * (1 + @CapacityIncreasePct) AS ProjectedThroughputRate,
        AvgDwellTime * (1 - (@CapacityIncreasePct * 0.6)) AS ProjectedDwellTime -- Assume 60% improvement efficiency
    FROM BaselineMetrics
),
FleetImpact AS (
    SELECT 
        AVG(f.IdleTimeMinutes) AS CurrentAvgIdleTime,
        AVG(f.IdleTimeMinutes) * (1 - (@CapacityIncreasePct * 0.4)) AS ProjectedAvgIdleTime -- Assume 40% reduction
    FROM FactFleetTrips f
    WHERE f.TimeKey >= CAST(CONVERT(VARCHAR(8), DATEADD(MONTH, -3, GETDATE()), 112) AS INT)
)
SELECT 
    sm.ProjectedThroughputRate,
    sm.ProjectedDwellTime / 60 AS ProjectedDwellHours,
    fi.CurrentAvgIdleTime,
    fi.ProjectedAvgIdleTime,
    (fi.CurrentAvgIdleTime - fi.ProjectedAvgIdleTime) AS IdleTimeReduction
FROM SimulatedMetrics sm
CROSS JOIN FleetImpact fi;
```

### Pattern 5: Role-Based Row-Level Security

```sql
-- Create security table
CREATE TABLE SecurityRoles (
    RoleID INT IDENTITY(1,1) PRIMARY KEY,
    RoleName VARCHAR(50),
    UserEmail VARCHAR(200),
    AllowedRegions VARCHAR(MAX), -- Comma-separated list
    AllowedCategories VARCHAR(MAX) -- Comma-separated list
);

-- Insert sample roles
INSERT INTO SecurityRoles (RoleName, UserEmail, AllowedRegions, AllowedCategories)
VALUES 
    ('RegionalManager', 'manager.us@example.com', 'North America', NULL),
    ('CategoryManager', 'category.fresh@example.com', NULL, 'Fresh,Frozen'),
    ('Executive', 'ceo@example.com', NULL, NULL); -- Full access

-- Create row-level security predicate function
CREATE FUNCTION fn_SecurityPredicate(@Region VARCHAR(100), @Category VARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS AccessGranted
    WHERE 
        -- Executives see everything
        EXISTS (SELECT 1 FROM dbo.SecurityRoles WHERE UserEmail = USER_NAME() AND RoleName = 'Executive')
        OR
        -- Regional managers see their regions
        EXISTS (SELECT 1 FROM dbo.SecurityRoles WHERE UserEmail = USER_NAME() AND ',' + AllowedRegions + ',' LIKE '%,' + @Region + ',%')
        OR
        -- Category managers see their categories
        EXISTS (SELECT 1 FROM dbo.SecurityRoles WHERE UserEmail = USER_NAME() AND ',' + AllowedCategories + ',' LIKE '%,' + @Category + ',%');
GO

-- Apply security policy to warehouse view
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(Region, Category)
ON dbo.vw_UnifiedLogisticsKPI
WITH (STATE = ON);
```

## Power BI DAX Formulas

### Key Measures for Dashboards

```dax
// Total Warehouse Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time in Hours
Avg Dwell Time (Hours) = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60

// Warehouse Throughput Rate
Throughput Rate = 
DIVIDE(
    SUM(FactWarehouseOperations[UnitsProcessed]),
    SUM(FactWarehouseOperations[ProcessingDuration]),
    0
)

// Fleet Fuel Efficiency (Km per Liter)
Fuel Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelLiters]),
    0
)

// Fleet Idle Time Percentage
Idle Time % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + 
    SUM(FactFleetTrips[LoadingTimeMinutes]) + 
    SUM(FactFleetTrips[UnloadingTimeMinutes]),
    0
) * 100

// Cross-Fact: Revenue Impact of Delays
Revenue Impact of Delays = 
VAR DelayedTrips = 
    FILTER(
        FactFleetTrips,
        FactFleetTrips[DelayMinutes] > 30
    )
VAR TotalDelayedUnits = 
    CALCULATE(
        SUM(FactCrossDock[UnitsTransferred]),
        DelayedTrips
    )
RETURN
    TotalDelayedUnits * RELATED(DimProductGravity[ValueScore])

// Time Intelligence: Month-over-Month Dwell Time Change
MoM Dwell Change = 
VAR CurrentMonthDwell = [Avg Dwell Time (Hours)]
VAR PreviousMonthDwell = 
    CALCULATE(
        [Avg Dwell Time (Hours)],
        DATEADD(DimTime[FullDateTime], -1, MONTH)
    )
RETURN
    DIVIDE(
        CurrentMonthDwell - PreviousMonthDwell,
        PreviousMonthDwell,
        0
    )

// Gravity Zone Compliance Percentage
Gravity Compliance % = 
VAR CorrectPlacements = 
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[ZoneGravityScore] = 
            RELATED(DimProductGravity[GravityScore])
        )
    )
RETURN
    DIVIDE(
        CorrectPlacements,
        [Total Operations],
        0
    ) * 100
```

## Troubleshooting

### Issue: Slow Power BI Dashboard Performance

**Solution:** Implement aggregation tables

```sql
-- Create pre-aggregated table for daily summaries
CREATE TABLE AggWarehouseDaily (
    DateKey INT,
    WarehouseKey INT,
    ProductKey INT,
    TotalOperations INT,
    AvgDwellTime DECIMAL(10,2),
    TotalUnitsProcessed INT,
    PRIMARY KEY (DateKey, WarehouseKey, ProductKey)
);

-- Populate via scheduled job
INSERT
