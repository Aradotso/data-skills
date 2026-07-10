---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform using MS SQL Server star schema and Power BI for warehouse, fleet, and supply chain analytics
triggers:
  - set up logistics analytics dashboard
  - create supply chain data warehouse
  - build fleet management power bi reports
  - implement warehouse operations tracking
  - design multi-fact star schema for logistics
  - configure cross-modal supply chain analytics
  - deploy logifleet pulse intelligence platform
  - integrate warehouse and fleet telemetry data
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and supply chain analytics into a single semantic data layer. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema architecture that enables cross-functional KPI analysis (e.g., correlating warehouse dwell time with fleet fuel consumption).

Key capabilities:
- **Multi-fact star schema** with time-phased dimensions for sub-second cross-fact queries
- **Warehouse Gravity Zones™** - spatial optimization based on pick frequency and item value
- **Adaptive Fleet Triage** - maintenance prioritization by revenue impact
- **Real-time dashboarding** with 15-minute refresh intervals
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Cross-dock operations tracking** with end-to-end visibility

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) recommended
- Access to data sources: WMS, TMS, telematics APIs

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy the dimensional model
-- Execute the provided schema scripts in order:
-- 1. Create dimension tables
-- 2. Create fact tables
-- 3. Create relationships and indexes
-- 4. Create views and stored procedures

-- Example: Create DimTime (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    Minute TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL, -- 1-4 (15-min buckets)
    DayOfWeek TINYINT NOT NULL,
    IsBusinessHour BIT NOT NULL,
    FiscalPeriod VARCHAR(10),
    INDEX IX_DimTime_DateTime (FullDateTime),
    INDEX IX_DimTime_DateKey (DateKey)
);
GO

-- Example: Create FactWarehouseOperations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'Receive', 'Putaway', 'Pick', 'Pack', 'Ship'
    QuantityMoved INT NOT NULL,
    DwellTimeMinutes INT,
    PickingTime DECIMAL(10,2),
    GravityZoneID INT,
    OperatorID INT,
    BatchID VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    INDEX IX_Fact_Time (TimeKey),
    INDEX IX_Fact_Warehouse (WarehouseKey),
    INDEX IX_Fact_Product (ProductKey),
    INDEX IX_Fact_OperationType (OperationType)
);
GO

-- Example: Create FactFleetTrips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginWarehouseKey INT NOT NULL,
    DestinationKey INT,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TripStatus VARCHAR(20), -- 'InProgress', 'Completed', 'Delayed'
    DelayReasonCode VARCHAR(10),
    FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    INDEX IX_FleetTrips_StartTime (StartTimeKey),
    INDEX IX_FleetTrips_Vehicle (VehicleKey),
    INDEX IX_FleetTrips_Route (RouteKey)
);
GO
```

### Step 2: Create Dimension Tables

```sql
-- DimProductGravity - assigns gravity score based on velocity and value
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(50),
    SubCategory VARCHAR(50),
    UnitValue DECIMAL(10,2),
    WeightKg DECIMAL(8,2),
    Fragility TINYINT, -- 1-5 scale
    AveragePickFrequency DECIMAL(10,2),
    GravityScore DECIMAL(10,2), -- Calculated composite score
    OptimalZoneType VARCHAR(20), -- 'HighGravity', 'MediumGravity', 'LowGravity'
    IsPerishable BIT,
    ShelfLifeDays INT,
    INDEX IX_Product_SKU (SKU),
    INDEX IX_Product_Gravity (GravityScore DESC)
);
GO

-- DimGeography - hierarchical location data
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationType VARCHAR(20), -- 'Warehouse', 'RouteNode', 'Customer'
    LocationID VARCHAR(50),
    LocationName VARCHAR(200),
    Address VARCHAR(500),
    City VARCHAR(100),
    StateProvince VARCHAR(50),
    PostalCode VARCHAR(20),
    Country VARCHAR(50),
    Region VARCHAR(50),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    TimeZone VARCHAR(50),
    INDEX IX_Geography_Type (LocationType),
    INDEX IX_Geography_City (City)
);
GO

