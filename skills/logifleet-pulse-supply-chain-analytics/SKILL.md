---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI dashboards for logistics and supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse dashboards"
  - "connect LogiFleet to SQL Server"
  - "implement cross-fact KPI queries for logistics"
  - "create warehouse gravity zone analysis"
  - "build multi-modal supply chain reports"
  - "integrate fleet telemetry with warehouse operations"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and supply chain analytics into a single semantic data layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse, fleet, and cross-dock operations
- **Time-phased dimensions** for temporal analysis at 15-minute granularity
- **Cross-modal KPI harmonization** (e.g., warehouse dwell time vs. fleet fuel costs)
- **Predictive bottleneck detection** using composite metrics
- **Warehouse Gravity Zones™** for spatial optimization
- **Role-based dashboards** with row-level security

The platform is designed for 3PL operators, retail chains, food distributors, and any organization needing unified logistics visibility.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telematics APIs)

### Step 1: Deploy SQL Schema

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME NOT NULL,
    TimeSlot TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    WeekOfYear INT NOT NULL,
    MonthNumber INT NOT NULL,
    MonthName VARCHAR(10) NOT NULL,
    Quarter INT NOT NULL,
    FiscalPeriod VARCHAR(7) NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0
);

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(FullDateTime);

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) NOT NULL UNIQUE,
    NodeType VARCHAR(20) NOT NULL, -- 'Warehouse', 'Route', 'Depot'
    NodeName VARCHAR(100) NOT NULL,
    Address VARCHAR(255),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    ZoneCode VARCHAR(20),
    IsActive BIT DEFAULT 1
);

CREATE INDEX IX_DimGeography_NodeType ON DimGeography(NodeType);
CREATE INDEX IX_DimGeography_Country ON DimGeography(Country);

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    GravityScore DECIMAL(5,2) NOT NULL, -- Composite metric
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2), -- 0-1 scale
    OptimalZone VARCHAR(10), -- Recommended storage zone
    ReplenishmentLeadDays INT,
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2)
);

CREATE INDEX IX_DimProductGravity_VelocityClass ON DimProductGravity(VelocityClass);
CREATE INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC);

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(20) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderID VARCHAR(50),
    BatchID VARCHAR(50),
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT, -- Time spent in warehouse
    ProcessingTimeMinutes DECIMAL(10,2),
    ZoneAssigned VARCHAR(10),
    OperatorID VARCHAR(50),
    QualityScore DECIMAL(3,2), -- 0-1 scale
    ExceptionFlag BIT DEFAULT 0,
    ExceptionReason VARCHAR(255)
);

CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactWarehouse_Type ON FactWarehouseOperations(OperationType);

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    RouteID VARCHAR(50),
    TripDistanceKm DECIMAL(10,2),
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    DeliveryCount INT,
    OnTimeDeliveries INT,
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT,
    MaintenanceFlag BIT DEFAULT 0,
    TirePressureAvg DECIMAL(5,2),
    EngineHealthScore DECIMAL(3,2) -- 0-1 scale
);

CREATE INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);
CREATE INDEX IX_FactFleet_Route ON FactFleetTrips(RouteID);

-- Create cross-dock operations fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    HubKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TransferTimeMinutes INT,
    QuantityTransferred INT,
    TemperatureCompliant BIT,
    DamageFlag BIT DEFAULT 0
);

