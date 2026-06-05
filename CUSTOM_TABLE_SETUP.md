# Creating ArconLogs_CL Custom Table in Log Analytics Workspace

Complete guide to create the custom log table for Arcon logs in Azure Log Analytics.

## 📋 Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Method 1: Automatic Creation via ARM Template](#method-1-automatic-creation-via-arm-template)
4. [Method 2: Manual Creation via KQL](#method-2-manual-creation-via-kql)
5. [Method 3: Create via Data Collector API](#method-3-create-via-data-collector-api)
6. [Table Schema Reference](#table-schema-reference)
7. [Verification & Testing](#verification--testing)
8. [Troubleshooting](#troubleshooting)

---

## Overview

The `ArconLogs_CL` is a custom table that stores logs ingested from the Arcon API. The `_CL` suffix indicates it's a custom log table.

### Table Details

| Property | Value |
|----------|-------|
| **Table Name** | `ArconLogs_CL` |
| **Retention** | 30 days (default, configurable) |
| **Data Format** | JSON |
| **Source** | Arcon API via Logic App |
| **Tier** | Analytics (Pay-as-you-go) |

### Key Features

✅ Automatic schema inference from JSON data  
✅ Fully queryable with Kusto Query Language (KQL)  
✅ Integrates with Sentinel analytics rules  
✅ Supports custom retention policies  
✅ Built-in cost management  

---

## Prerequisites

### Azure Permissions Required

- ✅ **Contributor** role on Log Analytics Workspace
- ✅ Access to Azure Portal or PowerShell
- ✅ Or access to data via Log Analytics API

### Log Analytics Workspace

- ✅ Log Analytics Workspace already exists
- ✅ Know your Workspace ID
- ✅ Know your Workspace Primary Key (for API method)

### Access Methods

**Choose one:**
1. Azure Portal (UI)
2. PowerShell (command-line)
3. KQL (query)
4. Log Analytics API (programmatic)

---

## Method 1: Automatic Creation via ARM Template

The table is **automatically created** when you deploy the `azuredeploy.json` Logic App template.

### How It Works

1. Logic App is deployed and enabled
2. Recurrence trigger fires (default: every 5 minutes)
3. First run calls Arcon API
4. Logs are sent to Log Analytics via Data Collector API
5. **Log Analytics automatically creates** `ArconLogs_CL` table
6. Data schema is inferred from first batch of logs

### Timeline

```
T+0    → Logic App deployment starts
T+1min → Logic App created and enabled
T+2min → First recurrence trigger fires
T+3min → Arcon API called
T+4min → Data sent to Log Analytics
T+5min → ArconLogs_CL table auto-created ✅
```

### Verify Automatic Creation

Once Logic App has run (wait 5+ minutes), verify the table:

```kusto
// Check if table exists
ArconLogs_CL
| count

// Expected result: 0 or more records
```

---

## Method 2: Manual Creation via KQL

Create the table explicitly before deploying Logic App (optional).

### Step 1: Open Log Analytics Workspace

1. Go to **Azure Portal**
2. Navigate to **Log Analytics workspaces**
3. Select your workspace
4. Click **Logs** (on the left sidebar)

### Step 2: Create Table with KQL

Run this KQL command in the query editor:

```kusto
// Create custom table with defined schema
.create table ArconLogs_CL (
    timestamp: datetime,
    event_type: string,
    severity: string,
    message: string,
    source_ip: string,
    user_id: string,
    action: string,
    status: string,
    details: dynamic
) with (docstring="Custom log table for Arcon security events")
```

### Step 3: Verify Creation

Run verification query:

```kusto
// Verify table exists
.show tables
| where TableName == "ArconLogs_CL"
```

**Expected Output:**
```
TableName      | TableId | Kind | CslCommand | DocString
ArconLogs_CL   | ...     | ... | ...        | Custom log table for Arcon security events
```

### Step 4: Configure Table Properties

```kusto
// Set retention policy (90 days)
.alter-merge table ArconLogs_CL policy retention softdelete = 90d

// Set ingestion time (optional)
.alter table ArconLogs_CL policy ingestiontime @'{"IsEnabled": true}'
```

---

## Method 3: Create via Data Collector API

Programmatically create table using PowerShell.

### Step 1: Prepare PowerShell Script

```powershell
# Log Analytics configuration
$workspaceId = "your-workspace-id"
$workspaceKey = "your-workspace-primary-key"
$customTableName = "ArconLogs_CL"

# Sample data for table creation (schema inference)
$sampleData = @(
    @{
        timestamp = (Get-Date).ToUniversalTime()
        event_type = "LOGIN"
        severity = "INFO"
        message = "User login successful"
        source_ip = "192.168.1.1"
        user_id = "user123"
        action = "LOGIN"
        status = "SUCCESS"
        details = @{
            application = "Arcon"
            version = "1.0"
        }
    }
)

# Convert to JSON
$json = $sampleData | ConvertTo-Json
```

### Step 2: Send Data to Create Table

```powershell
# Function to send data to Log Analytics
function Send-LogAnalyticsData {
    param(
        [string]$WorkspaceId,
        [string]$WorkspaceKey,
        [string]$TableName,
        [string]$JsonData,
        [string]$TimeStampField = "timestamp"
    )
    
    # Create signature
    $method = "POST"
    $contentType = "application/json"
    $resource = "/api/logs"
    $rfc1123date = [DateTime]::UtcNow.ToString("r")
    $contentLength = $JsonData.Length
    
    $signatureInput = "$($method)`n$($contentLength)`n$($contentType)`nx-ms-date:$($rfc1123date)`n$($resource)"
    $signature = New-Object System.Security.Cryptography.HMACSHA256
    $signature.Key = [Convert]::FromBase64String($WorkspaceKey)
    $signatureHash = $signature.ComputeHash([Text.Encoding]::UTF8.GetBytes($signatureInput))
    $signatureHashB64 = [Convert]::ToBase64String($signatureHash)
    $authorization = "SharedKey $WorkspaceId`:$signatureHashB64"
    
    # Create URI
    $uri = "https://$($WorkspaceId).ods.opinsights.azure.com$($resource)?api-version=2016-04-01"
    
    # Create headers
    $headers = @{
        "Authorization" = $authorization
        "Log-Type" = $TableName
        "x-ms-date" = $rfc1123date
        "x-ms-AzureResourceId" = "/subscriptions/{subscription-id}/resourcegroups/{resource-group}/providers/microsoft.operationalinsights/workspaces/{workspace-name}"
        "Content-Type" = $contentType
    }
    
    # Send request
    $response = Invoke-WebRequest -Uri $uri -Method $method -ContentType $contentType `
        -Headers $headers -Body $JsonData -UseBasicParsing
    
    if ($response.StatusCode -eq 200) {
        Write-Host "✅ Data successfully sent to Log Analytics"
        Write-Host "Table: $TableName"
    } else {
        Write-Host "❌ Error sending data: $($response.StatusCode)"
    }
    
    return $response
}

# Send the data
Send-LogAnalyticsData -WorkspaceId $workspaceId `
    -WorkspaceKey $workspaceKey `
    -TableName $customTableName `
    -JsonData $json
```

### Step 3: Verify Table Creation

Wait 5-10 minutes, then run:

```kusto
// Query the new table
ArconLogs_CL
| count
```

---

## Table Schema Reference

### Default Schema (Auto-Inferred)

When data is first ingested, Log Analytics creates columns based on the JSON structure:

```kusto
// View table schema
.show table ArconLogs_CL schema

// Output columns:
TimeGenerated         : datetime
data_d               : dynamic (contains entire JSON object)
SourceSystem         : string  (always "RestAPI")
_ResourceId          : string
Type                 : string  (always "ArconLogs_CL")
```

### Custom Schema (Recommended)

For better performance and querying, define a custom schema:

```kusto
// Create table with explicit schema
.create table ArconLogs_CL (
    TimeGenerated: datetime,
    event_id_s: string,
    event_type_s: string,
    severity_s: string,
    message_s: string,
    source_ip_s: string,
    destination_ip_s: string,
    user_id_s: string,
    user_name_s: string,
    action_s: string,
    status_s: string,
    duration_d: real,
    response_code_s: string,
    error_message_s: string,
    details_d: dynamic,
    metadata_d: dynamic
)
```

### Column Naming Convention

Log Analytics uses column naming suffixes for type inference:

| Suffix | Data Type | Example |
|--------|-----------|---------|
| `_s` | string | `severity_s` |
| `_d` | double/real | `duration_d` |
| `_b` | boolean | `success_b` |
| `_t` | datetime | `timestamp_t` |
| `_g` | guid | `correlation_id_g` |
| (none) | dynamic | `metadata` (JSON object) |

### Example Queries by Column Type

```kusto
// Query string columns
ArconLogs_CL
| where severity_s == "CRITICAL"

// Query numeric columns
ArconLogs_CL
| where duration_d > 5000

// Query dynamic (JSON) columns
ArconLogs_CL
| where details_d.error_code != ""

// Query datetime columns
ArconLogs_CL
| where TimeGenerated > ago(24h)
```

---

## Verification & Testing

### Verify Table Exists

```kusto
// Method 1: Try to query it
ArconLogs_CL
| take 1

// Method 2: List all custom tables
.show tables
| where TableName has "_CL"
```

### Verify Schema

```kusto
// Show table schema
.show table ArconLogs_CL schema

// Show table statistics
.show table ArconLogs_CL extents
```

### Verify Data Ingestion

```kusto
// Count records
ArconLogs_CL
| count

// Show latest records
ArconLogs_CL
| sort by TimeGenerated desc
| take 10

// Check ingestion timeline
ArconLogs_CL
| summarize 
    RecordCount = count(),
    FirstRecord = min(TimeGenerated),
    LastRecord = max(TimeGenerated)
    by bin(TimeGenerated, 1h)
```

### Test Data Ingestion

```kusto
// Verify all expected columns
ArconLogs_CL
| getschema
| project ColumnName, ColumnType

// Count by severity
ArconLogs_CL
| summarize count() by severity_s

// Find any ingestion errors
_LogOperation
| where TableName == "ArconLogs_CL"
| where Operation == "Data collection error"
```

---

## Configure Table Properties

### Set Retention Policy

```kusto
// 30 days (default)
.alter-merge table ArconLogs_CL policy retention softdelete = 30d

// 90 days
.alter-merge table ArconLogs_CL policy retention softdelete = 90d

// 1 year
.alter-merge table ArconLogs_CL policy retention softdelete = 365d

// Unlimited
.alter-merge table ArconLogs_CL policy retention softdelete = 0d
```

### Set Ingestion Time Property

```kusto
// Enable ingestion time for faster queries
.alter table ArconLogs_CL policy ingestiontime @'{"IsEnabled": true}'
```

### Set Docstring (Description)

```kusto
// Add table description
.alter table ArconLogs_CL docstring "Custom security logs from Arcon API"
```

### View Current Policies

```kusto
// Show all policies for table
.show table ArconLogs_CL

// Show retention policy
.show table ArconLogs_CL policy retention

// Show ingestion time policy
.show table ArconLogs_CL policy ingestiontime
```

---

## Troubleshooting

### Issue 1: Table Not Created After Logic App Deployment

**Symptoms:**
- Query returns "No data"
- Table doesn't appear in table list

**Diagnosis:**
```kusto
// Check if table exists
.show tables
| where TableName == "ArconLogs_CL"

// Check for errors
_LogOperation
| where Operation == "Data collection error"
| where TableName == "ArconLogs_CL"
```

**Solutions:**

1. **Wait longer:** Table takes 5-10 minutes to auto-create
   - Check Logic App run history
   - Verify first run completed successfully

2. **Check Logic App:**
   ```powershell
   $runs = Get-AzLogicAppRunHistory -ResourceGroupName "your-rg" `
       -Name "ArconToSentinel-Connector"
   $runs | Where-Object {$_.Status -eq "Failed"}
   ```

3. **Manually create table** using Method 2 or 3 above

4. **Check Log Analytics credentials:**
   - Verify Workspace ID is correct
   - Verify Workspace Primary Key is correct

### Issue 2: Schema Mismatch Errors

**Symptoms:**
- "Data type mismatch" errors
- Cannot query certain columns

**Solution:**
```kusto
// Drop and recreate table with correct schema
.drop table ArconLogs_CL ifexists

// Create with explicit schema
.create table ArconLogs_CL (
    TimeGenerated: datetime,
    event_type_s: string,
    severity_s: string,
    message_s: string,
    details_d: dynamic
)
```

### Issue 3: Ingestion Failures

**Symptoms:**
- Logic App shows success but no data in table
- Errors in Log Analytics

**Diagnosis:**
```kusto
// Check ingestion failures
_LogOperation
| where Operation contains "error" or Operation contains "fail"
| where TableName == "ArconLogs_CL"
| project TimeGenerated, Operation, Details
```

**Solutions:**

1. Verify JSON structure matches schema
2. Check for special characters that need escaping
3. Verify data types match defined schema
4. Check Log Analytics retention settings

### Issue 4: High Costs from Table

**Symptoms:**
- Unexpected Log Analytics charges
- Too much data ingested

**Solutions:**

1. **Reduce logging frequency:**
   ```powershell
   # Increase pull interval in Logic App
   # From 5 minutes to 15 minutes = 3x cost reduction
   ```

2. **Filter data at source:**
   ```
   # Add to Arcon API query
   /logs?filter=severity:CRITICAL,ERROR
   ```

3. **Implement data archival:**
   ```kusto
   // Archive data older than 30 days
   .alter-merge table ArconLogs_CL policy retention softdelete = 30d
   ```

4. **Set shorter retention:**
   ```kusto
   // Reduce from 90 to 30 days
   .alter-merge table ArconLogs_CL policy retention softdelete = 30d
   ```

---

## Using Table in Sentinel

### Query in Sentinel Logs

```kusto
// Basic query
ArconLogs_CL
| project TimeGenerated, severity_s, event_type_s, message_s

// Filter by severity
ArconLogs_CL
| where severity_s in ("CRITICAL", "ERROR")

// Summarize by event type
ArconLogs_CL
| summarize count() by event_type_s

// Join with other tables
ArconLogs_CL
| join (SecurityEvent) on $left.user_id_s == $right.Account
```

### Create Alert Rule

```kusto
// Alert on high severity events
ArconLogs_CL
| where severity_s == "CRITICAL"
| where TimeGenerated > ago(5m)
| summarize AlertCount = count() by event_type_s
| where AlertCount > 5
```

### Create Dashboard Widget

```kusto
// Events over time
ArconLogs_CL
| summarize EventCount = count() by bin(TimeGenerated, 1h), severity_s
| render timechart
```

---

## Cleanup & Deletion

### Delete Table

⚠️ **Warning:** This cannot be undone. Verify before deleting.

```kusto
// Delete table
.drop table ArconLogs_CL

// Delete with confirmation
.drop table ArconLogs_CL
| where 1 == 0  // Add safety check
```

### Clear Table Data (Keep Schema)

```kusto
// Clear all data but keep table structure
.clear table ArconLogs_CL data
```

---

## Best Practices

✅ **DO:**
- Define explicit schema for better performance
- Set appropriate retention policies
- Monitor table size and costs
- Use consistent naming conventions
- Document custom columns with descriptions
- Regularly test queries

❌ **DON'T:**
- Store sensitive data (passwords, tokens)
- Use overly broad retention (unless necessary)
- Create duplicate tables with similar names
- Ignore schema mismatches
- Forget to test ingestion pipeline

---

## Reference

| Resource | Link |
|----------|------|
| Log Analytics Tables | [Azure Docs](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/manage-cost-storage) |
| KQL Language | [Azure Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/) |
| Data Collector API | [Azure Docs](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/data-collector-api) |
| Sentinel Analytics | [Azure Docs](https://docs.microsoft.com/en-us/azure/sentinel/quickstart-onboard) |

---

## Summary

| Method | Time | Effort | Best For |
|--------|------|--------|----------|
| **Automatic (ARM)** | 5-10 min | None | Production deployment |
| **Manual (KQL)** | 2-5 min | Low | Pre-deployment setup |
| **API (PowerShell)** | 5-10 min | Medium | Programmatic control |

**Recommended:** Use Automatic creation (deployed with Logic App)

---

**For questions or issues, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md)**

**For deployment, see [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)**