-- DimSupplierReliability
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeDaysAvg DECIMAL(6,2),
    LeadTimeDaysStdDev DECIMAL(6,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(10), -- 'A', 'B', 'C'
    LastAuditDate DATE,
    INDEX IX_Supplier_ID (SupplierID),
    INDEX IX_Supplier_Tier (ReliabilityTier)
);
GO
```

### Step 3: Create Incremental Loading Procedures

```sql
-- Stored procedure for incremental warehouse operations loading
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging/source system
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        QuantityMoved, DwellTimeMinutes, PickingTime, GravityZoneID,
        OperatorID, BatchID
    )
    SELECT 
        t.TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        s.OperationType,
        s.QuantityMoved,
        s.DwellTimeMinutes,
        s.PickingTime,
        p.OptimalZoneType,
        s.OperatorID,
        s.BatchID
    FROM StagingWarehouseOps s
    INNER JOIN DimTime t ON DATEADD(MINUTE, (t.Minute / 15) * 15, 
        CAST(CAST(s.OperationDateTime AS DATE) AS DATETIME) + 
        CAST(CAST(s.OperationDateTime AS TIME) AS DATETIME)) = t.FullDateTime
    INNER JOIN DimGeography w ON s.WarehouseID = w.LocationID
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE s.OperationDateTime > @LastLoadDateTime;
    
    -- Update metadata
    UPDATE ETLMetadata 
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO

-- Stored procedure for cross-fact KPI calculation
CREATE PROCEDURE usp_CalculateCrossDockEfficiency
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    -- Calculate efficiency metrics linking warehouse and fleet operations
    SELECT 
        w.WarehouseKey,
        g.LocationName AS WarehouseName,
        COUNT(DISTINCT wo.BatchID) AS TotalBatches,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
        COUNT(ft.TripKey) AS AssociatedTrips,
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
        SUM(ft.FuelConsumedGallons) AS TotalFuelConsumed,
        -- Cross-fact KPI: Dwell-to-Idle correlation
        CASE 
            WHEN AVG(wo.DwellTimeMinutes) > 120 AND AVG(ft.IdleTimeMinutes) > 30 
            THEN 'HighFriction'
            WHEN AVG(wo.DwellTimeMinutes) < 60 AND AVG(ft.IdleTimeMinutes) < 15 
            THEN 'Optimized'
            ELSE 'Moderate'
        END AS EfficiencyTier
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    LEFT JOIN FactFleetTrips ft ON wo.WarehouseKey = ft.OriginWarehouseKey 
        AND wo.TimeKey = ft.StartTimeKey
    WHERE t.FullDateTime BETWEEN @StartDate AND @EndDate
    GROUP BY w.WarehouseKey, g.LocationName
    ORDER BY AvgDwellTimeMinutes DESC;
