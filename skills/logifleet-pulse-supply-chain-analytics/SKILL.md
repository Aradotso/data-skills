---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain KPI harmonization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "build multi-fact star schema for warehouse operations"
  - "implement cross-modal supply chain data warehouse"
  - "create fleet telemetry analytics system"
  - "deploy SQL Server logistics intelligence platform"
  - "integrate warehouse and fleet KPI dashboards"
  - "analyze supply chain bottlenecks with Power BI"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive MS SQL Server data warehouse and Power BI visualization platform designed for logistics intelligence. It implements a multi-fact star schema that harmonizes warehouse operations, fleet telemetry, inventory management, and supply chain KPIs into a unified semantic layer. The platform enables cross-fact analysis (e.g., correlating warehouse dwell time with fleet fuel consumption) through time-phased dimensions and bridge tables.

**Key capabilities:**
- Multi-fact star schema with shared dimensions (time, geography, product hierarchy)
- Real-time dashboards refreshed every 15 minutes
- Predictive bottleneck detection using historical patterns
- Warehouse "gravity zone" optimization for storage layout
- Fleet maintenance triage based on revenue impact
- Role-based access control with row-level security

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telemetry APIs, or flat file feeds

### Database Setup

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**

```sql
-- Connect to your SQL Server instance
-- Run the schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_stored_procedures.sql
```

3. **Configure data source connections:**

```json
// config.json (create from config_sample.json)
{
  "sql_server": {
    "host": "localhost",
    "database": "LogiFleetPulse",
    "user": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "refresh_interval_minutes": 15
  }
}
```

4. **Import Power BI template:**

- Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
- Enter SQL Server connection string when prompted
- Configure refresh schedule (minimum 15 minutes)

## Core Data Model

### Fact Tables

**FactWarehouseOperations** — Core warehouse activity metrics:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2),
    PackingTimeSeconds INT,
    EmployeeKey INT,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey) INCLUDE (DwellTimeMinutes, PickRate);
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey, OperationType);
```

**FactFleetTrips** — Fleet telemetry and route performance:

```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReasonKey INT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey) INCLUDE (IdleTimeMinutes, DelayMinutes);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey) INCLUDE (FuelLiters);
```

### Key Dimension Tables

**DimTime** — Time-phased dimension with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    DayOfMonth INT,
    DayOfWeek INT,
    HourOfDay INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    FiscalYear INT,
    FiscalPeriod INT,
    IsWeekend BIT,
    IsHoliday BIT
);
```

**DimProductGravity** — Product classification with velocity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: (PickFrequency * Value) / Fragility
    VelocityClass VARCHAR(20), -- 'FAST', 'MEDIUM', 'SLOW'
    FragilityIndex DECIMAL(3,2),
    UnitValue DECIMAL(10,2),
    OptimalZoneID INT,
    LastCalculated DATETIME
);
```

## Common Usage Patterns

### Cross-Fact KPI Analysis

**Query: Correlate warehouse dwell time with fleet idle time by product category:**

```sql
-- View joining warehouse operations and fleet trips through shared dimensions
CREATE VIEW vw_CrossFactLogistics AS
SELECT 
    t.FullDateTime,
    p.Category AS ProductCategory,
    w.WarehouseName,
    AVG(wh.DwellTimeMinutes) AS AvgDwellTime,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleTime,
    AVG(f.FuelLiters / NULLIF(f.DistanceKM, 0)) AS AvgFuelEfficiency,
    COUNT(DISTINCT wh.OperationID) AS WarehouseOps,
    COUNT(DISTINCT f.TripID) AS FleetTrips
FROM FactWarehouseOperations wh
INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
INNER JOIN DimWarehouse w ON wh.WarehouseKey = w.WarehouseKey
LEFT JOIN FactFleetTrips f ON f.TimeKey = t.TimeKey 
    AND f.RouteKey IN (SELECT RouteKey FROM DimRoute WHERE OriginWarehouseKey = w.WarehouseKey)
WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY t.FullDateTime, p.Category, w.WarehouseName;
```

### Warehouse Gravity Zone Optimization

**Stored procedure to recalculate product gravity scores:**

```sql
CREATE PROCEDURE sp_RecalculateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on last 90 days of activity
    UPDATE p
    SET 
        p.GravityScore = (
            (pick_freq.PicksPerDay * p.UnitValue) / NULLIF(p.FragilityIndex, 0)
        ),
        p.VelocityClass = CASE
            WHEN pick_freq.PicksPerDay > 50 THEN 'FAST'
            WHEN pick_freq.PicksPerDay > 10 THEN 'MEDIUM'
            ELSE 'SLOW'
        END,
        p.LastCalculated = GETDATE()
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / 90 AS PicksPerDay
        FROM FactWarehouseOperations
        WHERE OperationType = 'PICK'
            AND TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -90, GETDATE()))
        GROUP BY ProductKey
    ) pick_freq ON p.ProductKey = pick_freq.ProductKey;
    
    -- Suggest zone reassignments
    INSERT INTO ZoneRecommendations (ProductKey, CurrentZone, RecommendedZone, ReasonCode)
    SELECT 
        p.ProductKey,
        z.ZoneID AS CurrentZone,
        opt_z.ZoneID AS RecommendedZone,
        'HIGH_GRAVITY_RELOCATION' AS ReasonCode
    FROM DimProductGravity p
    INNER JOIN WarehouseZones z ON p.OptimalZoneID = z.ZoneID
    CROSS APPLY (
        SELECT TOP 1 ZoneID
        FROM WarehouseZones
        WHERE DistanceToShipping < z.DistanceToShipping
            AND AvailableSlots > 0
        ORDER BY DistanceToShipping ASC
    ) opt_z
    WHERE p.VelocityClass = 'FAST'
        AND z.DistanceToShipping > 50; -- meters from shipping dock
END;
```

### Fleet Maintenance Triage

**Query: Prioritize vehicle maintenance by revenue impact:**

```sql
WITH VehicleUtilization AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.MaintenanceStatus,
        AVG(ft.FuelLiters / NULLIF(ft.DistanceKM, 0)) AS FuelEfficiency,
        SUM(ft.IdleTimeMinutes) AS TotalIdleTime,
        COUNT(DISTINCT ft.TripID) AS TripCount,
        SUM(cargo.TotalValue) AS CargoValueTransported
    FROM DimVehicle v
    INNER JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
    LEFT JOIN (
        SELECT 
            TripID,
            SUM(p.UnitValue * cargo.Quantity) AS TotalValue
        FROM TripCargo cargo
        INNER JOIN DimProduct p ON cargo.ProductKey = p.ProductKey
        GROUP BY TripID
    ) cargo ON ft.TripID = cargo.TripID
    WHERE ft.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
    GROUP BY v.VehicleKey, v.VehicleID, v.MaintenanceStatus
),
MaintenanceIssues AS (
    SELECT 
        VehicleKey,
        IssueType,
        Severity,
        DetectedDate
    FROM VehicleTelemetry
    WHERE Severity IN ('WARNING', 'CRITICAL')
        AND ResolvedDate IS NULL
)
SELECT 
    u.VehicleID,
    u.MaintenanceStatus,
    mi.IssueType,
    mi.Severity,
    u.TripCount,
    u.CargoValueTransported,
    u.FuelEfficiency,
    u.TotalIdleTime,
    -- Priority score: higher cargo value + critical issues = higher priority
    (u.CargoValueTransported / 10000.0) + 
    (CASE mi.Severity WHEN 'CRITICAL' THEN 100 WHEN 'WARNING' THEN 50 ELSE 0 END) AS PriorityScore
FROM VehicleUtilization u
INNER JOIN MaintenanceIssues mi ON u.VehicleKey = mi.VehicleKey
ORDER BY PriorityScore DESC;
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

**Cross-Fact Efficiency Score:**

```dax
CrossFactEfficiency = 
VAR WarehouseDwellAvg = 
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR FleetIdleAvg = 
    AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR DwellNormalized = 
    DIVIDE(60, WarehouseDwellAvg, 1) -- Inverse: lower dwell = higher score
VAR IdleNormalized = 
    DIVIDE(60, FleetIdleAvg, 1) -- Inverse: lower idle = higher score
RETURN
    (DwellNormalized + IdleNormalized) / 2 * 100
```

**Predictive Bottleneck Index:**

```dax
BottleneckIndex = 
VAR CurrentHour = HOUR(NOW())
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FILTER(
            ALL(DimTime),
            DimTime[HourOfDay] = CurrentHour
        )
    )
VAR CurrentDwell = 
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
RETURN
    IF(
        CurrentDwell > HistoricalAvg * 1.5,
        "High Risk",
        IF(
            CurrentDwell > HistoricalAvg * 1.2,
            "Moderate Risk",
            "Normal"
        )
    )
```

