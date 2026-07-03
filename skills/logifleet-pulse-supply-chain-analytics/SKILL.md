---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics analytics platform for warehouse operations, fleet management, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "create warehouse and fleet data model"
  - "implement multi-fact star schema for logistics"
  - "build supply chain KPI dashboard"
  - "deploy LogiCore analytics warehouse"
  - "integrate warehouse management system with Power BI"
  - "set up real-time fleet tracking analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines MS SQL Server data warehousing with Power BI visualization to provide unified analytics across warehouse operations, fleet management, and supply chain orchestration. It features a multi-fact star schema design with time-phased dimensions, cross-fact KPI harmonization, and predictive analytics capabilities.

**Key Features:**
- Multi-fact star schema linking warehouse, fleet, and supplier data
- Real-time dashboards with 15-minute refresh intervals
- Warehouse Gravity Zones™ for spatial optimization
- Adaptive Fleet Triage Engine for maintenance prioritization
- Temporal elasticity modeling for scenario simulation
- Role-based access control with row-level security

## Installation

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- Access to source systems (WMS, TMS, telemetry APIs)

### Database Setup

1. **Deploy the SQL schema:**

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create staging schema for ETL
CREATE SCHEMA staging
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeValue DATETIME NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayOfMonth TINYINT,
    Month TINYINT,
    Quarter TINYINT,
    Year SMALLINT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, Distribution Center, Route Node
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    City NVARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    IsFragile BIT,
    IsPerishable BIT,
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    -- Gravity scoring components
    PickFrequencyScore INT, -- 1-100
    ValueScore INT, -- 1-100
    FragilityScore INT, -- 1-100
    GravityZone VARCHAR(20), -- High, Medium, Low
    RecommendedStorageZone VARCHAR(50),
    LastGravityCalculation DATETIME
)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(200),
    AverageLeadTimeDays DECIMAL(6,2),
    LeadTimeVarianceDays DECIMAL(6,2),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2),
    ComplianceScore INT, -- 1-100
    ReliabilityTier VARCHAR(20) -- Platinum, Gold, Silver, Bronze
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID VARCHAR(100),
    Quantity INT,
    CycleTimeMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50),
    ErrorCount INT DEFAULT 0,
    ProcessedDateTime DATETIME
)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    RouteID VARCHAR(100),
    TripDistanceKm DECIMAL(10,2),
    TripDurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeight DECIMAL(10,2),
    DelayMinutes DECIMAL(10,2),
    DelayReason VARCHAR(200),
    TripStartDateTime DATETIME,
    TripEndDateTime DATETIME
)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferID VARCHAR(100),
    Quantity INT,
    DwellTimeMinutes DECIMAL(10,2),
    ProcessedDateTime DATETIME
)
GO

-- Create indexes for performance
CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
CREATE INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey)
CREATE INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey)
CREATE INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey)
CREATE INDEX IX_FactFleet_Destination ON FactFleetTrips(DestinationGeographyKey)
```

2. **Create ETL stored procedures:**

```sql
-- Incremental load procedure for time dimension
CREATE PROCEDURE [dbo].[LoadDimTime]
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Generate 15-minute granularity time records
    DECLARE @CurrentDateTime DATETIME = @StartDate
    DECLARE @EndDateTime DATETIME = DATEADD(DAY, 1, @EndDate)
    
    WHILE @CurrentDateTime < @EndDateTime
    BEGIN
        INSERT INTO DimTime (
            TimeKey, DateTimeValue, HourOfDay, DayOfWeek, 
            DayOfMonth, Month, Quarter, Year, FiscalPeriod,
            IsWeekend, IsHoliday
        )
        SELECT 
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
            @CurrentDateTime,
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATEPART(DAY, @CurrentDateTime),
            DATEPART(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            DATEPART(YEAR, @CurrentDateTime),
            'FY' + CAST(YEAR(@CurrentDateTime) AS VARCHAR(4)) + 'Q' + CAST(DATEPART(QUARTER, @CurrentDateTime) AS VARCHAR(1)),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END,
            0 -- Holiday flag - update separately
        WHERE NOT EXISTS (
            SELECT 1 FROM DimTime 
            WHERE DateTimeValue = @CurrentDateTime
        )
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime)
    END
END
GO

