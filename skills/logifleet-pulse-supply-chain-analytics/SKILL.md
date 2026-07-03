---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing and fleet logistics intelligence platform for supply chain analytics
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - configure Power BI logistics dashboard
  - deploy SQL Server star schema for warehouse operations
  - implement fleet tracking data warehouse
  - create supply chain KPI dashboards
  - build multi-fact logistics analytics model
  - integrate warehouse and fleet telemetry data
  - set up predictive logistics bottleneck detection
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers implement LogiFleet Pulse, an advanced supply chain analytics platform that combines MS SQL Server data warehousing with Power BI visualization for unified warehouse, fleet, and logistics intelligence.

## What LogiFleet Pulse Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that:

- **Unifies disparate data sources**: Combines warehouse operations, fleet telemetry, supplier data, and external signals (weather, traffic) into a single semantic layer
- **Multi-fact star schema**: Custom data warehouse architecture linking warehouse operations, fleet trips, cross-dock activities, and inventory metrics
- **Real-time dashboards**: Power BI reports refreshed every 15 minutes for operational awareness
- **Predictive analytics**: Identifies bottlenecks, maintenance needs, and capacity constraints before they impact operations
- **Cross-domain KPI harmonization**: Links metrics like inventory turnover with fuel consumption and delivery performance

### Core Components

1. **SQL Server Data Warehouse**: Multi-fact star schema with time-phased dimensions
2. **Power BI Templates**: Pre-built dashboards for warehouse, fleet, and executive views
3. **ETL Procedures**: Stored procedures for incremental data loading
4. **Alerting System**: Automated threshold monitoring and notifications
5. **Security Layer**: Row-level security for role-based access

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, or flat file exports

### Step 1: Deploy SQL Schema

Clone the repository and deploy the database schema:

```sql
-- Create the main database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema deployment script
-- This creates all dimension and fact tables, views, and stored procedures
:r schema/01_create_dimensions.sql
:r schema/02_create_facts.sql
:r schema/03_create_relationships.sql
:r schema/04_create_views.sql
:r schema/05_create_etl_procedures.sql
```

### Step 2: Configure Data Sources

Create a configuration for your data connections:

```sql
-- Create a table to store connection metadata
CREATE TABLE dbo.DataSourceConfig (
    SourceID INT IDENTITY(1,1) PRIMARY KEY,
    SourceName NVARCHAR(100) NOT NULL,
    SourceType NVARCHAR(50), -- 'WMS', 'TMS', 'Telematics', 'API'
    ConnectionString NVARCHAR(500), -- Use environment variables for sensitive data
    RefreshIntervalMinutes INT DEFAULT 15,
    IsActive BIT DEFAULT 1,
    LastRefreshUTC DATETIME2,
    CreatedDate DATETIME2 DEFAULT GETUTCDATE()
);

-- Example: Register your WMS connection
INSERT INTO dbo.DataSourceConfig (SourceName, SourceType, ConnectionString, RefreshIntervalMinutes)
VALUES 
    ('Primary_WMS', 'WMS', 'Server=$(WMS_SERVER);Database=$(WMS_DB);Integrated Security=SSPI;', 15),
    ('Fleet_Telematics', 'API', 'https://api.telematics.example.com/v2?key=$(TELEMATICS_API_KEY)', 5);
```

### Step 3: Initialize Dimension Tables

```sql
-- Populate the time dimension (15-minute granularity)
EXEC dbo.sp_PopulateTimeDimension 
    @StartDate = '2024-01-01', 
    @EndDate = '2026-12-31';

-- Create geography dimension with your locations
INSERT INTO dbo.DimGeography (ContinentName, CountryName, RegionName, LocationName, LocationType, Latitude, Longitude)
VALUES 
    ('North America', 'USA', 'Midwest', 'Chicago Distribution Center', 'Warehouse', 41.8781, -87.6298),
    ('North America', 'USA', 'Midwest', 'Route_CHI_001', 'Route', 41.8781, -87.6298),
    ('North America', 'USA', 'Northeast', 'New York Hub', 'Warehouse', 40.7128, -74.0060);

-- Initialize product gravity zones
INSERT INTO dbo.DimProductGravity (SKU, ProductName, CategoryName, GravityScore, IsHighValue, IsFragile, AveragePicksPerDay)
VALUES 
    ('SKU-1001', 'Premium Electronics', 'Electronics', 95, 1, 1, 120),
    ('SKU-2034', 'Bulk Paper Goods', 'Office Supplies', 35, 0, 0, 15);
```

