---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine with MS SQL Server data warehousing and Power BI dashboards for supply chain optimization
triggers:
  - "set up LogiFleet Pulse analytics"
  - "create supply chain data warehouse"
  - "build logistics Power BI dashboard"
  - "implement multi-fact star schema for fleet tracking"
  - "configure warehouse operations analytics"
  - "query cross-modal supply chain data"
  - "optimize logistics KPI reporting"
  - "analyze fleet and warehouse metrics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What It Does

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a single semantic layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse, fleet, and supplier data
- **Real-time dashboards** with 15-minute refresh intervals
- **Predictive analytics** for bottleneck detection and maintenance prioritization
- **Cross-fact KPI harmonization** (e.g., inventory turnover vs. fuel efficiency)
- **Warehouse Gravity Zones™** for spatial optimization
- **Role-based access control** with row-level security

The platform is designed for 3PL operators, retailers, and distributors managing complex supply chains.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Access to data sources: WMS, TMS, telematics APIs

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script

USE master;
GO

CREATE DATABASE LogiFleetPulse
ON PRIMARY 
(
    NAME = LogiFleetPulse_Data,
    FILENAME = 'D:\SQLData\LogiFleetPulse_Data.mdf',
    SIZE = 10GB,
    FILEGROWTH = 1GB
)
LOG ON
(
    NAME = LogiFleetPulse_Log,
    FILENAME = 'D:\SQLLog\LogiFleetPulse_Log.ldf',
    SIZE = 2GB,
    FILEGROWTH = 512MB
);
GO

USE LogiFleetPulse;
GO

-- Enable snapshot isolation for consistent reads
ALTER DATABASE LogiFleetPulse 
SET ALLOW_SNAPSHOT_ISOLATION ON;
ALTER DATABASE LogiFleetPulse 
SET READ_COMMITTED_SNAPSHOT ON;
GO
```

### Step 2: Create Core Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthNumber TINYINT NOT NULL,
    QuarterNumber TINYINT NOT NULL,
    Year SMALLINT NOT NULL,
    FiscalPeriod NVARCHAR(10),
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0
);
CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON dbo.DimTime;
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE dbo.DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    NodeCode NVARCHAR(50) NOT NULL UNIQUE,
    NodeName NVARCHAR(200) NOT NULL,
    NodeType NVARCHAR(50) NOT NULL, -- 'Warehouse', 'Route', 'Customer', 'Supplier'
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    ParentNodeCode NVARCHAR(50),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    IsActive BIT DEFAULT 1,
    ValidFrom DATETIME2 DEFAULT SYSUTCDATETIME(),
    ValidTo DATETIME2
);
CREATE NONCLUSTERED INDEX IX_Geography_NodeType ON dbo.DimGeography(NodeType) INCLUDE (NodeCode, NodeName);
GO

-- DimProductGravity: Product classification with velocity scoring
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    Subcategory NVARCHAR(100),
    GravityScore DECIMAL(5,2) DEFAULT 0, -- 0-100 scale
    VelocityClass NVARCHAR(20), -- 'Fast', 'Medium', 'Slow', 'Dead'
    FragilityIndex DECIMAL(3,2), -- 0-1 scale
    UnitValue DECIMAL(18,2),
    UnitWeight DECIMAL(10,2),
    StorageRequirement NVARCHAR(50), -- 'Ambient', 'Refrigerated', 'Frozen', 'Hazmat'
    ReorderPoint INT,
    ValidFrom DATETIME2 DEFAULT SYSUTCDATETIME(),
    ValidTo DATETIME2
);
CREATE NONCLUSTERED INDEX IX_Product_Gravity ON dbo.DimProductGravity(GravityScore DESC, VelocityClass);
GO

-- FactWarehouseOperations: Granular warehouse activity tracking
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimProductGravity(ProductKey),
    OperationType NVARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2 NOT NULL,
    OperationEndTime DATETIME2,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    QuantityHandled INT NOT NULL,
    ZoneCode NVARCHAR(50),
    EmployeeID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    DwellTimeHours DECIMAL(10,2), -- Time in storage before next operation
    ErrorFlag BIT DEFAULT 0,
    ErrorReason NVARCHAR(500)
);
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouse ON dbo.FactWarehouseOperations;
CREATE NONCLUSTERED INDEX IX_Warehouse_Time ON dbo.FactWarehouseOperations(TimeKey, WarehouseKey);
GO

-- FactFleetTrips: Fleet telemetry and trip metrics
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimTime(TimeKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimGeography(GeographyKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    TripStartTime DATETIME2 NOT NULL,
    TripEndTime DATETIME2,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(10,2),
    IdleTimeMinutes INT,
    AverageSpeedMPH DECIMAL(5,2),
    LoadWeightLbs DECIMAL(10,2),
    LoadVolumePercent DECIMAL(5,2), -- Truck capacity utilization
    DelayMinutes INT DEFAULT 0,
    DelayReason NVARCHAR(200),
    MaintenanceFlag BIT DEFAULT 0,
    RevenueImpact DECIMAL(18,2)
);
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleet ON dbo.FactFleetTrips;
CREATE NONCLUSTERED INDEX IX_Fleet_Vehicle ON dbo.FactFleetTrips(VehicleID, TripStartTime);
GO
```

### Step 3: Create Bridge Tables

```sql
-- Bridge table for many-to-many relationships between trips and products
CREATE TABLE dbo.BridgeTripProduct (
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES dbo.FactFleetTrips(TripKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES dbo.DimProductGravity(ProductKey),
    Quantity INT NOT NULL,
    WeightLbs DECIMAL(10,2),
    PRIMARY KEY (TripKey, ProductKey)
);
CREATE NONCLUSTERED INDEX IX_Bridge_Product ON dbo.BridgeTripProduct(ProductKey);
GO
```

### Step 4: Configure Data Ingestion

```sql
-- Stored procedure for incremental warehouse data load
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Connect to external WMS source via linked server or OPENROWSET
    -- Replace with your actual connection details
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, QuantityHandled,
        ZoneCode, EmployeeID, EquipmentID, DwellTimeHours, ErrorFlag
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        src.OperationType,
        src.StartTime,
        src.EndTime,
        src.Quantity,
        src.ZoneCode,
        src.EmployeeID,
        src.EquipmentID,
        src.DwellHours,
        CASE WHEN src.ErrorCode IS NOT NULL THEN 1 ELSE 0 END
    FROM OPENROWSET(
        'SQLNCLI',
        'Server=$(WMS_SERVER);Trusted_Connection=yes;',
        'SELECT * FROM WMS.dbo.Operations WHERE LastModified > ''$(LastLoadTime)'''
    ) AS src
    INNER JOIN dbo.DimTime dt ON DATEADD(MINUTE, (DATEDIFF(MINUTE, 0, src.StartTime) / 15) * 15, 0) = dt.FullDateTime
    INNER JOIN dbo.DimGeography dg ON src.WarehouseCode = dg.NodeCode
    INNER JOIN dbo.DimProductGravity dp ON src.SKU = dp.SKU
    WHERE src.StartTime > @LastLoadTime;
    
    -- Update last load timestamp
    -- Store in metadata table (create separately)
END;
GO

-- Schedule the procedure via SQL Agent job
-- Run every 15 minutes to maintain real-time sync
```

### Step 5: Install Power BI Template

1. Download `LogiFleet_Pulse_Master.pbit` from the repository
2. Open in Power BI Desktop
3. Enter connection parameters:
   - SQL Server: `your-server.domain.com`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server
4. Apply credentials and refresh the data model

## Key SQL Queries

### Cross-Fact Analysis: Warehouse Dwell vs Fleet Idle

```sql
-- Identify products with high warehouse dwell time that correlate with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        wo.ProductKey,
        dp.SKU,
        dp.ProductName,
        AVG(wo.DwellTimeHours) AS AvgDwellHours,
        COUNT(*) AS Operations
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimProductGravity dp ON wo.ProductKey = dp.ProductKey
    WHERE wo.OperationStartTime >= DATEADD(MONTH, -3, GETDATE())
        AND wo.OperationType = 'Putaway'
    GROUP BY wo.ProductKey, dp.SKU, dp.ProductName
),
FleetIdle AS (
    SELECT 
        btp.ProductKey,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        COUNT(DISTINCT ft.VehicleID) AS VehiclesAffected
    FROM dbo.FactFleetTrips ft
    INNER JOIN dbo.BridgeTripProduct btp ON ft.TripKey = btp.TripKey
    WHERE ft.TripStartTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY btp.ProductKey
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellHours,
    wd.Operations,
    ISNULL(fi.AvgIdleMinutes, 0) AS AvgFleetIdle,
    ISNULL(fi.VehiclesAffected, 0) AS VehiclesAffected,
    -- Calculate correlation score (simplified)
    CASE 
        WHEN wd.AvgDwellHours > 72 AND ISNULL(fi.AvgIdleMinutes, 0) > 30 THEN 'High Risk'
        WHEN wd.AvgDwellHours > 48 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS RiskCategory
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi ON wd.ProductKey = fi.ProductKey
ORDER BY wd.AvgDwellHours DESC, fi.AvgIdleMinutes DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend product relocations based on gravity score vs current zone
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.VelocityClass,
    wo.ZoneCode AS CurrentZone,
    COUNT(*) AS PickFrequency,
    AVG(wo.DurationMinutes) AS AvgPickTimeMinutes,
    -- Recommended zone based on gravity
    CASE 
        WHEN dp.GravityScore >= 80 THEN 'A' -- High-gravity, near docks
        WHEN dp.GravityScore >= 50 THEN 'B' -- Medium-gravity
        WHEN dp.GravityScore >= 20 THEN 'C' -- Low-gravity
        ELSE 'D' -- Dead stock, deep storage
    END AS RecommendedZone,
    CASE 
        WHEN LEFT(wo.ZoneCode, 1) <> 
             CASE 
                WHEN dp.GravityScore >= 80 THEN 'A'
                WHEN dp.GravityScore >= 50 THEN 'B'
                WHEN dp.GravityScore >= 20 THEN 'C'
                ELSE 'D'
             END 
        THEN 'RELOCATE'
        ELSE 'OK'
    END AS Action
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimProductGravity dp ON wo.ProductKey = dp.ProductKey
WHERE wo.OperationType = 'Picking'
    AND wo.OperationStartTime >= DATEADD(MONTH, -1, GETDATE())
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, dp.VelocityClass, wo.ZoneCode
HAVING COUNT(*) > 10 -- Only products with sufficient pick history
ORDER BY 
    CASE WHEN LEFT(wo.ZoneCode, 1) <> 
         CASE 
            WHEN dp.GravityScore >= 80 THEN 'A'
            WHEN dp.GravityScore >= 50 THEN 'B'
            WHEN dp.GravityScore >= 20 THEN 'C'
            ELSE 'D'
         END 
    THEN 0 ELSE 1 END,
    PickFrequency DESC;
```

### Predictive Fleet Maintenance Queue

