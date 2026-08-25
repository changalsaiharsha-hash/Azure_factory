# Enterprise Integration Pipeline

A production-ready Azure Data Factory pipeline that automates data extraction, transformation, and loading from REST APIs to Azure SQL Database with incremental extraction, automatic duplicate removal, and comprehensive error handling.

## Overview

This project implements an enterprise-grade ETL pipeline that:
- Extracts data from REST APIs incrementally
- Transforms JSON data into relational format
- Removes duplicates automatically using MERGE logic
- Tracks progress with watermark-based extraction
- Logs errors for monitoring and debugging
- Provides complete audit trail and data lineage

## Key Features

- **Incremental Extraction** - Watermark-based extraction reduces API calls and costs
- **JSON Flattening** - Converts hierarchical JSON to clean relational format
- **Duplicate Removal** - Intelligent MERGE logic prevents duplicate records
- **Error Handling** - Automatic error logging and capture
- **Data Versioning** - Complete history of all changes
- **Idempotent Design** - Safe to re-run without data loss
- **Audit Trail** - Raw data backup in ADLS for compliance

## Architecture

```
REST API Orders
    |
    v
Copy data1 -> Azure Data Lake (raw zone)
    |
    v
TruncateStaging -> Clear old data
    |
    v
FlattenOrders -> JSON to relational
    |
    v
MergeToCurated -> Insert/Update/Deduplicate
    |
    v
UpdateWatermark -> Track progress
    |
    v
LogError (on failure) -> Error capture
```

## Pipeline Components

### Activities (6 Core + 1 Error Handling)

1. **GetWatermark** (Lookup)
   - Retrieves last extraction timestamp
   - Enables incremental extraction

2. **Copy data1** (Copy)
   - Fetches data from REST API
   - Stores raw JSON in ADLS

3. **TruncateStaging** (Script)
   - Clears old staging data
   - Ensures fresh load each run

4. **FlattenOrders** (Copy)
   - Transforms JSON to flat relational format
   - Loads to staging table

5. **MergeToCurated** (Stored Procedure)
   - MERGE logic handles INSERT/UPDATE/DELETE
   - Prevents duplicate records
   - Loads to production table

6. **UpdateWatermark** (Stored Procedure)
   - Updates progress marker
   - Only runs on success (idempotency)

7. **LogError** (Stored Procedure) - Error Path
   - Captures error details
   - Records to ETL_ErrorLog table

## Database Objects

### Tables
- `ETL_Watermark` - Tracks incremental extraction progress
- `ETL_ErrorLog` - Logs all pipeline errors
- `stg_Orders` - Staging table for data transformation
- `Orders` - Production curated table

### Stored Procedures
- `usp_UpdateWatermark` - Updates watermark on success
- `usp_LogError` - Captures error information
- `usp_MergeOrders` - MERGE logic for duplicate removal

## How It Works

### Execution Flow

```
1. GetWatermark reads: "Last processed: 10:00 AM"
   |
   v
2. Copy data1 fetches: "All orders since 10:00 AM"
   |
   v
3. TruncateStaging clears: Old staging data
   |
   v
4. FlattenOrders converts: JSON -> Table rows
   |
   v
5. MergeToCurated merges:
   - 900 NEW orders -> INSERT
   - 95 EXISTING orders -> UPDATE
   - 5 DUPLICATES -> IGNORE
   |
   v
6. UpdateWatermark sets: "Last processed: 12:30 PM"
   |
   v
SUCCESS - Next run starts from 12:30 PM
```

### Duplicate Removal Example

```
Incoming Data (with duplicates):
- Order 101: Pizza
- Order 101: Pizza  <- Duplicate
- Order 102: Pasta

After MERGE:
- Order 101: Pizza  (Kept)
- Order 102: Pasta  (Added)

Result: No duplicates, clean data
```

## Data Flow

```
Staging Table (stg_Orders)     Production Table (Orders)
+------------------+           +------------------+
| userid|id|title  |           |order_id|title    |
+------------------+  MERGE    +------------------+
|1|101|Order #1   +---------->|101|Order #1     |
|2|102|Order #2   |           |102|Order #2     |
|3|103|Order #3   |           |103|Order #3     |
+------------------+           +------------------+
  (Temporary)            (Production-Ready)
```

## Error Handling

The pipeline automatically captures errors:

```sql
ETL_ErrorLog captures:
- PipelineName: pipeline1
- ActivityName: UpdateWatermark
- ErrorCode: SqlOperationFailed
- ErrorMessage: Detailed error description
- ErrorTimeUtc: 2024-08-24 4:56 PM
```

**Key Feature:** Watermark is NOT updated on failure, so failed runs are automatically re-processed.

## Technologies Used

- **Azure Data Factory** - Orchestration & ETL
- **Azure Data Lake Storage Gen2** - Raw data landing
- **Azure SQL Database** - Data warehouse
- **T-SQL** - Stored procedures & transformations
- **Azure Key Vault** - Secrets management

## Performance Metrics

| Metric | Value |
|--------|-------|
| Pipeline Duration | ~100 seconds |
| Rows Processed | 1000+ per run |
| Data Quality | 100% (no duplicates) |
| Error Tracking | Automatic logging |
| Audit Trail | Complete history |

## Getting Started

### Prerequisites
- Azure subscription
- Azure Data Factory instance
- Azure SQL Database
- Azure Data Lake Storage Gen2
- REST API access

### Setup Steps

1. **Create Database Objects**
   ```sql
   -- Create tables
   CREATE TABLE dbo.ETL_Watermark (...)
   CREATE TABLE dbo.ETL_ErrorLog (...)
   CREATE TABLE dbo.stg_Orders (...)
   CREATE TABLE dbo.Orders (...)
   
   -- Create stored procedures
   CREATE PROCEDURE dbo.usp_LogError (...)
   CREATE PROCEDURE dbo.usp_UpdateWatermark (...)
   ```

2. **Configure Linked Services**
   - REST API connection
   - Azure SQL Database
   - Azure Data Lake

3. **Deploy Pipeline Activities**
   - Create 6 core activities
   - Add LogError for error handling
   - Configure dependencies

4. **Set Initial Watermark**
   ```sql
   INSERT INTO dbo.ETL_Watermark
   VALUES ('SourceAPI', 'Orders', '2024-01-01 00:00:00', ...)
   ```

5. **Test and Monitor**
   - Run Debug from Data Factory
   - Verify all activities succeed
   - Check data in production table

## Testing Results

All activities tested and verified:

```
SUCCESS - GetWatermark       - Lookup successful
SUCCESS - Copy data1         - 1000+ rows fetched
SUCCESS - TruncateStaging    - Staging cleared
SUCCESS - FlattenOrders      - JSON flattened
SUCCESS - MergeToCurated     - Duplicates removed
SUCCESS - UpdateWatermark    - Watermark advanced
SUCCESS - LogError           - Error capture working
```

## Security Features

- API key authentication via Linked Service
- Azure SQL Managed Identity for database access
- Key Vault integration for secrets
- No hardcoded credentials
- Audit trail for compliance

## Documentation

- Complete architecture documentation available
- SQL scripts for all objects
- Activity configurations and expressions
- Error handling procedures
- Troubleshooting guide

