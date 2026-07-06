---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for multi-modal logistics intelligence with warehouse operations, fleet telemetry, and cross-fact KPI harmonization
triggers:
  - set up logifleet pulse logistics dashboard
  - deploy supply chain analytics warehouse schema
  - configure power bi logistics intelligence template
  - implement warehouse gravity zone optimization
  - create multi-fact star schema for fleet data
  - build real-time logistics kpi dashboard
  - integrate warehouse and fleet analytics
  - set up predictive bottleneck detection
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data signals into a single semantic data layer. Built on MS SQL Server for data warehousing and Power BI for visualization, it provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Warehouse Gravity Zones™** that optimize storage layout based on pick frequency and item value
- **Cross-fact KPI harmonization** connecting inventory turnover to fuel consumption
- **Predictive bottleneck detection** using time-phased simulation scenarios
- **Real-time dashboards** with 15-minute refresh cycles
- **Role-based access control** with row-level security

The platform is designed for logistics operators, 3PL providers, retail chains, and distributors who need unified visibility across warehouse and fleet operations.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telemetry feeds, or CSV/Excel files

### Step 1: Deploy SQL Schema

Clone the repository and locate the SQL deployment script:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

Connect to your SQL Server instance and execute the schema script:

```sql
-- Run this in SQL Server Management Studio (SSMS) or Azure Data Studio
-- Ensure you're connected to the target database

:setvar DatabaseName "LogiFleetPulse"
USE master;
GO

IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = '$(DatabaseName)')
BEGIN
    CREATE DATABASE $(DatabaseName);
END
GO

USE $(DatabaseName);
GO

-- Deploy fact tables
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    PackingTimeSeconds INT,
    EmployeeKey INT,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT,
    DistanceMiles DECIMAL(10,2),
    FuelGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReasonKey INT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE TABLE FactCrossDock (
    CrossDockID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripID BIGINT,
    OutboundTripID BIGINT,
    TransferTimeMinutes INT,
    SkipStorage BIT DEFAULT 1,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey)
);

-- Deploy dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay INT,
    DayOfWeek NVARCHAR(20),
    WeekOfYear INT,
    MonthName NVARCHAR(20),
    Quarter INT,
    FiscalYear INT,
    FiscalPeriod INT,
    IsWeekend BIT,
    IsHoliday BIT
);

CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value weight / lead time
    PickFrequencyRank INT,
    ValueTier NVARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityScore DECIMAL(3,2),
    OptimalZone NVARCHAR(50) -- 'HighGravity', 'MediumGravity', 'LowGravity'
);

CREATE TABLE DimWarehouse (
    WarehouseKey INT IDENTITY(1,1) PRIMARY KEY,
    WarehouseID NVARCHAR(50) UNIQUE NOT NULL,
    WarehouseName NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    TotalCapacitySqFt INT,
    CurrentUtilizationPct DECIMAL(5,2)
);

CREATE TABLE DimVehicle (
    VehicleKey INT IDENTITY(1,1) PRIMARY KEY,
    VehicleID NVARCHAR(50) UNIQUE NOT NULL,
    VehicleType NVARCHAR(50), -- 'Box Truck', 'Flatbed', 'Refrigerated'
    Make NVARCHAR(50),
    Model NVARCHAR(50),
    Year INT,
    MaxLoadCapacityLbs INT,
    FuelType NVARCHAR(20),
    MaintenancePriorityScore DECIMAL(5,2)
);

CREATE TABLE DimRoute (
    RouteKey INT IDENTITY(1,1) PRIMARY KEY,
    RouteID NVARCHAR(50) UNIQUE NOT NULL,
    OriginWarehouseKey INT,
    DestinationKey INT,
    RouteName NVARCHAR(200),
    PlannedDistanceMiles DECIMAL(10,2),
    AverageTrafficDelayMinutes INT
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey);
```