```sql
-- Rank vehicles by maintenance urgency using weighted scoring
WITH VehicleMetrics AS (
    SELECT 
        VehicleID,
        COUNT(*) AS RecentTrips,
        SUM(TripDurationMinutes) AS TotalDriveTimeMinutes,
        SUM(IdleTimeMinutes) AS TotalIdleMinutes,
        AVG(LoadWeightLbs) AS AvgLoadWeight,
        SUM(CASE WHEN DelayMinutes > 0 THEN 1 ELSE 0 END) AS DelayCount,
        SUM(RevenueImpact) AS TotalRevenue
    FROM dbo.FactFleetTrips
    WHERE TripStartTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY VehicleID
),
MaintenanceScore AS (
    SELECT 
        VehicleID,
        RecentTrips,
        TotalDriveTimeMinutes,
        TotalIdleMinutes,
        AvgLoadWeight,
        DelayCount,
        TotalRevenue,
        -- Weighted urgency score (higher = more urgent)
        (
            (TotalDriveTimeMinutes / 60.0 / 100) * 0.3 + -- Age factor
            (TotalIdleMinutes / 60.0 / 10) * 0.2 + -- Idle wear
            (CASE WHEN AvgLoadWeight > 40000 THEN 1.5 ELSE 1.0 END) * 0.2 + -- Overload penalty
            (DelayCount * 0.2) + -- Reliability issues
            (TotalRevenue / 10000.0 * 0.1) -- Revenue at risk
        ) AS UrgencyScore
    FROM VehicleMetrics
)
SELECT 
    VehicleID,
    RecentTrips,
    ROUND(TotalDriveTimeMinutes / 60.0, 1) AS TotalDriveHours,
    ROUND(TotalIdleMinutes / 60.0, 1) AS TotalIdleHours,
    ROUND(AvgLoadWeight, 0) AS AvgLoadLbs,
    DelayCount,
    ROUND(TotalRevenue, 0) AS RevenueAtRisk,
    ROUND(UrgencyScore, 2) AS MaintenancePriority,
    CASE 
        WHEN UrgencyScore >= 5 THEN 'CRITICAL - Immediate'
        WHEN UrgencyScore >= 3 THEN 'HIGH - Within 48h'
        WHEN UrgencyScore >= 1.5 THEN 'MEDIUM - Within 1 week'
        ELSE 'LOW - Routine'
    END AS PriorityLevel
FROM MaintenanceScore
ORDER BY UrgencyScore DESC;
```

## Power BI DAX Measures

### Core KPI: Fleet Efficiency Ratio

```dax
// Measure: Fleet Efficiency Ratio
Fleet Efficiency = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceMiles])
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedGallons])
VAR TotalIdle = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalDuration = SUM(FactFleetTrips[TripDurationMinutes])
VAR IdlePercent = DIVIDE(TotalIdle, TotalDuration, 0)
VAR MilesPerGallon = DIVIDE(TotalDistance, TotalFuel, 0)
RETURN
    // Composite score: MPG adjusted for idle penalty
    MilesPerGallon * (1 - IdlePercent)
```

### Cross-Fact: Dwell Time Impact on Revenue

```dax
// Measure: Revenue Impact of Dwell Time
Dwell Revenue Impact = 
VAR HighDwellProducts = 
    FILTER(
        ALL(DimProductGravity),
        CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeHours])) > 72
    )
VAR AffectedTrips = 
    FILTER(
        FactFleetTrips,
        COUNTROWS(
            FILTER(
                BridgeTripProduct,
                BridgeTripProduct[TripKey] = FactFleetTrips[TripKey] &&
                BridgeTripProduct[ProductKey] IN VALUES(HighDwellProducts[ProductKey])
            )
        ) > 0
    )
RETURN
    SUMX(AffectedTrips, FactFleetTrips[RevenueImpact]) * -1
```

### Time Intelligence: Rolling 90-Day Average

```dax
// Measure: 90-Day Rolling Avg Operations
Ops Rolling 90D = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DurationMinutes]),
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -90,
        DAY
    )
)
```

## Configuration

### Environment Variables

Create a `.env` file or use SQL Server configuration:

```ini
# Database Connection
SQL_SERVER=your-server.domain.com
SQL_DATABASE=LogiFleetPulse
SQL_USERNAME=logifleet_svc
SQL_PASSWORD=${SQL_PASSWORD}  # Store in Azure Key Vault or similar

# Data Source Connections
WMS_SERVER=wms-prod.domain.com
TMS_API_ENDPOINT=https://api.tms-provider.com/v2
TELEMETRY_STREAM_URL=https://telemetry.fleet-provider.com/stream

# Alert Configuration
ALERT_EMAIL_SMTP=smtp.office365.com
ALERT_EMAIL_FROM=logistics-alerts@company.com
ALERT_TEAMS_WEBHOOK=${TEAMS_WEBHOOK_URL}

# Refresh Schedule (cron format)
POWERBI_REFRESH_CRON=*/15 * * * *  # Every 15 minutes
```

