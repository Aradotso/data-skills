---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet, warehouse, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "implement warehouse and fleet data model"
  - "create multi-fact star schema for logistics"
  - "deploy SQL Server supply chain warehouse"
  - "build logistics intelligence dashboard"
  - "integrate warehouse and fleet telemetry data"
  - "query cross-modal supply chain analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence platform that combines warehouse operations, fleet telemetry, and supply chain data into a unified MS SQL Server data warehouse with Power BI visualizations. It implements a custom multi-fact star schema with time-phased dimensions for cross-fact KPI analysis.

**Core capabilities:**
- Multi-fact star schema (warehouse operations, fleet trips, cross-dock)
- Time-aware dimensional modeling (15-minute granularity)
- Warehouse Gravity Zones™ spatial optimization
- Predictive bottleneck detection
- Real-time Power BI dashboards
- Role-based access with row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express/Standard/Enterprise)
- Power BI Desktop (latest version)
- Optional: Azure Synapse Analytics for big data enrichment

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Run the schema creation script
:r .\sql\01_create_database.sql
:r .\sql\02_create_dimensions.sql
:r .\sql\03_create_facts.sql
:r .\sql\04_create_views.sql
:r .\sql\05_create_procedures.sql
```

3. **Configure data sources:**
```json
{
  "connections": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    },
    "weather_api": {
      "endpoint": "${WEATHER_API_ENDPOINT}",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_interval_minutes": 15
}
```

4. **Import Power BI template:**
```powershell
# Open the .pbit template
Start-Process ".\powerbi\LogiFleet_Pulse_Master.pbit"

# Configure connection string when prompted
# Server: your-sql-server.database.windows.net
# Database: LogiFleetPulse
# Authentication: Windows/SQL Server/Azure AD
```

## Core Data Model

### Dimensional Structure

```sql
-- Time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    Hour INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayOfMonth INT NOT NULL,
    Month INT NOT NULL,
    Quarter INT NOT NULL,
    FiscalYear INT NOT NULL,
    IsBusinessHours BIT NOT NULL,
    IsWeekend BIT NOT NULL
);

-- Geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) UNIQUE NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, Hub, Route Node
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50)
);

-- Product Gravity dimension (velocity-based)
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite metric
    PickFrequency INT,
    UnitValue DECIMAL(10,2),
    FragilityIndex DECIMAL(3,2),
    OptimalZone NVARCHAR(10) -- A1, B2, etc.
);

-- Supplier Reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200) NOT NULL,
    LeadTimeMean INT, -- Days
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    LastAuditDate DATE
);
```

### Fact Tables

```sql
-- Warehouse Operations Fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    EmployeeID NVARCHAR(50),
    ZoneID NVARCHAR(10),
    BatchID NVARCHAR(50)
);

-- Fleet Trips Fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID NVARCHAR(50) NOT NULL,
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DriverID NVARCHAR(50),
    RouteID NVARCHAR(50),
    DistanceKm DECIMAL(8,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(8,2),
    LoadWeightKg DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason NVARCHAR(200)
);

-- Cross-Dock Fact
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    Quantity INT NOT NULL,
    TransferTimeMinutes INT,
    TemperatureCompliant BIT
);
```

## Common Query Patterns

### Cross-Fact KPI Analysis

```sql
-- Dwell time by SKU vs. fleet idling cost per route
SELECT 
    p.SKU,
    p.ProductName,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(DISTINCT ft.TripKey) AS TripCount,
    SUM(ft.IdleTimeMinutes) AS TotalIdleTime,
    SUM(ft.FuelConsumedLiters * 1.50) AS EstimatedIdleCost -- $1.50/liter