### Step 2: Create Stored Procedures for Data Loading

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartDateTime DATETIME,
    @EndDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external WMS staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        DwellTimeMinutes, PickRateUnitsPerHour, PackingTimeSeconds, EmployeeKey
    )
    SELECT 
        CONVERT(INT, FORMAT(op.OperationDateTime, 'yyyyMMddHHmm')) AS TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        op.OperationType,
        DATEDIFF(MINUTE, op.StartTime, op.EndTime) AS DwellTimeMinutes,
        CASE 
            WHEN op.OperationType = 'Pick' THEN op.UnitsProcessed / (DATEDIFF(MINUTE, op.StartTime, op.EndTime) / 60.0)
            ELSE NULL 
        END AS PickRateUnitsPerHour,
        DATEDIFF(SECOND, op.StartTime, op.EndTime) AS PackingTimeSeconds,
        e.EmployeeKey
    FROM Staging.WarehouseOperations op
    INNER JOIN DimWarehouse w ON op.WarehouseID = w.WarehouseID
    INNER JOIN DimProductGravity p ON op.SKU = p.SKU
    LEFT JOIN DimEmployee e ON op.EmployeeID = e.EmployeeID
    WHERE op.OperationDateTime BETWEEN @StartDateTime AND @EndDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.TimeKey = CONVERT(INT, FORMAT(op.OperationDateTime, 'yyyyMMddHHmm'))
                AND f.ProductKey = p.ProductKey
        );
END;
GO

-- Calculate gravity scores for products
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(f.OperationID) AS PickFrequency,
            AVG(f.DwellTimeMinutes) AS AvgDwellTime,
            p.ValueTier
        FROM DimProductGravity p
        LEFT JOIN FactWarehouseOperations f ON p.ProductKey = f.ProductKey
        WHERE f.OperationType = 'Pick'
            AND f.TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd0000'))
        GROUP BY p.ProductKey, p.ValueTier
    )
    UPDATE p
    SET 
        p.GravityScore = CASE 
            WHEN m.AvgDwellTime > 0 THEN 
                (m.PickFrequency * 
                 CASE p.ValueTier 
                    WHEN 'High' THEN 3.0 
                    WHEN 'Medium' THEN 2.0 
                    ELSE 1.0 
                 END) / (m.AvgDwellTime / 60.0)
            ELSE 0
        END,
        p.PickFrequencyRank = DENSE_RANK() OVER (ORDER BY m.PickFrequency DESC),
        p.OptimalZone = CASE 
            WHEN (m.PickFrequency * CASE p.ValueTier WHEN 'High' THEN 3.0 WHEN 'Medium' THEN 2.0 ELSE 1.0 END) > 100 THEN 'HighGravity'
            WHEN (m.PickFrequency * CASE p.ValueTier WHEN 'High' THEN 3.0 WHEN 'Medium' THEN 2.0 ELSE 1.0 END) > 30 THEN 'MediumGravity'
            ELSE 'LowGravity'
        END
    FROM DimProductGravity p
    INNER JOIN ProductMetrics m ON p.ProductKey = m.ProductKey;
END;
GO

-- Automated alerting procedure
CREATE PROCEDURE sp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check fleet idling threshold
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
            AND (f.IdleTimeMinutes * 1.0 / NULLIF(f.DistanceMiles / 50.0 * 60, 0)) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idling time exceeds 15% of trip duration for one or more vehicles today.';
        -- Insert into AlertLog table or send via email
        INSERT INTO AlertLog (AlertDateTime, AlertType, AlertMessage)
        VALUES (GETDATE(), 'Fleet Idling', @AlertMessage);
    END
    
    -- Check warehouse dwell time anomaly
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.Date >= CAST(DATEADD(DAY, -1, GETDATE()) AS DATE)
            AND f.OperationType = 'Pick'
            AND f.DwellTimeMinutes > 72 * 60
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Items with over 72 hours dwell time detected in warehouse.';
        INSERT INTO AlertLog (AlertDateTime, AlertType, AlertMessage)
        VALUES (GETDATE(), 'Warehouse Dwell', @AlertMessage);
    END
