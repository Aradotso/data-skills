---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain KPI harmonization
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "configure power bi logistics dashboard with sql server"
  - "implement multi-fact star schema for warehouse and fleet data"
  - "create supply chain analytics data warehouse"
  - "build logistics intelligence platform with power bi"
  - "integrate warehouse operations with fleet telemetry data"
  - "deploy logifleet pulse data model"
  - "configure cross-modal supply chain analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics platform for logistics operations that combines:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and supply chain data
- **MS SQL Server backend** with fact tables for operations, trips, and cross-docking
- **Power BI visualization layer** with real-time dashboards (15-minute refresh cycles)
- **Cross-fact KPI harmonization** to analyze relationships between inventory, fleet performance, and supplier reliability
- **Predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Role-based access control** with row-level security

The platform is designed for 3PL operators, retail chains, food distributors, and any organization managing complex warehouse-to-delivery logistics.

## Installation & Setup

### Prerequisites

```bash
# Required software
- MS SQL Server 2019 or later
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Network access to WMS, TMS, or telemetry data sources
```

### Step 1: Deploy SQL Schema

```sql
-- Execute the schema deployment script
-- Connect to your SQL Server instance first

USE master;
GO

-- Create the logistics database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeStamp DATETIME2(0) NOT NULL,
    [Year] INT NOT NULL,
    [Month] INT NOT NULL,
    [Day] INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    QuarterHourBucket INT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek VARCHAR(10) NOT NULL,
    FiscalPeriod VARCHAR(10) NOT NULL,
    IsBusinessHours BIT DEFAULT 0,
    CONSTRAINT UQ_DimTime_DateTime UNIQUE (DateTimeStamp)
);
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    NodeType VARCHAR(20), -- 'Warehouse', 'RoutePoint', 'Hub'
    NodeCode VARCHAR(50) UNIQUE NOT NULL,
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7)
);
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(255),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite score: velocity + value + fragility
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2),
    OptimalZoneType VARCHAR(50) -- Recommended storage zone
);
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(255),
    LeadTimeDaysAvg DECIMAL(5,2),
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,3),
    ComplianceScore DECIMAL(5,2), -- 0-100
    LastAuditDate DATE
);
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(20) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT NOT NULL,
    CycleTimeMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(8,2),
    ZoneCode VARCHAR(50),
    EmployeeID VARCHAR(50),
    BatchNumber VARCHAR(100)
);
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyStart INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyEnd INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    DistanceKm DECIMAL(10,2),
    FuelLiters DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    LoadingTimeMinutes DECIMAL(8,2),
    UnloadingTimeMinutes DECIMAL(8,2),
    WeightKg DECIMAL(10,2),
    DelayMinutes INT DEFAULT 0,
    DelayReason VARCHAR(255)
);
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyInbound INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyOutbound INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT NOT NULL FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    QuantityUnits INT NOT NULL,
    TransferTimeMinutes DECIMAL(8,2),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey)
);
GO

-- Create indexes for performance
CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactFleet_TimeStart ON FactFleetTrips(TimeKeyStart);
CREATE INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);
CREATE INDEX IX_FactCrossDock_Product ON FactCrossDock(ProductKey);
GO
```

### Step 2: Create Data Loading Stored Procedures