### Dashboard Refresh Configuration

```python
# Python script for scheduled Power BI refresh via Power BI REST API
import requests
import os
from datetime import datetime

def trigger_powerbi_refresh():
    """Trigger dataset refresh using Power BI REST API"""
    
    # Authentication
    tenant_id = os.getenv('AZURE_TENANT_ID')
    client_id = os.getenv('POWERBI_CLIENT_ID')
    client_secret = os.getenv('POWERBI_CLIENT_SECRET')
    
    # Get access token
    token_url = f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token"
    token_data = {
        'grant_type': 'client_credentials',
        'client_id': client_id,
        'client_secret': client_secret,
        'scope': 'https://analysis.windows.net/powerbi/api/.default'
    }
    
    token_response = requests.post(token_url, data=token_data)
    access_token = token_response.json()['access_token']
    
    # Trigger refresh
    workspace_id = os.getenv('POWERBI_WORKSPACE_ID')
    dataset_id = os.getenv('POWERBI_DATASET_ID')
    
    refresh_url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/refreshes"
    
    headers = {
        'Authorization': f'Bearer {access_token}',
        'Content-Type': 'application/json'
    }
    
    refresh_response = requests.post(refresh_url, headers=headers)
    
    if refresh_response.status_code == 202:
        print(f"Refresh triggered successfully at {datetime.now()}")
    else:
        print(f"Refresh failed: {refresh_response.text}")

if __name__ == "__main__":
    trigger_powerbi_refresh()
```

## Data Loading Patterns

### Incremental Load Stored Procedure

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    -- Stage new data from external source
    INSERT INTO Stage_WarehouseOperations (
        OperationDateTime,
        WarehouseCode,
        SKU,
        OperationType,
        DwellTimeMinutes,
        PickRate,
        PackingTimeSeconds,
        EmployeeID
    )
    SELECT 
        OperationDateTime,
        WarehouseCode,
        SKU,
        OperationType,
        DwellTimeMinutes,
        PickRate,
        PackingTimeSeconds,
        EmployeeID
    FROM OPENROWSET(
        BULK '${WMS_EXPORT_PATH}/*.csv',
        DATA_SOURCE = 'WMS_External_Source',
        FORMAT = 'CSV',
        FIRSTROW = 2
    ) AS ext
    WHERE ext.OperationDateTime > @LastLoadDateTime;
    
    -- Transform and load to fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        WarehouseKey,
        ProductKey,
        OperationType,
        DwellTimeMinutes,
        PickRate,
        PackingTimeSeconds,
        EmployeeKey
    )
    SELECT 
        t.TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        stage.OperationType,
        stage.DwellTimeMinutes,
        stage.PickRate,
        stage.PackingTimeSeconds,
        e.EmployeeKey
    FROM Stage_WarehouseOperations stage
    INNER JOIN DimTime t ON 
        t.FullDateTime = DATEADD(MINUTE, 
            (DATEPART(MINUTE, stage.OperationDateTime) / 15) * 15, 
            DATEADD(MINUTE, -DATEPART(MINUTE, stage.OperationDateTime), stage.OperationDateTime)
        )
    INNER JOIN DimWarehouse w ON stage.WarehouseCode = w.WarehouseCode
    INNER JOIN DimProduct p ON stage.SKU = p.SKU
    LEFT JOIN DimEmployee e ON stage.EmployeeID = e.EmployeeID;
    
    -- Clean staging table
    TRUNCATE TABLE Stage_WarehouseOperations;
    
    COMMIT TRANSACTION;
    
    -- Log load completion
    INSERT INTO ETL_LoadLog (ProcedureName, LoadDateTime, RowsProcessed)
    VALUES ('sp_IncrementalLoadWarehouseOps', GETDATE(), @@ROWCOUNT);
END;
```

### Real-Time Streaming via External Tables

```sql
-- Create external data source for streaming telemetry
CREATE EXTERNAL DATA SOURCE Fleet_Telemetry_Stream
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${FLEET_BLOB_STORAGE_URL}',
    CREDENTIAL = Fleet_Storage_Credential
);

