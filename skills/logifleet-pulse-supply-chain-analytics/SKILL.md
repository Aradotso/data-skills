---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - create logistics data warehouse with star schema
  - implement Power BI fleet optimization dashboard
  - build warehouse and fleet analytics integration
  - deploy multi-fact supply chain reporting system
  - configure real-time logistics KPI dashboards
  - integrate warehouse operations with fleet telemetry
  - design cross-modal supply chain data model
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a **multi-modal logistics intelligence platform** that combines:
- **MS SQL Server data warehouse** with multi-fact star schema
- **Power BI dashboards** for real-time supply chain visualization
- **Integrated analytics** across warehouse operations, fleet telemetry, and inventory management
- **Predictive modeling** for bottleneck detection and fleet optimization
- **Cross-fact KPI harmonization** linking warehouse metrics with fleet performance

The platform fuses warehouse micro-operations, fleet telemetry, inventory aging, and external macroeconomic signals into a unified semantic layer for end-to-end logistics visibility.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- Access to warehouse management system (WMS) and fleet telemetry data sources
- Optional: Azure Synapse Analytics for external big data enrichment

### Step 1: Deploy SQL Schema

```sql
-- Clone repository and navigate to SQL scripts
-- Execute in order:

-- 1. Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- 2. Deploy dimension tables
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    Hour INT,
    Minute INT,
    DayOfWeek VARCHAR(20),
    DayOfMonth INT,
    Month INT,
    Quarter INT,
    FiscalYear INT,
    FiscalPeriod INT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- Create clustered columnstore index for fact tables
CREATE NONCLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime);
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, Route Node, Distribution Center
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    ParentLocationCode VARCHAR(50),
    IsActive BIT DEFAULT 1
);
GO

-- DimProduct: Product hierarchy with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(300),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    IsPerishable BIT,
    IsFragile BIT,
    WeightKg DECIMAL(10,3),
    VolumeM3 DECIMAL(10,6),
    UnitValue DECIMAL(12,2),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility
    OptimalStorageZone VARCHAR(50),
    LastGravityUpdate DATETIME
);
GO

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    Country VARCHAR(100),
    AvgLeadTimeDays DECIMAL(6,2),
    LeadTimeVariance DECIMAL(6,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier VARCHAR(20), -- Platinum, Gold, Silver, Bronze
    LastUpdated DATETIME
);
GO
```

### Step 2: Create Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME NOT NULL,
    OperationEndTime DATETIME,
    DurationMinutes INT,
    DwellTimeHours DECIMAL(8,2),
    QuantityUnits INT,
    PalletCount INT,
    LabourHours DECIMAL(6,2),
    OperationCost DECIMAL(12,2),
    ErrorFlag BIT,
    ErrorType VARCHAR(100),
    CONSTRAINT FK_Warehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Warehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Warehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_Warehouse_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
);

-- Columnstore index for analytical queries
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations ON FactWarehouseOperations;
GO

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripStartTime DATETIME NOT NULL,
    TripEndTime DATETIME,
    DistanceKm DECIMAL(10,2),
    FuelConsumedLitres DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    AverageSpeedKmh DECIMAL(6,2),
    DelayMinutes INT,
    DelayReason VARCHAR(200),
    WeatherCondition VARCHAR(50),
    MaintenanceFlag BIT,
    TripCost DECIMAL(12,2),
    RevenueGenerated DECIMAL(12,2),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips ON FactFleetTrips;
GO

-- FactCrossDock: Cross-dock transfer operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    ReceiveTime DATETIME NOT NULL,
    ShipTime DATETIME,
    DwellMinutes INT,
    QuantityUnits INT,
    TransferCost DECIMAL(12,2),
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactCrossDock ON FactCrossDock;
GO
```

### Step 3: Create Views for Cross-Fact Analysis

```sql
-- View: Unified Logistics KPI Dashboard
CREATE VIEW vw_UnifiedLogisticsKPI AS
SELECT 
    t.DateTime,
    t.FiscalPeriod,
    g.LocationName AS Warehouse,
    g.Country,
    p.Category AS ProductCategory,
    
    -- Warehouse metrics
    SUM(w.QuantityUnits) AS TotalUnitsProcessed,
    AVG(w.DwellTimeHours) AS AvgDwellHours,
    SUM(w.LabourHours) AS TotalLabourHours,
    SUM(w.OperationCost) AS WarehouseCost,
    
    -- Fleet metrics (linked via geography)
    COUNT(DISTINCT f.TripKey) AS FleetTripCount,
    SUM(f.DistanceKm) AS TotalDistanceKm,
    SUM(f.FuelConsumedLitres) AS TotalFuelLitres,
    AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(f.TripCost) AS FleetCost,
    SUM(f.RevenueGenerated) AS FleetRevenue,
    
    -- Cross-fact calculations
    (SUM(f.IdleTimeMinutes) / NULLIF(SUM(DATEDIFF(MINUTE, f.TripStartTime, f.TripEndTime)), 0)) * 100 AS IdleTimePercent,
    SUM(w.OperationCost + f.TripCost) AS TotalLogisticsCost,
    SUM(f.RevenueGenerated) - SUM(w.OperationCost + f.TripCost) AS NetLogisticsMargin