END;
GO
```

### Step 4: Configure Data Source Connections

Create a configuration file for external data integration:

```json
{
  "dataSources": {
    "wms": {
      "type": "SQL",
      "connectionString": "Server=${WMS_SERVER};Database=${WMS_DB};User Id=${WMS_USER};Password=${WMS_PASSWORD};"
    },
    "telematics": {
      "type": "REST_API",
      "baseUrl": "${TELEMATICS_API_URL}",
      "apiKey": "${TELEMATICS_API_KEY}",
      "refreshIntervalMinutes": 15
    },
    "weatherApi": {
      "type": "REST_API",
      "baseUrl": "https://api.weatherapi.com/v1",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "etl": {
    "incrementalLoadInterval": "15min",
    "fullRefreshSchedule": "0 2 * * 0",
    "errorRetryAttempts": 3
  }
}
```

### Step 5: Import Power BI Template

```powershell
# Open Power BI Desktop
# File > Import > Power BI Template (.pbit)
# Navigate to LogiFleet_Pulse_Master.pbit

# Configure data source connection
# Enter SQL Server connection details:
# - Server: your-sql-server.database.windows.net
# - Database: LogiFleetPulse
# - Authentication: SQL Server / Windows / Azure AD
```

## Key SQL Queries for Analytics

### Query 1: Warehouse Gravity Zone Optimization

```sql
-- Identify products misaligned with their optimal gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneType AS RecommendedZone,
    wo.GravityZoneID AS CurrentZone,
    AVG(wo.PickingTime) AS AvgPickingTimeSeconds,
    COUNT(*) AS PickCount,
    -- Calculate potential time savings
    CASE 
        WHEN p.OptimalZoneType = 'HighGravity' AND wo.GravityZoneID != 1 
        THEN AVG(wo.PickingTime) * 0.35 -- 35% potential improvement
        ELSE 0 
    END AS EstimatedTimeSavings
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Pick'
    AND wo.TimeKey >= (SELECT MAX(TimeKey) - 10080 FROM DimTime) -- Last week
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZoneType, wo.GravityZoneID
HAVING p.OptimalZoneType != 
    CASE wo.GravityZoneID 
        WHEN 1 THEN 'HighGravity'
        WHEN 2 THEN 'MediumGravity'
        ELSE 'LowGravity'
    END
ORDER BY EstimatedTimeSavings DESC;
```

### Query 2: Fleet Idle Time Correlation with Warehouse Dwell

```sql
-- Cross-fact analysis: warehouse delays impact on fleet efficiency
WITH WarehouseDwell AS (
    SELECT 
        wo.WarehouseKey,
        t.DateKey,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.OperationType IN ('Receive', 'Ship')
    GROUP BY wo.WarehouseKey, t.DateKey
),
FleetIdle AS (
    SELECT 
        ft.OriginWarehouseKey,
        t.DateKey,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        AVG(ft.LoadingTimeMinutes + ft.UnloadingTimeMinutes) AS AvgLoadUnloadMinutes,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    GROUP BY ft.OriginWarehouseKey, t.DateKey
)
SELECT 
    g.LocationName AS Warehouse,
    wd.DateKey AS Date,
    wd.AvgDwellMinutes,
    fi.AvgIdleMinutes,
    fi.AvgLoadUnloadMinutes,
    -- Correlation indicator
    CASE 
        WHEN wd.AvgDwellMinutes > 90 AND fi.AvgIdleMinutes > 25 
        THEN 'StrongCorrelation'
        WHEN wd.AvgDwellMinutes > 60 AND fi.AvgIdleMinutes > 15 
        THEN 'ModerateCorrelation'
        ELSE 'WeakCorrelation'
    END AS CorrelationStrength,
    -- Estimated cost impact (assuming $45/hour idle cost)
    (fi.AvgIdleMinutes / 60.0) * 45 * fi.TripCount AS EstimatedIdleCost
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.WarehouseKey = fi.OriginWarehouseKey 
    AND wd.DateKey = fi.DateKey
INNER JOIN DimGeography g ON wd.WarehouseKey = g.GeographyKey
ORDER BY EstimatedIdleCost DESC;
```

### Query 3: Predictive Bottleneck Detection

```sql
-- Identify emerging bottlenecks using time-series analysis
WITH HourlyMetrics AS (
    SELECT 
        wo.WarehouseKey,
        t.DateKey,
        t.Hour,
        wo.OperationType,
        COUNT(*) AS OperationVolume,
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        STDEV(wo.DwellTimeMinutes) AS StdDevDwell
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateKey >= (SELECT MAX(DateKey) - 30 FROM DimTime) -- Last 30 days
    GROUP BY wo.WarehouseKey, t.DateKey, t.Hour, wo.OperationType
),
Baseline AS (
    SELECT 
        WarehouseKey,
        Hour,
        OperationType,
        AVG(OperationVolume) AS BaselineVolume,
        AVG(AvgDwell) AS BaselineDwell,
        AVG(StdDevDwell) AS BaselineVariance
    FROM HourlyMetrics
    WHERE DateKey < (SELECT MAX(DateKey) - 7 FROM DimTime) -- Exclude last week
    GROUP BY WarehouseKey, Hour, OperationType
)
SELECT 
    g.LocationName AS Warehouse,
    hm.Hour,
    hm.OperationType,
    hm.OperationVolume AS CurrentVolume,
    b.BaselineVolume,
    hm.AvgDwell AS CurrentDwell,
    b.BaselineDwell,
    -- Bottleneck score (0-100)
    CAST((
        (hm.OperationVolume / NULLIF(b.BaselineVolume, 0) - 1) * 30 +
        (hm.AvgDwell / NULLIF(b.BaselineDwell, 0) - 1) * 40 +
        (hm.StdDevDwell / NULLIF(b.BaselineVariance, 0) - 1) * 30
    ) * 100 AS DECIMAL(5,2)) AS BottleneckScore
FROM HourlyMetrics hm
INNER JOIN Baseline b ON hm.WarehouseKey = b.WarehouseKey 
    AND hm.Hour = b.Hour 
    AND hm.OperationType = b.OperationType
INNER JOIN DimGeography g ON hm.WarehouseKey = g.GeographyKey
WHERE hm.DateKey = (SELECT MAX(DateKey) FROM DimTime) -- Today
    AND (
        hm.OperationVolume > b.BaselineVolume * 1.2 OR
        hm.AvgDwell > b.BaselineDwell * 1.3
    )
ORDER BY BottleneckScore DESC;
```

## Power BI DAX Measures

### Cross-Fact KPI: Warehouse-Fleet Efficiency Index

```dax
// Composite measure linking warehouse and fleet performance
Warehouse_Fleet_Efficiency_Index = 
VAR WarehouseEfficiency = 
    DIVIDE(
        CALCULATE(SUM(FactWarehouseOperations[QuantityMoved]), 
                  FactWarehouseOperations[OperationType] = "Pick"),
        CALCULATE(SUM(FactWarehouseOperations[PickingTime]))
    ) * 100
VAR FleetEfficiency = 
    DIVIDE(
        CALCULATE(SUM(FactFleetTrips[DistanceMiles])),
        CALCULATE(SUM(FactFleetTrips[FuelConsumedGallons]))
    )
VAR IdlePenalty = 
    1 - (CALCULATE(AVERAGE(FactFleetTrips[IdleTimeMinutes])) / 100)
RETURN 
    (WarehouseEfficiency * 0.6 + FleetEfficiency * 0.4) * IdlePenalty
```

### Dynamic Gravity Zone Recommendation

```dax
Gravity_Zone_Recommendation = 
VAR CurrentZone = SELECTEDVALUE(FactWarehouseOperations[GravityZoneID])
VAR ProductGravity = SELECTEDVALUE(DimProductGravity[GravityScore])
VAR OptimalZone = 
    SWITCH(
        TRUE(),
        ProductGravity >= 80, 1, // HighGravity
        ProductGravity >= 50, 2, // MediumGravity
        3 // LowGravity
    )
VAR TimeSavingsPotential = 
    IF(
        CurrentZone <> OptimalZone,
        CALCULATE(AVERAGE(FactWarehouseOperations[PickingTime])) * 0.35,
        0
    )
RETURN 
    IF(
        CurrentZone = OptimalZone,
        "Optimized",
        "Relocate to Zone " & OptimalZone & " (Save " & 
        FORMAT(TimeSavingsPotential, "0.0") & "s per pick)"
    )
```

### Predictive Bottleneck Score

```dax
Bottleneck_Risk_Score = 
VAR CurrentVolume = 
    CALCULATE(COUNT(FactWarehouseOperations[OperationKey]))
VAR BaselineVolume = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        DATEADD(DimTime[FullDateTime], -7, DAY)
    )
VAR CurrentDwell = 
    CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeMinutes]))
VAR BaselineDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATEADD(DimTime[FullDateTime], -7, DAY)
    )
VAR VolumeScore = 
    DIVIDE(CurrentVolume, BaselineVolume, 1) - 1
VAR DwellScore = 
    DIVIDE(CurrentDwell, BaselineDwell, 1) - 1
RETURN 
    (VolumeScore * 0.4 + DwellScore * 0.6) * 100
```

## Automated Alerting Setup

### Create Alert Stored Procedure

```sql
-- Alert procedure for critical KPI breaches
CREATE PROCEDURE usp_MonitorCriticalKPIs
AS
BEGIN
    DECLARE @AlertTable TABLE (
        AlertType VARCHAR(50),
        Severity VARCHAR(10),
        Message VARCHAR(500),
        AffectedEntity VARCHAR(200),
        CurrentValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2)
    );
    
    -- Alert 1: Excessive fleet idling
    INSERT INTO @AlertTable
    SELECT 
        'FleetIdling' AS AlertType,
        'HIGH' AS Severity,
        'Vehicle idle time exceeds 15% of trip duration' AS Message,
        v.VehicleID AS AffectedEntity,
        AVG(ft.IdleTimeMinutes) / NULLIF(AVG(DATEDIFF(MINUTE, st.FullDateTime, et.FullDateTime)), 0) * 100 AS CurrentValue,
        15.0 AS ThresholdValue
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    INNER JOIN DimTime st ON ft.StartTimeKey = st.TimeKey
    INNER JOIN DimTime et ON ft.EndTimeKey = et.TimeKey
    WHERE st.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
    GROUP BY v.VehicleID
    HAVING AVG(ft.IdleTimeMinutes) / NULLIF(AVG(DATEDIFF(MINUTE, st.FullDateTime, et.FullDateTime)), 0) * 100 > 15;
    
    -- Alert 2: Warehouse dwell time spike
    INSERT INTO @AlertTable
    SELECT 
        'WarehouseDwell' AS AlertType,
        'MEDIUM' AS Severity,
        'Product dwell time exceeds 72 hours in high-gravity zone' AS Message,
        p.SKU AS AffectedEntity,
        AVG(wo.DwellTimeMinutes) / 60.0 AS CurrentValue,
        72.0 AS ThresholdValue
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -3, GETDATE())
        AND p.OptimalZoneType = 'HighGravity'
        AND wo.DwellTimeMinutes IS NOT NULL
    GROUP BY p.SKU
    HAVING AVG(wo.DwellTimeMinutes) / 60.0 > 72;
    
    -- Alert 3: Cross-fact anomaly
    INSERT INTO @AlertTable
    SELECT 
        'CrossFactAnomaly' AS AlertType,
        'HIGH' AS Severity,
        'Warehouse delay correlating with fleet disruption' AS Message,
        g.LocationName AS AffectedEntity,
        AVG(wo.DwellTimeMinutes) AS CurrentValue,
        90.0 AS ThresholdValue
    FROM FactWarehouseOperations wo
    INNER JOIN FactFleetTrips ft ON wo.WarehouseKey = ft.OriginWarehouseKey
    INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -2, GETDATE())
        AND wo.DwellTimeMinutes > 90
        AND ft.IdleTimeMinutes > 25
    GROUP BY g.LocationName
    HAVING COUNT(*) > 3;
    
    -- Send alerts (integrate with email/SMS service)
    SELECT * FROM @AlertTable ORDER BY Severity, CurrentValue DESC;
    
    -- Log to alert history table
    INSERT INTO AlertHistory (AlertType, Severity, Message, AffectedEntity, 
                              CurrentValue, ThresholdValue, AlertDateTime)
    SELECT AlertType, Severity, Message, AffectedEntity, 
           CurrentValue, ThresholdValue, GETDATE()
    FROM @AlertTable;