### Row-Level Security Setup

```sql
-- Create role-based security table
CREATE TABLE dbo.SecurityRoles (
    UserEmail NVARCHAR(200) NOT NULL,
    Role NVARCHAR(50) NOT NULL, -- 'Executive', 'Manager', 'Operator'
    WarehouseAccess NVARCHAR(MAX), -- Comma-separated warehouse codes
    FleetAccess NVARCHAR(MAX), -- Comma-separated vehicle IDs
    PRIMARY KEY (UserEmail, Role)
);
GO

-- Insert sample roles
INSERT INTO dbo.SecurityRoles VALUES 
('john.doe@company.com', 'Manager', 'WH01,WH02', NULL),
('jane.smith@company.com', 'Executive', NULL, NULL); -- NULL = all access
GO

-- Create RLS function
CREATE FUNCTION dbo.fn_SecurityPredicate(@UserEmail NVARCHAR(200))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS AccessGranted
    WHERE 
        @UserEmail IN (SELECT UserEmail FROM dbo.SecurityRoles WHERE Role = 'Executive')
        OR EXISTS (
            SELECT 1 FROM dbo.SecurityRoles sr
            INNER JOIN dbo.DimGeography dg ON 
                ',' + sr.WarehouseAccess + ',' LIKE '%,' + dg.NodeCode + ',%'
            WHERE sr.UserEmail = @UserEmail
        );
GO

-- Apply RLS policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(SYSTEM_USER)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
GO
```

## Common Patterns

### Pattern 1: Streaming Data Ingestion

```sql
-- Use Service Broker or Azure Stream Analytics for real-time feeds
-- Example: Telemetry data from IoT devices

CREATE QUEUE FleetTelemetryQueue;
CREATE SERVICE FleetTelemetryService ON QUEUE FleetTelemetryQueue;
GO

CREATE PROCEDURE dbo.usp_ProcessTelemetryMessage
AS
BEGIN
    DECLARE @message_body NVARCHAR(MAX);
    
    RECEIVE TOP(100) @message_body = CAST(message_body AS NVARCHAR(MAX))
    FROM FleetTelemetryQueue;
    
    -- Parse JSON telemetry
    INSERT INTO dbo.FactFleetTrips (...)
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        JSON_VALUE(@message_body, '$.vehicleId'),
        ...
    FROM OPENJSON(@message_body);
END;
GO
```

### Pattern 2: Automated Alerting

```sql
-- Stored procedure for KPI breach detection
CREATE PROCEDURE dbo.usp_CheckKPIAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check fleet idle time threshold
    IF EXISTS (
        SELECT 1 FROM dbo.FactFleetTrips
        WHERE TripStartTime >= DATEADD(HOUR, -1, GETDATE())
            AND (IdleTimeMinutes * 1.0 / NULLIF(TripDurationMinutes, 0)) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 15% in the last hour';
        
        -- Send email via Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'operations@company.com',
            @subject = 'LogiFleet KPI Alert',
            @body = @AlertMessage;
            
        -- Log to alerts table
        INSERT INTO dbo.AlertLog (AlertTime, AlertType, Message)
        VALUES (GETDATE(), 'Fleet_Idle', @AlertMessage);
    END;
END;
GO

-- Schedule via SQL Agent: run every 15 minutes
```

### Pattern 3: Time-Phased Simulation