-- Warehouse operations ETL
CREATE PROCEDURE [dbo].[LoadFactWarehouseOperations]
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -1, GETDATE())
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationID, Quantity, CycleTimeMinutes, DwellTimeHours,
        StorageZone, OperatorID, ErrorCount, ProcessedDateTime
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OperationID,
        s.Quantity,
        s.CycleTimeMinutes,
        s.DwellTimeHours,
        s.StorageZone,
        s.OperatorID,
        s.ErrorCount,
        s.ProcessedDateTime
    FROM staging.WarehouseOperations s
    INNER JOIN DimTime t ON t.DateTimeValue = DATEADD(MINUTE, 
        (DATEPART(MINUTE, s.ProcessedDateTime) / 15) * 15 - DATEPART(MINUTE, s.ProcessedDateTime),
        s.ProcessedDateTime)
    INNER JOIN DimGeography g ON g.LocationID = s.WarehouseID
    INNER JOIN DimProductGravity p ON p.SKU = s.SKU
    WHERE s.ProcessedDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.OperationID = s.OperationID
        )
END
GO

-- Calculate product gravity scores
CREATE PROCEDURE [dbo].[CalculateProductGravity]
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on recent activity
    UPDATE p
    SET 
        PickFrequencyScore = CASE 
            WHEN pick_count > 1000 THEN 100
            WHEN pick_count > 500 THEN 75
            WHEN pick_count > 100 THEN 50
            WHEN pick_count > 10 THEN 25
            ELSE 10
        END,
        ValueScore = CASE 
            WHEN avg_value > 10000 THEN 100
            WHEN avg_value > 1000 THEN 75
            WHEN avg_value > 100 THEN 50
            ELSE 25
        END,
        FragilityScore = CASE 
            WHEN p.IsFragile = 1 THEN 100
            WHEN p.IsPerishable = 1 THEN 75
            ELSE 25
        END,
        LastGravityCalculation = GETDATE()
    FROM DimProductGravity p
    CROSS APPLY (
        SELECT 
            COUNT(*) as pick_count,
            AVG(CAST(Quantity AS DECIMAL(10,2))) as avg_value
        FROM FactWarehouseOperations f
        WHERE f.ProductKey = p.ProductKey
            AND f.OperationType = 'Picking'
            AND f.ProcessedDateTime > DATEADD(DAY, -30, GETDATE())
    ) stats
    
    -- Assign gravity zones
    UPDATE DimProductGravity
    SET GravityZone = CASE 
        WHEN (PickFrequencyScore + ValueScore + FragilityScore) / 3.0 > 75 THEN 'High'
        WHEN (PickFrequencyScore + ValueScore + FragilityScore) / 3.0 > 40 THEN 'Medium'
        ELSE 'Low'
    END,
    RecommendedStorageZone = CASE 
        WHEN (PickFrequencyScore + ValueScore + FragilityScore) / 3.0 > 75 THEN 'Zone-A-FastPick'
        WHEN (PickFrequencyScore + ValueScore + FragilityScore) / 3.0 > 40 THEN 'Zone-B-Standard'
        ELSE 'Zone-C-BulkStorage'
    END
END
GO
```

3. **Set up row-level security:**

```sql
-- Create security schema
CREATE SCHEMA security
GO

CREATE TABLE security.UserRoles (
    UserID VARCHAR(100) PRIMARY KEY,
    UserEmail VARCHAR(200),
    RoleName VARCHAR(50), -- Executive, Supervisor, User
    GeographyFilter VARCHAR(200), -- NULL for all access
    DepartmentFilter VARCHAR(100) -- Fleet, Warehouse, Both
)
GO

-- Create security function
CREATE FUNCTION security.fn_SecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    WHERE 
        -- Admins see everything
        EXISTS (
            SELECT 1 FROM security.UserRoles
            WHERE UserEmail = USER_NAME()
                AND RoleName = 'Executive'
        )
        OR
        -- Filtered users see their geography
        EXISTS (
            SELECT 1 
            FROM security.UserRoles ur
            INNER JOIN DimGeography g ON g.Region = ur.GeographyFilter
            WHERE ur.UserEmail = USER_NAME()
                AND g.GeographyKey = @GeographyKey
        )
        OR
        -- Users with no filter see everything
        EXISTS (
            SELECT 1 FROM security.UserRoles
            WHERE UserEmail = USER_NAME()
                AND GeographyFilter IS NULL
        )
)
GO

-- Apply security policy
CREATE SECURITY POLICY security.WarehouseSecurityPolicy
ADD FILTER PREDICATE security.fn_SecurityPredicate(GeographyKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON)
GO
```

## Power BI Configuration

### Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Configure connection:

```
Server: ${SQL_SERVER_HOSTNAME}
Database: LogiFleetPulse
Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)
```

### Create Key Measures (DAX)

```dax
// Total Warehouse Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Average Cycle Time
Avg Cycle Time = 
AVERAGE(FactWarehouseOperations[CycleTimeMinutes])