## Core Data Model

### Key Fact Tables

#### FactWarehouseOperations

Tracks warehouse micro-operations:

```sql
CREATE TABLE dbo.FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimProductGravity(ProductKey),
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    ZoneID NVARCHAR(50),
    OperatorID NVARCHAR(50),
    LoadTimestamp DATETIME2 DEFAULT GETUTCDATE(),
    INDEX IX_WH_Time_Geography CLUSTERED (TimeKey, GeographyKey)
);
```

#### FactFleetTrips

Captures fleet and route performance:

```sql
CREATE TABLE dbo.FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimGeography(GeographyKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    TripDurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    DistanceKM DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    OnTimeDelivery BIT,
    DelayReasonCode NVARCHAR(50), -- 'Weather', 'Traffic', 'Mechanical', NULL if on-time
    LoadTimestamp DATETIME2 DEFAULT GETUTCDATE(),
    INDEX IX_Fleet_Time_Route CLUSTERED (TimeKey, OriginGeographyKey, DestinationGeographyKey)
);
```

### ETL Procedures

#### Incremental Load from WMS

```sql
CREATE PROCEDURE dbo.sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- If no timestamp provided, get the last successful load
    IF @LastLoadTimestamp IS NULL
    BEGIN
        SELECT @LastLoadTimestamp = ISNULL(MAX(LoadTimestamp), '1900-01-01')
        FROM dbo.FactWarehouseOperations;
    END
    
    -- Insert new operations from staging or linked server
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, 
        QuantityHandled, DurationMinutes, DwellTimeHours, ZoneID, OperatorID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        src.OperationType,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DurationMinutes,
        DATEDIFF(HOUR, src.ArrivalTime, src.StartTime) AS DwellTimeHours,
        src.ZoneID,
        src.OperatorID
    FROM WMS_LinkedServer.dbo.Operations src
    INNER JOIN dbo.DimTime t ON t.DateValue = CAST(src.OperationDate AS DATE)
        AND t.HourOfDay = DATEPART(HOUR, src.OperationDate)
        AND t.MinuteBucket = (DATEPART(MINUTE, src.OperationDate) / 15) * 15
    INNER JOIN dbo.DimGeography g ON g.LocationName = src.WarehouseName
    INNER JOIN dbo.DimProductGravity p ON p.SKU = src.SKU
    WHERE src.LastModified > @LastLoadTimestamp;
    
    -- Update source config with last refresh time
    UPDATE dbo.DataSourceConfig
    SET LastRefreshUTC = GETUTCDATE()
    WHERE SourceName = 'Primary_WMS';
    
    RETURN @@ROWCOUNT;
END;
GO
```

#### Cross-Fact KPI Calculation

```sql
CREATE VIEW dbo.vw_WarehouseFleetKPIs AS
SELECT 
    t.DateValue,
    t.FiscalWeek,
    g.LocationName,
    
    -- Warehouse metrics
    SUM(CASE WHEN w.OperationType = 'Picking' THEN w.QuantityHandled ELSE 0 END) AS TotalPicks,
    AVG(CASE WHEN w.OperationType = 'Picking' THEN w.DurationMinutes ELSE NULL END) AS AvgPickTime,
    AVG(w.DwellTimeHours) AS AvgDwellTime,
    
    -- Fleet metrics linked to same warehouse
    COUNT(DISTINCT f.TripID) AS TotalTrips,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    SUM(f.FuelConsumedLiters) AS TotalFuelConsumed,
    
    -- Cross-domain KPI: Dwell time impact on fuel efficiency
    CASE 
        WHEN AVG(w.DwellTimeHours) > 48 THEN 'High Dwell - Potential Rush Delivery'
        WHEN AVG(w.DwellTimeHours) < 12 THEN 'Optimal Flow'
        ELSE 'Normal'
    END AS DwellImpactFlag,
    
    -- Fuel efficiency adjusted for warehouse dwell
    CASE 
        WHEN AVG(w.DwellTimeHours) > 0 AND SUM(f.DistanceKM) > 0 
        THEN (SUM(f.FuelConsumedLiters) / SUM(f.DistanceKM)) * (1 + (AVG(w.DwellTimeHours) / 100))
        ELSE NULL
    END AS AdjustedFuelEfficiency

FROM dbo.DimTime t
LEFT JOIN dbo.FactWarehouseOperations w ON w.TimeKey = t.TimeKey
LEFT JOIN dbo.DimGeography g ON g.GeographyKey = w.GeographyKey
LEFT JOIN dbo.FactFleetTrips f ON f.TimeKey = t.TimeKey 
    AND (f.OriginGeographyKey = g.GeographyKey OR f.DestinationGeographyKey = g.GeographyKey)
WHERE t.DateValue >= DATEADD(MONTH, -3, GETDATE())
    AND g.LocationType = 'Warehouse'
GROUP BY t.DateValue, t.FiscalWeek, g.LocationName;
GO
```