END;
GO

-- Schedule the alert procedure (SQL Server Agent)
-- Run every 15 minutes: */15 * * * *
```

## Common Patterns & Best Practices

### Pattern 1: Time-Phased Dimension Usage

```sql
-- Always join through DimTime for accurate time bucketing
SELECT 
    t.DateKey,
    t.Hour,
    t.QuarterHour,
    COUNT(wo.OperationKey) AS OperationCount
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey -- Use TimeKey, not raw timestamp
WHERE t.FullDateTime BETWEEN '2026-07-01' AND '2026-07-10'
    AND t.IsBusinessHour = 1
GROUP BY t.DateKey, t.Hour, t.QuarterHour;
```

### Pattern 2: Cross-Fact Bridge Queries

```sql
-- Use common dimensions to bridge multiple fact tables
SELECT 
    g.LocationName,
    t.DateKey,
    SUM(wo.QuantityMoved) AS WarehouseVolume,
    COUNT(DISTINCT ft.TripKey) AS FleetTrips,
    AVG(ft.FuelConsumedGallons) AS AvgFuelPerTrip
FROM DimGeography g
INNER JOIN FactWarehouseOperations wo ON g.GeographyKey = wo.WarehouseKey
INNER JOIN FactFleetTrips ft ON g.GeographyKey = ft.OriginWarehouseKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateKey >= 20260701
GROUP BY g.LocationName, t.DateKey;
```

### Pattern 3: Incremental ETL Pattern

```sql
-- Create ETL metadata table
CREATE TABLE ETLMetadata (
    TableName VARCHAR(100) PRIMARY KEY,
    LastLoadDateTime DATETIME NOT NULL,
    LastLoadRecordCount INT,
    LastLoadStatus VARCHAR(20)
);

