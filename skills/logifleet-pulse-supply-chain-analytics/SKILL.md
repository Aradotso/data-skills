---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehouse for real-time logistics, fleet management, and supply chain intelligence with multi-fact star schema modeling
triggers:
  - "set up logistics analytics dashboard"
  - "build supply chain data warehouse"
  - "create fleet management reporting system"
  - "implement warehouse operations analytics"
  - "configure Power BI for logistics KPIs"
  - "design multi-fact star schema for supply chain"
  - "track fleet telemetry and warehouse metrics"
  - "analyze cross-dock and inventory operations"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a comprehensive data warehousing and analytics platform for logistics and supply chain management. It integrates warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer using MS SQL Server and Power BI. The platform uses a multi-fact star schema with time-phased dimensions to enable cross-fact KPI analysis and predictive bottleneck detection.

## Core Capabilities

- **Multi-Fact Star Schema**: Warehouse operations, fleet trips, cross-dock activities linked through shared dimensions
- **Real-Time Dashboards**: Power BI visualizations with 15-minute refresh intervals
- **Predictive Analytics**: Bottleneck detection, fleet maintenance prioritization, warehouse optimization
- **Unified Semantic Layer**: Single source of truth across logistics operations
- **Role-Based Access**: Row-level security for different organizational roles

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Enterprise or Standard edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Access to warehouse management system (WMS) and fleet telemetry data sources

### Step 1: Deploy SQL Schema

```sql
-- Create the main database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTime DATETIME2 NOT NULL,
    TimeSlot15Min VARCHAR(5),
    HourOfDay INT,
    DayOfWeek VARCHAR(10),
    DayOfMonth INT,
    MonthName VARCHAR(10),
    QuarterName VARCHAR(10),
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);

CREATE UNIQUE NONCLUSTERED INDEX IX_DimTime_DateTime 
ON DimTime(DateTime);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) NOT NULL UNIQUE,
    NodeType VARCHAR(20), -- 'Warehouse', 'Route', 'Hub'
    LocationName VARCHAR(100),
    City VARCHAR(50),
    Region VARCHAR(50),
    Country VARCHAR(50),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50)
);

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(50),
    SubCategory VARCHAR(50),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT,
    IsPerishable BIT,
    RequiresRefrigeration BIT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity × value × fragility
    GravityZone VARCHAR(20) -- 'High', 'Medium', 'Low'
);

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    Country VARCHAR(50),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(3,2),
    ReliabilityTier VARCHAR(10) -- 'Gold', 'Silver', 'Bronze'
);

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) NOT NULL UNIQUE,
    VehicleType VARCHAR(50),
    Capacity DECIMAL(10,2),
    FuelType VARCHAR(20),
    YearManufactured INT,
    MaintenanceStatus VARCHAR(20),
    LastMaintenanceDate DATE
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityProcessed INT,
    ProcessingTimeMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    StorageZone VARCHAR(50),
    BinLocation VARCHAR(50),
    OperatorID VARCHAR(50),
    BatchID VARCHAR(50),
    QualityScore DECIMAL(3,2),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time 
ON FactWarehouseOperations(TimeKey) INCLUDE (GeographyKey, ProductKey);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    FleetKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripDistanceKm DECIMAL(10,2),
    TripDurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    LoadUtilizationPct DECIMAL(5,2),
    AverageSpeedKmh DECIMAL(5,2),
    RouteSegmentID VARCHAR(50),
    WeatherCondition VARCHAR(50),
    TrafficDelay BIT,
    OnTimeDelivery BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (FleetKey) REFERENCES DimFleet(FleetKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time 
ON FactFleetTrips(TimeKey) INCLUDE (FleetKey, OriginGeographyKey);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundQuantity INT,
    OutboundQuantity INT,
    CrossDockTimeMinutes DECIMAL(10,2),
    TemperatureCompliance BIT,
    DamageReported BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey)
);
```

### Step 2: Create ETL Stored Procedures

