---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain dashboard"
  - "configure Power BI logistics data warehouse"
  - "create multi-fact star schema for warehouse analytics"
  - "implement fleet telemetry tracking with SQL Server"
  - "build cross-modal supply chain KPI dashboard"
  - "integrate warehouse and fleet data models"
  - "deploy LogiFleet Pulse analytics engine"
  - "optimize logistics data modeling with temporal dimensions"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers implement **LogiFleet Pulse**, an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain metrics into a single multi-fact star schema data warehouse using MS SQL Server and Power BI.

## What LogiFleet Pulse Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that:

- Fuses warehouse micro-operations, fleet telemetry, inventory aging, and macroeconomic signals
- Implements a custom multi-fact star schema with time-phased dimensions
- Provides cross-fact KPI harmonization (e.g., dwell time per SKU vs. fleet idling cost per route)
- Enables predictive bottleneck detection and adaptive fleet triage
- Delivers real-time Power BI dashboards with 15-minute refresh cycles
- Supports role-based access and multilingual interfaces

**Key Innovation**: Uses composite shared dimensions to mathematically link inventory turnover with fuel burn rates, warehouse gravity zones for spatial optimization, and temporal elasticity modeling for predictive scenarios.

## Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version recommended)
- Access to data sources: WMS, telemetry APIs, ERP feeds
- SQL Server Management Studio (SSMS) for schema deployment

## Installation & Setup

### 1. Deploy SQL Schema

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfWeek INT NOT NULL,
    Hour INT NOT NULL,
    Minute INT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0
);

-- Create clustered columnstore index for time dimension
CREATE CLUSTERED COLUMNSTORE INDEX IX_DimTime_CCS ON DimTime;
GO

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    ContinentName NVARCHAR(50) NOT NULL,
    CountryName NVARCHAR(100) NOT NULL,
    RegionName NVARCHAR(100) NOT NULL,
    CityName NVARCHAR(100) NOT NULL,
    WarehouseNodeID NVARCHAR(50),
    RouteNodeID NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZoneOffset INT,
    IsActive BIT DEFAULT 1
);

CREATE NONCLUSTERED INDEX IX_DimGeography_Country 
ON DimGeography(CountryName) INCLUDE (RegionName, CityName);
GO

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(255) NOT NULL,
    Category NVARCHAR(100),
    Subcategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite score: velocity + value + fragility
    PickFrequency INT DEFAULT 0,
    AverageValue DECIMAL(10,2),
    FragilityIndex DECIMAL(3,2), -- 0.00 to 1.00
    OptimalStorageZone NVARCHAR(50),
    LastGravityUpdate DATETIME2
);

CREATE NONCLUSTERED INDEX IX_DimProductGravity_Gravity 
ON DimProductGravity(GravityScore DESC) INCLUDE (OptimalStorageZone);
GO

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(255) NOT NULL,
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0.00 to 100.00
    LastReliabilityUpdate DATETIME2
);
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseGeographyKey INT NOT NULL,
    SupplierKey INT,
    OperationType NVARCHAR(20) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT,
    LabourMinutes DECIMAL(10,2),
    OperationCost DECIMAL(10,2),
    StorageZone NVARCHAR(50),
    BatchNumber NVARCHAR(50),
    CONSTRAINT FK_FactWH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_FactWH_Geography FOREIGN KEY (WarehouseGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWH_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
);

-- Partition by month for better query performance
CREATE PARTITION FUNCTION PF_FactWH_Monthly (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, 202604, 202605, 202606, 202607, 202608, 202609, 202610, 202611, 202612);

CREATE PARTITION SCHEME PS_FactWH_Monthly
AS PARTITION PF_FactWH_Monthly ALL TO ([PRIMARY]);

-- Create clustered index on partitioned scheme
CREATE CLUSTERED INDEX IX_FactWH_TimeKey 
ON FactWarehouseOperations(TimeKey)
ON PS_FactWH_Monthly(TimeKey);
GO

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    RouteSegmentID NVARCHAR(100),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TotalTripMinutes INT,
    FuelCost DECIMAL(10,2),
    MaintenanceAlerts INT DEFAULT 0,
    WeatherDelayMinutes INT DEFAULT 0,
    TrafficDelayMinutes INT DEFAULT 0,
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE CLUSTERED INDEX IX_FactFleet_TimeKey 
ON FactFleetTrips(TimeKey);

CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle 
ON FactFleetTrips(VehicleID) INCLUDE (TimeKey, FuelConsumedLiters, IdleTimeMinutes);
GO