```sql
-- Incremental load procedure for time dimension
CREATE PROCEDURE usp_LoadDimTime
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        IF NOT EXISTS (SELECT 1 FROM DimTime WHERE DateTimeStamp = @CurrentDateTime)
        BEGIN
            INSERT INTO DimTime (
                DateTimeStamp, [Year], [Month], [Day], [Hour], [Minute],
                QuarterHourBucket, DayOfWeek, FiscalPeriod, IsBusinessHours
            )
            VALUES (
                @CurrentDateTime,
                YEAR(@CurrentDateTime),
                MONTH(@CurrentDateTime),
                DAY(@CurrentDateTime),
                DATEPART(HOUR, @CurrentDateTime),
                DATEPART(MINUTE, @CurrentDateTime),
                (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15,
                DATENAME(WEEKDAY, @CurrentDateTime),
                'FY' + CAST(YEAR(@CurrentDateTime) AS VARCHAR(4)) + 'Q' + 
                    CAST(CEILING(MONTH(@CurrentDateTime) / 3.0) AS VARCHAR(1)),
                CASE WHEN DATEPART(HOUR, @CurrentDateTime) BETWEEN 8 AND 17 
                     AND DATEPART(WEEKDAY, @CurrentDateTime) NOT IN (1, 7) 
                     THEN 1 ELSE 0 END
            );
        END
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute to populate time dimension for 2 years
EXEC usp_LoadDimTime 
    @StartDate = '2025-01-01 00:00:00',
    @EndDate = '2026-12-31 23:45:00';
GO

-- Data ingestion procedure from external WMS
CREATE PROCEDURE usp_LoadWarehouseOperations
    @SourceConnectionString NVARCHAR(500)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external WMS via linked server or OPENROWSET
    -- Adjust based on your actual data source
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        QuantityHandled, CycleTimeMinutes, DwellTimeHours, 
        ZoneCode, EmployeeID, BatchNumber
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        wms.OperationType,
        wms.Quantity,
        wms.CycleTime,
        wms.DwellTime,
        wms.Zone,
        wms.EmployeeID,
        wms.BatchNumber
    FROM OPENROWSET(
        'SQLNCLI',
        @SourceConnectionString,
        'SELECT * FROM WMS.WarehouseTransactions WHERE ProcessedFlag = 0'
    ) AS wms
    INNER JOIN DimTime dt ON CAST(wms.TransactionDateTime AS DATETIME2(0)) = dt.DateTimeStamp
    INNER JOIN DimGeography dg ON wms.WarehouseCode = dg.NodeCode
    INNER JOIN DimProductGravity dp ON wms.SKU = dp.SKU
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.TimeKey = dt.TimeKey 
          AND f.ProductKey = dp.ProductKey
          AND f.BatchNumber = wms.BatchNumber
    );
END;
GO
```

### Step 3: Configure Power BI Connection

Create a `config.json` file (do not commit to repository):

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "Windows",
    "connectionTimeout": 30
  },
  "refreshSchedule": {
    "intervalMinutes": 15,
    "enableIncrementalRefresh": true
  },
  "security": {
    "enableRLS": true,
    "defaultRole": "Viewer"
  }
}
```

### Step 4: Import Power BI Template

```powershell
# Open Power BI Desktop and load the template
# File > Open > LogiFleet_Pulse_Master.pbit

# Update connection parameters in Power Query
# Replace placeholders with actual values from environment variables
```

## Key Data Model Patterns

### Cross-Fact KPI Calculation

```sql
-- Example: Correlate warehouse dwell time with fleet idle time
-- This shows products with high warehouse dwell AND delivery route inefficiency

CREATE VIEW vw_DwellTimeVsFleetIdling AS
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.Category,
    AVG(fwo.DwellTimeHours) AS AvgDwellTimeHours,
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    COUNT(DISTINCT fwo.OperationKey) AS WarehouseOps,
    COUNT(DISTINCT fft.TripKey) AS FleetTrips,
    -- Composite inefficiency score
    (AVG(fwo.DwellTimeHours) * 10 + AVG(fft.IdleTimeMinutes) / 6.0) AS InefficiencyScore
FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN FactFleetTrips fft ON 
    fft.TimeKeyStart >= fwo.TimeKey 
    AND fft.TimeKeyStart <= DATEADD(DAY, 3, fwo.TimeKey)
WHERE fwo.OperationType = 'Shipping'
GROUP BY dp.SKU, dp.ProductName, dp.Category
HAVING AVG(fwo.DwellTimeHours) > 24;
GO
```

### Warehouse Gravity Zone Optimization

```sql
-- Recalculate gravity scores based on recent velocity
CREATE PROCEDURE usp_UpdateGravityScores
    @LookbackDays INT = 30
AS
BEGIN
    SET NOCOUNT ON;
    
    ;WITH VelocityMetrics AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            SUM(QuantityHandled) AS TotalQuantity,
            AVG(CycleTimeMinutes) AS AvgCycleTime
        FROM FactWarehouseOperations
        WHERE OperationType IN ('Picking', 'Shipping')
          AND TimeKey >= (SELECT TimeKey FROM DimTime 
                          WHERE DateTimeStamp >= DATEADD(DAY, -@LookbackDays, GETDATE())
                          ORDER BY TimeKey ASC OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY)
        GROUP BY ProductKey
    )
    UPDATE dp
    SET 
        GravityScore = (
            (vm.PickCount / 100.0) * 0.4 +  -- Pick frequency weight
            (dp.ValueTier CASE WHEN 'High' THEN 3 WHEN 'Medium' THEN 2 ELSE 1 END) * 0.3 +
            (1.0 / NULLIF(vm.AvgCycleTime, 0)) * 10 * 0.2 +  -- Inverse cycle time
            (1.0 - dp.FragilityIndex) * 0.1  -- Lower fragility = higher gravity
        ),
        VelocityClass = CASE 
            WHEN vm.PickCount > 100 THEN 'Fast'
            WHEN vm.PickCount > 30 THEN 'Medium'
            ELSE 'Slow'
        END,
        OptimalZoneType = CASE 
            WHEN vm.PickCount > 100 AND dp.ValueTier = 'High' THEN 'Zone_A_Express'
            WHEN vm.PickCount > 100 THEN 'Zone_B_FastMover'
            WHEN vm.PickCount > 30 THEN 'Zone_C_Standard'
            ELSE 'Zone_D_SlowMover'
        END
    FROM DimProductGravity dp
    INNER JOIN VelocityMetrics vm ON dp.ProductKey = vm.ProductKey;