FROM DimTime t
LEFT JOIN FactWarehouseOperations w ON t.TimeKey = w.TimeKey
LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
LEFT JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
LEFT JOIN DimProductGravity p ON w.ProductKey = p.ProductKey

WHERE t.DateTime >= DATEADD(MONTH, -6, GETDATE())
GROUP BY 
    t.DateTime, t.FiscalPeriod, g.LocationName, g.Country, p.Category;
GO
```

### Step 4: Create Stored Procedures for Data Loading

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- If no last load time provided, get from control table
    IF @LastLoadDateTime IS NULL
        SELECT @LastLoadDateTime = ISNULL(MAX(OperationEndTime), '2020-01-01')
        FROM FactWarehouseOperations;
    
    -- Insert new records from staging table (assumes ETL populates this)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OperationStartTime, OperationEndTime,
        DurationMinutes, DwellTimeHours, QuantityUnits,
        PalletCount, LabourHours, OperationCost, ErrorFlag, ErrorType
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.OperationStartTime,
        stg.OperationEndTime,
        DATEDIFF(MINUTE, stg.OperationStartTime, stg.OperationEndTime),
        stg.DwellTimeHours,
        stg.QuantityUnits,
        stg.PalletCount,
        stg.LabourHours,
        stg.OperationCost,
        stg.ErrorFlag,
        stg.ErrorType
    FROM dbo.StagingWarehouseOperations stg
    INNER JOIN DimTime t ON CAST(stg.OperationStartTime AS DATE) = CAST(t.DateTime AS DATE)
        AND DATEPART(HOUR, stg.OperationStartTime) = t.Hour
        AND (DATEPART(MINUTE, stg.OperationStartTime) / 15) * 15 = t.Minute
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON stg.SupplierCode = s.SupplierCode
    WHERE stg.OperationEndTime > @LastLoadDateTime;
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR(10)) + ' warehouse operation records.';
END;
GO

-- Automated alerting procedure
CREATE PROCEDURE usp_CheckLogisticsAlerts
AS
BEGIN
    DECLARE @AlertMessage VARCHAR(1000);
    
    -- Alert 1: High dwell time
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations 
        WHERE DwellTimeHours > 72 
        AND OperationEndTime > DATEADD(HOUR, -24, GETDATE())
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Items with dwell time > 72 hours detected in last 24 hours.';
        -- Insert into alert log or send notification
        INSERT INTO AlertLog (AlertType, AlertMessage, AlertDateTime)
        VALUES ('HighDwellTime', @AlertMessage, GETDATE());
    END;
    
    -- Alert 2: Fleet idle time exceeds threshold
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips
        WHERE (IdleTimeMinutes * 1.0 / NULLIF(DATEDIFF(MINUTE, TripStartTime, TripEndTime), 0)) > 0.15
        AND TripEndTime > DATEADD(HOUR, -24, GETDATE())
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeds 15% of trip duration.';
        INSERT INTO AlertLog (AlertType, AlertMessage, AlertDateTime)
        VALUES ('HighIdleTime', @AlertMessage, GETDATE());
    END;
END;
GO
```

## Power BI Configuration

### Step 1: Import Template

1. Download `LogiFleet_Pulse_Master.pbit` from repository
2. Open in Power BI Desktop
3. Enter connection parameters:
   - SQL Server: `${SQL_SERVER_NAME}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server (use environment variables for credentials)

### Step 2: Configure Data Refresh

```powerquery
// Power Query M code for incremental refresh
let
    Source = Sql.Database(
        #"SQL_SERVER_NAME", 
        "LogiFleetPulse",
        [Query="
            SELECT *
            FROM vw_UnifiedLogisticsKPI
            WHERE DateTime >= DATEADD(MONTH, -12, GETDATE())
        "]
    ),
    #"Changed Type" = Table.TransformColumnTypes(Source, {
        {"DateTime", type datetime},
        {"TotalUnitsProcessed", Int64.Type},
        {"AvgDwellHours", type number},
        {"TotalLogisticsCost", type number}
    })