-- Create cross-dock fact table (transfers without long-term storage)
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripID BIGINT,
    OutboundTripID BIGINT,
    TransferTimeMinutes INT,
    QuantityTransferred INT,
    CONSTRAINT FK_FactCD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactCD_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

CREATE CLUSTERED INDEX IX_FactCD_TimeKey 
ON FactCrossDock(TimeKey);
GO
```

### 2. Populate Time Dimension

```sql
-- Stored procedure to populate time dimension
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE = '2026-01-01',
    @EndDate DATE = '2028-12-31'
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @EndDateTime DATETIME2 = DATEADD(DAY, 1, @EndDate);
    
    WHILE @CurrentDateTime < @EndDateTime
    BEGIN
        INSERT INTO DimTime (
            TimeKey, FullDateTime, Date, Year, Quarter, Month, Week, 
            DayOfWeek, Hour, Minute, FiscalYear, FiscalQuarter, IsWeekend
        )
        VALUES (
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime),
            -- Fiscal year starts in April
            CASE WHEN MONTH(@CurrentDateTime) >= 4 
                THEN YEAR(@CurrentDateTime) 
                ELSE YEAR(@CurrentDateTime) - 1 END,
            CASE WHEN MONTH(@CurrentDateTime) >= 4 
                THEN DATEPART(QUARTER, @CurrentDateTime) 
                ELSE DATEPART(QUARTER, @CurrentDateTime) + 3 END,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) 
                THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute population
EXEC PopulateTimeDimension;
GO
```

### 3. Create Cross-Fact KPI View

```sql
-- Unified view for cross-fact analysis
CREATE VIEW vw_CrossFactLogisticsKPI
AS
SELECT 
    dt.Date,
    dt.Year,
    dt.Quarter,
    dt.Month,
    dg.CountryName,
    dg.RegionName,
    dp.Category AS ProductCategory,
    dp.GravityScore,
    
    -- Warehouse KPIs
    SUM(fwh.QuantityHandled) AS TotalWarehouseVolume,
    AVG(fwh.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    SUM(fwh.OperationCost) AS TotalWarehouseCost,
    
    -- Fleet KPIs
    SUM(fft.DistanceKM) AS TotalFleetDistanceKM,
    SUM(fft.FuelConsumedLiters) AS TotalFuelConsumed,
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    SUM(fft.FuelCost) AS TotalFuelCost,
    
    -- Cross-fact composite KPI: Cost per unit moved per km
    CASE 
        WHEN SUM(fft.DistanceKM) > 0 
        THEN (SUM(fwh.OperationCost) + SUM(fft.FuelCost)) / 
             (SUM(fwh.QuantityHandled) * SUM(fft.DistanceKM))
        ELSE 0 
    END AS CostPerUnitKM

FROM FactWarehouseOperations fwh
INNER JOIN DimTime dt ON fwh.TimeKey = dt.TimeKey
INNER JOIN DimProductGravity dp ON fwh.ProductKey = dp.ProductKey
INNER JOIN DimGeography dg ON fwh.WarehouseGeographyKey = dg.GeographyKey
LEFT JOIN FactFleetTrips fft ON dt.TimeKey = fft.TimeKey 
    AND dg.GeographyKey = fft.OriginGeographyKey

GROUP BY 
    dt.Date, dt.Year, dt.Quarter, dt.Month,
    dg.CountryName, dg.RegionName,
    dp.Category, dp.GravityScore;
GO
```

## Power BI Configuration

### Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server name and database: `LogiFleetPulse`
4. Select DirectQuery or Import mode (DirectQuery for real-time, Import for performance)

### Power BI DAX Measures

```dax
// Total Warehouse Operations
TotalOperations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time with Context
AvgDwellTime = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    FactWarehouseOperations[DwellTimeMinutes] > 0
)

// Fleet Efficiency Score (0-100)
FleetEfficiencyScore = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKM])
VAR TotalIdle = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalTrip = SUM(FactFleetTrips[TotalTripMinutes])
VAR IdlePercentage = DIVIDE(TotalIdle, TotalTrip, 0)
RETURN
    (1 - IdlePercentage) * 100

// Warehouse Gravity Score Compliance
GravityCompliance = 
VAR ActualZone = FactWarehouseOperations[StorageZone]
VAR OptimalZone = RELATED(DimProductGravity[OptimalStorageZone])
RETURN
    DIVIDE(
        COUNTROWS(
            FILTER(
                FactWarehouseOperations,
                ActualZone = OptimalZone
            )
        ),
        COUNTROWS(FactWarehouseOperations),
        0
    ) * 100