```sql
-- Procedure to populate DimTime with 15-minute granularity
CREATE PROCEDURE sp_LoadDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (
            DateTime, 
            TimeSlot15Min, 
            HourOfDay, 
            DayOfWeek, 
            DayOfMonth, 
            MonthName, 
            QuarterName, 
            FiscalYear, 
            IsWeekend, 
            IsHoliday
        )
        SELECT 
            @CurrentDateTime,
            FORMAT(@CurrentDateTime, 'HH:mm'),
            DATEPART(HOUR, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DAY(@CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            'Q' + CAST(DATEPART(QUARTER, @CurrentDateTime) AS VARCHAR),
            YEAR(@CurrentDateTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END,
            0; -- Update with holiday logic
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Procedure to calculate product gravity scores
CREATE PROCEDURE sp_UpdateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(DISTINCT f.OperationKey) AS PickFrequency,
            AVG(f.DwellTimeHours) AS AvgDwellTime,
            p.IsFragile,
            p.UnitWeight * p.UnitVolume AS ProductDensity
        FROM DimProduct p
        LEFT JOIN FactWarehouseOperations f ON p.ProductKey = f.ProductKey
        WHERE f.OperationType = 'Picking'
            AND f.TimeKey IN (
                SELECT TimeKey FROM DimTime 
                WHERE DateTime >= DATEADD(DAY, -90, GETDATE())
            )
        GROUP BY 
            p.ProductKey, 
            p.IsFragile, 
            p.UnitWeight, 
            p.UnitVolume
    )
    UPDATE p
    SET 
        GravityScore = (
            (ISNULL(m.PickFrequency, 0) * 0.5) + 
            ((1.0 / NULLIF(m.AvgDwellTime, 0)) * 0.3) + 
            (CASE WHEN m.IsFragile = 1 THEN 10 ELSE 0 END * 0.2)
        ),
        GravityZone = CASE 
            WHEN (ISNULL(m.PickFrequency, 0) * 0.5 + (1.0 / NULLIF(m.AvgDwellTime, 0)) * 0.3) > 50 THEN 'High'
            WHEN (ISNULL(m.PickFrequency, 0) * 0.5 + (1.0 / NULLIF(m.AvgDwellTime, 0)) * 0.3) > 20 THEN 'Medium'
            ELSE 'Low'
        END
    FROM DimProduct p
    INNER JOIN ProductMetrics m ON p.ProductKey = m.ProductKey;
END;
GO

-- Procedure for incremental warehouse data load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assumes external staging table from WMS
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        GeographyKey,
        ProductKey,
        OperationType,
        QuantityProcessed,
        ProcessingTimeMinutes,
        DwellTimeHours,
        StorageZone,
        BinLocation,
        OperatorID,
        BatchID,
        QualityScore
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        stg.OperationType,
        stg.Quantity,
        stg.ProcessingTimeMinutes,
        stg.DwellTimeHours,
        stg.StorageZone,
        stg.BinLocation,
        stg.OperatorID,
        stg.BatchID,
        stg.QualityScore
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON t.DateTime = stg.OperationDateTime
    INNER JOIN DimGeography g ON g.NodeID = stg.WarehouseID
    INNER JOIN DimProduct p ON p.SKU = stg.SKU
    WHERE stg.OperationDateTime > @LastLoadDateTime;
END;
GO
```

### Step 3: Configure Power BI Template

```powerquery
// Power Query M for dynamic date filtering
let
    Source = Sql.Database(
        #"SQL_SERVER_NAME", 
        "LogiFleetPulse", 
        [Query="
            SELECT 
                t.DateTime,
                t.HourOfDay,
                t.DayOfWeek,
                g.LocationName AS Warehouse,
                g.Region,
                p.ProductName,
                p.Category,
                p.GravityZone,
                f.OperationType,
                f.QuantityProcessed,
                f.ProcessingTimeMinutes,
                f.DwellTimeHours,
                f.QualityScore
            FROM FactWarehouseOperations f
            INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
            INNER JOIN DimGeography g ON f.GeographyKey = g.GeographyKey
            INNER JOIN DimProduct p ON f.ProductKey = p.ProductKey
            WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
        "]
    ),
    ChangedType = Table.TransformColumnTypes(
        Source,
        {
            {"DateTime", type datetime},
            {"QuantityProcessed", Int64.Type},
            {"ProcessingTimeMinutes", type number},
            {"DwellTimeHours", type number},
            {"QualityScore", type number}
        }
    )
in
    ChangedType
```

## Key Queries & Analytics

### Cross-Fact KPI: Warehouse Dwell Time vs Fleet Idle Costs