-- Create external table for real-time fleet data
CREATE EXTERNAL TABLE Ext_FleetTelemetryStream (
    VehicleID VARCHAR(20),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp INT,
    TirePressure VARCHAR(50)
)
WITH (
    LOCATION = '/telemetry/',
    DATA_SOURCE = Fleet_Telemetry_Stream,
    FILE_FORMAT = JSON_Format
);

-- Continuous insert from external stream to staging
CREATE PROCEDURE sp_StreamFleetTelemetry
AS
BEGIN
    INSERT INTO Stage_FleetTelemetry
    SELECT * FROM Ext_FleetTelemetryStream
    WHERE Timestamp > (SELECT MAX(LastProcessedTime) FROM ETL_StreamCheckpoint);
    
    UPDATE ETL_StreamCheckpoint
    SET LastProcessedTime = GETDATE()
    WHERE StreamName = 'FleetTelemetry';
END;
```

## Alert Configuration

### Automated KPI Threshold Alerts

```sql
CREATE PROCEDURE sp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Alerts TABLE (
        AlertType VARCHAR(50),
        EntityName NVARCHAR(200),
        CurrentValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2),
        Severity VARCHAR(20)
    );
    
    -- Check fleet idle time threshold
    INSERT INTO @Alerts (AlertType, EntityName, CurrentValue, ThresholdValue, Severity)
    SELECT 
        'FLEET_IDLE_TIME',
        v.VehicleID,
        AVG(ft.IdleTimeMinutes) * 1.0 / NULLIF(AVG(ft.DistanceKM / 60.0), 0) * 100 AS IdlePercent,
        15.0, -- 15% threshold
        'HIGH'
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    WHERE ft.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(HOUR, -4, GETDATE()))
    GROUP BY v.VehicleID
    HAVING AVG(ft.IdleTimeMinutes) * 1.0 / NULLIF(AVG(ft.DistanceKM / 60.0), 0) * 100 > 15;
    
    -- Check warehouse dwell time threshold
    INSERT INTO @Alerts (AlertType, EntityName, CurrentValue, ThresholdValue, Severity)
    SELECT 
        'WAREHOUSE_DWELL_TIME',
        p.SKU,
        AVG(wh.DwellTimeMinutes),
        72.0, -- 72 hours threshold
        CASE 
            WHEN AVG(wh.DwellTimeMinutes) > 120 THEN 'CRITICAL'
            ELSE 'HIGH'
        END
    FROM FactWarehouseOperations wh
    INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    WHERE wh.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -1, GETDATE()))
        AND p.VelocityClass = 'FAST'
    GROUP BY p.SKU
    HAVING AVG(wh.DwellTimeMinutes) > 72;
    
    -- Send alerts via email (requires Database Mail configured)
    IF EXISTS (SELECT 1 FROM @Alerts)
    BEGIN
        DECLARE @EmailBody NVARCHAR(MAX);
        
        SET @EmailBody = N'<html><body><h2>LogiFleet Pulse KPI Alerts</h2><table border="1">' +
            N'<tr><th>Alert Type</th><th>Entity</th><th>Current</th><th>Threshold</th><th>Severity</th></tr>';
        
        SELECT @EmailBody = @EmailBody + 
            N'<tr><td>' + AlertType + '</td>' +
            N'<td>' + EntityName + '</td>' +
            N'<td>' + CAST(CurrentValue AS NVARCHAR(20)) + '</td>' +
            N'<td>' + CAST(ThresholdValue AS NVARCHAR(20)) + '</td>' +
            N'<td>' + Severity + '</td></tr>'
        FROM @Alerts;
        
        SET @EmailBody = @EmailBody + N'</table></body></html>';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: KPI Threshold Breach',
            @body = @EmailBody,
            @body_format = 'HTML';
    END;
END;
```

## Troubleshooting

### Performance Optimization

**Issue: Slow cross-fact queries**

```sql
-- Check for missing indexes on join columns
SELECT 
    t.name AS TableName,
    i.name AS IndexName,
    dm_ius.user_seeks,
    dm_ius.user_scans,
    dm_ius.user_lookups
FROM sys.dm_db_index_usage_stats dm_ius
INNER JOIN sys.tables t ON dm_ius.object_id = t.object_id
INNER JOIN sys.indexes i ON dm_ius.object_id = i.object_id AND dm_ius.index_id = i.index_id
WHERE t.name IN ('FactWarehouseOperations', 'FactFleetTrips')
ORDER BY dm_ius.user_seeks DESC;