in
    #"Changed Type"
```

### Step 3: Create Key Measures

```dax
// DAX measure: Warehouse Efficiency Score
WarehouseEfficiency = 
VAR TotalUnits = SUM(FactWarehouseOperations[QuantityUnits])
VAR TotalLabour = SUM(FactWarehouseOperations[LabourHours])
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
RETURN
    IF(
        TotalLabour > 0,
        (TotalUnits / TotalLabour) * (1 - (AvgDwell / 168)) * 100,
        BLANK()
    )

// DAX measure: Fleet Utilization Rate
FleetUtilization = 
VAR TotalTripTime = SUMX(
    FactFleetTrips,
    DATEDIFF(FactFleetTrips[TripStartTime], FactFleetTrips[TripEndTime], MINUTE)
)
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
    IF(
        TotalTripTime > 0,
        ((TotalTripTime - TotalIdleTime) / TotalTripTime) * 100,
        BLANK()
    )

// DAX measure: Cross-Fact Logistics Cost per Unit
CostPerUnit = 
DIVIDE(
    SUM(FactWarehouseOperations[OperationCost]) + SUM(FactFleetTrips[TripCost]),
    SUM(FactWarehouseOperations[QuantityUnits]),
    0
)

// DAX measure: Predictive Bottleneck Index
BottleneckIndex = 
VAR HighDwellCount = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[DwellTimeHours] > 48
)
VAR HighIdleCount = CALCULATE(
    COUNTROWS(FactFleetTrips),
    FactFleetTrips[IdleTimeMinutes] > 30
)
VAR TotalOps = COUNTROWS(FactWarehouseOperations) + COUNTROWS(FactFleetTrips)
RETURN
    IF(
        TotalOps > 0,
        ((HighDwellCount + HighIdleCount) / TotalOps) * 100,
        0
    )
```

## Configuration Files

### config.json (Template)

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_NAME}",
    "database": "LogiFleetPulse",
    "connectionTimeout": 30,
    "commandTimeout": 300,
    "authentication": "IntegratedSecurity"
  },
  "dataSources": {
    "wms": {
      "type": "REST_API",
      "endpoint": "${WMS_API_ENDPOINT}",
      "authHeader": "Bearer ${WMS_API_KEY}",
      "refreshIntervalMinutes": 15
    },
    "telemetry": {
      "type": "MQTT",
      "broker": "${TELEMETRY_BROKER_URL}",
      "topic": "fleet/vehicles/+/telemetry",
      "qos": 1
    },
    "weather": {
      "type": "REST_API",
      "endpoint": "https://api.openweathermap.org/data/2.5/forecast",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "powerBI": {
    "workspaceId": "${POWERBI_WORKSPACE_ID}",
    "reportId": "${POWERBI_REPORT_ID}",
    "refreshSchedule": "0 */15 * * * *"
  },
  "alerts": {
    "email": {
      "enabled": true,
      "smtpServer": "${SMTP_SERVER}",
      "smtpPort": 587,
      "fromAddress": "${ALERT_FROM_EMAIL}",
      "recipients": ["${ALERT_RECIPIENT_1}", "${ALERT_RECIPIENT_2}"]
    },
    "teams": {
      "enabled": true,
      "webhookUrl": "${TEAMS_WEBHOOK_URL}"
    }
  },
  "thresholds": {
    "dwellTimeHoursWarning": 48,
    "dwellTimeHoursCritical": 72,
    "idleTimePercentWarning": 10,
    "idleTimePercentCritical": 15,
    "fuelEfficiencyThresholdKmL": 8.5
  }
}
```

## Common Patterns & Use Cases

### Pattern 1: Cross-Fact Analysis Query