FROM FactWarehouseOperations wo
JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
JOIN FactCrossDock cd ON wo.ProductKey = cd.ProductKey
JOIN FactFleetTrips ft ON cd.OutboundTripKey = ft.TripKey
WHERE wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(quarter, -1, GETDATE()))
GROUP BY p.SKU, p.ProductName
HAVING AVG(wo.DwellTimeMinutes) > 72
ORDER BY TotalIdleTime DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products needing zone reassignment
WITH ProductVelocity AS (
    SELECT 
        ProductKey,
        COUNT(*) AS PickCount,
        AVG(CycleTimeSeconds) AS AvgCycleTime
    FROM FactWarehouseOperations
    WHERE OperationType = 'Picking'
      AND TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(month, -1, GETDATE()))
    GROUP BY ProductKey
)
SELECT 
    p.SKU,
    p.ProductName,
    p.OptimalZone AS CurrentZone,
    pv.PickCount,
    pv.AvgCycleTime,
    CASE 
        WHEN pv.PickCount > 100 AND p.OptimalZone NOT LIKE 'A%' THEN 'Move to A-Zone'
        WHEN pv.PickCount < 20 AND p.OptimalZone LIKE 'A%' THEN 'Move to C-Zone'
        ELSE 'No Change'
    END AS Recommendation
FROM DimProductGravity p
JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
WHERE pv.PickCount > 0
ORDER BY pv.PickCount DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential congestion points
CREATE PROCEDURE sp_PredictBottlenecks
AS
BEGIN
    WITH TimeSlotMetrics AS (
        SELECT 
            t.Hour,
            g.LocationName,
            COUNT(wo.OperationKey) AS OperationCount,
            AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
            STDEV(wo.CycleTimeSeconds) AS StdDevCycleTime
        FROM FactWarehouseOperations wo
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
        WHERE t.DateTime >= DATEADD(day, -30, GETDATE())
        GROUP BY t.Hour, g.LocationName
    )
    SELECT 
        Hour,
        LocationName,
        OperationCount,
        AvgCycleTime,
        CASE 
            WHEN AvgCycleTime > (SELECT AVG(AvgCycleTime) * 1.5 FROM TimeSlotMetrics) 
            THEN 'High Risk'
            WHEN StdDevCycleTime > AvgCycleTime * 0.5
            THEN 'Moderate Risk'
            ELSE 'Low Risk'
        END AS BottleneckRisk
    FROM TimeSlotMetrics
    WHERE OperationCount > 10
    ORDER BY AvgCycleTime DESC;
END;
GO
```

### Fleet Maintenance Priority Queue

```sql
-- Adaptive fleet triage based on revenue impact
SELECT 
    ft.VehicleID,
    COUNT(DISTINCT ft.TripKey) AS TripCount,
    SUM(ft.DistanceKm) AS TotalDistanceKm,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS FuelEfficiency,
    SUM(p.UnitValue * cd.Quantity) AS TotalCargoValue,
    CASE 
        WHEN SUM(p.UnitValue * cd.Quantity) > 100000 AND AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) > 0.15 
        THEN 'Critical Priority'
        WHEN SUM(ft.DelayMinutes) > 180
        THEN 'High Priority'
        ELSE 'Standard Priority'
    END AS MaintenancePriority
FROM FactFleetTrips ft
JOIN FactCrossDock cd ON ft.TripKey = cd.OutboundTripKey
JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
WHERE ft.TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(month, -3, GETDATE()))
GROUP BY ft.VehicleID
ORDER BY TotalCargoValue DESC;
```

## Data Loading & ETL

### Incremental Load Procedure

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Staging table for incremental data
    CREATE TABLE #StagingOperations (
        OperationID NVARCHAR(50),
        Timestamp DATETIME2,
        SKU NVARCHAR(50),
        LocationID NVARCHAR(50),
        OperationType NVARCHAR(50),
        Quantity INT,
        CycleTimeSeconds INT,
        EmployeeID NVARCHAR(50)
    );
    
    -- Extract from source (external table, API, etc.)
    INSERT INTO #StagingOperations
    SELECT * FROM OPENQUERY([WMS_LINKED_SERVER], 
        'SELECT * FROM operations WHERE timestamp > ''' + CONVERT(VARCHAR, @LastLoadTimestamp, 120) + '''');
    
    -- Transform and load
    INSERT INTO FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, OperationType, Quantity, CycleTimeSeconds, EmployeeID)
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.OperationType,
        s.Quantity,
        s.CycleTimeSeconds,
        s.EmployeeID
    FROM #StagingOperations s
    JOIN DimTime t ON CAST(s.Timestamp AS DATETIME2(0)) = t.DateTime
    JOIN DimProductGravity p ON s.SKU = p.SKU
    JOIN DimGeography g ON s.LocationID = g.LocationID;
    
    DROP TABLE #StagingOperations;
END;
GO
```