END;
GO
```

### Predictive Bottleneck Detection

```sql
-- Identify probable congestion points using historical patterns
CREATE VIEW vw_BottleneckPrediction AS
WITH HourlyPatterns AS (
    SELECT 
        dt.[Hour],
        dt.DayOfWeek,
        dg.NodeCode,
        AVG(fwo.CycleTimeMinutes) AS AvgCycleTime,
        STDEV(fwo.CycleTimeMinutes) AS StdDevCycleTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    WHERE dt.DateTimeStamp >= DATEADD(DAY, -60, GETDATE())
    GROUP BY dt.[Hour], dt.DayOfWeek, dg.NodeCode
)
SELECT 
    [Hour],
    DayOfWeek,
    NodeCode,
    AvgCycleTime,
    StdDevCycleTime,
    OperationCount,
    -- Bottleneck probability: high volume + high variance + slow cycle time
    (OperationCount / 100.0 * 0.3 + 
     StdDevCycleTime / NULLIF(AvgCycleTime, 0) * 0.4 +
     AvgCycleTime / 10.0 * 0.3) AS BottleneckScore
FROM HourlyPatterns
WHERE OperationCount > 10
ORDER BY BottleneckScore DESC;
GO
```

## Power BI DAX Measures

```dax
// Total Fleet Idle Cost
TotalFleetIdleCost = 
SUMX(
    FactFleetTrips,
    FactFleetTrips[IdleTimeMinutes] / 60 * 25  // $25/hour idle cost assumption
)

// Warehouse Throughput Rate (units per hour)
WarehouseThroughput = 
DIVIDE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    SUM(FactWarehouseOperations[CycleTimeMinutes]) / 60,
    0
)