// Dwell Time by Gravity Zone
Dwell Time by Zone = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    ALLEXCEPT(DimProductGravity, DimProductGravity[GravityZone])
)

// Fleet Efficiency Ratio
Fleet Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
)

// Cross-Fact KPI: Dwell Impact on Fleet Delay
Dwell Impact Score = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgDelay = AVERAGE(FactFleetTrips[DelayMinutes])
RETURN
    AvgDwell * AvgDelay / 60 -- Hours of combined inefficiency

// Predictive Bottleneck Index
Bottleneck Index = 
VAR CurrentUtilization = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactWarehouseOperations), 
            FactWarehouseOperations[OperationType] = "Picking"),
        CALCULATE(COUNTROWS(FactWarehouseOperations))
    )
VAR HistoricalPeak = 
    CALCULATE(
        MAX(FactWarehouseOperations[Quantity]),
        DATESINPERIOD(DimTime[DateTimeValue], MAX(DimTime[DateTimeValue]), -30, DAY)
    )
RETURN
    CurrentUtilization * 100 + (HistoricalPeak * 0.1)

// Gravity Zone Compliance
Zone Compliance % = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[StorageZone] = DimProductGravity[RecommendedStorageZone]
    ),
    COUNTROWS(FactWarehouseOperations)
) * 100
```

### Time Intelligence Measures

```dax
// Previous Period Comparison
Cycle Time PP = 
CALCULATE(
    [Avg Cycle Time],
    DATEADD(DimTime[DateTimeValue], -1, MONTH)
)

Cycle Time Change % = 
DIVIDE(
    [Avg Cycle Time] - [Cycle Time PP],
    [Cycle Time PP]
) * 100

// Rolling Averages
Avg Dwell 7D = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    DATESINPERIOD(DimTime[DateTimeValue], LASTDATE(DimTime[DateTimeValue]), -7, DAY)
)

// Year-over-Year
Fleet Trips YoY = 
CALCULATE(
    COUNTROWS(FactFleetTrips),
    SAMEPERIODLASTYEAR(DimTime[DateTimeValue])
)
```

## Real-World Usage Patterns

### 1. Daily Operations Dashboard

Create a report page with:

```dax
// KPI Cards
Today's Operations = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    DimTime[DateTimeValue] >= TODAY()
)

Active Fleet Count = 
CALCULATE(
    DISTINCTCOUNT(FactFleetTrips[VehicleID]),
    DimTime[DateTimeValue] >= TODAY()
)

// Heat Map Visual (Matrix)
// Rows: Hour of Day
// Columns: Day of Week
// Values: Operation Count
```

### 2. Gravity Zone Optimization Query

```sql
-- Identify misplaced products
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityZone,
    p.RecommendedStorageZone,
    COUNT(f.OperationKey) AS PickCount,
    AVG(f.CycleTimeMinutes) AS AvgPickTime,
    CASE 
        WHEN f.StorageZone <> p.RecommendedStorageZone 
        THEN 'Relocate to ' + p.RecommendedStorageZone
        ELSE 'Optimized'
    END AS Recommendation
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations f ON f.ProductKey = p.ProductKey
WHERE f.OperationType = 'Picking'
    AND f.ProcessedDateTime > DATEADD(DAY, -7, GETDATE())
GROUP BY 
    p.SKU, p.ProductName, p.GravityZone, 
    p.RecommendedStorageZone, f.StorageZone
HAVING COUNT(f.OperationKey) > 10
ORDER BY AvgPickTime DESC
```

### 3. Fleet Maintenance Triage

```sql
-- Adaptive triage scoring
WITH FleetHealth AS (
    SELECT 
        VehicleID,
        COUNT(*) AS TripCount,
        AVG(IdleTimeMinutes) AS AvgIdleTime,
        AVG(FuelConsumedLiters / NULLIF(TripDistanceKm, 0)) AS FuelEfficiency,
        SUM(CASE WHEN DelayMinutes > 0 THEN 1 ELSE 0 END) AS DelayCount,
        MAX(TripEndDateTime) AS LastTrip
    FROM FactFleetTrips
    WHERE TripStartDateTime > DATEADD(DAY, -30, GETDATE())
    GROUP BY VehicleID
)
SELECT 
    VehicleID,
    TripCount,
    AvgIdleTime,
    FuelEfficiency,
    DelayCount,
    -- Priority score: higher = more urgent
    (
        (CASE WHEN AvgIdleTime > 30 THEN 25 ELSE 0 END) +
        (CASE WHEN FuelEfficiency > 0.15 THEN 25 ELSE 0 END) +
        (CASE WHEN DelayCount > 5 THEN 30 ELSE 0 END) +
        (CASE WHEN DATEDIFF(DAY, LastTrip, GETDATE()) > 7 THEN 20 ELSE 0 END)
    ) AS MaintenancePriority,
    CASE 
        WHEN AvgIdleTime > 30 THEN 'High idle time'
        WHEN FuelEfficiency > 0.15 THEN 'Poor fuel efficiency'
        WHEN DelayCount > 5 THEN 'Frequent delays'
        ELSE 'Routine check'
    END AS PrimaryIssue
