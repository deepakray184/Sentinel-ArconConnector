# Configuration Template - Arcon to Microsoft Sentinel Connector

Pre-deployment configuration checklist and reference guide.

## 📋 Pre-Deployment Checklist

Complete this checklist before deploying the connector.

### Azure Environment
- [ ] Azure subscription selected and active
- [ ] Contributor role assigned to your user account
- [ ] Target resource group identified (or will be created)
- [ ] Target Azure region selected (e.g., eastus)

### Log Analytics Workspace
- [ ] Log Analytics Workspace already exists
- [ ] Workspace location selected (note: should match Logic App region for best performance)
- [ ] Workspace Workspace ID obtained
  ```
  Location: Azure Portal → Log Analytics workspaces → [Your Workspace] → Overview
  Copy: Workspace ID
  ```
- [ ] Workspace Primary Key obtained
  ```
  Location: Azure Portal → Log Analytics workspaces → [Your Workspace] → Agents management
  Copy: Primary Key
  ```
- [ ] Workspace has sufficient data retention configured
  - Recommended: 90+ days for security logs
  - Can be configured under Workspace Settings

### Arcon API Configuration
- [ ] Arcon API endpoint URL verified (e.g., https://api.arcon.example.com/v1)
- [ ] Arcon API Key obtained and securely stored
  - Store temporarily in a password manager or Key Vault
  - Do NOT commit to source code
- [ ] API Key has permissions for `/logs` endpoint
- [ ] API Key is NOT expired
- [ ] Arcon API is publicly accessible (or accessible from Azure)
- [ ] Tested API connectivity with curl or Postman:
  ```bash
  curl -H "X-API-Key: your-api-key" \
       https://api.arcon.example.com/v1/logs?limit=1
  ```

### Deployment Method
- [ ] Deployment method selected (Portal / PowerShell / Azure CLI)
- [ ] Necessary tools installed:
  - **Portal**: Web browser
  - **PowerShell**: PowerShell 7.0+ and Az module
  - **Azure CLI**: Azure CLI 2.0+
- [ ] User authenticated to Azure account

### Network & Security
- [ ] Firewall allows outbound HTTPS (port 443) to Arcon API
- [ ] Firewall allows outbound HTTPS to Log Analytics
- [ ] VPN or proxy configuration (if needed) configured
- [ ] No IP whitelist blocking required
- [ ] API Key will be stored securely after deployment

---

## Deployment Parameters Reference

### Required Parameters

#### Azure Resource Parameters

| Parameter | Type | Example | Notes |
|-----------|------|---------|-------|
| **location** | string | `eastus` | Azure region for Logic App |
| **Subscription ID** | string | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | Your Azure subscription |
| **Resource Group** | string | `my-resource-group` | Where Logic App will be deployed |

#### Logic App Parameters

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| **logicAppName** | string | `ArconToSentinel-Connector` | Name of the Logic App (must be unique) |
| **pullIntervalMinutes** | integer | `5` | How often to pull logs (1-60 recommended) |

#### Arcon API Parameters

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| **arconApiUrl** | string | ✅ YES | Base URL without trailing slash |
| **arconApiKey** | securestring | ✅ YES | API authentication key (kept secure) |

**Examples:**
```
✅ Correct: https://api.arcon.example.com/v1
❌ Incorrect: https://api.arcon.example.com/v1/
❌ Incorrect: https://api.arcon.example.com
```

#### Log Analytics Parameters

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| **workspaceId** | string | ✅ YES | Log Analytics Workspace ID (GUID) |
| **workspaceKey** | securestring | ✅ YES | Workspace Primary Key (kept secure) |
| **logTableName** | string | `ArconLogs_CL` | Custom log table name (must end in _CL) |

---

## Parameter Configuration Guide

### How to Find Parameters

#### Arcon API Configuration

```powershell
# Example: Arcon API configuration
$arconConfig = @{
    ApiUrl = "https://api.arcon.example.com/v1"
    ApiKey = "sk-abc123def456ghi789jkl012"
    LogsEndpoint = "/logs"
}

# Test the configuration
$headers = @{
    "X-API-Key" = $arconConfig.ApiKey
    "Content-Type" = "application/json"
}

$response = Invoke-WebRequest -Uri "$($arconConfig.ApiUrl)$($arconConfig.LogsEndpoint)?limit=1" `
    -Headers $headers
```

#### Log Analytics Configuration

```powershell
# Get Log Analytics Workspace details
$workspace = Get-AzOperationalInsightsWorkspace `
    -ResourceGroupName "your-rg" `
    -Name "your-workspace-name"

Write-Host "Workspace ID: $($workspace.CustomerId)"

# Get Primary Key from Portal:
# Azure Portal → Log Analytics workspaces → [Name] → Agents management → Primary Key
```

#### Azure Configuration

```powershell
# Get your subscription and location
$context = Get-AzContext
Write-Host "Subscription ID: $($context.Subscription.Id)"

# List available locations
Get-AzLocation | Where-Object {$_.DisplayName -like "*East*"} | 
    Select-Object DisplayName, Location | Format-Table
```

---

## Configuration Scenarios

### Scenario 1: Small Environment (< 1000 logs/hour)

**Configuration:**
```json
{
  "pullIntervalMinutes": 5,
  "arconBatchSize": 100,
  "logTableName": "ArconLogs_CL",
  "workspaceRetention": 30
}
```

**Estimated Cost:**
- ~8,640 Logic App runs/month
- Minimal Log Analytics ingestion costs
- Recommended for Dev/Test environments

### Scenario 2: Medium Environment (1000-10000 logs/hour)

**Configuration:**
```json
{
  "pullIntervalMinutes": 10,
  "arconBatchSize": 100,
  "logTableName": "ArconLogs_CL",
  "workspaceRetention": 90
}
```

**Estimated Cost:**
- ~4,320 Logic App runs/month
- Moderate Log Analytics ingestion costs
- Recommended for Production environments

### Scenario 3: Large Environment (> 10000 logs/hour)

**Configuration:**
```json
{
  "pullIntervalMinutes": 15,
  "arconBatchSize": 100,
  "logTableName": "ArconLogs_CL",
  "workspaceRetention": 180,
  "enableParallelProcessing": true
}
```

**Estimated Cost:**
- ~2,880 Logic App runs/month
- High Log Analytics ingestion costs
- Consider archive strategy
- Recommended for Enterprise environments

---

## Advanced Configuration

### Custom Log Table Schema

The connector creates a custom table with this schema:

```kusto
// Table: ArconLogs_CL
// Data Types:
ArconLogs_CL
| take 0
| project 
    TimeGenerated,              // datetime - ingestion time
    data_d,                     // dynamic - raw JSON data from Arcon
    SourceSystem                // string - "RestAPI"
```

### Enable Key Vault Integration

For enhanced security, store secrets in Azure Key Vault:

```powershell
# Create Key Vault
$kv = New-AzKeyVault -ResourceGroupName "your-rg" `
    -VaultName "arcon-secrets" `
    -Location "eastus"

# Store Arcon API Key
$apiKeySecret = ConvertTo-SecureString "sk-xxxxx" -AsPlainText -Force
Set-AzKeyVaultSecret -VaultName "arcon-secrets" `
    -Name "ArconApiKey" -SecretValue $apiKeySecret

# Reference in Logic App:
# Use @variables('arconApiKey') pointing to Key Vault
```

### Configure Data Retention

```kusto
// Set retention to 90 days
.alter-merge table ArconLogs_CL policy retention softdelete = 90d

// Archive older data to storage
.alter table ArconLogs_CL policy externaldata 
    @'[{"storageConnectionString":"DefaultEndpointsProtocol=https;..."}]'
```

### Enable Diagnostics & Monitoring

```powershell
# Enable Logic App diagnostics
$workspace = Get-AzOperationalInsightsWorkspace `
    -ResourceGroupName "your-rg" -Name "your-workspace"

Set-AzDiagnosticSetting -ResourceId "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Logic/workflows/{name}" `
    -WorkspaceId $workspace.ResourceId `
    -Enabled $true `
    -Categories "WorkflowRuntime"
```

---

## Environment Variables Template

Create a `.env` file (for reference, don't commit to git):

```
# Azure Configuration
AZURE_SUBSCRIPTION_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_RESOURCE_GROUP=my-resource-group
AZURE_LOCATION=eastus

# Logic App Configuration
LOGIC_APP_NAME=ArconToSentinel-Connector
PULL_INTERVAL_MINUTES=5

# Arcon API Configuration
ARCON_API_URL=https://api.arcon.example.com/v1
ARCON_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxx

# Log Analytics Configuration
LOG_ANALYTICS_WORKSPACE_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
LOG_ANALYTICS_WORKSPACE_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
LOG_TABLE_NAME=ArconLogs_CL

# Optional Security
KEY_VAULT_NAME=arcon-secrets
KEY_VAULT_RESOURCE_GROUP=my-resource-group
```

---

## Deployment Configuration JSON

Use this template for your deployment parameters:

```json
{
  "deploymentMetadata": {
    "projectName": "Arcon-Sentinel-Connector",
    "deploymentDate": "2026-06-05",
    "deploymentBy": "your-name",
    "environment": "production"
  },
  "azureConfiguration": {
    "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "resourceGroup": "my-resource-group",
    "location": "eastus"
  },
  "logicAppConfiguration": {
    "name": "ArconToSentinel-Connector",
    "sku": "Standard",
    "pullIntervalMinutes": 5
  },
  "arconConfiguration": {
    "apiUrl": "https://api.arcon.example.com/v1",
    "apiKeyVaultReference": "ArconApiKey",
    "apiKeyVaultName": "arcon-secrets",
    "endpoints": {
      "logs": "/logs",
      "health": "/health"
    }
  },
  "logAnalyticsConfiguration": {
    "workspaceName": "my-log-analytics",
    "workspaceId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "customTableName": "ArconLogs_CL",
    "retentionDays": 90,
    "archiveEnabled": true
  },
  "monitoringConfiguration": {
    "enableDiagnostics": true,
    "enableAlerts": true,
    "alertingEmail": "admin@example.com"
  }
}
```

---

## Post-Deployment Configuration

### Update Pull Interval

If you need to change how often the Logic App runs:

1. Go to **Azure Portal** → **Logic Apps** → **Your Logic App**
2. Click **Edit** in the designer
3. Select the **Recurrence** trigger
4. Update **Interval** and **Frequency**
5. Click **Save**

### Modify Batch Size

To process more or fewer logs per run:

1. Edit Logic App designer
2. Find **Call_Arcon_API** action
3. Update query parameter: `limit=100` → `limit=50` (or desired number)
4. Save and test

### Change Log Table Name

⚠️ **Warning:** Cannot change after deployment without manual migration

To use a different table name:
1. Create new Logic App with desired table name
2. Migrate historical data if needed
3. Update Sentinel queries and rules

---

## Cost Estimation

### Monthly Cost Breakdown

**Logic App Costs:**
- Recurrence trigger: $0.000002 per execution
- Actions: ~$0.000001 per action
- ~8,640 runs/month (5-minute interval) = ~$0.02/month

**Log Analytics Costs:**
- Ingestion: $2.99 per GB
- Example: 1000 logs/hour × 30 days = ~1-5 GB = $3-15/month

**Typical Total:** $5-20/month

### Cost Optimization Tips

1. ✅ Increase pull interval from 5 to 15 minutes → 60% cost reduction
2. ✅ Filter logs at source (only important events) → 50% reduction
3. ✅ Archive old data to storage → 70% cost reduction
4. ✅ Use 2-hour retention instead of 90-day → 30% reduction

---

## Security Checklist

- [ ] API keys NOT hardcoded in templates
- [ ] Secrets stored in Azure Key Vault
- [ ] HTTPS enforced for all API calls
- [ ] Log Analytics data encrypted at rest
- [ ] Minimal permissions assigned (least privilege)
- [ ] Audit logging enabled
- [ ] Firewall rules reviewed
- [ ] Secrets rotated every 90 days
- [ ] Deployment history reviewed

---

## Validation Commands

Use these commands to validate your configuration:

```powershell
# Validate Arcon API configuration
Write-Host "Testing Arcon API..."
$response = Invoke-WebRequest -Uri "https://api.arcon.example.com/v1/health" `
    -Headers @{"X-API-Key" = $apiKey}
if ($response.StatusCode -eq 200) { Write-Host "✅ Arcon API OK" }

# Validate Log Analytics configuration
Write-Host "Testing Log Analytics..."
$workspace = Get-AzOperationalInsightsWorkspace `
    -ResourceGroupName "your-rg" -Name "your-workspace"
if ($workspace) { Write-Host "✅ Log Analytics OK" }

# Validate Azure permissions
Write-Host "Checking Azure permissions..."
$context = Get-AzContext
Write-Host "✅ Connected as: $($context.Account.Id)"
```

---

## Reference Links

- [Azure Logic Apps Documentation](https://docs.microsoft.com/en-us/azure/logic-apps/)
- [Log Analytics Data Collector API](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/data-collector-api)
- [Azure Key Vault Documentation](https://docs.microsoft.com/en-us/azure/key-vault/)
- [Microsoft Sentinel Documentation](https://docs.microsoft.com/en-us/azure/sentinel/)

---

## Support Resources

For issues with:
- **Deployment**: See [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)
- **Troubleshooting**: See [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- **General Info**: See [README.md](README.md)
- **Arcon API**: Contact Arcon support
- **Azure Services**: Contact Azure support

---

**Version**: 1.0.0  
**Last Updated**: June 2026  
**Document Type**: Configuration Reference