// Predictive Bottleneck Index (higher = more risk)
BottleneckIndex = 
VAR CurrentDwell = [AvgDwellTime]
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -30, DAY)
    )
VAR CurrentFleetIdle = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[TotalTripMinutes])
    )
RETURN
    (CurrentDwell / HistoricalAvg) * 50 + (CurrentFleetIdle * 100) * 50

// Cost Per Unit Shipped
CostPerUnit = 
DIVIDE(
    SUM(FactWarehouseOperations[OperationCost]) + SUM(FactFleetTrips[FuelCost]),
    SUM(FactWarehouseOperations[QuantityHandled]),
    0
)
```

### Power BI Visualization Best Practices

```dax
// Time intelligence for YoY comparison
SamePeriodLastYear = 
CALCULATE(
    [TotalOperations],
    SAMEPERIODLASTYEAR(DimTime[Date])
)

YoYGrowth = 
DIVIDE(
    [TotalOperations] - [SamePeriodLastYear],
    [SamePeriodLastYear],
    0
) * 100

// Dynamic ranking for top routes
RouteRank = 
RANKX(
    ALL(FactFleetTrips[RouteSegmentID]),
    [TotalFleetDistanceKM],
    ,
    DESC,
    DENSE
)

// Alert threshold flag
IsHighDwellTime = 
IF(
    [AvgDwellTime] > 72, // 72 hours threshold
    "🔴 Critical",
    IF(
        [AvgDwellTime] > 48,
        "🟡 Warning",
        "🟢 Normal"
    )
)
```

## Data Integration Patterns

### Load Warehouse Operations from WMS

```sql
-- Stored procedure for incremental ETL
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseGeographyKey, SupplierKey,
        OperationType, QuantityHandled, DwellTimeMinutes, 
        LabourMinutes, OperationCost, StorageZone, BatchNumber
    )
    SELECT 
        CAST(FORMAT(wms.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        dp.ProductKey,
        dg.GeographyKey,
        ds.SupplierKey,
        wms.OperationType,
        wms.Quantity,
        DATEDIFF(MINUTE, wms.ArrivalDateTime, wms.OperationDateTime) AS DwellTimeMinutes,
        wms.LabourHours * 60 AS LabourMinutes,
        wms.TotalCost,
        wms.StorageZone,
        wms.BatchNumber
    FROM 
        OPENQUERY(
            WMS_LinkedServer, 
            'SELECT * FROM operations WHERE last_updated > ''' + 
            CONVERT(VARCHAR(23), @LastLoadDateTime, 121) + ''''
        ) AS wms
    INNER JOIN DimProductGravity dp ON wms.SKU = dp.SKU
    INNER JOIN DimGeography dg ON wms.WarehouseID = dg.WarehouseNodeID
    LEFT JOIN DimSupplierReliability ds ON wms.SupplierID = ds.SupplierID;
    
    -- Update last load timestamp
    UPDATE ETL_Control 
    SET LastLoadDateTime = GETDATE() 
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Load Fleet Telemetry from API

```sql
-- External table for streaming telemetry (requires PolyBase)
CREATE EXTERNAL DATA SOURCE TelemetryAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://telemetry-api.example.com/feed',
    CREDENTIAL = TelemetryAPICredential
);

-- Create external file format
CREATE EXTERNAL FILE FORMAT JSONFormat
WITH (FORMAT_TYPE = JSON);

-- External table pointing to API feed
CREATE EXTERNAL TABLE ext_FleetTelemetry (
    VehicleID NVARCHAR(50),
    Timestamp DATETIME2,
    OriginLocation NVARCHAR(100),
    DestinationLocation NVARCHAR(100),
    DistanceKM DECIMAL(10,2),
    FuelUsed DECIMAL(10,2),
    IdleMinutes INT,
    LoadingMinutes INT,
    UnloadingMinutes INT
)
WITH (
    DATA_SOURCE = TelemetryAPI,
    FILE_FORMAT = JSONFormat,
    LOCATION = '/stream'
);

-- Load into fact table
INSERT INTO FactFleetTrips (
    TimeKey, VehicleID, OriginGeographyKey, DestinationGeographyKey,
    DistanceKM, FuelConsumedLiters, IdleTimeMinutes, 
    LoadingTimeMinutes, UnloadingTimeMinutes, TotalTripMinutes
)
SELECT 
    CAST(FORMAT(t.Timestamp, 'yyyyMMddHHmm') AS INT),
    t.VehicleID,
    og.GeographyKey,
    dg.GeographyKey,
    t.DistanceKM,
    t.FuelUsed,
    t.IdleMinutes,
    t.LoadingMinutes,
    t.UnloadingMinutes,
    t.IdleMinutes + t.LoadingMinutes + t.UnloadingMinutes + 
        (t.DistanceKM / 60) -- Approximate drive time
FROM ext_FleetTelemetry t
INNER JOIN DimGeography og ON t.OriginLocation = og.CityName
INNER JOIN DimGeography dg ON t.DestinationLocation = dg.CityName
WHERE NOT EXISTS (
    SELECT 1 FROM FactFleetTrips fft
    WHERE fft.VehicleID = t.VehicleID 
    AND fft.TimeKey = CAST(FORMAT(t.Timestamp, 'yyyyMMddHHmm') AS INT)
);
```

## Automated Alerting System

```sql
-- Alert configuration table
CREATE TABLE AlertThresholds (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName NVARCHAR(100) NOT NULL,
    MetricName NVARCHAR(100) NOT NULL,
    ThresholdValue DECIMAL(10,2) NOT NULL,
    ComparisonOperator NVARCHAR(10) NOT NULL, -- >, <, =, >=, <=
    AlertRecipients NVARCHAR(MAX), -- JSON array of emails
    IsActive BIT DEFAULT 1
);

-- Insert sample alerts
INSERT INTO AlertThresholds (AlertName, MetricName, ThresholdValue, ComparisonOperator, AlertRecipients)
VALUES 
    ('High Dwell Time', 'AvgDwellTimeMinutes', 72, '>', '["logistics@company.com", "warehouse@company.com"]'),
    ('Excessive Fleet Idle', 'AvgIdlePercentage', 15, '>', '["fleet@company.com"]'),
    ('Low Gravity Compliance', 'GravityCompliancePercent', 80, '<', '["operations@company.com"]');

-- Scheduled job to check thresholds (SQL Server Agent)
CREATE PROCEDURE CheckAlertThresholds
AS
BEGIN
    DECLARE @AlertID INT, @AlertName NVARCHAR(100), @MetricName NVARCHAR(100);
    DECLARE @ThresholdValue DECIMAL(10,2), @ComparisonOperator NVARCHAR(10);
    DECLARE @Recipients NVARCHAR(MAX);
    DECLARE @CurrentValue DECIMAL(10,2);
    DECLARE @BreachDetected BIT;
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, AlertName, MetricName, ThresholdValue, ComparisonOperator, AlertRecipients
    FROM AlertThresholds WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @AlertID, @AlertName, @MetricName, @ThresholdValue, @ComparisonOperator, @Recipients;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Calculate current metric value
        SET @CurrentValue = CASE @MetricName
            WHEN 'AvgDwellTimeMinutes' THEN (
                SELECT AVG(DwellTimeMinutes) 
                FROM FactWarehouseOperations fwh
                INNER JOIN DimTime dt ON fwh.TimeKey = dt.TimeKey
                WHERE dt.Date >= DATEADD(DAY, -1, GETDATE())
            )
            WHEN 'AvgIdlePercentage' THEN (
                SELECT AVG(CAST(IdleTimeMinutes AS DECIMAL) / TotalTripMinutes) * 100
                FROM FactFleetTrips fft
                INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
                WHERE dt.Date >= DATEADD(DAY, -1, GETDATE())
            )
            WHEN 'GravityCompliancePercent' THEN (
                SELECT CAST(COUNT(CASE WHEN fwh.StorageZone = dp.OptimalStorageZone THEN 1 END) AS DECIMAL) / COUNT(*) * 100
                FROM FactWarehouseOperations fwh
                INNER JOIN DimProductGravity dp ON fwh.ProductKey = dp.ProductKey
                INNER JOIN DimTime dt ON fwh.TimeKey = dt.TimeKey
                WHERE dt.Date >= DATEADD(DAY, -1, GETDATE())
            )
        END;
        
        -- Check threshold breach
        SET @BreachDetected = CASE @ComparisonOperator
            WHEN '>' THEN CASE WHEN @CurrentValue > @ThresholdValue THEN 1 ELSE 0 END
            WHEN '<' THEN CASE WHEN @CurrentValue < @ThresholdValue THEN 1 ELSE 0 END
            WHEN '>=' THEN CASE WHEN @CurrentValue >= @ThresholdValue THEN 1 ELSE 0 END
            WHEN '<=' THEN CASE WHEN @CurrentValue <= @ThresholdValue THEN 1 ELSE 0 END
            WHEN '=' THEN CASE WHEN @CurrentValue = @ThresholdValue THEN 1 ELSE 0 END
        END;
        
        -- Send alert if breached
        IF @BreachDetected = 1
        BEGIN
            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = 'LogiFleet Alerts',
                @recipients = @Recipients,
                @subject = @AlertName,
                @body = CONCAT(
                    'Alert: ', @AlertName, CHAR(13), CHAR(10),
                    'Metric: ', @MetricName, CHAR(13), CHAR(10),
                    'Current Value: ', @CurrentValue, CHAR(13), CHAR(10),
                    'Threshold: ', @ComparisonOperator, ' ', @ThresholdValue, CHAR(13), CHAR(10),
                    'Timestamp: ', GETDATE()
                );
        END;
        
        FETCH NEXT FROM alert_cursor INTO @AlertID, @AlertName, @MetricName, @ThresholdValue, @ComparisonOperator, @Recipients;
    END;
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END;
GO
```

## Warehouse Gravity Zone Optimization

```sql
-- Calculate and update gravity scores
CREATE PROCEDURE UpdateProductGravityScores
AS
BEGIN
    UPDATE DimProductGravity
    SET 
        PickFrequency = stats.PickCount,
        AverageValue = stats.AvgValue,
        GravityScore = (
            -- Weighted formula: 50% frequency, 30% value, 20% fragility
            (CAST(stats.PickCount AS DECIMAL) / stats.MaxPicks * 50) +
            (stats.AvgValue / stats.MaxValue * 30) +
            ((1 - DimProductGravity.FragilityIndex) * 20)
        ),
        OptimalStorageZone = CASE
            WHEN (
                (CAST(stats.PickCount AS DECIMAL) / stats.MaxPicks * 50) +
                (stats.AvgValue / stats.MaxValue * 30) +
                ((1 - DimProductGravity.FragilityIndex) * 20)
            ) >= 80 THEN 'HIGH-GRAVITY-A'
            WHEN (
                (CAST(stats.PickCount AS DECIMAL) / stats.MaxPicks * 50) +
                (stats.AvgValue / stats.MaxValue * 30) +
                ((1 - DimProductGravity.FragilityIndex) * 20)
            ) >= 60 THEN 'MEDIUM-GRAVITY-B'
            ELSE 'LOW-GRAVITY-C'
        END,
        LastGravityUpdate = GETDATE()
    FROM DimProductGravity
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            AVG(OperationCost / NULLIF(QuantityHandled, 0)) AS AvgValue,
            MAX(COUNT(*)) OVER () AS MaxPicks,
            MAX(AVG(OperationCost / NULLIF(QuantityHandled, 0))) OVER () AS MaxValue
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
        AND TimeKey >= CAST(FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY ProductKey
    ) AS stats ON DimProductGravity.ProductKey = stats.ProductKey;
END;
GO

-- Schedule to run weekly
```

## Row-Level Security Implementation

```sql
-- Create security policy
CREATE TABLE UserRegionAccess (
    UserEmail NVARCHAR(255) PRIMARY KEY,
    AllowedRegions NVARCHAR(MAX) -- Comma-separated list
);

-- Insert sample access rules
INSERT INTO UserRegionAccess VALUES 
    ('manager.na@company.com', 'North America'),
    ('manager.eu@company.com', 'Europe,Asia'),
    ('executive@company.com', NULL); -- NULL = all regions

-- Create security predicate function
CREATE FUNCTION fn_SecurityPredicate(@RegionName NVARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS result
    WHERE 
        EXISTS (
            SELECT 1 FROM dbo.UserRegionAccess
            WHERE UserEmail = USER_NAME()
            AND (AllowedRegions IS NULL OR @RegionName IN (
                SELECT value FROM STRING_SPLIT(AllowedRegions, ',')
            ))
        )
        OR IS_MEMBER('db_owner') = 1
);
GO

-- Apply security policy to geography dimension
CREATE SECURITY POLICY RegionAccessPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(RegionName)
ON dbo.DimGeography
WITH (STATE = ON);
GO
```

## Common Troubleshooting

### Issue: Power BI Dashboard Slow Refresh

**Solution**: Switch from Import to DirectQuery mode, or create aggregation tables