CREATE INDEX IX_FactCrossDock_Time ON FactCrossDock(TimeKey);
CREATE INDEX IX_FactCrossDock_Hub ON FactCrossDock(HubKey);
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime with 15-minute intervals
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (
            FullDateTime,
            TimeSlot,
            HourOfDay,
            DayOfWeek,
            DayName,
            WeekOfYear,
            MonthNumber,
            MonthName,
            Quarter,
            FiscalPeriod,
            IsWeekend
        )
        VALUES (
            @CurrentDateTime,
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(MONTH, @CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            CONCAT(YEAR(@CurrentDateTime), '-', RIGHT('0' + CAST(DATEPART(MONTH, @CurrentDateTime) AS VARCHAR), 2)),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute for 2 years of data
EXEC PopulateTimeDimension '2024-01-01', '2026-12-31';
```

### Step 3: Create Views for Cross-Fact KPIs

```sql
-- View: Warehouse efficiency with fleet impact
CREATE VIEW vw_WarehouseFleetImpact AS
SELECT 
    dt.FiscalPeriod,
    dg.NodeName AS Warehouse,
    dpg.Category,
    dpg.VelocityClass,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(fwo.ProcessingTimeMinutes) AS AvgProcessingTime,
    SUM(fwo.QuantityHandled) AS TotalUnitsHandled,
    COUNT(DISTINCT CASE WHEN fwo.ExceptionFlag = 1 THEN fwo.OperationKey END) AS ExceptionCount,
    -- Correlated fleet metrics for same time/location
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleTime,
    AVG(fft.FuelConsumedLiters) AS AvgFuelConsumption
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fwo.WarehouseKey = dg.GeographyKey
INNER JOIN DimProductGravity dpg ON fwo.ProductKey = dpg.ProductKey
LEFT JOIN FactFleetTrips fft ON fft.OriginKey = fwo.WarehouseKey 
    AND fft.TimeKey BETWEEN fwo.TimeKey - 4 AND fwo.TimeKey + 4 -- 1-hour window
GROUP BY dt.FiscalPeriod, dg.NodeName, dpg.Category, dpg.VelocityClass;
GO

-- View: Gravity zone optimization recommendations
CREATE VIEW vw_GravityZoneAnalysis AS
SELECT 
    dpg.SKU,
    dpg.ProductName,
    dpg.GravityScore,
    dpg.OptimalZone AS RecommendedZone,
    fwo.ZoneAssigned AS CurrentZone,
    COUNT(*) AS PickOperations,
    AVG(fwo.ProcessingTimeMinutes) AS AvgPickTime,
    CASE 
        WHEN dpg.OptimalZone != fwo.ZoneAssigned THEN 'Misaligned'
        ELSE 'Optimal'
    END AS ZoneAlignment,
    AVG(fwo.ProcessingTimeMinutes) - 
        AVG(AVG(fwo.ProcessingTimeMinutes)) OVER (PARTITION BY dpg.OptimalZone) AS TimeDeltaVsOptimal
FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dpg ON fwo.ProductKey = dpg.ProductKey
WHERE fwo.OperationType = 'Picking'
GROUP BY dpg.SKU, dpg.ProductName, dpg.GravityScore, dpg.OptimalZone, fwo.ZoneAssigned;
GO
```

### Step 4: Configure Data Loading

```sql
-- Stored procedure for incremental warehouse data load
CREATE PROCEDURE LoadWarehouseOperations
    @SourceConnectionString VARCHAR(500) = NULL
AS
BEGIN
    -- Example using OPENROWSET for external data
    -- Replace with your WMS connection details from environment variable
    -- Assumes data is staged in a JSON or CSV format
    
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        WarehouseKey,
        ProductKey,
        OperationType,
        OrderID,
        QuantityHandled,
        DwellTimeMinutes,
        ProcessingTimeMinutes,
        ZoneAssigned,
        OperatorID,
        QualityScore
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dpg.ProductKey,
        src.operation_type,
        src.order_id,
        src.quantity,
        src.dwell_time,
        src.processing_time,
        src.zone,
        src.operator_id,
        src.quality_score
    FROM OPENROWSET(
        BULK 'warehouse_staging.csv',
        DATA_SOURCE = 'ExternalDataSource',
        FORMAT = 'CSV',
        FIRSTROW = 2
    ) AS src
    INNER JOIN DimTime dt ON dt.FullDateTime = src.operation_datetime
    INNER JOIN DimGeography dg ON dg.NodeID = src.warehouse_id
    INNER JOIN DimProductGravity dpg ON dpg.SKU = src.sku
    WHERE src.operation_datetime > (SELECT MAX(dt2.FullDateTime) FROM FactWarehouseOperations fwo2 INNER JOIN DimTime dt2 ON fwo2.TimeKey = dt2.TimeKey);
END;
GO
```

### Step 5: Connect Power BI

1. Open Power BI Desktop
2. Download the `.pbit` template from the repository
3. When prompted, enter your SQL Server connection string:
   - Server: `your-server.database.windows.net` or `localhost`
   - Database: `LogiFleetPulse`
   - Use Windows or SQL authentication (store credentials in environment variables)

4. The template auto-loads relationships and creates these dashboards:
   - **Executive Overview**: High-level KPIs across warehouse and fleet
   - **Warehouse Heatmap**: Gravity zone utilization and dwell time analysis
   - **Fleet Performance**: Route efficiency, fuel consumption, idle time
   - **Cross-Dock Monitor**: Real-time transfer tracking
   - **Predictive Bottlenecks**: ML-powered congestion forecasting

## Key Queries and API Patterns

### Calculate Composite Gravity Score

```sql
-- Update gravity scores based on velocity, value, and fragility
UPDATE dpg
SET GravityScore = (
    (VelocityScore * 0.4) + 
    (ValueScore * 0.35) + 
    (FragilityPenalty * 0.25)
)
FROM DimProductGravity dpg
CROSS APPLY (
    SELECT 
        CASE 
            WHEN VelocityClass = 'Fast' THEN 10
            WHEN VelocityClass = 'Medium' THEN 5
            ELSE 1
        END AS VelocityScore,
        CASE 
            WHEN ValueTier = 'High' THEN 10
            WHEN ValueTier = 'Medium' THEN 5
            ELSE 1
        END AS ValueScore,
        (10 - (FragilityIndex * 10)) AS FragilityPenalty
) calc;

-- Reassign optimal zones based on gravity
UPDATE DimProductGravity
SET OptimalZone = 
    CASE 
        WHEN GravityScore >= 8 THEN 'A' -- Closest to shipping
        WHEN GravityScore >= 5 THEN 'B'
        WHEN GravityScore >= 3 THEN 'C'
        ELSE 'D' -- Farthest from shipping
    END;
```

### Cross-Fact Analysis: Dwell Time vs. Fleet Idle

```sql
-- Find correlation between warehouse dwell and fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        dt.FullDateTime,
        dg.GeographyKey,
        AVG(fwo.DwellTimeMinutes) AS AvgDwell
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fwo.WarehouseKey = dg.GeographyKey
    WHERE dt.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY dt.FullDateTime, dg.GeographyKey
),
FleetIdle AS (
    SELECT 
        dt.FullDateTime,
        dg.GeographyKey,
        AVG(fft.IdleTimeMinutes) AS AvgIdle
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fft.OriginKey = dg.GeographyKey
    WHERE dt.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY dt.FullDateTime, dg.GeographyKey
)
SELECT 
    wd.FullDateTime,
    wd.GeographyKey,
    wd.AvgDwell,
    fi.AvgIdle,
    (wd.AvgDwell * fi.AvgIdle) / 100.0 AS CompositeInefficiencyIndex
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.FullDateTime = fi.FullDateTime AND wd.GeographyKey = fi.GeographyKey
WHERE wd.AvgDwell > 120 OR fi.AvgIdle > 20 -- Threshold for alerts
ORDER BY CompositeInefficiencyIndex DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify future bottlenecks using historical patterns
WITH HistoricalBottlenecks AS (
    SELECT 
        dt.HourOfDay,
        dt.DayOfWeek,
        dg.NodeName,
        AVG(fwo.DwellTimeMinutes) AS AvgHistoricalDwell,
        STDEV(fwo.DwellTimeMinutes) AS DwellStdDev
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fwo.WarehouseKey = dg.GeographyKey
    WHERE dt.FullDateTime BETWEEN DATEADD(MONTH, -6, GETDATE()) AND DATEADD(MONTH, -1, GETDATE())
    GROUP BY dt.HourOfDay, dt.DayOfWeek, dg.NodeName
),
CurrentTrend AS (
    SELECT 
        dt.HourOfDay,
        dt.DayOfWeek,
        dg.NodeName,
        AVG(fwo.DwellTimeMinutes) AS CurrentDwell
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fwo.WarehouseKey = dg.GeographyKey
    WHERE dt.FullDateTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY dt.HourOfDay, dt.DayOfWeek, dg.NodeName
)
SELECT 
    hb.NodeName,
    hb.HourOfDay,
    hb.DayOfWeek,
    hb.AvgHistoricalDwell,
    ct.CurrentDwell,
    (ct.CurrentDwell - hb.AvgHistoricalDwell) / NULLIF(hb.DwellStdDev, 0) AS ZScore,
    CASE 
        WHEN (ct.CurrentDwell - hb.AvgHistoricalDwell) / NULLIF(hb.DwellStdDev, 0) > 2 THEN 'High Risk'
        WHEN (ct.CurrentDwell - hb.AvgHistoricalDwell) / NULLIF(hb.DwellStdDev, 0) > 1 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM HistoricalBottlenecks hb
INNER JOIN CurrentTrend ct ON hb.HourOfDay = ct.HourOfDay 
    AND hb.DayOfWeek = ct.DayOfWeek 
    AND hb.NodeName = ct.NodeName
WHERE (ct.CurrentDwell - hb.AvgHistoricalDwell) / NULLIF(hb.DwellStdDev, 0) > 1
ORDER BY ZScore DESC;
```

### Fleet Maintenance Priority Queue

```sql
-- Rank fleet maintenance needs by revenue impact
WITH VehicleHealth AS (
    SELECT 
        VehicleID,
        AVG(EngineHealthScore) AS AvgHealthScore,
        AVG(TirePressureAvg) AS AvgTirePressure,
        SUM(CASE WHEN MaintenanceFlag = 1 THEN 1 ELSE 0 END) AS MaintenanceEvents,
        COUNT(*) AS TotalTrips
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(MONTH, -1, GETDATE()) ORDER BY TimeKey OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY)
    GROUP BY VehicleID
),
VehicleRevenue AS (
    SELECT 
        fft.VehicleID,
        SUM(fwo.QuantityHandled * dpg.GravityScore) AS RevenueProxyScore
    FROM FactFleetTrips fft
    INNER JOIN FactWarehouseOperations fwo ON fwo.TimeKey BETWEEN fft.TimeKey - 4 AND fft.TimeKey + 4
    INNER JOIN DimProductGravity dpg ON fwo.ProductKey = dpg.ProductKey
    GROUP BY fft.VehicleID
)
SELECT 
    vh.VehicleID,
    vh.AvgHealthScore,
    vh.AvgTirePressure,
    vh.MaintenanceEvents,
    vr.RevenueProxyScore,
    (1 - vh.AvgHealthScore) * vr.RevenueProxyScore AS MaintenancePriorityScore,
    CASE 
        WHEN (1 - vh.AvgHealthScore) * vr.RevenueProxyScore > 500 THEN 'Critical'
        WHEN (1 - vh.AvgHealthScore) * vr.RevenueProxyScore > 200 THEN 'High'
        WHEN (1 - vh.AvgHealthScore) * vr.RevenueProxyScore > 50 THEN 'Medium'
        ELSE 'Low'
    END AS PriorityTier