-- Create recommended indexes
CREATE NONCLUSTERED INDEX IX_FactWH_Composite 
ON FactWarehouseOperations(TimeKey, ProductKey, WarehouseKey)
INCLUDE (DwellTimeMinutes, PickRate);

CREATE NONCLUSTERED INDEX IX_FactFleet_Composite
ON FactFleetTrips(TimeKey, VehicleKey, RouteKey)
INCLUDE (IdleTimeMinutes, FuelLiters);
```

**Issue: Power BI refresh timeouts**

- Enable incremental refresh in Power BI with RangeStart/RangeEnd parameters:

```powerquery
let
    Source = Sql.Database("${SQL_SERVER}", "${DATABASE}"),
    FilteredRows = Table.SelectRows(Source, 
        each [OperationDateTime] >= RangeStart and [OperationDateTime] < RangeEnd)
in
    FilteredRows
```

### Data Quality Checks

**Identify orphaned dimension references:**

```sql
-- Find fact records with missing dimension keys
SELECT 
    'FactWarehouseOperations' AS FactTable,
    COUNT(*) AS OrphanedRecords
FROM FactWarehouseOperations fwh
WHERE NOT EXISTS (SELECT 1 FROM DimProduct p WHERE p.ProductKey = fwh.ProductKey)
    OR NOT EXISTS (SELECT 1 FROM DimWarehouse w WHERE w.WarehouseKey = fwh.WarehouseKey)

UNION ALL

SELECT 
    'FactFleetTrips',
    COUNT(*)
FROM FactFleetTrips fft
WHERE NOT EXISTS (SELECT 1 FROM DimVehicle v WHERE v.VehicleKey = fft.VehicleKey)
    OR NOT EXISTS (SELECT 1 FROM DimTime t WHERE t.TimeKey = fft.TimeKey);
```

### Common Integration Issues

**Issue: External data source connection failures**

```sql
-- Test external table connectivity
BEGIN TRY
    SELECT TOP 10 * FROM Ext_FleetTelemetryStream;
    PRINT 'External data source connected successfully';
END TRY
BEGIN CATCH
    PRINT 'Connection error: ' + ERROR_MESSAGE();
    -- Check credential configuration
    SELECT * FROM sys.database_scoped_credentials;
END CATCH;
```

**Issue: Power BI gateway timeout**

- Increase gateway timeout in Power BI settings (default 10 minutes)
- Implement query folding to push computation to SQL Server
- Use DirectQuery mode for large datasets instead of Import mode

## Best Practices

1. **Partitioning for large fact tables:**

```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION SCHEME PS_FactWH_Monthly
AS PARTITION PF_Monthly_Range TO ([PRIMARY], [FG_2025_01], [FG_2025_02], ...);

ALTER TABLE FactWarehouseOperations DROP CONSTRAINT PK_FactWH;

ALTER TABLE FactWarehouseOperations ADD CONSTRAINT PK_FactWH 
PRIMARY KEY CLUSTERED (OperationID, TimeKey) ON PS_FactWH_Monthly(TimeKey);
```

2. **Maintain surrogate key integrity:**

```sql
-- Use IDENTITY columns for fact tables
-- Use MERGE for dimension updates to preserve historical keys
MERGE DimProduct AS target
USING Stage_Product AS source
ON target.SKU = source.SKU
WHEN MATCHED THEN
    UPDATE SET 
        ProductName = source.ProductName,
        Category = source.Category
WHEN NOT MATCHED THEN
    INSERT (SKU, ProductName, Category)
    VALUES (source.SKU, source.ProductName, source.Category);
```

3. **Regular index maintenance:**

```sql
-- Weekly index rebuild job
DECLARE @TableName NVARCHAR(255);
DECLARE @SQL NVARCHAR(MAX);

DECLARE table_cursor CURSOR FOR
SELECT name FROM sys.tables WHERE name LIKE 'Fact%';

OPEN table_cursor;
FETCH NEXT FROM table_cursor INTO @TableName;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @SQL = 'ALTER INDEX ALL ON ' + @TableName + ' REBUILD WITH (ONLINE = ON);';
    EXEC sp_executesql @SQL;
    FETCH NEXT FROM table_cursor INTO @TableName;
END;

CLOSE table_cursor;
DEALLOCATE table_cursor;
```