FROM FleetHealth
WHERE TripCount > 10
ORDER BY MaintenancePriority DESC
```

### 4. Automated Alerting

```sql
CREATE PROCEDURE [dbo].[CheckAlertsAndNotify]
AS
BEGIN
    -- Create alert log table if not exists
    IF OBJECT_ID('dbo.AlertLog', 'U') IS NULL
    BEGIN
        CREATE TABLE dbo.AlertLog (
            AlertID INT IDENTITY(1,1) PRIMARY KEY,
            AlertType VARCHAR(100),
            AlertMessage NVARCHAR(500),
            Severity VARCHAR(20),
            TriggeredDateTime DATETIME DEFAULT GETDATE(),
            IsSent BIT DEFAULT 0
        )
    END
    
    -- Check for high dwell time
    INSERT INTO dbo.AlertLog (AlertType, AlertMessage, Severity)
    SELECT 
        'High Dwell Time',
        'Product ' + p.SKU + ' has dwell time of ' + 
        CAST(AVG(f.DwellTimeHours) AS VARCHAR(10)) + ' hours',
        'Warning'
    FROM FactWarehouseOperations f
    INNER JOIN DimProductGravity p ON p.ProductKey = f.ProductKey
    WHERE f.ProcessedDateTime > DATEADD(HOUR, -1, GETDATE())
        AND f.DwellTimeHours > 72
    GROUP BY p.SKU
    HAVING AVG(f.DwellTimeHours) > 72
    
    -- Check for fleet idle time threshold
    INSERT INTO dbo.AlertLog (AlertType, AlertMessage, Severity)
    SELECT 
        'Fleet Idle Threshold Exceeded',
        'Vehicle ' + VehicleID + ' idle time: ' + 
        CAST(IdleTimeMinutes AS VARCHAR(10)) + ' minutes',
        'Critical'
    FROM FactFleetTrips
    WHERE TripEndDateTime > DATEADD(HOUR, -1, GETDATE())
        AND (IdleTimeMinutes / NULLIF(TripDurationMinutes, 0)) > 0.15
    
    -- Check for gravity zone misalignment
    INSERT INTO dbo.AlertLog (AlertType, AlertMessage, Severity)
    SELECT TOP 10
        'Gravity Zone Mismatch',
        'Product ' + p.SKU + ' in ' + f.StorageZone + 
        ' should be in ' + p.RecommendedStorageZone,
        'Info'
    FROM FactWarehouseOperations f
    INNER JOIN DimProductGravity p ON p.ProductKey = f.ProductKey
    WHERE f.ProcessedDateTime > DATEADD(HOUR, -4, GETDATE())
        AND f.StorageZone <> p.RecommendedStorageZone
        AND p.GravityZone = 'High'
    GROUP BY p.SKU, f.StorageZone, p.RecommendedStorageZone
END
GO
```

## Configuration

### Connection String Template

```json
{
  "sql_server": {
    "hostname": "${SQL_SERVER_HOSTNAME}",
    "database": "LogiFleetPulse",
    "authentication": "ActiveDirectoryIntegrated",
    "encrypt": true,
    "trustServerCertificate": false
  },
  "power_bi": {
    "workspace_id": "${POWERBI_WORKSPACE_ID}",
    "refresh_schedule": "*/15 * * * *",
    "gateway_id": "${POWERBI_GATEWAY_ID}"
  },
  "external_apis": {
    "weather_api_url": "${WEATHER_API_URL}",
    "weather_api_key": "${WEATHER_API_KEY}",
    "traffic_api_url": "${TRAFFIC_API_URL}",
    "traffic_api_key": "${TRAFFIC_API_KEY}"
  },
  "alerting": {
    "smtp_server": "${SMTP_SERVER}",
    "smtp_port": 587,
    "from_email": "${ALERT_FROM_EMAIL}",
    "alert_recipients": "${ALERT_RECIPIENTS}"
  }
}
```

### Environment Variables

```bash
# SQL Server
export SQL_SERVER_HOSTNAME="logifleet-prod.database.windows.net"
export SQL_SERVER_USERNAME="${DB_USERNAME}"
export SQL_SERVER_PASSWORD="${DB_PASSWORD}"