FROM VehicleHealth vh
INNER JOIN VehicleRevenue vr ON vh.VehicleID = vr.VehicleID
WHERE vh.AvgHealthScore < 0.85 OR vh.AvgTirePressure < 30
ORDER BY MaintenancePriorityScore DESC;
```

## Configuration

### Environment Variables

Store sensitive configuration in environment variables or Azure Key Vault:

```bash
# SQL Server connection
export LOGIFLEET_SQL_SERVER="your-server.database.windows.net"
export LOGIFLEET_SQL_DATABASE="LogiFleetPulse"
export LOGIFLEET_SQL_USER="sql_admin"
export LOGIFLEET_SQL_PASSWORD="your-secure-password"

# External API endpoints
export LOGIFLEET_WMS_API_URL="https://api.yourdomain.com/wms"
export LOGIFLEET_WMS_API_KEY="your-wms-api-key"
export LOGIFLEET_TELEMATICS_API_URL="https://api.telematics-provider.com/v2"
export LOGIFLEET_TELEMATICS_API_KEY="your-telematics-key"

# Power BI service
export LOGIFLEET_PBI_WORKSPACE_ID="your-workspace-guid"
export LOGIFLEET_PBI_DATASET_ID="your-dataset-guid"
```

### Automated Alerts Configuration

```sql
-- Create alert configuration table
CREATE TABLE AlertConfig (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(100) NOT NULL,
    MetricType VARCHAR(50) NOT NULL,
    ThresholdValue DECIMAL(10,2) NOT NULL,
    ComparisonOperator VARCHAR(10) NOT NULL, -- '>', '<', '>=', '<=', '='
    RecipientEmail VARCHAR(255) NOT NULL,
    IsActive BIT DEFAULT 1
);