```sql
-- Find products with high dwell time and associated fleet delays
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    w.AvgDwellHours,
    f.AvgDelayMinutes,
    w.TotalCost AS WarehouseCost,
    f.TotalCost AS FleetCost
FROM (
    SELECT 
        ProductKey,
        AVG(DwellTimeHours) AS AvgDwellHours,
        SUM(OperationCost) AS TotalCost
    FROM FactWarehouseOperations
    WHERE OperationEndTime > DATEADD(DAY, -30, GETDATE())
    GROUP BY ProductKey
) w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
LEFT JOIN (
    SELECT 
        p2.ProductKey,
        AVG(f.DelayMinutes) AS AvgDelayMinutes,
        SUM(f.TripCost) AS TotalCost
    FROM FactFleetTrips f
    INNER JOIN FactWarehouseOperations wo ON wo.GeographyKey = f.OriginGeographyKey
    INNER JOIN DimProductGravity p2 ON wo.ProductKey = p2.ProductKey
    WHERE f.TripEndTime > DATEADD(DAY, -30, GETDATE())
    GROUP BY p2.ProductKey
) f ON w.ProductKey = f.ProductKey
WHERE w.AvgDwellHours > 48
ORDER BY w.AvgDwellHours DESC, f.AvgDelayMinutes DESC;
```

### Pattern 2: Warehouse Gravity Zone Optimization

```sql
-- Recalculate product gravity scores based on recent activity
UPDATE p
SET 
    GravityScore = (
        (PickFrequency.Picks30Days / 30.0) * -- Velocity component
        (p.UnitValue / 100.0) * -- Value component (normalized)
        (CASE WHEN p.IsFragile = 1 THEN 1.5 ELSE 1.0 END) -- Fragility multiplier
    ),
    OptimalStorageZone = CASE
        WHEN (PickFrequency.Picks30Days / 30.0) * p.UnitValue > 500 THEN 'High-Gravity-Zone-A'
        WHEN (PickFrequency.Picks30Days / 30.0) * p.UnitValue > 200 THEN 'Medium-Gravity-Zone-B'
        ELSE 'Low-Gravity-Zone-C'
    END,
    LastGravityUpdate = GETDATE()
FROM DimProductGravity p
INNER JOIN (
    SELECT 
        ProductKey,
        COUNT(*) AS Picks30Days
    FROM FactWarehouseOperations
    WHERE OperationType = 'Picking'
    AND OperationEndTime > DATEADD(DAY, -30, GETDATE())
    GROUP BY ProductKey
) PickFrequency ON p.ProductKey = PickFrequency.ProductKey;
```

### Pattern 3: Fleet Predictive Maintenance Queue

```sql
-- Generate maintenance priority queue based on telemetry and revenue impact
WITH VehicleMetrics AS (
    SELECT 
        VehicleID,
        COUNT(*) AS TripsLast30Days,
        AVG(IdleTimeMinutes) AS AvgIdleTime,
        SUM(DistanceKm) AS TotalDistance,
        AVG(FuelConsumedLitres / NULLIF(DistanceKm, 0)) AS FuelEfficiency,
        SUM(RevenueGenerated) AS TotalRevenue,
        MAX(TripEndTime) AS LastTripDate
    FROM FactFleetTrips
    WHERE TripEndTime > DATEADD(DAY, -30, GETDATE())
    GROUP BY VehicleID
),
MaintenanceFlags AS (
    SELECT 
        VehicleID,
        CASE 
            WHEN FuelEfficiency > 12 THEN 1 -- Poor fuel efficiency
            WHEN AvgIdleTime > 45 THEN 1 -- High idle time
            WHEN TotalDistance > 5000 THEN 1 -- High mileage
            ELSE 0
        END AS MaintenanceFlag,
        CASE
            WHEN FuelEfficiency > 12 THEN 'Fuel System Check'
            WHEN AvgIdleTime > 45 THEN 'Engine Diagnostics'
            WHEN TotalDistance > 5000 THEN 'Scheduled Service'
            ELSE NULL
        END AS MaintenanceType,
        TotalRevenue * 0.1 AS RevenueAtRisk -- Estimate 10% revenue loss if vehicle fails
    FROM VehicleMetrics
)
SELECT 
    m.VehicleID,
    m.MaintenanceType,
    m.RevenueAtRisk,
    v.TripsLast30Days,
    v.FuelEfficiency,
    v.AvgIdleTime,
    v.LastTripDate,
    RANK() OVER (ORDER BY m.RevenueAtRisk DESC, v.AvgIdleTime DESC) AS PriorityRank
FROM MaintenanceFlags m
INNER JOIN VehicleMetrics v ON m.VehicleID = v.VehicleID
WHERE m.MaintenanceFlag = 1
ORDER BY PriorityRank;
```

### Pattern 4: Temporal Elasticity Simulation