// Cross-Fact Delivery Efficiency
DeliveryEfficiency = 
VAR TotalDwellTime = 
    CALCULATE(
        SUM(FactWarehouseOperations[DwellTimeHours]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
VAR TotalTripTime = 
    SUMX(
        FactFleetTrips,
        DATEDIFF(
            RELATED(DimTime[DateTimeStamp]),
            RELATEDTABLE(DimTime[DateTimeStamp]),
            MINUTE
        ) / 60
    )
RETURN
DIVIDE(
    TotalTripTime,
    TotalDwellTime + TotalTripTime,
    0
) * 100

// Supplier Reliability Index (0-100)
SupplierReliabilityIndex = 
AVERAGEX(
    DimSupplierReliability,
    (100 - DimSupplierReliability[DefectRatePercent] * 10) * 0.5 +
    DimSupplierReliability[ComplianceScore] * 0.5
)

// Dynamic Gravity Zone Recommendation
RecommendedZoneShift = 
VAR CurrentZone = SELECTEDVALUE(FactWarehouseOperations[ZoneCode])
VAR OptimalZone = SELECTEDVALUE(DimProductGravity[OptimalZoneType])
RETURN
IF(
    CurrentZone <> OptimalZone,
    "Move to " & OptimalZone,
    "Optimal"
)
```

## Row-Level Security (RLS) Implementation

```sql
-- Create security table
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY(1,1),
    RoleName VARCHAR(50) UNIQUE NOT NULL,
    GeographyFilter VARCHAR(MAX),  -- JSON array of allowed GeographyKeys
    CanViewFleetData BIT DEFAULT 0,
    CanViewFinancials BIT DEFAULT 0
);
GO

CREATE TABLE UserRoleAssignments (
    UserEmail VARCHAR(255) NOT NULL,
    RoleID INT NOT NULL FOREIGN KEY REFERENCES SecurityRoles(RoleID),
    PRIMARY KEY (UserEmail, RoleID)
);
GO

-- Sample role setup
INSERT INTO SecurityRoles (RoleName, GeographyFilter, CanViewFleetData, CanViewFinancials)
VALUES 
    ('GlobalExecutive', NULL, 1, 1),  -- NULL = all access
    ('RegionalManager', '["US-West", "US-Central"]', 1, 0),
    ('WarehouseSupervisor', '["WH-001", "WH-002"]', 0, 0);
GO
```

Power BI RLS DAX filter:

```dax
// Apply to DimGeography table
[NodeCode] IN 
    JSONTABLE(
        LOOKUPVALUE(
            SecurityRoles[GeographyFilter],
            SecurityRoles[RoleName],
            USERPRINCIPALNAME()
        )
    )
```

## Configuration Examples

### External Data Source Integration

```sql
-- Create linked server to WMS (adjust for your system)
EXEC sp_addlinkedserver 
    @server = 'WMS_PROD',
    @srvproduct = '',
    @provider = 'SQLNCLI',
    @datasrc = '${WMS_SERVER_HOST}',
    @catalog = 'WarehouseDB';
GO

EXEC sp_addlinkedsrvlogin 
    @rmtsrvname = 'WMS_PROD',
    @useself = 'FALSE',
    @rmtuser = '${WMS_USER}',
    @rmtpassword = '${WMS_PASSWORD}';
GO

-- Configure external table for polybase ingestion
CREATE EXTERNAL DATA SOURCE TelemetryAPI
WITH (
    TYPE = HADOOP,
    LOCATION = '${TELEMETRY_API_ENDPOINT}',
    CREDENTIAL = TelemetryAPICredential
);
GO
```

### Automated Alert Configuration

```sql
-- Alert when idle time exceeds threshold
CREATE PROCEDURE usp_CheckFleetIdleAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertThresholdPercent DECIMAL(5,2) = 15.0;
    
    ;WITH IdleRates AS (
        SELECT 
            VehicleID,
            SUM(IdleTimeMinutes) AS TotalIdle,
            SUM(DATEDIFF(MINUTE, ts.DateTimeStamp, te.DateTimeStamp)) AS TotalTrip,
            CAST(SUM(IdleTimeMinutes) * 100.0 / 
                 NULLIF(SUM(DATEDIFF(MINUTE, ts.DateTimeStamp, te.DateTimeStamp)), 0) 
                 AS DECIMAL(5,2)) AS IdlePercent
        FROM FactFleetTrips fft
        INNER JOIN DimTime ts ON fft.TimeKeyStart = ts.TimeKey
        INNER JOIN DimTime te ON fft.TimeKeyEnd = te.TimeKey
        WHERE ts.DateTimeStamp >= DATEADD(DAY, -7, GETDATE())
        GROUP BY VehicleID
    )
    INSERT INTO AlertLog (AlertType, AlertMessage, CreatedAt)
    SELECT 
        'FleetIdle',
        'Vehicle ' + VehicleID + ' idle time: ' + CAST(IdlePercent AS VARCHAR(10)) + '%',
        GETDATE()
    FROM IdleRates
    WHERE IdlePercent > @AlertThresholdPercent;
    
    -- Send notifications (integrate with email/SMS service)
    -- Example: Call external API via CLR or SQL Agent job
END;
GO

-- Schedule via SQL Agent (run every 15 minutes)
```

## Common Patterns

### Time-Phased Scenario Modeling

```sql
-- Simulate impact of capacity increase
CREATE PROCEDURE usp_SimulateCapacityIncrease
    @CapacityIncreasePercent DECIMAL(5,2) = 20.0,
    @SimulationDays INT = 30
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Create temporary scenario table
    CREATE TABLE #ScenarioResults (
        ScenarioDate DATE,
        BaselineThroughput DECIMAL(12,2),
        ProjectedThroughput DECIMAL(12,2),
        FleetUtilizationChange DECIMAL(5,2)
    );
    
    ;WITH DailyBaseline AS (
        SELECT 
            CAST(dt.DateTimeStamp AS DATE) AS OpDate,
            SUM(QuantityHandled) AS DailyQuantity,
            SUM(CycleTimeMinutes) / 60.0 AS DailyHours
        FROM FactWarehouseOperations fwo
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.DateTimeStamp >= DATEADD(DAY, -@SimulationDays, GETDATE())
        GROUP BY CAST(dt.DateTimeStamp AS DATE)
    )
    INSERT INTO #ScenarioResults
    SELECT 
        OpDate,
        DailyQuantity / DailyHours AS BaselineThroughput,
        (DailyQuantity * (1 + @CapacityIncreasePercent / 100.0)) / DailyHours AS ProjectedThroughput,
        -- Estimate fleet utilization reduction (inverse relationship)
        -(@CapacityIncreasePercent * 0.6) AS FleetUtilizationChange
    FROM DailyBaseline;
    
    SELECT * FROM #ScenarioResults ORDER BY ScenarioDate;
    
    DROP TABLE #ScenarioResults;