```sql
-- Correlate slow-moving inventory with delivery route inefficiency
WITH WarehouseDwell AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        AVG(f.DwellTimeHours) AS AvgDwellTime,
        SUM(f.QuantityProcessed) AS TotalVolume
    FROM FactWarehouseOperations f
    INNER JOIN DimProduct p ON f.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -90, GETDATE())
        AND f.OperationType IN ('Putaway', 'Picking')
    GROUP BY p.ProductKey, p.SKU, p.ProductName
),
FleetIdle AS (
    SELECT 
        ft.OriginGeographyKey,
        g.LocationName AS WarehouseName,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(ft.LoadUtilizationPct) AS AvgLoadUtilization,
        SUM(ft.FuelConsumedLiters * 1.5) AS EstimatedIdleCost -- $1.50 per liter
    FROM FactFleetTrips ft
    INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -90, GETDATE())
        AND g.NodeType = 'Warehouse'
    GROUP BY ft.OriginGeographyKey, g.LocationName
)
SELECT 
    w.SKU,
    w.ProductName,
    w.AvgDwellTime,
    w.TotalVolume,
    f.WarehouseName,
    f.AvgIdleTime,
    f.AvgLoadUtilization,
    f.EstimatedIdleCost,
    -- Composite inefficiency score
    (w.AvgDwellTime * 0.4 + f.AvgIdleTime * 0.6) AS InefficiencyIndex
FROM WarehouseDwell w
CROSS JOIN FleetIdle f
WHERE w.TotalVolume > 100
ORDER BY InefficiencyIndex DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify probable congestion points in next 7 days
CREATE PROCEDURE sp_PredictBottlenecks
AS
BEGIN
    WITH HistoricalPatterns AS (
        SELECT 
            g.GeographyKey,
            g.LocationName,
            t.DayOfWeek,
            t.HourOfDay,
            AVG(f.ProcessingTimeMinutes) AS AvgProcessingTime,
            STDEV(f.ProcessingTimeMinutes) AS StdDevProcessingTime,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        INNER JOIN DimGeography g ON f.GeographyKey = g.GeographyKey
        WHERE t.DateTime >= DATEADD(DAY, -180, GETDATE())
        GROUP BY 
            g.GeographyKey,
            g.LocationName,
            t.DayOfWeek,
            t.HourOfDay
    ),
    FutureTimeSlots AS (
        SELECT 
            TimeKey,
            DateTime,
            DayOfWeek,
            HourOfDay
        FROM DimTime
        WHERE DateTime >= GETDATE()
            AND DateTime < DATEADD(DAY, 7, GETDATE())
    )
    SELECT 
        f.DateTime AS PredictedDateTime,
        h.LocationName,
        h.AvgProcessingTime,
        h.StdDevProcessingTime,
        h.OperationCount,
        -- Risk score: operations above 2 std devs
        CASE 
            WHEN h.AvgProcessingTime > (
                SELECT AVG(AvgProcessingTime) + 2 * STDEV(AvgProcessingTime)
                FROM HistoricalPatterns
            ) THEN 'High Risk'
            WHEN h.AvgProcessingTime > (
                SELECT AVG(AvgProcessingTime) + STDEV(AvgProcessingTime)
                FROM HistoricalPatterns
            ) THEN 'Medium Risk'
            ELSE 'Low Risk'
        END AS BottleneckRisk
    FROM FutureTimeSlots f
    INNER JOIN HistoricalPatterns h 
        ON f.DayOfWeek = h.DayOfWeek 
        AND f.HourOfDay = h.HourOfDay
    WHERE h.OperationCount > 10
    ORDER BY 
        f.DateTime,
        h.AvgProcessingTime DESC;
END;
GO
```

### Fleet Maintenance Priority Queue

```sql
-- Rank vehicles by revenue-impact-adjusted maintenance urgency
WITH FleetDiagnostics AS (
    SELECT 
        fl.FleetKey,
        fl.VehicleID,
        fl.VehicleType,
        fl.LastMaintenanceDate,
        DATEDIFF(DAY, fl.LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(ft.LoadUtilizationPct) AS AvgLoadUtilization,
        SUM(ft.LoadWeightKg * 0.01) AS EstimatedRevenuePerTrip -- $0.01/kg estimate
    FROM DimFleet fl
    LEFT JOIN FactFleetTrips ft ON fl.FleetKey = ft.FleetKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY 
        fl.FleetKey,
        fl.VehicleID,
        fl.VehicleType,
        fl.LastMaintenanceDate
)
SELECT 
    VehicleID,
    VehicleType,
    DaysSinceLastMaintenance,
    AvgIdleTime,
    AvgLoadUtilization,
    EstimatedRevenuePerTrip,
    -- Priority score: urgency × revenue impact
    (
        (DaysSinceLastMaintenance / 365.0 * 100) * 0.5 +
        (AvgIdleTime / 60.0 * 100) * 0.2 +
        (EstimatedRevenuePerTrip / 100.0) * 0.3
    ) AS MaintenancePriorityScore
FROM FleetDiagnostics
ORDER BY MaintenancePriorityScore DESC;
```