-- Insert sample alerts
INSERT INTO AlertConfig (AlertName, MetricType, ThresholdValue, ComparisonOperator, RecipientEmail)
VALUES 
    ('High Fleet Idle Time', 'IdleTimePercent', 15, '>', 'ops-manager@company.com'),
    ('Warehouse Dwell Exceeded', 'DwellTimeMinutes', 120, '>', 'warehouse-lead@company.com'),
    ('Low Engine Health', 'EngineHealthScore', 0.75, '<', 'maintenance@company.com');

-- Stored procedure to check and send alerts
CREATE PROCEDURE CheckAndSendAlerts
AS
BEGIN
    DECLARE @AlertMessage VARCHAR(MAX);
    
    -- Example: Check fleet idle time
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips fft
        INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
        INNER JOIN AlertConfig ac ON ac.MetricType = 'IdleTimePercent' AND ac.IsActive = 1
        WHERE dt.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
        AND (CAST(fft.IdleTimeMinutes AS DECIMAL) / NULLIF(fft.TripDurationMinutes, 0) * 100) > ac.ThresholdValue
    )
    BEGIN
        SET @AlertMessage = 'Alert: Fleet idle time exceeded threshold in the last hour.';
        -- Send via SQL Server Database Mail or external API
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = 'ops-manager@company.com',
            @subject = 'LogiFleet Pulse Alert: High Fleet Idle Time',
            @body = @AlertMessage;
    END