## Power BI Integration

### Connecting to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter your SQL Server connection details:
   - Server: `$(SQL_SERVER_NAME)`
   - Database: `LogiFleetPulse`
   - Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)

### DAX Measures for Advanced KPIs

Create calculated measures in Power BI:

```dax
// Warehouse Gravity Score (higher = more urgent attention needed)
WarehouseGravityScore = 
VAR HighDwellCount = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[DwellTimeHours] > 72
    )
VAR HighValueItems = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[IsHighValue] = TRUE
    )
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
RETURN
    IF(TotalOps > 0, 
       (HighDwellCount * 2 + HighValueItems) / TotalOps * 100,
       0
    )

// Fleet Efficiency Index
FleetEfficiencyIndex = 
VAR ActualDistance = SUM(FactFleetTrips[DistanceKM])
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR OnTimeRate = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeDelivery] = TRUE),
        COUNTROWS(FactFleetTrips)
    )
RETURN
    IF(TotalTime > 0,
       (ActualDistance / TotalTime) * (1 - (IdleTime / TotalTime)) * OnTimeRate * 100,
       0
    )

// Cross-Domain Bottleneck Score
BottleneckScore = 
VAR WarehouseScore = [WarehouseGravityScore]
VAR FleetScore = 100 - [FleetEfficiencyIndex]
VAR WeatherDelayPct = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[DelayReasonCode] = "Weather"),
        COUNTROWS(FactFleetTrips)
    ) * 100
RETURN
    (WarehouseScore * 0.4) + (FleetScore * 0.4) + (WeatherDelayPct * 0.2)
```

### Row-Level Security Setup

```dax
// Create a security role for regional managers
// In Power BI Desktop: Modeling > Manage Roles > New

[Region] = USERNAME()

// Or for more complex scenarios with a security table:
// Create DimUserSecurity table in SQL with columns: UserEmail, AllowedRegions

VAR UserEmail = USERPRINCIPALNAME()
VAR AllowedRegions = 
    CALCULATETABLE(
        VALUES(DimUserSecurity[AllowedRegion]),
        DimUserSecurity[UserEmail] = UserEmail
    )
RETURN
    [RegionName] IN AllowedRegions
```

## Automated Alerting

### Create Alert Threshold Table

```sql
CREATE TABLE dbo.AlertThresholds (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName NVARCHAR(200),
    MetricType NVARCHAR(100), -- 'DwellTime', 'IdleTime', 'FuelEfficiency', etc.
    ThresholdValue DECIMAL(18,2),
    ThresholdOperator NVARCHAR(10), -- '>', '<', '=', '>=', '<='
    NotificationEmail NVARCHAR(255),
    NotificationSMS NVARCHAR(50),
    IsActive BIT DEFAULT 1
);

INSERT INTO dbo.AlertThresholds (AlertName, MetricType, ThresholdValue, ThresholdOperator, NotificationEmail)
VALUES 
    ('High Dwell Time Alert', 'DwellTime', 72.0, '>', '$(ALERT_EMAIL)'),
    ('Fleet Idle Warning', 'IdleTimePercent', 15.0, '>', '$(ALERT_EMAIL)'),
    ('Low Fuel Efficiency', 'KmPerLiter', 6.0, '<', '$(ALERT_EMAIL)');
```

### Alert Monitoring Procedure

```sql
CREATE PROCEDURE dbo.sp_CheckAlertThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check dwell time alerts
    INSERT INTO dbo.AlertLog (AlertID, LocationName, MetricValue, TriggeredDate)
    SELECT 
        a.AlertID,
        g.LocationName,
        AVG(w.DwellTimeHours) AS AvgDwellTime,
        GETUTCDATE()
    FROM dbo.AlertThresholds a
    CROSS APPLY (
        SELECT 
            g.LocationName,
            AVG(w.DwellTimeHours) AS AvgDwellTime
        FROM dbo.FactWarehouseOperations w
        INNER JOIN dbo.DimGeography g ON g.GeographyKey = w.GeographyKey
        INNER JOIN dbo.DimTime t ON t.TimeKey = w.TimeKey
        WHERE t.DateValue >= CAST(GETDATE() AS DATE)
            AND g.LocationType = 'Warehouse'
        GROUP BY g.LocationName
        HAVING AVG(w.DwellTimeHours) > a.ThresholdValue
    ) g
    WHERE a.MetricType = 'DwellTime'
        AND a.IsActive = 1;
    
    -- Check fleet idle time alerts
    INSERT INTO dbo.AlertLog (AlertID, LocationName, MetricValue, TriggeredDate)
    SELECT 
        a.AlertID,
        g.LocationName,
        (SUM(f.IdleTimeMinutes) * 100.0 / NULLIF(SUM(f.TripDurationMinutes), 0)) AS IdlePercent,
        GETUTCDATE()
    FROM dbo.AlertThresholds a
    CROSS APPLY (
        SELECT 
            g.LocationName,
            SUM(f.IdleTimeMinutes) AS TotalIdle,
            SUM(f.TripDurationMinutes) AS TotalDuration
        FROM dbo.FactFleetTrips f
        INNER JOIN dbo.DimGeography g ON g.GeographyKey = f.OriginGeographyKey
        INNER JOIN dbo.DimTime t ON t.TimeKey = f.TimeKey
        WHERE t.DateValue >= CAST(GETDATE() AS DATE)
        GROUP BY g.LocationName
        HAVING (SUM(f.IdleTimeMinutes) * 100.0 / NULLIF(SUM(f.TripDurationMinutes), 0)) > a.ThresholdValue
    ) g
    WHERE a.MetricType = 'IdleTimePercent'
        AND a.IsActive = 1;
    
    -- Send notifications (integrate with SQL Server Database Mail or external API)
    -- This is a placeholder - implement based on your notification system
    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = 'LogiFleet Alerts',
        @recipients = '$(ALERT_EMAIL)',
        @subject = 'LogiFleet Pulse Alert Triggered',
        @body = 'One or more thresholds have been exceeded. Check AlertLog table for details.';
END;
GO

-- Schedule this to run every 15 minutes using SQL Server Agent
```

## Common Usage Patterns

### Pattern 1: Onboard a New Data Source

```sql
-- 1. Register the source
INSERT INTO dbo.DataSourceConfig (SourceName, SourceType, ConnectionString, RefreshIntervalMinutes)
VALUES ('New_Supplier_Portal', 'API', 'https://api.supplier.com/v1?token=$(SUPPLIER_API_TOKEN)', 60);

-- 2. Create a staging table for the new data
CREATE TABLE dbo.Staging_SupplierData (
    SupplierID NVARCHAR(50),
    OrderID NVARCHAR(50),
    SKU NVARCHAR(50),
    LeadTimeDays INT,
    DefectRate DECIMAL(5,2),
    ImportTimestamp DATETIME2 DEFAULT GETUTCDATE()
);

-- 3. Create ETL procedure to load into dimension
CREATE PROCEDURE dbo.sp_LoadSupplierReliability
AS
BEGIN
    MERGE INTO dbo.DimSupplierReliability AS Target
    USING (
        SELECT 
            SupplierID,
            AVG(LeadTimeDays) AS AvgLeadTime,
            AVG(DefectRate) AS AvgDefectRate,
            COUNT(*) AS TotalOrders
        FROM dbo.Staging_SupplierData
        WHERE ImportTimestamp > DATEADD(DAY, -30, GETUTCDATE())
        GROUP BY SupplierID
    ) AS Source
    ON Target.SupplierID = Source.SupplierID
    WHEN MATCHED THEN
        UPDATE SET 
            Target.AvgLeadTimeDays = Source.AvgLeadTime,
            Target.DefectPercentage = Source.AvgDefectRate,
            Target.LastUpdated = GETUTCDATE()
    WHEN NOT MATCHED THEN
        INSERT (SupplierID, AvgLeadTimeDays, DefectPercentage)
        VALUES (Source.SupplierID, Source.AvgLeadTime, Source.AvgDefectRate);
END;
GO
```

### Pattern 2: Create a Custom Cross-Fact Analysis

```sql
-- Example: Analyze correlation between warehouse dwell time and delivery delays
SELECT 
    t.FiscalWeek,
    g.LocationName AS WarehouseName,
    
    -- Warehouse metrics (this week)
    AVG(w.DwellTimeHours) AS AvgWarehouseDwellTime,
    SUM(CASE WHEN w.OperationType = 'Shipping' THEN w.QuantityHandled ELSE 0 END) AS ShipmentsOut,
    
    -- Fleet metrics (same week, same origin)
    COUNT(f.TripID) AS TripsOriginated,
    AVG(f.TripDurationMinutes) AS AvgTripDuration,
    SUM(CASE WHEN f.OnTimeDelivery = 0 THEN 1 ELSE 0 END) AS DelayedDeliveries,
    
    -- Correlation score (simplified)
    CASE 
        WHEN AVG(w.DwellTimeHours) > 48 AND 
             SUM(CASE WHEN f.OnTimeDelivery = 0 THEN 1 ELSE 0 END) > 
             COUNT(f.TripID) * 0.15
        THEN 'High Dwell Correlates with Delays'
        ELSE 'No Strong Correlation'
    END AS CorrelationFlag

FROM dbo.DimTime t
INNER JOIN dbo.FactWarehouseOperations w ON w.TimeKey = t.TimeKey
INNER JOIN dbo.DimGeography g ON g.GeographyKey = w.GeographyKey
LEFT JOIN dbo.FactFleetTrips f ON f.TimeKey = t.TimeKey 
    AND f.OriginGeographyKey = g.GeographyKey
WHERE t.DateValue >= DATEADD(WEEK, -12, GETDATE())
    AND g.LocationType = 'Warehouse'
GROUP BY t.FiscalWeek, g.LocationName
ORDER BY t.FiscalWeek DESC, AvgWarehouseDwellTime DESC;
```

### Pattern 3: Implement Warehouse Gravity Zone Rebalancing

```sql
CREATE PROCEDURE dbo.sp_RecommendGravityRebalance
    @WarehouseID NVARCHAR(50)
AS
BEGIN
    -- Identify SKUs that have changed velocity category
    WITH CurrentVelocity AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            p.ProductName,
            p.GravityScore AS CurrentGravityScore,
            COUNT(w.OperationID) / NULLIF(DATEDIFF(DAY, MIN(t.DateValue), MAX(t.DateValue)), 0) AS CurrentPicksPerDay,
            CASE 
                WHEN COUNT(w.OperationID) / NULLIF(DATEDIFF(DAY, MIN(t.DateValue), MAX(t.DateValue)), 0) > 100 THEN 90
                WHEN COUNT(w.OperationID) / NULLIF(DATEDIFF(DAY, MIN(t.DateValue), MAX(t.DateValue)), 0) > 50 THEN 70
                WHEN COUNT(w.OperationID) / NULLIF(DATEDIFF(DAY, MIN(t.DateValue), MAX(t.DateValue)), 0) > 20 THEN 50
                ELSE 30
            END AS RecommendedGravityScore
        FROM dbo.DimProductGravity p
        LEFT JOIN dbo.FactWarehouseOperations w ON w.ProductKey = p.ProductKey
        LEFT JOIN dbo.DimTime t ON t.TimeKey = w.TimeKey
        INNER JOIN dbo.DimGeography g ON g.GeographyKey = w.GeographyKey
        WHERE g.LocationName = @WarehouseID
            AND t.DateValue >= DATEADD(DAY, -30, GETDATE())
            AND w.OperationType = 'Picking'
        GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore
    )
    SELECT 
        SKU,
        ProductName,
        CurrentGravityScore,
        RecommendedGravityScore,
        CurrentPicksPerDay,
        CASE 
            WHEN RecommendedGravityScore > CurrentGravityScore + 10 
            THEN 'Move Closer to Shipping'
            WHEN RecommendedGravityScore < CurrentGravityScore - 10 
            THEN 'Move to Back Storage'
            ELSE 'No Change Needed'
        END AS RecommendedAction
    FROM CurrentVelocity
    WHERE ABS(RecommendedGravityScore - CurrentGravityScore) > 10
    ORDER BY ABS(RecommendedGravityScore - CurrentGravityScore) DESC;
END;
GO
```