-- Incremental load logic
DECLARE @LastLoad DATETIME;
SELECT @LastLoad = LastLoadDateTime 
FROM ETLMetadata 
WHERE TableName = 'FactWarehouseOperations';

-- Load only new/changed records
BEGIN TRANSACTION;
    INSERT INTO FactWarehouseOperations (...)
    SELECT ... 
    FROM SourceSystem.WarehouseEvents
    WHERE ModifiedDate > @LastLoad;
    
    UPDATE ETLMetadata 
    SET LastLoadDateTime = GETDATE(),
        LastLoadRecordCount = @@ROWCOUNT,
        LastLoadStatus = 'Success'
    WHERE TableName = 'FactWarehouseOperations';
COMMIT TRANSACTION;
```

## Configuration Examples

### Row-Level Security Setup

```sql
-- Create security table
CREATE TABLE UserWarehouseAccess (
    Username VARCHAR(100),
    WarehouseKey INT,
    AccessLevel VARCHAR(20) -- 'Read', 'Write', 'Admin'
);

-- Create RLS function
CREATE FUNCTION fn_WarehouseSecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS AccessGranted
    WHERE EXISTS (
        SELECT 1 
        FROM dbo.UserWarehouseAccess
        WHERE Username = USER_NAME()
            AND WarehouseKey = @WarehouseKey
    )
    OR IS_MEMBER('LogisticsAdmin') = 1;
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_WarehouseSecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_WarehouseSecurityPredicate(OriginWarehouseKey)
ON dbo.FactFleetTrips
WITH (STATE = ON);
GO
```

### Power BI Refresh Configuration

```json
{
  "refreshSchedule": {
    "enabled": true,
    "times": ["02:00", "08:00", "14:00", "20:00"],
    "timeZone": "UTC",
    "notifyOnFailure": true,
    "email": "${ADMIN_EMAIL}"
  },
  "credentials": {
    "database": {
      "connectionType": "sql",
      "server": "${SQL_SERVER}",
      "database": "LogiFleetPulse",
      "authenticationType": "ServicePrincipal",
      "tenantId": "${AZURE_TENANT_ID}",
      "clientId":