END;
GO

-- Schedule this procedure to run every 15 minutes via SQL Server Agent
```

### Row-Level Security Setup

```sql
-- Create security table for role-based access
CREATE TABLE UserRoleSecurity (
    UserID VARCHAR(100) NOT NULL,
    RoleType VARCHAR(50) NOT NULL, -- 'Executive', 'Supervisor', 'Operator'
    AllowedWarehouseKeys VARCHAR(MAX), -- Comma-separated GeographyKeys
    AllowedVehicleIDs VARCHAR(MAX) -- Comma-separated VehicleIDs
);

-- Create security predicate function
CREATE FUNCTION fn_SecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessGranted
WHERE 
    @WarehouseKey IN (
        SELECT value 
        FROM STRING_SPLIT((
            SELECT AllowedWarehouseKeys 
            FROM dbo.UserRoleSecurity 
            WHERE UserID = USER_NAME()
        ), ',')
    )
    OR EXISTS (
        SELECT 1 
        FROM dbo.UserRoleSecurity 
        WHERE UserID = USER_NAME() 
        AND RoleType = 'Executive'
    );
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
```

## Common Patterns

### Pattern 1: Incremental ETL with Change Tracking

```sql
-- Enable change tracking on source table
ALTER DATABASE LogiFleetPulse
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 14 DAYS, AUTO_CLEANUP = ON);

ALTER TABLE FactWarehouseOperations
ENABLE CHANGE_TRACKING;

-- Incremental load procedure
CREATE PROCEDURE IncrementalLoadWarehouse
AS
BEGIN
    DECLARE @LastSyncVersion BIGINT = (SELECT ISNULL(MAX(SyncVersion), 0) FROM SyncMetadata WHERE TableName = 'FactWarehouseOperations');
    DECLARE @CurrentSyncVersion BIGINT = CHANGE_TRACKING_CURRENT_VERSION();
    
    -- Load changed records
    INSERT INTO FactWarehouseOperations (...)
    SELECT ...
    FROM StagingWarehouseOperations src
    INNER JOIN CHANGETABLE(CHANGES StagingWarehouseOperations, @LastSyncVersion) ct
        ON src.RowID = ct.RowID
    WHERE ct.SYS_CHANGE_VERSION <= @CurrentSyncVersion;
    
    -- Update sync metadata
    UPDATE SyncMetadata
    SET SyncVersion = @CurrentSyncVersion, LastSyncTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Pattern 2: Time-Series Forecasting for Capacity Planning

```sql
-- Calculate moving averages for capacity trends
WITH DailyCapacity AS (
    SELECT 
        CAST(dt.FullDateTime AS DATE) AS OperationDate,
        dg.NodeName,
        SUM(fwo.QuantityHandled) AS DailyVolume
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fwo.WarehouseKey = dg.GeographyKey
    GROUP BY CAST(dt.FullDateTime AS DATE), dg.NodeName
)
SELECT 
    OperationDate,
    NodeName,
    DailyVolume,
    AVG(DailyVolume) OVER (
        PARTITION BY NodeName 
        ORDER BY OperationDate 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS MA7,
    AVG(DailyVolume) OVER (
        PARTITION BY NodeName 
        ORDER BY OperationDate 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
