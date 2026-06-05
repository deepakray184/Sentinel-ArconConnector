# Troubleshooting Guide - Arcon to Microsoft Sentinel Connector

Comprehensive guide to diagnose and resolve common issues.

## 📋 Table of Contents

1. [Quick Diagnostics](#quick-diagnostics)
2. [Common Issues & Solutions](#common-issues--solutions)
3. [Logic App Errors](#logic-app-errors)
4. [API Connection Issues](#api-connection-issues)
5. [Log Analytics Issues](#log-analytics-issues)
6. [Performance Problems](#performance-problems)
7. [Security & Authentication](#security--authentication)
8. [Advanced Debugging](#advanced-debugging)

---

## Quick Diagnostics

### Check Logic App Status

```powershell
# Get overall status
$logicApp = Get-AzLogicApp -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector"

Write-Host "Logic App Status:" $logicApp.State
Write-Host "Created:" $logicApp.CreatedTime
Write-Host "Location:" $logicApp.Location
```

### View Recent Run History

```powershell
# Get last 10 runs
Get-AzLogicAppRunHistory -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector" -MaximumItemCount 10 | 
    Select-Object Name, Status, StartTime, EndTime, TriggerName | 
    Format-Table

# Get only failed runs
Get-AzLogicAppRunHistory -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector" | 
    Where-Object {$_.Status -eq "Failed"} | 
    Select-Object -First 5 | 
    Format-Table
```

### Get Detailed Error Information

```powershell
# Get details of a failed run
$run = Get-AzLogicAppRunHistory -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector" | 
    Where-Object {$_.Status -eq "Failed"} | 
    Select-Object -First 1

# Get full run details
Get-AzLogicAppRun -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector" `
    -RunName $run.Name | 
    ConvertTo-Json -Depth 10
```

---

## Common Issues & Solutions

### Issue 1: Logic App Never Runs

**Symptoms:**
- Run history is empty
- No executions shown in Azure Portal

**Diagnosis:**
```powershell
# Check if Logic App is enabled
$logicApp = Get-AzLogicApp -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector"

if ($logicApp.State -eq "Enabled") {
    Write-Host "✅ Logic App is enabled"
} else {
    Write-Host "❌ Logic App is disabled"
}
```

**Solutions:**

1. **Enable the Logic App:**
   ```powershell
   # Enable Logic App
   $logicApp | Set-AzLogicApp -State Enabled
   ```

2. **Check recurrence trigger:**
   - Go to Azure Portal → Logic App → Edit Designer
   - Verify the Recurrence trigger is configured correctly
   - Check that the interval and frequency are set properly

3. **Wait for next scheduled run:**
   - If interval is 5 minutes, wait up to 5 minutes for next run

4. **Manually trigger the run:**
   ```powershell
   # Trigger manual run
   Invoke-AzLogicAppRun -ResourceGroupName "your-rg" `
       -Name "ArconToSentinel-Connector" `
       -TriggerName "Recurrence"
   ```

---

### Issue 2: All Runs Fail Immediately

**Symptoms:**
- Every run shows "Failed" status
- Errors appear in first action (Initialize variables)

**Diagnosis:**
```powershell
# Get detailed error from run
$failedRun = Get-AzLogicAppRunHistory -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector" | 
    Where-Object {$_.Status -eq "Failed"} | 
    Select-Object -First 1

$runDetails = Get-AzLogicAppRun -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector" `
    -RunName $failedRun.Name

$runDetails.Outputs
```

**Common Causes & Solutions:**

#### A. Template Deployment Error

**Error:** `The template deployment failed...`

**Solution:**
1. Verify azuredeploy.json is valid JSON
   ```powershell
   $json = Get-Content azuredeploy.json | ConvertFrom-Json
   Write-Host "✅ Valid JSON"
   ```

2. Redeploy the Logic App:
   ```powershell
   Remove-AzLogicApp -ResourceGroupName "your-rg" `
       -Name "ArconToSentinel-Connector" -Force
   
   # Then redeploy using DEPLOYMENT_GUIDE.md
   ```

#### B. Parameter Reference Error

**Error:** `Reference to parameter 'arconApiUrl' is not valid...`

**Solution:**
1. Check parameter names match exactly
2. Verify all required parameters are provided
3. Ensure parameter values are not empty

---

### Issue 3: Authentication Failures

**Symptoms:**
- "Unauthorized" or "401" errors in API calls
- "Invalid API Key" errors

**Diagnosis:**
```powershell
# Test Arcon API directly
$apiUrl = "https://api.arcon.example.com/v1"
$apiKey = "your-api-key"

$headers = @{
    "X-API-Key" = $apiKey
    "Content-Type" = "application/json"
}

try {
    $response = Invoke-WebRequest -Uri "$apiUrl/logs?limit=1" `
        -Headers $headers -Method GET -TimeoutSec 10
    Write-Host "✅ API accessible"
} catch {
    Write-Host "❌ API error: $_"
}
```

**Solutions:**

1. **Verify API Key:**
   - Check API key hasn't expired
   - Confirm key has `/logs` endpoint permission
   - Test key format (sometimes requires decoding)

2. **Update API Key in Logic App:**
   ```powershell
   # Edit Logic App definition
   $logicApp = Get-AzLogicApp -ResourceGroupName "your-rg" `
       -Name "ArconToSentinel-Connector"
   
   # Update parameters in the definition
   # Then save using Set-AzLogicApp
   ```

3. **Store in Key Vault:**
   - Move API key to Azure Key Vault
   - Reference from Logic App using Key Vault connector

---

### Issue 4: No Logs Appearing in Log Analytics

**Symptoms:**
- Logs not showing in ArconLogs_CL table
- Logic App runs successfully but no data ingested

**Diagnosis:**

```kusto
// Check if table exists and has data
ArconLogs_CL
| count

// Check for any errors in ingestion
_LogOperation
| where Operation == "Data collection error"
| where TimeGenerated > ago(24h)
```

**Solutions:**

1. **Verify Log Analytics credentials:**
   ```powershell
   # Test Log Analytics access
   $workspaceId = "your-workspace-id"
   $workspaceKey = "your-workspace-key"
   
   # Try sending test data
   # (Use script from Log Analytics documentation)
   ```

2. **Check table name:**
   - Verify `logTableName` parameter matches expected table
   - Custom tables must end with `_CL` suffix
   - Table names are case-sensitive

3. **Monitor send action:**
   - In Logic App run details, check "Send_to_Log_Analytics" action
   - Look for error messages in action output
   - Verify request body is properly formatted

4. **Check data retention:**
   ```kusto
   // Check retention settings
   .show retention
   
   // Ensure table has proper retention
   .alter-merge table ArconLogs_CL policy retention softdelete = 90d
   ```

---

### Issue 5: Timeout Errors

**Symptoms:**
- "Request timeout" or "Connection timeout" errors
- Runs fail after 5 minutes

**Diagnosis:**
```powershell
# Check API response time
$apiUrl = "https://api.arcon.example.com/v1"
$apiKey = "your-api-key"

$timer = [System.Diagnostics.Stopwatch]::StartNew()

try {
    $response = Invoke-WebRequest -Uri "$apiUrl/logs?limit=100" `
        -Headers @{"X-API-Key" = $apiKey} -Method GET
    $timer.Stop()
    Write-Host "Response time: $($timer.ElapsedMilliseconds)ms"
} catch {
    Write-Host "Error: $_"
}
```

**Solutions:**

1. **Increase timeout in Logic App:**
   - Edit Call_Arcon_API action
   - Increase timeout from 5 minutes (PT5M) to 10 minutes (PT10M)

2. **Reduce batch size:**
   - Change `limit` from 100 to 50 in API call
   - Reduces processing time per request

3. **Increase pull interval:**
   - Change pull interval from 5 to 10 minutes
   - Reduces load on Arcon API

4. **Check network connectivity:**
   ```powershell
   # Test network path to Arcon API
   Test-NetConnection -ComputerName "api.arcon.example.com" -Port 443
   ```

---

### Issue 6: High Costs or Excessive Runs

**Symptoms:**
- Logic App runs very frequently
- Unexpected Azure costs

**Diagnosis:**
```powershell
# Count runs in past 24 hours
$runs = Get-AzLogicAppRunHistory -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector" | 
    Where-Object {$_.StartTime -gt (Get-Date).AddDays(-1)}

Write-Host "Runs in past 24 hours: $($runs.Count)"
```

**Solutions:**

1. **Adjust recurrence interval:**
   - Increase from 5 minutes to 10 or 15 minutes
   - Edit trigger in Logic App designer

2. **Implement filtering:**
   - Add API query filters: `?filter=severity:ERROR,CRITICAL`
   - Reduce data volume per run

3. **Monitor costs:**
   ```powershell
   # Get Logic App billing info
   Get-AzLogicApp -ResourceGroupName "your-rg" `
       -Name "ArconToSentinel-Connector" | 
       Select-Object Name, SkuName
   ```

---

## Logic App Errors

### Error: "Self reference is not supported"

**Cause:** Trying to assign variable to itself  
**Solution:** Use Compose action to calculate, then SetVariable to assign

```json
{
  "Compose_IncrementedPage": {
    "type": "Compose",
    "inputs": "@add(variables('CurrentPage'), 1)"
  },
  "Set_CurrentPage": {
    "type": "SetVariable",
    "inputs": {
      "name": "CurrentPage",
      "value": "@outputs('Compose_IncrementedPage')"
    }
  }
}
```

### Error: "Cannot read property 'pagination' of null"

**Cause:** API response doesn't have pagination object  
**Solution:** Verify API response schema matches expected format

```kusto
// Expected format:
{
  "success": true,
  "data": [ /* logs */ ],
  "pagination": {
    "has_next": false
  }
}
```

### Error: "The workflow with 'Response' action type should not have triggers"

**Cause:** Response action incompatible with recurrence triggers  
**Solution:** Remove Response action from workflow

---

## API Connection Issues

### Test Arcon API Connectivity

```powershell
# Comprehensive API test
function Test-ArconAPI {
    param(
        [string]$ApiUrl,
        [string]$ApiKey
    )
    
    $headers = @{
        "X-API-Key" = $ApiKey
        "Content-Type" = "application/json"
    }
    
    # Test 1: Basic connectivity
    Write-Host "Test 1: Basic connectivity..." -ForegroundColor Cyan
    try {
        $response = Invoke-WebRequest -Uri $ApiUrl `
            -Headers $headers -Method GET -TimeoutSec 10
        Write-Host "✅ Connected" -ForegroundColor Green
    } catch {
        Write-Host "❌ Failed: $_" -ForegroundColor Red
        return
    }
    
    # Test 2: Logs endpoint
    Write-Host "Test 2: Logs endpoint..." -ForegroundColor Cyan
    try {
        $response = Invoke-WebRequest -Uri "$ApiUrl/logs?limit=1" `
            -Headers $headers -Method GET -TimeoutSec 10
        $data = $response.Content | ConvertFrom-Json
        Write-Host "✅ Logs endpoint working" -ForegroundColor Green
        Write-Host "   Success: $($data.success)"
        Write-Host "   Records: $($data.data.Count)"
    } catch {
        Write-Host "❌ Failed: $_" -ForegroundColor Red
    }
    
    # Test 3: Pagination
    Write-Host "Test 3: Pagination support..." -ForegroundColor Cyan
    try {
        $response = Invoke-WebRequest -Uri "$ApiUrl/logs?limit=1&offset=0" `
            -Headers $headers -Method GET -TimeoutSec 10
        $data = $response.Content | ConvertFrom-Json
        if ($data.pagination) {
            Write-Host "✅ Pagination supported" -ForegroundColor Green
        } else {
            Write-Host "⚠️ Pagination not found in response" -ForegroundColor Yellow
        }
    } catch {
        Write-Host "❌ Failed: $_" -ForegroundColor Red
    }
}

# Run tests
Test-ArconAPI -ApiUrl "https://api.arcon.example.com/v1" `
    -ApiKey "your-api-key"
```

### Network Troubleshooting

```powershell
# Check firewall/network access
$apiHost = "api.arcon.example.com"
$apiPort = 443

# Test DNS resolution
Write-Host "Resolving DNS..."
$dns = Resolve-DnsName -Name $apiHost -ErrorAction SilentlyContinue
if ($dns) {
    Write-Host "✅ DNS resolved to: $($dns.IPAddress)"
} else {
    Write-Host "❌ DNS resolution failed"
}

# Test port connectivity
Write-Host "Testing port connectivity..."
$connection = Test-NetConnection -ComputerName $apiHost -Port $apiPort
if ($connection.TcpTestSucceeded) {
    Write-Host "✅ Port $apiPort is accessible"
} else {
    Write-Host "❌ Cannot reach port $apiPort"
}
```

---

## Log Analytics Issues

### Verify Log Analytics Workspace

```kusto
// Check workspace health
.show databases

// Check table statistics
ArconLogs_CL
| summarize 
    RecordCount = count(),
    FirstRecord = min(TimeGenerated),
    LastRecord = max(TimeGenerated)

// Check for ingestion errors
_LogOperation
| where Operation == "Data collection error"
| summarize ErrorCount = count() by Details
```

### Troubleshoot Log Ingestion

```powershell
# Get Log Analytics workspace details
$workspace = Get-AzOperationalInsightsWorkspace `
    -ResourceGroupName "your-rg" `
    -Name "your-workspace-name"

Write-Host "Workspace ID: $($workspace.CustomerId)"
Write-Host "Region: $($workspace.Location)"
Write-Host "SKU: $($workspace.Sku)"
```

---

## Performance Problems

### Logic App Running Slowly

**Diagnosis:**
```powershell
# Get average run duration
$runs = Get-AzLogicAppRunHistory -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector" -MaximumItemCount 50

$durations = $runs | ForEach-Object {
    if ($_.EndTime) {
        ($_.EndTime - $_.StartTime).TotalSeconds
    }
}

$avgDuration = ($durations | Measure-Object -Average).Average
Write-Host "Average run duration: $($avgDuration) seconds"
```

**Optimization Steps:**

1. **Reduce batch size:**
   - Change API `limit` from 100 to 50

2. **Implement filtering:**
   - Query only necessary severity levels
   - Use date range filters

3. **Enable parallel processing:**
   - In For_Each loop, increase concurrency

4. **Monitor Arcon API:**
   - Check if API is slow
   - May need to increase pull interval

---

## Security & Authentication

### Store Secrets in Key Vault

```powershell
# Create Azure Key Vault
$keyVault = New-AzKeyVault -ResourceGroupName "your-rg" `
    -VaultName "arcon-secrets" `
    -Location "eastus"

# Store API key
$apiKeySecret = ConvertTo-SecureString "your-api-key" `
    -AsPlainText -Force

Set-AzKeyVaultSecret -VaultName "arcon-secrets" `
    -Name "ArconApiKey" `
    -SecretValue $apiKeySecret

# Store workspace key
$workspaceKeySecret = ConvertTo-SecureString "your-workspace-key" `
    -AsPlainText -Force

Set-AzKeyVaultSecret -VaultName "arcon-secrets" `
    -Name "WorkspaceKey" `
    -SecretValue $workspaceKeySecret
```

### Audit API Key Usage

```kusto
// Monitor API key authentication failures
_LogOperation
| where Operation == "Data collection authentication error"
| where TimeGenerated > ago(24h)
| summarize FailureCount = count() by Details, bin(TimeGenerated, 1h)
```

---

## Advanced Debugging

### Enable Logic App Diagnostic Logs

```powershell
# Enable diagnostics
$diagSettings = New-AzDiagnosticSetting `
    -Name "ArconLogicAppDiagnostics" `
    -ResourceId "/subscriptions/{subId}/resourceGroups/{rg}/providers/Microsoft.Logic/workflows/{name}" `
    -WorkspaceId $workspace.ResourceId `
    -Enabled $true

# All logs
$diagSettings | Enable-AzDiagnosticSetting -Category @("WorkflowRuntime") `
    -Enabled $true
```

### Export Run Data for Analysis

```powershell
# Export all runs to CSV
$runs = Get-AzLogicAppRunHistory -ResourceGroupName "your-rg" `
    -Name "ArconToSentinel-Connector" -MaximumItemCount 100

$runs | Select-Object Name, Status, StartTime, EndTime, TriggerName | 
    Export-Csv -Path "logic-app-runs.csv" -NoTypeInformation

# Analyze in PowerShell
$csv = Import-Csv "logic-app-runs.csv"
$failed = $csv | Where-Object {$_.Status -eq "Failed"} | Measure-Object
Write-Host "Failed runs: $($failed.Count)"
```

---

## Contact Support

If issues persist:

1. ✅ Check all steps above
2. ✅ Collect diagnostic logs
3. ✅ Document error messages and run IDs
4. ✅ Contact Azure Support or Arcon support

**Information to provide:**
- Logic App run ID
- Full error message
- Steps taken for troubleshooting
- Arcon API version
- Azure subscription ID and resource group name

---

**For deployment help, see [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)**

**For general information, see [README.md](README.md)**