## Troubleshooting

### Issue: Power BI Dashboard Refresh Fails

**Symptoms**: Dashboard shows "Unable to connect to data source" or stale data

**Solutions**:
```sql
-- Check if ETL procedures have run recently
SELECT 
    SourceName,
    LastRefreshUTC,
    DATEDIFF(MINUTE, LastRefreshUTC, GETUTCDATE()) AS MinutesSinceRefresh
FROM dbo.DataSourceConfig
WHERE IsActive = 1
ORDER BY LastRefreshUTC DESC;

-- Manually trigger refresh
EXEC dbo.sp_LoadWarehouseOperations;
EXEC dbo.sp_LoadFleetTrips;

-- Check for errors in SQL Server Agent job history
SELECT 
    job.name AS JobName,
    run_status,
    run_date,
    run_time,
    message
FROM msdb.dbo.sysjobhistory history
INNER JOIN msdb.dbo.sysjobs job ON job.job_id = history.job_id
WHERE job.name LIKE '%LogiFleet%'
ORDER BY run_date DESC, run_time DESC;
```

### Issue: Slow Cross-Fact Queries

**Symptoms**: Dashboard takes >30 seconds to load, timeout errors

**Solutions**:
```sql
-- Add missing indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product_Time
ON dbo.FactWarehouseOperations(ProductKey, TimeKey)
INCLUDE (QuantityHandled, DurationMinutes, DwellTimeHours);

CREATE NONCLUSTERED INDEX IX_FactFleet_Origin_Dest_Time
ON dbo.FactFleetTrips(OriginGeographyKey, DestinationGeographyKey, TimeKey)
INCLUDE (TripDurationMinutes, IdleTimeMinutes, FuelConsumedLiters);

-- Implement table partitioning by month for large fact tables
CREATE PARTITION FUNCTION PF_MonthlyPartition (DATE)
AS RANGE RIGHT FOR VALUES (
    '2025-01-01', '2025-02-01', '2025-03-01', 
    '2025-04-01', '2025-05-01', '2025-06-01',
    '2025-07-01', '2025-08-01', '2025-09-01',
    '2025-10-01', '2025-11-01', '2025-12-01',
    '2026-01-01'
);

CREATE PARTITION SCHEME PS_MonthlyPartition
AS PARTITION PF_MonthlyPartition
ALL TO ([PRIMARY]);

-- Rebuild fact table on partition scheme (requires downtime)
-- See documentation for full partitioning migration steps
```

### Issue: Alert Notifications Not Sending

**Symptoms**: Thresholds breached but no emails/SMS received

**Solutions**:
```sql
-- Check if Database Mail is configured
EXEC msdb.dbo.sysmail_help_status_sp;

-- Verify alert log is being populated
SELECT TOP 10 *
FROM dbo.AlertLog
ORDER BY TriggeredDate DESC;

-- Test Database Mail directly
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'LogiFleet Alerts',
    @recipients = '$(ALERT_EMAIL)',
    @subject = 'Test Alert',
    @body = 'This is a test message from LogiFleet Pulse.';

-- Check mail queue for errors
SELECT * FROM msdb.dbo.sysmail_allitems WHERE send_request_date > DATEADD(HOUR, -1, GETUTCDATE());
SELECT * FROM msdb.dbo.sysmail_faileditems WHERE send_request_date > DATEADD(HOUR, -1, GETUTCDATE());
```

### Issue: Row-Level Security Not Working

**Symptoms**: Users see data from regions they shouldn't access

**Solutions**:
```dax
// Verify the security table is correctly populated
// In Power BI, go to View > Performance Analyzer and check if RLS filters are applied

// Test RLS in Power BI Desktop:
// Modeling > View As > Select the role and specific user

// Common fix: Ensure bidirectional filtering is enabled between DimUserSecurity and DimGeography
// In Power BI Model view, double-click the relationship line and set:
// Cross filter direction: Both
// Apply security filter in both directions: Yes
```

```sql
-- SQL-side RLS (alternative to Power BI RLS)
CREATE FUNCTION dbo.fn_SecurityPredicate(@UserRegion NVARCHAR(