END;
GO
```

### Natural Language Query Support (Power BI Q&A)

Configure synonyms in Power BI model:

```yaml
# Add to Power BI model metadata
Synonyms:
  - Table: FactFleetTrips
    Column: DistanceKm
    Synonyms: ["miles", "route length", "trip distance"]
  - Table: FactWarehouseOperations
    Column: DwellTimeHours
    Synonyms: ["storage time", "waiting period", "time in warehouse"]
  - Measure: DeliveryEfficiency
    Synonyms: ["on-time performance", "delivery success rate"]
```

## Troubleshooting

### Performance Issues

```sql
-- Check missing indexes
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    ips.avg_user_impact AS AvgImpact,
    ips.avg_total_user_cost AS AvgCost,
    'CREATE INDEX IX_' + OBJECT_NAME(ips.object_id) + '_' + 
        REPLACE(REPLACE(id.equality_columns, '[', ''), ']', '') + 
    ' ON ' + OBJECT_NAME(ips.object_id) + ' (' + id.equality_columns + ')' AS CreateStatement
FROM sys.dm_db_missing_index_details id
INNER JOIN sys.dm_db_missing_index_groups ig ON id.index_handle = ig.index_handle
INNER JOIN sys.dm_db_missing_index_group_stats ips ON ig.index_group_handle = ips.group_handle
WHERE ips.avg_user_impact > 50
ORDER BY ips.avg_user_impact DESC;
GO

-- Rebuild fragmented indexes
EXEC sp_MSforeachtable 'ALTER INDEX ALL ON ? REBUILD WITH (ONLINE = ON)';
GO
```

### Data Quality Validation

```sql
-- Check for orphaned records
SELECT 'FactWarehouseOperations' AS TableName, COUNT(*) AS OrphanCount
FROM FactWarehouseOperations fwo
LEFT JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
WHERE dp.ProductKey IS NULL

UNION ALL

SELECT 'FactFleetTrips', COUNT(*)
FROM FactFleetTrips fft
LEFT JOIN DimGeography dg ON fft.OriginGeographyKey = dg.GeographyKey
WHERE dg.GeographyKey IS NULL;
GO

-- Fix missing dimension references
UPDATE FactWarehouseOperations
SET ProductKey = (SELECT ProductKey FROM DimProductGravity WHERE SKU = 'UNKNOWN')
WHERE ProductKey NOT IN (SELECT ProductKey FROM DimProductGravity);
GO
```

### Power BI Refresh Failures

```powershell
# Check last refresh status via Power BI REST API
$headers = @{
    "Authorization" = "Bearer ${POWERBI_ACCESS_TOKEN}"
}

$response = Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/datasets/${DATASET_ID}/refreshes" `
    -Headers $headers -Method Get

$response.value | Select-Object -First 5 | Format-Table requestId, status, endTime
```

```sql
-- Verify incremental refresh partitions
SELECT 
    t.name AS TableName,
    p.partition_number,
    p.rows,
    p.data_compression_desc
FROM sys.tables t
INNER JOIN sys.partitions p ON t.object_id = p.object_id
WHERE t.name LIKE 'Fact%'
ORDER BY t.name, p.partition_number;
GO
```

### Connection Timeout Issues

```json
// Update Power BI connection string
{
  "connectionString": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;Connection Timeout=120;",
  "commandTimeout": 600
}
```

## Best Practices

1. **Always use parameterized queries** to prevent SQL injection
2. **Enable query store** for performance monitoring:
   ```sql
   ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;
   ```
3. **Partition large fact tables** by date:
   ```sql
   CREATE PARTITION FUNCTION pfTimeKey (INT)
   AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301);
   ```
4. **Schedule gravity score updates** weekly via SQL Agent
5. **Monitor dashboard usage** via Power BI admin portal
6. **Backup before schema changes**:
   ```sql
   BACKUP DATABASE LogiFleetPulse TO DISK = '${BACKUP_PATH}/LogiFleetPulse_PreChange.bak';
   ```