### Automated Refresh Schedule

```sql
-- Create SQL Server Agent job for scheduled refresh
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleet_IncrementalRefresh',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalRefresh',
    @step_name = N'Load Warehouse Data',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_LoadWarehouseOperations @LastLoadTimestamp = (SELECT MAX(DateTime) FROM DimTime WHERE TimeKey IN (SELECT TimeKey FROM FactWarehouseOperations))';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalRefresh',
    @schedule_name = N'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = N'LogiFleet_IncrementalRefresh',
    @server_name = N'(local)';
```

## Power BI Configuration

### DAX Measures for Cross-Fact KPIs

```dax
-- Total Dwell Time per SKU
TotalDwellTime = 
SUM(FactWarehouseOperations[DwellTimeMinutes])

-- Average Fleet Efficiency
AvgFleetEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

-- Cross-Fact: Dwell Impact on Delivery Delay
DwellImpactOnDelay = 
SUMX(
    RELATEDTABLE(FactCrossDock),
    RELATED(FactWarehouseOperations[DwellTimeMinutes]) * 
    RELATED(FactFleetTrips[DelayMinutes])
) / 
SUMX(
    RELATEDTABLE(FactCrossDock),
    RELATED(FactFleetTrips[DelayMinutes])
)

-- Warehouse Gravity Score (composite)
GravityScore = 
VAR PickFreq = DIVIDE(COUNTROWS(FactWarehouseOperations), DISTINCTCOUNT(FactWarehouseOperations[TimeKey]), 0)
VAR AvgValue = AVERAGE(DimProductGravity[UnitValue])
VAR Fragility = AVERAGE(DimProductGravity[FragilityIndex])
RETURN (PickFreq * 0.5) + (AvgValue / 100 * 0.3) + (Fragility * 0.2)
```

### Row-Level Security

```dax
-- RLS filter for regional managers
[Region] = USERPRINCIPALNAME()

-- Implementation in Power BI
1. Modeling > Manage Roles > Create Role: "RegionalManager"
2. Table: DimGeography
3. Filter: [Region] = USERNAME()
4. Assign users to role in Power BI Service
```

## Alerting & Monitoring

### Threshold-Based Alert Procedure

```sql
CREATE PROCEDURE sp_CheckAlertThresholds
AS
BEGIN
    DECLARE @AlertRecipients NVARCHAR(500) = '${ALERT_EMAIL_RECIPIENTS}';
    DECLARE @AlertSubject NVARCHAR(200);
    DECLARE @AlertBody NVARCHAR(MAX);
    
    -- Check fleet idle time threshold
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips
        WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(hour, -4, GETDATE()))
        GROUP BY VehicleID
        HAVING SUM(IdleTimeMinutes) > (SUM(DurationMinutes) * 0.15)
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Fleet Idle Time Exceeded 15%';
        SET @AlertBody = 'One or more vehicles exceeded idle time threshold in the last 4 hours.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @AlertRecipients,
            @subject = @AlertSubject,
            @body = @AlertBody;
    END;
    
    -- Check warehouse dwell time threshold
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations wo
        JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        WHERE wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(day, -1, GETDATE()))
          AND wo.DwellTimeMinutes > 72
          AND p.Category = 'Perishables'
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Perishable Dwell Time Exceeded 72 Hours';
        SET @AlertBody = 'Perishable products have exceeded safe dwell time.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @AlertRecipients,
            @subject = @AlertSubject,
            @body = @AlertBody;
    END;
END;
GO
```

## Performance Optimization

### Indexing Strategy

```sql
-- Time dimension clustered index
CREATE CLUSTERED INDEX IX_DimTime_TimeKey ON DimTime(TimeKey);
CREATE NONCLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime);

-- Fact table columnstore indexes (for large datasets)
CREATE CLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOps_CS 
ON FactWarehouseOperations;

CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (Quantity, DwellTimeMinutes);

-- Fleet trips covering index
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Vehicle_Time
ON FactFleetTrips(VehicleID, TimeKey)
INCLUDE (DistanceKm, IdleTimeMinutes, FuelConsumedLiters);
```

### Partitioning for Large Datasets

```sql
-- Partition by time (monthly)
CREATE PARTITION FUNCTION PF_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, 202604); -- YYYYMM format

CREATE PARTITION SCHEME PS_TimeKey
AS PARTITION PF_TimeKey ALL TO ([PRIMARY]);

-- Apply to fact table
CREATE TABLE FactWarehouseOperations (
    -- columns as before
) ON PS_TimeKey(TimeKey);
```

## Troubleshooting

### Common Issues

**Power BI connection timeout:**
```sql
-- Increase query timeout in connection string
Data Source=your-server;Initial Catalog=LogiFleetPulse;Connection Timeout=120;Command Timeout=300
```

**Slow cross-fact queries:**
```sql
-- Verify indexes exist
SELECT 
    i.name AS IndexName,
    OBJECT_NAME(i.object_id) AS TableName,
    i.type_desc
FROM sys.indexes i
WHERE OBJECT_NAME(i.object_id) LIKE 'Fact%'
ORDER BY TableName, IndexName;

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

**Data refresh failures:**
```sql
-- Check SQL Agent job history
SELECT 
    j.name AS JobName,
    h.run_date,
    h.run_time,
    h.run_status,
    h.message
FROM msdb.dbo.sysjobhistory h
JOIN msdb.dbo.sysjobs j ON h.job_id = j.job_id
WHERE j.name = 'LogiFleet_IncrementalRefresh'
ORDER BY h.run_date DESC, h.run_time DESC;
```

**Gravity zone calculations incorrect:**
```sql
-- Recalculate gravity scores
UPDATE DimProductGravity
SET GravityScore = (
    (SELECT COUNT(*) FROM FactWarehouseOperations wo 
     WHERE wo.ProductKey = DimProductGravity.ProductKey 
       AND wo.OperationType = 'Picking') * 0.5 +
    (UnitValue / 100) * 0.3 +
    FragilityIndex * 0.2
);
```

## Integration Examples

### REST API Data Ingestion

```sql
-- Enable external scripts if using R/Python for API calls
EXEC sp_configure 'external scripts enabled', 1;
RECONFIGURE;

-- Python script to ingest fleet telemetry
CREATE PROCEDURE sp_IngestFleetTelemetry
AS
BEGIN
    EXEC sp_execute_external_script
        @language = N'Python',
        @script = N'
import requests
import pandas as pd
import os

api_endpoint = os.environ["FLEET_TELEMETRY_ENDPOINT"]
api_key = os.environ["FLEET_API_KEY"]

response = requests.get(api_endpoint, headers={"Authorization": f"Bearer {api_key}"})
data = response.json()

df = pd.DataFrame(data["trips"])
OutputDataSet = df
',
        @output_data_1_name = N'df';
END;
```

### Azure Synapse Integration

```sql
-- Create external data source
CREATE EXTERNAL DATA SOURCE AzureSynapse
WITH (
    TYPE = HADOOP,
    LOCATION = 'wasbs://logifleet@${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net',
    CREDENTIAL = AzureStorageCredential
);

-- Query external big data
SELECT 
    p.SKU,
    ext.MachineLearningPrediction
FROM DimProductGravity p
JOIN EXTERNAL TABLE dbo.SynapsePredictions ext
    ON p.SKU = ext.SKU;
```

## Best Practices

1. **Time dimension population:** Always populate DimTime at least 2 years ahead to avoid lookup failures
2. **Incremental loads:** Use watermark timestamps stored in a control table
3. **Data validation:** Implement CHECK constraints on fact tables for data quality
4. **Security:** Always use environment variables for credentials, never hardcode
5. **Documentation:** Annotate complex DAX measures with comments explaining business logic
6. **Testing:** Validate cross-fact queries against known manual calculations before deployment
7. **Backup strategy:** Schedule nightly full backups with transaction log backups every 15 minutes