END;
GO
```

### Step 3: Configure Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter your SQL Server connection details:
   - Server: `your-sql-server.database.windows.net` or `localhost`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server credentials

3. The template will automatically detect relationships between fact and dimension tables

4. Configure data refresh schedule (Power BI Service):
   - Navigate to dataset settings
   - Set refresh frequency to every 15 minutes (requires Pro or Premium)
   - Configure gateway if using on-premises SQL Server

## Key Configuration Files

### config.json (Data Source Configuration)

Create a `config.json` file for external data source connections:

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SqlPassword",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "telemetryAPI": {
    "endpoint": "https://api.fleet-telemetry.example.com/v1",
    "apiKey": "${TELEMETRY_API_KEY}",
    "refreshIntervalSeconds": 900
  },
  "wmsIntegration": {
    "type": "REST",
    "endpoint": "${WMS_API_ENDPOINT}",
    "authToken": "${WMS_AUTH_TOKEN}"
  },
  "weatherAPI": {
    "endpoint": "https://api.weather.gov",
    "enableCorrelation": true
  },
  "alerting": {
    "smtpServer": "${SMTP_SERVER}",
    "smtpPort": 587,
    "fromEmail": "${ALERT_FROM_EMAIL}",
    "toEmails": ["logistics@example.com", "operations@example.com"]
  }
}
```

### Row-Level Security Configuration

```sql
-- Create role-based security
CREATE ROLE WarehouseManager;
CREATE ROLE FleetSupervisor;
CREATE ROLE ExecutiveView;

-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@UserRole NVARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    WHERE 
        @UserRole = USER_NAME()
        OR IS_MEMBER(@UserRole) = 1
);
GO

-- Apply security policy to fact tables
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate('WarehouseManager')
    ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate('FleetSupervisor')
    ON dbo.FactFleetTrips
WITH (STATE = ON);
GO
```

## Common Usage Patterns

### Pattern 1: Cross-Fact KPI Analysis (SQL Query)

Query linking warehouse dwell time with fleet idle time for same SKU:

```sql
-- Find products with high dwell time AND high fleet idle time on delivery
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(w.OperationID) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
        AND w.OperationType IN ('Putaway', 'Pick')
    GROUP BY p.SKU, p.ProductName
    HAVING AVG(w.DwellTimeMinutes) > 240
),
FleetIdle AS (
    SELECT 
        p.SKU,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        COUNT(f.TripID) AS TripCount
    FROM FactFleetTrips f
    INNER JOIN FactCrossDock cd ON f.TripID = cd.OutboundTripID
    INNER JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.SKU
    HAVING AVG(f.IdleTimeMinutes) > 20
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellMinutes,
    wd.OperationCount AS WarehouseOps,
    fi.AvgIdleMinutes,
    fi.TripCount AS FleetTrips,
    (wd.AvgDwellMinutes + fi.AvgIdleMinutes) AS TotalFrictionMinutes
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.SKU = fi.SKU
ORDER BY TotalFrictionMinutes DESC;
```

### Pattern 2: Warehouse Gravity Zone Recommendations

```sql
-- Generate rearrangement recommendations based on gravity score changes
WITH CurrentLayout AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.OptimalZone AS CurrentZone,
        p.GravityScore AS CurrentGravity,
        wl.ActualZone,
        wl.StorageLocationID
    FROM DimProductGravity p
    LEFT JOIN WarehouseLayout wl ON p.SKU = wl.SKU
),
GravityShift AS (
    SELECT 
        cl.*,
        CASE 
            WHEN cl.CurrentGravity > 80 THEN 'HighGravity'
            WHEN cl.CurrentGravity > 30 THEN 'MediumGravity'
            ELSE 'LowGravity'
        END AS RecommendedZone
    FROM CurrentLayout cl
)
SELECT 
    gs.SKU,
    gs.CurrentZone,
    gs.ActualZone,
    gs.RecommendedZone,
    gs.CurrentGravity,
    CASE 
        WHEN gs.ActualZone <> gs.RecommendedZone THEN 'RELOCATE'
        ELSE 'OK'
    END AS Action,
    CASE 
        WHEN gs.ActualZone = 'LowGravity' AND gs.RecommendedZone = 'HighGravity' 
            THEN 'PRIORITY: High-value fast-mover in wrong zone'
        WHEN gs.ActualZone = 'HighGravity' AND gs.RecommendedZone = 'LowGravity'
            THEN 'Opportunity: Free up premium space'
        ELSE 'Standard relocation'
    END AS RecommendationNote
FROM GravityShift gs
WHERE gs.ActualZone <> gs.RecommendedZone
ORDER BY 
    CASE 
        WHEN gs.ActualZone = 'LowGravity' AND gs.RecommendedZone = 'HighGravity' THEN 1
        WHEN gs.ActualZone = 'MediumGravity' AND gs.RecommendedZone = 'HighGravity' THEN 2
        ELSE 3
    END,
    gs.CurrentGravity DESC;
```