## Configuration & Integration

### Connecting External Data Sources

```sql
-- Example: External table for weather API data
CREATE EXTERNAL DATA SOURCE WeatherAPISource
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://STORAGE_ACCOUNT_NAME.blob.core.windows.net/weather-data',
    CREDENTIAL = AzureStorageCredential
);

CREATE EXTERNAL TABLE ExternalWeatherData (
    DateTime DATETIME2,
    LocationID VARCHAR(50),
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2),
    WindSpeed DECIMAL(5,2),
    Condition VARCHAR(50)
)
WITH (
    LOCATION = '/current/',
    DATA_SOURCE = WeatherAPISource,
    FILE_FORMAT = JSONFileFormat
);

-- Merge weather data into fleet fact table
MERGE INTO FactFleetTrips AS target
USING (
    SELECT 
        ft.TripKey,
        w.Condition AS WeatherCondition
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
    LEFT JOIN ExternalWeatherData w 
        ON t.DateTime = w.DateTime 
        AND g.NodeID = w.LocationID
    WHERE ft.WeatherCondition IS NULL
) AS source
ON target.TripKey = source.TripKey
WHEN MATCHED THEN 
    UPDATE SET target.WeatherCondition = source.WeatherCondition;
```

### Row-Level Security for Multi-Tenant Access

```sql
-- Create security roles
CREATE ROLE WarehouseManager;
CREATE ROLE FleetOperator;
CREATE ROLE ExecutiveViewer;

-- Create security policy function
CREATE FUNCTION fn_SecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessResult
WHERE 
    -- Warehouse managers see only their warehouse
    (IS_MEMBER('WarehouseManager') = 1 AND @GeographyKey IN (
        SELECT GeographyKey FROM dbo.UserGeographyAccess 
        WHERE UserName = USER_NAME()
    ))
    OR
    -- Executives see everything
    IS_MEMBER('ExecutiveViewer') = 1;
GO

-- Apply security policy to fact table
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(GeographyKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
```

### Automated Alert Configuration

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    MetricName VARCHAR(100),
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '='
    AlertRecipients VARCHAR(500), -- Email addresses
    IsActive BIT DEFAULT 1
);

-- Insert sample thresholds
INSERT INTO AlertThresholds (MetricName, ThresholdValue, ComparisonOperator, AlertRecipients)
VALUES 
    ('FleetIdleTimePercent', 15.0, '>', 'fleet-manager@example.com'),
    ('WarehouseDwellTimeHours', 72.0, '>', 'warehouse-ops@example.com'),
    ('OnTimeDeliveryPercent', 90.0, '<', 'logistics-director@example.com');

-- Stored procedure to check alerts and send notifications
CREATE PROCEDURE sp_CheckAndTriggerAlerts
AS
BEGIN
    DECLARE @MetricName VARCHAR(100);
    DECLARE @ThresholdValue DECIMAL(10,2);
    DECLARE @ComparisonOperator VARCHAR(10);
    DECLARE @Recipients VARCHAR(500);
    DECLARE @ActualValue DECIMAL(10,2);
    DECLARE @AlertMessage VARCHAR(2000);
    
    DECLARE alert_cursor CURSOR FOR
    SELECT MetricName, ThresholdValue, ComparisonOperator, AlertRecipients
    FROM AlertThresholds
    WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @MetricName, @ThresholdValue, @ComparisonOperator, @Recipients;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Check FleetIdleTimePercent
        IF @MetricName = 'FleetIdleTimePercent'
        BEGIN
            SELECT @ActualValue = AVG((IdleTimeMinutes / NULLIF(TripDurationMinutes, 0)) * 100)
            FROM FactFleetTrips ft
            INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
            WHERE t.DateTime >= DATEADD(HOUR, -24, GETDATE());
            
            IF (@ComparisonOperator = '>' AND @ActualValue > @ThresholdValue)
            BEGIN
                SET @AlertMessage = 'ALERT: Fleet idle time is ' + CAST(@ActualValue AS VARCHAR) + 
                    '%, exceeding threshold of ' + CAST(@ThresholdValue AS VARCHAR) + '%';
                
                EXEC msdb.dbo.sp_send_dbmail
                    @profile_name = 'LogiFleetAlerts',
                    @recipients = @Recipients,
                    @subject = 'LogiFleet Pulse - Fleet Idle Time Alert',
                    @body = @AlertMessage;
            END
        END
        
        FETCH NEXT FROM alert_cursor INTO @MetricName, @ThresholdValue, @ComparisonOperator, @Recipients;
    END
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END;
GO