```sql
-- Scenario analysis: What if warehouse capacity increases by 20%?
WITH BaselineMetrics AS (
    SELECT 
        AVG(wo.DurationMinutes) AS AvgPickTime,
        COUNT(*) AS TotalOps
    FROM dbo.FactWarehouseOperations wo
    WHERE wo.OperationType = 'Picking'
        AND wo.OperationStartTime >= DATEADD(MONTH, -3, GETDATE())
),
SimulatedMetrics AS (
    SELECT 
        -- Assume 20% capacity increase reduces congestion by 12%
        AVG(wo.DurationMinutes) * 0.88 AS SimulatedPickTime,
        COUNT(*) * 1.15 AS SimulatedOps -- 15% more throughput
    FROM dbo.FactWarehouseOperations wo
    WHERE wo.OperationType = 'Picking'
        AND wo.OperationStartTime >= DATEADD(MONTH, -3, GETDATE())
)
SELECT 
    b.AvgPickTime AS CurrentAvgPickTime,
    s.SimulatedPickTime AS ProjectedPickTime,
    b.AvgPickTime - s.SimulatedPickTime AS TimeSavingsMinutes,
    b.TotalOps AS CurrentMonthlyOps,
    s.SimulatedOps AS ProjectedMonthlyOps,
    s.SimulatedOps - b.TotalOps AS AdditionalThroughput
FROM BaselineMetrics b, SimulatedMetrics s;
```

## Troubleshooting

### Issue: Power BI Refresh Fails

**Symptom**: "Data source error: Unable to connect to SQL Server"

**Solution**:
1. Verify SQL Server allows remote connections:
   ```sql
   EXEC sp_configure 'remote access', 1;
   RECONFIGURE;
   ```
2. Check firewall rules for port 1433
3. Ensure Power BI gateway is installed and configured
4. Test connection via SSMS from the Power BI machine

### Issue: Slow Query Performance

**Symptom**: Cross-fact queries take >30 seconds

**Solution**:
```sql
-- Add columnstore indexes to fact tables (already in schema)
-- Add covering indexes for common filter patterns
CREATE NONCLUSTERED INDEX IX_Warehouse_Operation_Time 
ON dbo.FactWarehouseOperations(OperationType, OperationStartTime)
INCLUDE (ProductKey, DurationMinutes, DwellTimeHours);

-- Update statistics regularly
CREATE PROCEDURE dbo.usp_UpdateStatistics
AS
BEGIN
    UPDATE STATISTICS dbo.FactWarehouseOperations WITH FULLSCAN;
    UPDATE STATISTICS dbo.FactFleetTrips WITH FULLSCAN;
END;
GO

-- Schedule nightly via SQL Agent
```

### Issue: Gravity Score Not Updating

**Symptom**: Products remain in wrong velocity class

**Solution**:
```sql
-- Refresh gravity scores based on recent activity
CREATE PROCEDURE dbo.usp_RecalculateGravityScores
AS
BEGIN
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            AVG(DurationMinutes) AS AvgPickTime,
            SUM(QuantityHandled) AS TotalVolume
        FROM dbo.FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND OperationStartTime >= DATEADD(MONTH, -1, GETDATE())
        GROUP BY ProductKey
    )
    UPDATE dp
    SET 
        GravityScore = 
            (pv.PickCount * 0.4) + 
            ((100 - pv.AvgPickTime) * 0.3) + 
            (LOG(pv.TotalVolume + 1) * 0.3),
        VelocityClass = 
            CASE 
                WHEN pv.PickCount >= 50 THEN 'Fast'
                WHEN pv.PickCount >= 20 THEN 'Medium'
                WHEN pv.PickCount >= 5 THEN 'Slow'
                ELSE 'Dead'
            END,
        ValidFrom = GETDATE()
    FROM dbo.DimProductGravity dp
    INNER JOIN ProductVelocity pv ON dp.ProductKey = pv.ProductKey;
END;
GO

-- Run weekly via SQL Agent
```

### Issue: Row-Level Security Not Working

**Symptom**: Users see all data despite restrictions

**Solution**:
```sql
-- Verify RLS policy is enabled
SELECT name, is_enabled FROM sys.security_policies;

-- Check user context
SELECT SYSTEM_USER, ORIGINAL_LOGIN();

-- Test RLS function directly
SELECT * FROM dbo.fn_SecurityPredicate('test.user@company.com');

-- Ensure Power BI uses correct user context
-- In Power BI Desktop: File > Options > Security > 
-- Set "Use my current security context"
```

## Advanced Usage

### Integrating External APIs

```sql
-- Example: Weather API integration for delay correlation