### Pattern 3: Predictive Bottleneck Detection

```sql
-- Simulate capacity impact on fleet utilization
WITH CapacityScenario AS (
    SELECT 
        w.WarehouseKey,
        w.WarehouseName,
        w.CurrentUtilizationPct,
        0.95 AS TargetUtilizationPct -- Simulate 95% capacity
    FROM DimWarehouse w
),
HistoricalFleetLoad AS (
    SELECT 
        w.WarehouseKey,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        AVG(f.LoadingTimeMinutes) AS AvgLoadingMinutes,
        COUNT(f.TripID) AS TripCount
    FROM FactFleetTrips f
    INNER JOIN DimRoute r ON f.RouteKey = r.RouteKey
    INNER JOIN DimWarehouse w ON r.OriginWarehouseKey = w.WarehouseKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
    GROUP BY w.WarehouseKey
)
SELECT 
    cs.WarehouseName,
    cs.CurrentUtilizationPct,
    cs.TargetUtilizationPct,
    hfl.AvgLoadingMinutes AS CurrentAvgLoading,
    -- Model: Loading time increases exponentially above 85% capacity
    hfl.AvgLoadingMinutes * 
        POWER(1.5, (cs.TargetUtilizationPct - cs.CurrentUtilizationPct) / 0.1) 
        AS ProjectedAvgLoading,
    hfl.AvgIdleMinutes AS CurrentAvgIdle,
    hfl.AvgIdleMinutes * 
        POWER(1.3, (cs.TargetUtilizationPct - cs.CurrentUtilizationPct) / 0.1)
        AS ProjectedAvgIdle,
    hfl.TripCount,
    CASE 
        WHEN (hfl.AvgLoadingMinutes * POWER(1.5, (cs.TargetUtilizationPct - cs.CurrentUtilizationPct) / 0.1)) > 45
            THEN 'HIGH RISK: Projected loading time exceeds 45 min threshold'
        WHEN (hfl.AvgLoadingMinutes * POWER(1.5, (cs.TargetUtilizationPct - cs.CurrentUtilizationPct) / 0.1)) > 30
            THEN 'MODERATE RISK: Consider expansion or process optimization'
        ELSE 'LOW RISK'
    END AS BottleneckAssessment
FROM CapacityScenario cs
INNER JOIN HistoricalFleetLoad hfl ON cs.WarehouseKey = hfl.WarehouseKey
ORDER BY ProjectedAvgLoading DESC;
```

### Pattern 4: Power BI DAX Measures

Common DAX measures to add to your Power BI model:

```dax
// Cross-Fact: Warehouse Efficiency Ratio
Warehouse_Efficiency_Ratio = 
VAR TotalPickTime = SUM(FactWarehouseOperations[DwellTimeMinutes])
VAR TotalFleetIdle = 
    CALCULATE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        TREATAS(
            VALUES(FactWarehouseOperations[TimeKey]),
            DimTime[TimeKey]
        )
    )
RETURN
DIVIDE(TotalPickTime, TotalPickTime + TotalFleetIdle, 0)

// Adaptive Fleet Triage Score
Fleet_Triage_Score = 
VAR LoadPriority = 
    CALCULATE(
        AVERAGE(DimProductGravity[GravityScore]),
        TREATAS(
            VALUES(FactCrossDock[ProductKey]),
            DimProductGravity[ProductKey]
        )
    )
VAR VehicleHealth = 1 - (DimVehicle[MaintenancePriorityScore] / 100)
VAR DelayImpact = 
    DIVIDE(
        SUM(FactFleetTrips[DelayMinutes]),
        SUM(FactFleetTrips[DistanceMiles]) / 50 * 60,
        0
    )
RETURN
(LoadPriority * 0.5 + VehicleHealth * 0.3 + (1 - DelayImpact) * 0.2) * 100

// Time Intelligence: Rolling 7-Day Dwell Average
Rolling_7Day_Dwell = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(
        DimTime[Date],
        LASTDATE(DimTime[Date]),
        -7,
        DAY
    )
)

// Gravity Zone Compliance Rate
Gravity_Zone_Compliance = 
VAR ProductsInCorrectZone = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        WarehouseLayout[ActualZone] = DimProductGravity[OptimalZone]
    )
VAR TotalProducts = COUNTROWS(FactWarehouseOperations)
RETURN
DIVIDE(ProductsInCorrectZone, TotalProducts, 0)
```