-- Schedule via SQL Agent Job (run every 15 minutes)
```

## Power BI DAX Measures

```dax
// Total Warehouse Throughput
TotalThroughput = 
SUM(FactWarehouseOperations[QuantityProcessed])

// Average Dwell Time with Filter Context
AvgDwellTime = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    FactWarehouseOperations[OperationType] IN {"Putaway", "Picking"}
)

// Fleet Utilization Rate
FleetUtilization = 
DIVIDE(
    AVERAGE(FactFleetTrips[LoadUtilizationPct]),
    100,
    0
)

// On-Time Delivery Rate
OnTimeDeliveryRate = 
DIVIDE(
    CALCULATE(COUNT(FactFleetTrips[TripKey]), FactFleetTrips[OnTimeDelivery] = TRUE()),
    COUNT(FactFleetTrips[TripKey]),
    0
)

// Cross-Fact Efficiency Score
EfficiencyScore = 
VAR WarehouseEfficiency = 
    DIVIDE(
        [TotalThroughput],
        SUM(FactWarehouseOperations[ProcessingTimeMinutes]),
        0
    )
VAR FleetEfficiency = 
    1 - DIVIDE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        AVERAGE(FactFleetTrips[TripDurationMinutes]),
        1
    )
RETURN
    (WarehouseEfficiency * 0.5 + FleetEfficiency * 0.5) * 100

// Time Intelligence: Same Period Last Year
ThroughputSamePeriodLastYear = 
CALCULATE(
    [TotalThroughput],
    SAMEPERIODLASTYEAR(DimTime[DateTime])
)

// Bottleneck Risk Indicator
BottleneckRisk = 
VAR AvgProcessingTime = AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes])
VAR StdDev = STDEV.P(FactWarehouseOperations[ProcessingTimeMinutes])
VAR Threshold = AvgProcessingTime + (2 * StdDev)
RETURN
    IF(
        AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes]) > Threshold,
        "High Risk",
        IF(
            AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes]) > (AvgProcessingTime + StdDev),
            "Medium Risk",
            "Low Risk"
        )
    )
```

## Common Patterns

### Pattern 1: Daily Data Refresh Workflow

```sql
-- Execute in sequence via SQL Agent Job
EXEC sp_UpdateProductGravity;
EXEC sp_LoadWarehouseOperations @LastLoadDateTime = '2026-07-10 00:00:00';
EXEC sp_CheckAndTriggerAlerts;

-- Refresh Power BI dataset via REST API (PowerShell)
-- $token = Get-PowerBIAccessToken
-- Invoke-PowerBIRestMethod -Url "https://api.powerbi.com/v1.0/myorg/datasets/{DATASET_ID}/refreshes" -Method POST -Headers @{Authorization = $token}
```

### Pattern 2: Warehouse Zone Optimization Analysis

```sql
-- Identify products that should be moved to higher gravity zones
WITH ProductPerformance AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityZone AS CurrentZone,
        f.StorageZone,
        COUNT(*) AS PickCount,
        AVG(f.ProcessingTimeMinutes) AS AvgPickTime,
        AVG(f.DwellTimeHours) AS AvgDwellTime
    FROM FactWarehouseOperations f
    INNER JOIN DimProduct p ON f.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE f.OperationType = 'Picking'
        AND t.DateTime >= DATEADD(DAY, -60, GETDATE())
    GROUP BY 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityZone,
        f.StorageZone
)
SELECT 
    SKU,
    ProductName,
    CurrentZone,
    StorageZone,
    PickCount,
    AvgPickTime,
    AvgDwellTime
