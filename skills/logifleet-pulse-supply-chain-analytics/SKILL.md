---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing template for multi-modal logistics intelligence with fleet, warehouse, and supply chain KPI harmonization
triggers:
  - "set up logistics analytics dashboard"
  - "configure supply chain data warehouse"
  - "implement fleet and warehouse KPI tracking"
  - "deploy LogiFleet Pulse analytics"
  - "create multi-fact star schema for logistics"
  - "build Power BI supply chain dashboard"
  - "integrate warehouse and fleet telemetry data"
  - "configure cross-modal supply chain analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics template for supply chain operations. It provides a multi-fact star schema combining warehouse operations, fleet telemetry, and inventory management into a unified semantic layer. The platform uses MS SQL Server for data storage and Power BI for visualization, enabling cross-functional KPI analysis across logistics operations.

**Core Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Real-time warehouse and fleet operations tracking
- Predictive bottleneck detection
- Cross-fact KPI harmonization (e.g., inventory dwell time vs. fleet idle costs)
- Role-based access control and row-level security
- Adaptive visual dashboards with 15-minute refresh intervals

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQLCMD or SQL Server Management Studio (SSMS)
- Access to data sources (WMS, TMS, telemetry APIs)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script

USE master;
GO

-- Create the LogiFleet database
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = 'LogiFleetPulse')
BEGIN
    CREATE DATABASE LogiFleetPulse;
END
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    TimeSlot15Min VARCHAR(5),
    HourOfDay TINYINT,
    DayOfWeek VARCHAR(10),
    WeekOfYear TINYINT,
    MonthName VARCHAR(10),
    Quarter TINYINT,
    FiscalYear SMALLINT,
    IsWeekend BIT,
    IsHoliday BIT
);

CREATE INDEX IX_DimTime_DateTime ON DimTime(FullDateTime);
CREATE INDEX IX_DimTime_FiscalYear_Quarter ON DimTime(FiscalYear, Quarter);

-- Geography dimension with hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(20) UNIQUE NOT NULL,
    LocationName VARCHAR(100),
    LocationType VARCHAR(20), -- 'Warehouse', 'Route Node', 'CrossDock'
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    EffectiveDate DATE,
    ExpiryDate DATE
);

CREATE INDEX IX_DimGeography_LocationType ON DimGeography(LocationType);
CREATE INDEX IX_DimGeography_Country_City ON DimGeography(Country, City);

-- Product with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    IsPerishable BIT,
    ShelfLifeDays SMALLINT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity + value + fragility
    RecommendedZoneType VARCHAR(50),
    LastGravityCalculation DATETIME2
);

CREATE INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC);
CREATE INDEX IX_DimProductGravity_Category ON DimProductGravity(Category);

-- Supplier reliability tracking
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,3),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    LastEvaluationDate DATE
);

-- Fact table: Warehouse operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    DurationMinutes DECIMAL(10,2),
    Quantity INT,
    DwellTimeHours DECIMAL(10,2),
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50),
    BatchNumber VARCHAR(50),
    OrderID VARCHAR(50),
    CONSTRAINT FK_FactWH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_FactWH_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
);

CREATE INDEX IX_FactWH_TimeGeo ON FactWarehouseOperations(TimeKey, GeographyKey);
CREATE INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactWH_OperationType ON FactWarehouseOperations(OperationType);

-- Fact table: Fleet trips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelLiters DECIMAL(10,3),
    FuelCost DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    DeliveryStatus VARCHAR(20), -- 'OnTime', 'Delayed', 'Failed'
    DelayReasonCode VARCHAR(50),
    WeatherCondition VARCHAR(50),
    TrafficCondition VARCHAR(50),
    MaintenanceFlag BIT,
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE INDEX IX_FactFleet_TimeOrigin ON FactFleetTrips(TimeKey, OriginGeographyKey);
CREATE INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);
CREATE INDEX IX_FactFleet_DeliveryStatus ON FactFleetTrips(DeliveryStatus);

-- Bridge table for many-to-many: Routes to Storage Zones
CREATE TABLE BridgeRouteStorageZone (
    BridgeKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripKey BIGINT NOT NULL,
    OperationKey BIGINT NOT NULL,
    LinkageType VARCHAR(20), -- 'Pickup', 'Delivery'
    LinkageTimestamp DATETIME2,
    CONSTRAINT FK_Bridge_Trip FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_Bridge_Operation FOREIGN KEY (OperationKey) REFERENCES FactWarehouseOperations(OperationKey)
);
```

### Step 2: Configure Data Ingestion

Create stored procedures for incremental data loading:

```sql
-- Stored procedure: Incremental load from WMS
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if no parameter provided
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -24, GETDATE());
    
    -- Insert new warehouse operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OperationStartTime, OperationEndTime,
        DurationMinutes, Quantity, DwellTimeHours,
        StorageZone, OperatorID, BatchNumber, OrderID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.StartTime,
        stg.EndTime,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime),
        stg.Quantity,
        stg.DwellTimeHours,
        stg.Zone,
        stg.OperatorID,
        stg.BatchNumber,
        stg.OrderID
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime t ON CAST(stg.StartTime AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, stg.StartTime) = t.HourOfDay
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON stg.SupplierCode = s.SupplierCode
    WHERE stg.LoadTimestamp > @LastLoadDateTime;
    
    -- Update gravity scores based on recent velocity
    EXEC usp_RecalculateGravityScores;
END;
GO

-- Stored procedure: Calculate product gravity scores
CREATE PROCEDURE usp_RecalculateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH RecentVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) as OperationCount,
            SUM(Quantity) as TotalQuantity,
            AVG(DwellTimeHours) as AvgDwellTime
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 28 days
        GROUP BY ProductKey
    ),
    GravityCalc AS (
        SELECT 
            p.ProductKey,
            -- Gravity formula: (velocity_score * 40) + (value_score * 30) + (urgency_score * 30)
            (
                (CASE 
                    WHEN rv.OperationCount > 100 THEN 10
                    WHEN rv.OperationCount > 50 THEN 7
                    WHEN rv.OperationCount > 20 THEN 4
                    ELSE 1
                END * 4.0) +
                (CASE 
                    WHEN p.IsPerishable = 1 THEN 9
                    WHEN p.IsFragile = 1 THEN 6
                    ELSE 3
                END * 3.0) +
                (CASE 
                    WHEN rv.AvgDwellTime < 24 THEN 10
                    WHEN rv.AvgDwellTime < 72 THEN 6
                    ELSE 2
                END * 3.0)
            ) AS NewGravityScore
        FROM DimProductGravity p
        LEFT JOIN RecentVelocity rv ON p.ProductKey = rv.ProductKey
    )
    UPDATE p
    SET 
        p.GravityScore = gc.NewGravityScore,
        p.RecommendedZoneType = CASE 
            WHEN gc.NewGravityScore >= 80 THEN 'High-Gravity-Express'
            WHEN gc.NewGravityScore >= 50 THEN 'Medium-Gravity-Standard'
            ELSE 'Low-Gravity-Bulk'
        END,
        p.LastGravityCalculation = GETDATE()
    FROM DimProductGravity p
    INNER JOIN GravityCalc gc ON p.ProductKey = gc.ProductKey;
END;
GO
```

### Step 3: Set Up Automated Alerts

```sql
-- Alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(100),
    MetricType VARCHAR(50), -- 'FleetIdle', 'DwellTime', 'DeliveryDelay'
    ThresholdValue DECIMAL(10,2),
    ThresholdOperator VARCHAR(10), -- '>', '<', '='
    AlertRecipients VARCHAR(500), -- Comma-separated emails
    IsActive BIT DEFAULT 1,
    LastTriggered DATETIME2
);

-- Stored procedure: Check alerts and send notifications
CREATE PROCEDURE usp_CheckAndTriggerAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @AlertRecipients VARCHAR(500);
    
    -- Check fleet idle time alerts
    DECLARE @HighIdleVehicles TABLE (VehicleID VARCHAR(50), IdlePercent DECIMAL(5,2));
    
    INSERT INTO @HighIdleVehicles
    SELECT 
        VehicleID,
        (SUM(IdleTimeMinutes) / NULLIF(SUM(DurationMinutes), 0)) * 100 AS IdlePercent
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
    GROUP BY VehicleID
    HAVING (SUM(IdleTimeMinutes) / NULLIF(SUM(DurationMinutes), 0)) * 100 > 15;
    
    IF EXISTS (SELECT 1 FROM @HighIdleVehicles)
    BEGIN
        SELECT @AlertRecipients = AlertRecipients 
        FROM AlertConfiguration 
        WHERE MetricType = 'FleetIdle' AND IsActive = 1;
        
        SET @AlertMessage = 'ALERT: High fleet idle time detected. Vehicles: ' + 
            (SELECT STRING_AGG(VehicleID + ' (' + CAST(IdlePercent AS VARCHAR) + '%)', ', ')
             FROM @HighIdleVehicles);
        
        -- Send email via Database Mail (configure separately)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @AlertRecipients,
            @subject = 'LogiFleet Pulse: High Idle Time Alert',
            @body = @AlertMessage;
            
        UPDATE AlertConfiguration 
        SET LastTriggered = GETDATE() 
        WHERE MetricType = 'FleetIdle';
    END;
END;
GO

-- Schedule this procedure to run every 15 minutes via SQL Server Agent
```

## Power BI Configuration

### Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `YOUR_SQL_SERVER_INSTANCE`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for performance)

### Create Key Measures (DAX)

```dax
// Measure: Total Warehouse Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Measure: Average Dwell Time
Avg Dwell Time (Hours) = 
AVERAGE(FactWarehouseOperations[DwellTimeHours])

// Measure: Fleet Idle Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Measure: Cross-Fact KPI - Dwell Cost Impact
Dwell Cost Impact = 
VAR HighDwellOps = 
    FILTER(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeHours] > 72
    )
VAR LinkedTrips = 
    CALCULATETABLE(
        FactFleetTrips,
        TREATAS(
            SELECTCOLUMNS(
                RELATEDTABLE(BridgeRouteStorageZone),
                "TripKey", BridgeRouteStorageZone[TripKey]
            ),
            FactFleetTrips[TripKey]
        )
    )
RETURN
    SUMX(LinkedTrips, LinkedTrips[FuelCost] * (LinkedTrips[IdleTimeMinutes] / 60))

// Measure: On-Time Delivery Rate
On-Time Delivery % = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[DeliveryStatus] = "OnTime"
    ),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Measure: Gravity Zone Compliance
Gravity Zone Compliance % = 
VAR ActualPlacements = 
    ADDCOLUMNS(
        FactWarehouseOperations,
        "IsCompliant",
        IF(
            RELATED(DimProductGravity[RecommendedZoneType]) = FactWarehouseOperations[StorageZone],
            1,
            0
        )
    )
RETURN
    DIVIDE(
        SUMX(ActualPlacements, [IsCompliant]),
        COUNTROWS(FactWarehouseOperations),
        0
    ) * 100

// Measure: Predictive Bottleneck Score
Bottleneck Risk Score = 
VAR CurrentCapacity = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] IN {"Putaway", "Picking"}
    )
VAR HistoricalAvg = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DATESINPERIOD(
            DimTime[FullDateTime],
            MAX(DimTime[FullDateTime]),
            -28,
            DAY
        )
    ) / 28
VAR FleetDelayRate = [Fleet Idle %] / 100
RETURN
    (CurrentCapacity / HistoricalAvg) * 50 + FleetDelayRate * 50
```

### Create Time Intelligence Measures

```dax
// Measure: Operations vs. Last Period
Operations vs Last Period = 
VAR CurrentPeriod = [Total Operations]
VAR LastPeriod = 
    CALCULATE(
        [Total Operations],
        DATEADD(DimTime[FullDateTime], -1, MONTH)
    )
RETURN
    CurrentPeriod - LastPeriod

// Measure: YTD Fleet Fuel Cost
YTD Fuel Cost = 
TOTALYTD(
    SUM(FactFleetTrips[FuelCost]),
    DimTime[FullDateTime]
)

// Measure: Rolling 7-Day Avg Dwell Time
Rolling 7-Day Avg Dwell = 
AVERAGEX(
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -7,
        DAY
    ),
    [Avg Dwell Time (Hours)]
)
```

## Common Usage Patterns

### Pattern 1: Cross-Fact Analysis Query

Analyze how warehouse dwell time affects fleet delivery performance:

```sql
-- SQL query for cross-fact analysis
SELECT 
    g.LocationName AS Warehouse,
    p.Category AS ProductCategory,
    AVG(wh.DwellTimeHours) AS AvgDwellTime,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdle,
    AVG(ft.FuelCost) AS AvgFuelCost,
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    SUM(CASE WHEN ft.DeliveryStatus = 'Delayed' THEN 1 ELSE 0 END) AS DelayedTrips
FROM FactWarehouseOperations wh
INNER JOIN BridgeRouteStorageZone br ON wh.OperationKey = br.OperationKey
INNER JOIN FactFleetTrips ft ON br.TripKey = ft.TripKey
INNER JOIN DimGeography g ON wh.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
WHERE wh.TimeKey >= (SELECT MAX(TimeKey) - 2016 FROM DimTime) -- Last 3 months
GROUP BY g.LocationName, p.Category
HAVING AVG(wh.DwellTimeHours) > 48
ORDER BY AvgDwellTime DESC, DelayedTrips DESC;
```

### Pattern 2: Gravity Zone Optimization

Identify products that should be moved to different zones:

```sql
-- Find misaligned product placements
WITH ProductZoneAnalysis AS (
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.RecommendedZoneType,
        wh.StorageZone AS CurrentZone,
        COUNT(*) AS OperationCount,
        AVG(wh.DurationMinutes) AS AvgPickTime,
        AVG(wh.DwellTimeHours) AS AvgDwellTime
    FROM FactWarehouseOperations wh
    INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
    WHERE wh.OperationType = 'Picking'
        AND wh.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
    GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZoneType, wh.StorageZone
)
SELECT 
    SKU,
    ProductName,
    GravityScore,
    RecommendedZoneType,
    CurrentZone,
    OperationCount,
    AvgPickTime,
    AvgDwellTime,
    CASE 
        WHEN CurrentZone <> RecommendedZoneType THEN 'RELOCATE'
        ELSE 'OK'
    END AS RecommendationStatus
FROM ProductZoneAnalysis
WHERE CurrentZone <> RecommendedZoneType
    AND OperationCount > 10
ORDER BY GravityScore DESC;
```

### Pattern 3: Predictive Maintenance Triage

Prioritize vehicle maintenance by revenue impact:

```sql
-- Fleet maintenance priority queue
WITH TripImpact AS (
    SELECT 
        ft.VehicleID,
        COUNT(*) AS TripCount,
        SUM(ft.FuelCost) AS TotalFuelCost,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(CASE WHEN ft.MaintenanceFlag = 1 THEN 1 ELSE 0 END) AS MaintenanceFlags,
        -- Calculate revenue impact based on high-value products carried
        SUM(
            CASE 
                WHEN p.GravityScore > 70 THEN ft.LoadWeightKG * 10 -- High-value estimate
                ELSE ft.LoadWeightKG * 2
            END
        ) AS EstimatedRevenueImpact
    FROM FactFleetTrips ft
    LEFT JOIN BridgeRouteStorageZone br ON ft.TripKey = br.TripKey
    LEFT JOIN FactWarehouseOperations wh ON br.OperationKey = wh.OperationKey
    LEFT JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
    WHERE ft.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
    GROUP BY ft.VehicleID
)
SELECT 
    VehicleID,
    TripCount,
    TotalFuelCost,
    AvgIdleTime,
    MaintenanceFlags,
    EstimatedRevenueImpact,
    -- Priority score: (maintenance urgency * 50) + (revenue impact * 50)
    (MaintenanceFlags * 50.0 / NULLIF(TripCount, 0)) + 
    (EstimatedRevenueImpact / 1000.0) AS PriorityScore
FROM TripImpact
WHERE MaintenanceFlags > 0
ORDER BY PriorityScore DESC;
```

### Pattern 4: Row-Level Security Setup

Implement role-based access control:

```sql
-- Create security roles
CREATE ROLE WarehouseManager;
CREATE ROLE FleetSupervisor;
CREATE ROLE Executive;

-- Create RLS function for warehouse managers
CREATE FUNCTION dbo.fn_WarehouseSecurityPredicate(@LocationCode VARCHAR(20))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessPermitted
    WHERE 
        @LocationCode IN (
            SELECT LocationCode 
            FROM dbo.UserLocationAccess 
            WHERE UserLogin = USER_NAME()
        )
        OR IS_MEMBER('Executive') = 1
);
GO

-- Apply security policy to DimGeography
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_WarehouseSecurityPredicate(LocationCode)
ON dbo.DimGeography
WITH (STATE = ON);
GO

-- Create user access mapping table
CREATE TABLE UserLocationAccess (
    UserLogin VARCHAR(100),
    LocationCode VARCHAR(20),
    AccessLevel VARCHAR(20), -- 'Read', 'Write', 'Admin'
    CONSTRAINT PK_UserAccess PRIMARY KEY (UserLogin, LocationCode)
);

-- Grant permissions
GRANT SELECT ON FactWarehouseOperations TO WarehouseManager;
GRANT SELECT ON FactFleetTrips TO FleetSupervisor;
GRANT SELECT ON SCHEMA::dbo TO Executive;
```

## Configuration Files

### Connection Configuration (JSON)

Create a `config.json` file for data source connections:

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SqlPassword",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}",
    "trustServerCertificate": true
  },
  "dataSources": {
    "wms": {
      "type": "REST_API",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}",
      "refreshIntervalMinutes": 15
    },
    "telemetry": {
      "type": "MQTT",
      "broker": "${TELEMETRY_BROKER_URL}",
      "topic": "fleet/telemetry/#",
      "username": "${TELEMETRY_USERNAME}",
      "password": "${TELEMETRY_PASSWORD}"
    },
    "weather": {
      "type": "REST_API",
      "endpoint": "https://api.openweathermap.org/data/2.5/forecast",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "powerBI": {
    "workspaceId": "${POWERBI_WORKSPACE_ID}",
    "datasetId": "${POWERBI_DATASET_ID}",
    "refreshSchedule": "*/15 * * * *"
  },
  "alerts": {
    "emailProfile": "LogiFleetAlerts",
    "smtpServer": "${SMTP_SERVER}",
    "smtpPort": 587,
    "defaultRecipients": [
      "${ALERT_EMAIL_1}",
      "${ALERT_EMAIL_2}"
    ]
  }
}
```

## Troubleshooting

### Issue: Slow Dashboard Performance

**Symptoms:** Power BI reports take >10 seconds to load

**Solutions:**

1. **Check indexing:**
```sql
-- Verify indexes exist
SELECT 
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc AS IndexType,
    s.avg_fragmentation_in_percent
FROM sys.indexes i
INNER JOIN sys.dm_db_index_physical_stats(
    DB_ID(), NULL, NULL, NULL, 'LIMITED'
) s ON i.object_id = s.object_id AND i.index_id = s.index_id
WHERE OBJECT_NAME(i.object_id) LIKE 'Fact%'
ORDER BY s.avg_fragmentation_in_percent DESC;

-- Rebuild fragmented indexes
ALTER INDEX ALL ON FactWarehouseOperations REBUILD;
ALTER INDEX ALL ON FactFleetTrips REBUILD;
```

2. **Implement columnstore indexes for analytical queries:**
```sql
-- Create columnstore index on fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, DwellTimeHours, Quantity)
WHERE OperationType IN ('Putaway', 'Picking');
```

3. **Switch to aggregated views:**
```sql
-- Create aggregated view for common queries
CREATE VIEW vw_DailyWarehouseSummary
WITH SCHEMABINDING
AS
SELECT 
    CAST(t.FullDateTime AS DATE) AS OperationDate,
    wh.GeographyKey,
    wh.ProductKey,
    wh.OperationType,
    COUNT_BIG(*) AS OperationCount,
    AVG(wh.DurationMinutes) AS AvgDuration,
    AVG(wh.DwellTimeHours) AS AvgDwell
FROM dbo.FactWarehouseOperations wh
INNER JOIN dbo.DimTime t ON wh.TimeKey = t.TimeKey
GROUP BY CAST(t.FullDateTime AS DATE), wh.GeographyKey, wh.ProductKey, wh.OperationType;
GO

-- Create indexed view
CREATE UNIQUE CLUSTERED INDEX UCI_Daily