```sql
-- Simulate impact of increasing warehouse capacity utilization
DECLARE @TargetUtilization DECIMAL(5,2) = 0.95;

WITH CurrentCapacity AS (
    SELECT 
        GeographyKey,
        SUM(QuantityUnits) AS CurrentUnits,
        10000 AS MaxCapacity, -- Adjust per warehouse
        SUM(QuantityUnits) * 1.0 / 10000 AS CurrentUtilization
    FROM FactWarehouseOperations
    WHERE OperationEndTime > DATEADD(DAY, -7, GETDATE())
    GROUP BY GeographyKey
),
SimulatedImpact AS (
    SELECT 
        cc.GeographyKey,
        cc.CurrentUnits,
        cc.CurrentUtilization,
        @TargetUtilization AS TargetUtilization,
        (MaxCapacity * @TargetUtilization) - CurrentUnits AS AdditionalUnitsCapacity,
        -- Assume 10% increase in dwell time per 10% capacity increase
        AVG(w.DwellTimeHours) * (1 + ((@TargetUtilization - cc.CurrentUtilization) * 1.0)) AS ProjectedDwellHours,
        -- Assume 5% increase in labour hours per 10% capacity increase
        SUM(w.LabourHours) * (1 + ((@TargetUtilization - cc.CurrentUtilization) * 0.5)) AS ProjectedLabourHours
    FROM CurrentCapacity cc
    INNER JOIN FactWarehouseOperations w ON cc.GeographyKey = w.GeographyKey
    WHERE w.OperationEndTime > DATEADD(DAY, -7, GETDATE())
    GROUP BY cc.GeographyKey, cc.CurrentUnits, cc.CurrentUtilization, cc.MaxCapacity
)
SELECT 
    g.LocationName,
    si.CurrentUtilization,
    si.TargetUtilization,
    si.AdditionalUnitsCapacity,
    si.ProjectedDwellHours,
    si.ProjectedLabourHours,
    si.ProjectedLabourHours - SUM(w.LabourHours) AS AdditionalLabourRequired
FROM SimulatedImpact si
INNER JOIN DimGeography g ON si.GeographyKey = g.GeographyKey
INNER JOIN FactWarehouseOperations w ON si.GeographyKey = w.GeographyKey
WHERE w.OperationEndTime > DATEADD(DAY, -7, GETDATE())
GROUP BY g.LocationName, si.CurrentUtilization, si.TargetUtilization, 
    si.AdditionalUnitsCapacity, si.ProjectedDwellHours, si.ProjectedLabourHours;
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptoms**: Queries joining multiple fact tables take > 30 seconds

**Solutions**:
1. Ensure columnstore indexes are created on all fact tables
2. Update statistics regularly:
   ```sql
   UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
   UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
   ```
3. Use appropriate date filters in WHERE clauses to limit scan range
4. Consider table partitioning for fact tables > 100M rows:
   ```sql
   -- Partition by month
   CREATE PARTITION FUNCTION pf_MonthlyPartition (DATETIME)
   AS RANGE RIGHT FOR VALUES (
       '2025-01-01', '2025-02-01', '2025-03-01' -- ... add monthly boundaries
   );
   ```

### Issue: Power BI Report Refresh Failures

**Symptoms**: Scheduled refresh fails with timeout errors

**Solutions**:
1. Reduce refresh scope using incremental refresh
2. Create aggregate tables for common queries:
   ```sql
   CREATE TABLE AggMonthlyLogisticsKPI AS
   SELECT 
       YEAR(t.DateTime) AS Year,
       MONTH(t.DateTime) AS Month,
       g.Country,
       p.Category,
       SUM(w.QuantityUnits) AS TotalUnits,
       AVG(w.DwellTimeHours) AS AvgDwell,
       SUM(f.TripCost) AS FleetCost
   FROM FactWarehouseOperations w
   INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
   INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
   INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
   LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
   GROUP BY YEAR(t.DateTime), MONTH(t.DateTime), g.Country, p.Category;
   ```
3. Use Power BI Premium for enhanced capacity

### Issue: Dimension Table Drift

**Symptoms**: New locations/products not appearing in reports

**Solutions**:
1. Implement slowly changing dimensions (SCD Type 2):
   ```sql
   -- Add SCD tracking columns
   ALTER TABLE DimGeography ADD 
       ValidFrom DATETIME DEFAULT GETDATE(),
       ValidTo DATETIME DEFAULT '9999-12-31',
       IsCurrent BIT DEFAULT 1;
   
   -- Create update procedure
   CREATE PROCEDURE usp_UpdateDimGeography
       @LocationCode VARCHAR(50),
       @NewLocationName VARCHAR(200)
   AS
   BEGIN
       -- Expire current record
       UPDATE Dim