## Troubleshooting

### Issue: Power BI Connection Timeout

**Symptom**: Power BI reports "Query timeout expired" when refreshing data.

**Solution**: 
1. Increase query timeout in Power BI:
   - File → Options and settings → Options → Data Load
   - Set "Command timeout in minutes" to 10 or higher

2. Optimize SQL queries by adding indexes:
```sql
-- Add covering indexes for common query patterns
CREATE NONCLUSTERED INDEX IX_FactWH_Time_Product_Includes
ON FactWarehouseOperations(TimeKey, ProductKey)
INCLUDE (DwellTimeMinutes, PickRateUnitsPerHour);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time_Vehicle_Includes
ON FactFleetTrips(TimeKey, VehicleKey)
INCLUDE (IdleTimeMinutes, FuelGallons, DistanceMiles);
```

### Issue: Gravity Scores Not Updating

**Symptom**: `DimProductGravity.GravityScore` remains static despite new operations.

**Solution**: Schedule the gravity score update procedure:

```sql
-- Create SQL Server Agent job (or use Azure Automation)
USE msdb;
GO

EXEC sp_add_job 
    @job_name = N'Update Product Gravity Scores',
    @enabled = 1,
    @description = N'Recalculate gravity scores daily based on 90-day rolling window';

EXEC sp_add_jobstep 
    @job_name = N'Update Product Gravity Scores',
    @step_name = N'Execute Gravity Procedure',
    @subsystem = N'TSQL',
    @database_name = N'LogiFleetPulse',
    @command = N'EXEC sp_UpdateProductGravityScores;';

EXEC sp_add_schedule 
    @schedule_name = N'Daily at 2 AM',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @active_start_time = 020000; -- 2:00 AM

EXEC sp_attach_schedule 
    @job_name = N'Update Product Gravity Scores',
    @schedule_name = N'Daily at 2 AM';
GO
```

### Issue: Cross-Fact Queries Running Slowly

**Symptom**: Queries joining `FactWarehouseOperations` and `FactFleetTrips` take over 30 seconds.

**Solution**: Create indexed views for common joins:

```sql
-- Create indexed view for warehouse-fleet correlation
CREATE VIEW vw_WarehouseFleetCorrelation
WITH SCHEMABINDING
AS
SELECT 
    t.Date,
    w.WarehouseKey,
    p.ProductKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(ISNULL(wo.DwellTimeMinutes, 0)) AS TotalDwellMinutes,
    AVG(ISNULL(wo.PickRateUnitsPerHour, 0)) AS AvgPickRate
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN dbo.DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
GROUP BY t.Date, w.WarehouseKey, p.ProductKey;
GO

CREATE UNIQUE CLUSTERED INDEX IX_WHFleet_Date_Warehouse_Product
ON vw_WarehouseFleetCorrelation(Date, WarehouseKey, ProductKey);
GO
```

### Issue: Row-Level Security Not Applied in Power BI

**Symptom**: Users see data outside their assigned warehouse/region.

**Solution**: 
1. Ensure security policy is enabled in SQL:
```sql
-- Verify security policy state
SELECT * FROM sys.security_policies WHERE name = 'WarehouseSecurityPolicy';

-- If STATE = 0, enable it
ALTER SECURITY POLICY WarehouseSecurityPolicy WITH (STATE = ON);
```

2. Configure Power BI to use user context:
   - Dataset settings → Data source credentials
   - Select "OAuth2" or "Windows authentication"
   - Enable "DirectQuery" mode (security doesn't work with Import mode for SQL RLS)

3. Map Power BI users to SQL roles:
```sql
-- Add Power BI service account to role
EXEC sp_addrolemember 'WarehouseManager', 'user@company.com';
```

### Issue: Alerts Not Triggering

**Symptom**: `sp_CheckKPIThresholds` runs but no alerts appear in inbox.

**Solution**:
1. Verify alert log is populated:
