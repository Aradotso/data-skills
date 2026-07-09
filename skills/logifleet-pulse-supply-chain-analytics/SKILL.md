---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing and analytics platform for logistics, fleet management, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "implement multi-fact star schema for warehouse data"
  - "create fleet optimization SQL queries"
  - "build supply chain KPI dashboard"
  - "integrate warehouse management system with Power BI"
  - "deploy logistics data warehouse"
  - "analyze fleet telemetry with SQL Server"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced data warehousing and analytics platform designed for logistics and supply chain management. It combines MS SQL Server for transactional data storage with Power BI for visualization, implementing a multi-fact star schema architecture that harmonizes warehouse operations, fleet telemetry, inventory management, and external data sources.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time dashboard refresh (15-minute granularity)
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Fleet maintenance prioritization

**Primary language:** SQL (MS SQL Server)
**Visualization:** Power BI (.pbit templates)

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Enterprise or Standard edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, TMS, telematics APIs

### Step 1: Deploy SQL Schema

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create core dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2(0) NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME(0) NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthOfYear TINYINT NOT NULL,
    QuarterOfYear TINYINT NOT NULL,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0
)
GO

-- Clustered columnstore index for performance
CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Route Node, Distribution Center
    Street VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    PostalCode VARCHAR(20),
    Country VARCHAR(100),
    Region VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500) NOT NULL,
    ProductCategory VARCHAR(100),
    ProductSubcategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    ShelfLifeDays INT,
    VelocityScore DECIMAL(5,2), -- Calculated: picks per day
    ValueScore DECIMAL(10,2), -- Unit price
    FragilityScore DECIMAL(3,2), -- 0.0 to 1.0
    GravityScore AS (VelocityScore * ValueScore * (1 + FragilityScore)) PERSISTED, -- Computed column
    GravityZone VARCHAR(20) AS (
        CASE 
            WHEN VelocityScore * ValueScore * (1 + FragilityScore) > 100 THEN 'High'
            WHEN VelocityScore * ValueScore * (1 + FragilityScore) > 50 THEN 'Medium'
            ELSE 'Low'
        END
    ) PERSISTED
)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(100) NOT NULL UNIQUE,
    SupplierName VARCHAR(300) NOT NULL,
    SupplierType VARCHAR(100),
    Country VARCHAR(100),
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeDaysStdDev DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityIndex AS (
        (OnTimeDeliveryPercent * 0.4) + 
        ((100 - DefectRatePercent) * 0.3) + 
        (ComplianceScore * 0.3)
    ) PERSISTED
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderNumber VARCHAR(100),
    BatchNumber VARCHAR(100),
    QuantityUnits INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    DistanceMeters DECIMAL(10,2),
    LabourHours DECIMAL(6,2),
    OperationCost DECIMAL(10,2),
    ErrorCount INT DEFAULT 0,
    TemperatureCelsius DECIMAL(5,2), -- For cold storage tracking
    IsAnomaly BIT DEFAULT 0,
    CONSTRAINT FK_WHOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WHOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WHOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WHOps_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
)
GO

-- Partition by month for performance
CREATE PARTITION FUNCTION PF_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, 202604, 202605, 202606)
GO

CREATE PARTITION SCHEME PS_TimeKey
AS PARTITION PF_TimeKey ALL TO ([PRIMARY])
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyStart INT NOT NULL,
    TimeKeyEnd INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripNumber VARCHAR(100),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AvgSpeedKmh AS (
        CASE WHEN DurationMinutes > 0 
        THEN (DistanceKm / (DurationMinutes / 60.0)) 
        ELSE 0 END
    ) PERSISTED,
    FuelEfficiency AS (
        CASE WHEN FuelLiters > 0 
        THEN (DistanceKm / FuelLiters) 
        ELSE 0 END
    ) PERSISTED,
    IdlePercentage AS (
        CASE WHEN DurationMinutes > 0 
        THEN (IdleTimeMinutes * 100.0 / DurationMinutes) 
        ELSE 0 END
    ) PERSISTED,
    TripCost DECIMAL(10,2),
    DelayMinutes INT DEFAULT 0,
    IsDelayed BIT DEFAULT 0,
    WeatherCondition VARCHAR(100),
    TrafficCondition VARCHAR(100),
    CONSTRAINT FK_Fleet_TimeStart FOREIGN KEY (TimeKeyStart) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_TimeEnd FOREIGN KEY (TimeKeyEnd) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

-- Indexes for common query patterns
CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle_Time 
ON FactFleetTrips(VehicleID, TimeKeyStart) 
INCLUDE (DistanceKm, FuelLiters, IdleTimeMinutes)
GO

CREATE NONCLUSTERED INDEX IX_WHOps_Product_Time 
ON FactWarehouseOperations(ProductKey, TimeKey, OperationType) 
INCLUDE (QuantityUnits, DwellTimeMinutes, ProcessingTimeMinutes)
GO
```

### Step 2: Populate Time Dimension

```sql
-- Generate time dimension records (15-minute granularity)
DECLARE @StartDate DATETIME2 = '2026-01-01 00:00:00'
DECLARE @EndDate DATETIME2 = '2026-12-31 23:45:00'
DECLARE @CurrentDate DATETIME2 = @StartDate

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey,
        DateTime,
        Date,
        TimeOfDay,
        HourOfDay,
        DayOfWeek,
        DayName,
        WeekOfYear,
        MonthOfYear,
        QuarterOfYear,
        IsWeekend
    )
    VALUES (
        CAST(FORMAT(@CurrentDate, 'yyyyMMddHHmm') AS INT),
        @CurrentDate,
        CAST(@CurrentDate AS DATE),
        CAST(@CurrentDate AS TIME),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        DATEPART(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END
    )
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END
GO
```

### Step 3: Create ETL Stored Procedures

```sql
-- Incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assuming source data from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        GeographyKey,
        ProductKey,
        SupplierKey,
        OperationType,
        OrderNumber,
        QuantityUnits,
        DwellTimeMinutes,
        ProcessingTimeMinutes,
        OperationCost
    )
    SELECT 
        CAST(FORMAT(s.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        g.GeographyKey,
        p.ProductKey,
        sup.SupplierKey,
        s.OperationType,
        s.OrderNumber,
        s.QuantityUnits,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.ProcessingTimeMinutes,
        s.OperationCost
    FROM Staging_WarehouseOperations s
    INNER JOIN DimGeography g ON s.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    LEFT JOIN DimSupplierReliability sup ON s.SupplierCode = sup.SupplierCode
    WHERE s.OperationDateTime > @LastLoadTime
        AND s.IsProcessed = 0;
    
    -- Mark as processed
    UPDATE Staging_WarehouseOperations
    SET IsProcessed = 1
    WHERE OperationDateTime > @LastLoadTime;
    
END
GO

-- Calculate product gravity scores
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE p
    SET 
        p.VelocityScore = ISNULL(ops.PicksPerDay, 0),
        p.ValueScore = ISNULL(ops.AvgUnitValue, 0)
    FROM DimProductGravity p
    LEFT JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / DATEDIFF(DAY, MIN(t.Date), MAX(t.Date)) AS PicksPerDay,
            AVG(OperationCost / NULLIF(QuantityUnits, 0)) AS AvgUnitValue
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE OperationType = 'Picking'
            AND t.Date >= DATEADD(DAY, -90, GETDATE())
        GROUP BY ProductKey
    ) ops ON p.ProductKey = ops.ProductKey;
    
END
GO
```

### Step 4: Configure Power BI Connection

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Connection string:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server (use env vars for credentials)
4. Import tables:
   - All Dim* tables
   - Fact* tables (DirectQuery for real-time, Import for performance)

## Key SQL Query Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Correlate warehouse dwell time with fleet idle costs by route
SELECT 
    g.LocationName AS Warehouse,
    p.ProductCategory,
    AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
    COUNT(DISTINCT ft.TripKey) AS TripCount,
    AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(ft.TripCost * (ft.IdleTimeMinutes * 1.0 / NULLIF(ft.DurationMinutes, 0))) AS EstimatedIdleCost
FROM FactWarehouseOperations wo
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN FactFleetTrips ft ON ft.OriginGeographyKey = g.GeographyKey
    AND ft.TimeKeyStart >= wo.TimeKey
    AND ft.TimeKeyStart <= DATEADD(HOUR, 24, wo.TimeKey)
WHERE wo.OperationType = 'Shipping'
    AND t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY g.LocationName, p.ProductCategory
HAVING AVG(wo.DwellTimeMinutes) > 60
ORDER BY EstimatedIdleCost DESC
GO
```

### Predictive Bottleneck Detection

```sql
-- Identify operations with increasing processing times (early bottleneck indicator)
WITH ProcessingTrends AS (
    SELECT 
        wo.GeographyKey,
        wo.ProductKey,
        wo.OperationType,
        DATEPART(WEEK, t.Date) AS WeekNum,
        AVG(wo.ProcessingTimeMinutes) AS AvgProcessingTime,
        STDEV(wo.ProcessingTimeMinutes) AS StdDevProcessingTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -60, GETDATE())
    GROUP BY wo.GeographyKey, wo.ProductKey, wo.OperationType, DATEPART(WEEK, t.Date)
),
TrendAnalysis AS (
    SELECT 
        pt1.GeographyKey,
        pt1.ProductKey,
        pt1.OperationType,
        pt1.AvgProcessingTime AS CurrentWeekAvg,
        pt2.AvgProcessingTime AS PriorWeekAvg,
        ((pt1.AvgProcessingTime - pt2.AvgProcessingTime) * 100.0 / NULLIF(pt2.AvgProcessingTime, 0)) AS PercentChange
    FROM ProcessingTrends pt1
    INNER JOIN ProcessingTrends pt2 
        ON pt1.GeographyKey = pt2.GeographyKey
        AND pt1.ProductKey = pt2.ProductKey
        AND pt1.OperationType = pt2.OperationType
        AND pt1.WeekNum = pt2.WeekNum + 1
)
SELECT 
    g.LocationName,
    p.ProductName,
    ta.OperationType,
    ta.CurrentWeekAvg,
    ta.PriorWeekAvg,
    ta.PercentChange,
    CASE 
        WHEN ta.PercentChange > 20 THEN 'Critical'
        WHEN ta.PercentChange > 10 THEN 'Warning'
        ELSE 'Normal'
    END AS AlertLevel
FROM TrendAnalysis ta
INNER JOIN DimGeography g ON ta.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON ta.ProductKey = p.ProductKey
WHERE ta.PercentChange > 10
ORDER BY ta.PercentChange DESC
GO
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend warehouse zone reassignments based on gravity scores
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.GravityZone AS CurrentZone,
    CASE 
        WHEN p.GravityScore > 100 THEN 'High'
        WHEN p.GravityScore > 50 THEN 'Medium'
        ELSE 'Low'
    END AS RecommendedZone,
    g.LocationName AS CurrentWarehouse,
    AVG(wo.ProcessingTimeMinutes) AS AvgPickTime,
    COUNT(*) AS PickFrequency
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE wo.OperationType = 'Picking'
    AND t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.GravityZone,
    g.LocationName
HAVING p.GravityZone <> CASE 
        WHEN p.GravityScore > 100 THEN 'High'
        WHEN p.GravityScore > 50 THEN 'Medium'
        ELSE 'Low'
    END
ORDER BY p.GravityScore DESC
GO
```

### Fleet Maintenance Priority Queue

```sql
-- Prioritize fleet maintenance by revenue impact
SELECT 
    ft.VehicleID,
    COUNT(DISTINCT ft.TripKey) AS TripCount,
    SUM(ft.DistanceKm) AS TotalDistanceKm,
    AVG(ft.FuelEfficiency) AS AvgFuelEfficiency,
    AVG(ft.IdlePercentage) AS AvgIdlePercent,
    SUM(ft.TripCost) AS TotalRevenue,
    CASE 
        WHEN AVG(ft.IdlePercentage) > 20 THEN 'Engine Diagnostics'
        WHEN AVG(ft.FuelEfficiency) < 8 THEN 'Fuel System Check'
        WHEN SUM(ft.DistanceKm) > 50000 THEN 'Scheduled Maintenance'
        ELSE 'Normal Operation'
    END AS MaintenanceRecommendation,
    CASE 
        WHEN AVG(ft.IdlePercentage) > 20 THEN SUM(ft.TripCost) * 0.15
        WHEN AVG(ft.FuelEfficiency) < 8 THEN SUM(ft.TripCost) * 0.12
        ELSE SUM(ft.TripCost) * 0.05
    END AS EstimatedRevenueAtRisk
FROM FactFleetTrips ft
INNER JOIN DimTime t ON ft.TimeKeyStart = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
GROUP BY ft.VehicleID
HAVING AVG(ft.IdlePercentage) > 15
    OR AVG(ft.FuelEfficiency) < 8
    OR SUM(ft.DistanceKm) > 50000
ORDER BY EstimatedRevenueAtRisk DESC
GO
```

## Power BI DAX Measures

### Create measures in Power BI for cross-fact analysis:

```dax
// Warehouse Velocity Score
WarehouseVelocity = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[OperationType] = "Picking"
) / 
CALCULATE(
    DISTINCTCOUNT(DimTime[Date])
)

// Fleet Utilization Rate
FleetUtilizationRate = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Cross-Fact: Cost per Unit Delivered
CostPerUnitDelivered = 
DIVIDE(
    SUM(FactFleetTrips[TripCost]),
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityUnits]),
        FactWarehouseOperations[OperationType] = "Shipping"
    ),
    0
)

// Temporal Elasticity: Projected Bottleneck Index
BottleneckIndex = 
VAR CurrentAvgProcessing = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes]),
        DimTime[Date] >= TODAY() - 7
    )
VAR HistoricalAvgProcessing = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes]),
        DimTime[Date] < TODAY() - 7,
        DimTime[Date] >= TODAY() - 30
    )
RETURN
    DIVIDE(
        CurrentAvgProcessing - HistoricalAvgProcessing,
        HistoricalAvgProcessing,
        0
    ) * 100

// Gravity Zone Efficiency
GravityZoneEfficiency = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes]),
    FILTER(
        DimProductGravity,
        DimProductGravity[GravityZone] = "High"
    )
) / 
CALCULATE(
    AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes]),
    FILTER(
        DimProductGravity,
        DimProductGravity[GravityZone] = "Low"
    )
)
```

## Configuration Files

### Connection Configuration (config.json)

```json
{
  "database": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SqlPassword",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}",
    "encrypt": true,
    "trustServerCertificate": false
  },
  "etl": {
    "refreshIntervalMinutes": 15,
    "historicalLoadDays": 90,
    "batchSize": 10000
  },
  "dataSources": {
    "wms": {
      "apiEndpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telemetry": {
      "apiEndpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}"
    },
    "weather": {
      "apiEndpoint": "https://api.openweathermap.org/data/2.5",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "alerts": {
    "email": {
      "smtpServer": "${SMTP_SERVER}",
      "smtpPort": 587,
      "fromAddress": "${ALERT_EMAIL_FROM}",
      "toAddresses": ["${ALERT_EMAIL_TO}"]
    },
    "thresholds": {
      "dwellTimeMinutes": 120,
      "idlePercentage": 15,
      "processingTimeIncreasePercent": 20
    }
  }
}
```

### Scheduled Refresh Script (PowerShell)

```powershell
# refresh_dashboards.ps1
# Schedule this script via Windows Task Scheduler or Azure Automation

$configPath = "config.json"
$config = Get-Content $configPath | ConvertFrom-Json

# Connection string
$connectionString = "Server=$($env:SQL_SERVER_HOST);Database=LogiFleetPulse;User Id=$($env:SQL_USERNAME);Password=$($env:SQL_PASSWORD);Encrypt=True;"

# Execute ETL procedures
$sqlCommands = @(
    "EXEC usp_LoadWarehouseOperations @LastLoadTime = '$(Get-Date (Get-Date).AddMinutes(-15) -Format "yyyy-MM-dd HH:mm:ss")'",
    "EXEC usp_UpdateProductGravityScores"
)

foreach ($cmd in $sqlCommands) {
    Invoke-Sqlcmd -ConnectionString $connectionString -Query $cmd -Verbose
}

# Refresh Power BI dataset (requires Power BI REST API)
$pbiWorkspaceId = $env:PBI_WORKSPACE_ID
$pbiDatasetId = $env:PBI_DATASET_ID
$pbiAccessToken = $env:PBI_ACCESS_TOKEN

$refreshUrl = "https://api.powerbi.com/v1.0/myorg/groups/$pbiWorkspaceId/datasets/$pbiDatasetId/refreshes"
$headers = @{
    "Authorization" = "Bearer $pbiAccessToken"
    "Content-Type" = "application/json"
}

Invoke-RestMethod -Method Post -Uri $refreshUrl -Headers $headers

Write-Host "Dashboard refresh completed at $(Get-Date)"
```

## Common Patterns & Best Practices

### 1. Incremental Data Loading

Always use time-based incremental loads to avoid full table scans:

```sql
-- Track last successful load
CREATE TABLE ETL_LoadLog (
    LoadID INT PRIMARY KEY IDENTITY(1,1),
    TableName VARCHAR(100) NOT NULL,
    LoadDateTime DATETIME2 NOT NULL DEFAULT GETDATE(),
    RecordsLoaded INT,
    Status VARCHAR(50)
)
GO

-- Modified load procedure with logging
CREATE PROCEDURE usp_LoadWarehouseOperations_WithLog
AS
BEGIN
    DECLARE @LastLoadTime DATETIME2
    DECLARE @RecordsLoaded INT
    
    -- Get last successful load time
    SELECT @LastLoadTime = MAX(LoadDateTime)
    FROM ETL_LoadLog
    WHERE TableName = 'FactWarehouseOperations'
        AND Status = 'Success'
    
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(DAY, -90, GETDATE())
    
    BEGIN TRY
        -- Perform load
        EXEC usp_LoadWarehouseOperations @LastLoadTime
        
        SET @RecordsLoaded = @@ROWCOUNT
        
        -- Log success
        INSERT INTO ETL_LoadLog (TableName, RecordsLoaded, Status)
        VALUES ('FactWarehouseOperations', @RecordsLoaded, 'Success')
    END TRY
    BEGIN CATCH
        -- Log failure
        INSERT INTO ETL_LoadLog (TableName, RecordsLoaded, Status)
        VALUES ('FactWarehouseOperations', 0, 'Failed: ' + ERROR_MESSAGE())
    END CATCH
END
GO
```

### 2. Row-Level Security for Multi-Tenant Access

```sql
-- Create security predicate for geography-based filtering
CREATE SCHEMA Security
GO

CREATE FUNCTION Security.fn_SecurityPredicate_Geography(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS result
    WHERE 
        @GeographyKey IN (
            SELECT g.GeographyKey
            FROM dbo.DimGeography g
            INNER JOIN dbo.UserGeographyAccess uga ON g.GeographyKey = uga.GeographyKey
            WHERE uga.UserName = USER_NAME()
        )
        OR IS_MEMBER('LogisticsAdmin') = 1
GO

-- Apply to fact tables
CREATE SECURITY POLICY Security.GeographySecurityPolicy
ADD FILTER PREDICATE Security.fn_SecurityPredicate_Geography(GeographyKey) 
    ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE Security.fn_SecurityPredicate_Geography(OriginGeographyKey) 
    ON dbo.FactFleetTrips
WITH (STATE = ON)
GO
```

### 3. Automated Anomaly Detection

```sql
-- Flag outliers using statistical methods
CREATE PROCEDURE usp_DetectAnomalies
AS
BEGIN
    WITH Stats AS (
        SELECT 
            ProductKey,
            GeographyKey,
            OperationType,
            AVG(ProcessingTimeMinutes) AS AvgTime,
            STDEV(ProcessingTimeMinutes) AS StdDevTime
        FROM FactWarehouseOperations
        WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY ProductKey, GeographyKey, OperationType
    )
    UPDATE wo
    SET wo.IsAnomaly = 1
    FROM FactWarehouseOperations wo
    INNER JOIN Stats s 
        ON wo.ProductKey = s.ProductKey
        AND wo.GeographyKey = s.Ge