# Power BI
export POWERBI_WORKSPACE_ID="your-workspace-guid"
export POWERBI_GATEWAY_ID="your-gateway-guid"
export POWERBI_CLIENT_ID="${AZURE_CLIENT_ID}"
export POWERBI_CLIENT_SECRET="${AZURE_CLIENT_SECRET}"

# External APIs
export WEATHER_API_KEY="${YOUR_WEATHER_API_KEY}"
export TRAFFIC_API_KEY="${YOUR_TRAFFIC_API_KEY}"

# Alerts
export SMTP_SERVER="smtp.office365.com"
export ALERT_FROM_EMAIL="logifleet-alerts@yourcompany.com"
export ALERT_RECIPIENTS="ops-team@yourcompany.com,logistics-mgr@yourcompany.com"
```

## Troubleshooting

### Common Issues

**1. Slow dashboard refresh times:**

```sql
-- Add columnstore indexes for large fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, Quantity, 
    CycleTimeMinutes, DwellTimeHours
)

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleet
ON FactFleetTrips (
    TimeKey, OriginGeographyKey, DestinationGeographyKey,
    TripDistanceKm, TripDurationMinutes, IdleTimeMinutes
)
```

**2. Time dimension mismatch (15-minute buckets):**

```sql
-- Verify time key generation
SELECT 
    ProcessedDateTime,
    DATEADD(MINUTE, 
        (DATEPART(MINUTE, ProcessedDateTime) / 15) * 15 - DATEPART(MINUTE, ProcessedDateTime),
        ProcessedDateTime) AS RoundedTime,
    CAST(FORMAT(
        DATEADD(MINUTE, 
            (DATEPART(MINUTE, ProcessedDateTime) / 15) * 15 - DATEPART(MINUTE, ProcessedDateTime),
            ProcessedDateTime), 
        'yyyyMMddHHmm') AS INT) AS TimeKey
FROM FactWarehouseOperations
WHERE ProcessedDateTime > DATEADD(DAY, -1, GETDATE())
```

**3. Row-level security not working:**

```sql
-- Debug security predicate
SELECT 
    USER_NAME() AS CurrentUser,
    ur.RoleName,
    ur.GeographyFilter,
    g.Region,
    g.GeographyKey
FROM security.UserRoles ur
CROSS JOIN DimGeography g
WHERE ur.UserEmail = USER_NAME()
    AND (ur.GeographyFilter IS NULL OR g.Region = ur.GeographyFilter)
```

**4. Gravity scores not updating:**

```sql
-- Force recalculation
EXEC dbo.CalculateProductGravity

-- Schedule as SQL Agent job
USE msdb
GO

EXEC sp_add_job @job_name = 'Update Product Gravity Scores'
EXEC sp_add_jobstep 
    @job_name = 'Update Product Gravity Scores',
    @step_name = 'Calculate Gravity',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC dbo.CalculateProductGravity'

EXEC sp_add_schedule 
    @schedule_name = 'Daily at 2 AM',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @active_start_time = 020000

EXEC sp_attach_schedule 
    @job_name = 'Update Product Gravity Scores',
    @schedule_name = 'Daily at 2 AM'
```

**5. Power BI gateway timeout:**

```dax
// Use query folding-friendly measures
// Avoid calculated columns in DirectQuery mode

// Instead of calculated column:
// ProductValue = Product[UnitPrice] * Warehouse[Quantity]

// Use measure:
Total Product Value = 
SUMX(
    FactWarehouseOperations,
    RELATED(DimProductGravity[UnitValue]) * FactWarehouseOperations[Quantity]
)
```

## Best Practices

1. **Always use time-phased dimensions** for temporal analysis
2. **Run gravity recalculation daily** to keep zone recommendations current
3. **Set up incremental refresh** in Power BI for large fact tables
4. **Use DirectQuery** for operational dashboards, Import for executive reports
5. **Monitor query performance** with SQL Server DMVs
6. **Test row-level security** with different user roles before deployment
7. **Document custom measures** with business context in descriptions
8. **Create composite indexes** on commonly filtered columns (TimeKey + GeographyKey)

## Advanced Pattern: Cross-Fact Analysis

```sql
-- Combined warehouse and fleet performance
WITH WarehouseMetrics AS (
    SELECT 
        g.LocationName,
        AVG(f.DwellTimeHours) AS AvgDwell,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations f
    INNER JOIN DimGeography g ON g.GeographyKey = f.GeographyKey
    WHERE f.ProcessedDateTime > DATEADD(DAY, -7, GETDATE())
    GROUP BY g.LocationName
),
Fleet
