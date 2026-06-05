# Deployment Guide - Arcon to Microsoft Sentinel Connector

Complete step-by-step instructions for deploying the Arcon to Sentinel connector.

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Pre-Deployment Setup](#pre-deployment-setup)
3. [Option 1: Deploy via Azure Portal](#option-1-deploy-via-azure-portal)
4. [Option 2: Deploy via PowerShell](#option-2-deploy-via-powershell)
5. [Option 3: Deploy via Azure CLI](#option-3-deploy-via-azure-cli)
6. [Post-Deployment Verification](#post-deployment-verification)
7. [Monitoring Setup](#monitoring-setup)
8. [Rollback Procedure](#rollback-procedure)

---

## Prerequisites

### Required Access & Permissions

- ✅ **Azure Account** with active subscription
- ✅ **Contributor role** on the subscription or resource group
- ✅ **Log Analytics Workspace** (Standard tier or higher recommended)
- ✅ **Arcon API credentials** (API URL and API Key)
- ✅ **API key permissions** to access `/logs` endpoint in Arcon

### Software Requirements

**For PowerShell Deployment:**
- PowerShell 7.0+ or PowerShell Core
- Azure PowerShell module (Az)

**For Azure CLI Deployment:**
- Azure CLI 2.0+

**For Portal Deployment:**
- Web browser (Chrome, Edge, or Firefox)

### Install Required Tools

#### Option A: Install Azure PowerShell

```powershell
# Install Azure PowerShell module
Install-Module -Name Az -AllowClobber -Force

# Verify installation
Get-Module -Name Az

# Update to latest version (optional)
Update-Module -Name Az
```

#### Option B: Install Azure CLI

```bash
# For Windows (using Chocolatey)
choco install azure-cli

# For macOS (using Homebrew)
brew install azure-cli

# For Linux (using apt)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Verify installation
az --version
```

---

## Pre-Deployment Setup

### Step 1: Gather Required Information

Create a deployment configuration file with the following information:

```text
AZURE_SUBSCRIPTION_ID = <your-subscription-id>
AZURE_RESOURCE_GROUP = <your-resource-group-name>
AZURE_LOCATION = eastus  # or your preferred location
LOGIC_APP_NAME = ArconToSentinel-Connector
ARCON_API_URL = https://api.arcon.example.com/v1
ARCON_API_KEY = <your-arcon-api-key>
LOG_ANALYTICS_WORKSPACE_ID = <your-workspace-id>
LOG_ANALYTICS_WORKSPACE_KEY = <your-workspace-key>
LOG_TABLE_NAME = ArconLogs_CL
PULL_INTERVAL_MINUTES = 5
```

### Step 2: Find Your Log Analytics Workspace Details

1. Go to [Azure Portal](https://portal.azure.com)
2. Navigate to **Log Analytics workspaces**
3. Select your workspace
4. In the left panel, click **Agents management**
5. Copy the following:
   - **Workspace ID** (on the Windows tab)
   - **Primary Key** (on the Windows tab)

### Step 3: Verify Arcon API Access

```powershell
# Test Arcon API connectivity
$apiUrl = "https://api.arcon.example.com/v1"
$apiKey = "your-api-key"

$headers = @{
    "X-API-Key" = $apiKey
    "Content-Type" = "application/json"
}

try {
    $response = Invoke-WebRequest -Uri "$apiUrl/logs?limit=1" -Headers $headers -Method GET
    Write-Host "✅ Arcon API is accessible"
    Write-Host "Status Code: $($response.StatusCode)"
} catch {
    Write-Host "❌ Failed to connect to Arcon API"
    Write-Host "Error: $_"
}
```

### Step 4: Create Azure Resource Group (if needed)

```powershell
# Connect to Azure
Connect-AzAccount

# Create resource group
New-AzResourceGroup `
    -Name "your-resource-group" `
    -Location "eastus"
```

---

## Option 1: Deploy via Azure Portal

### Step-by-Step Portal Deployment

#### 1. Navigate to Template Deployment

1. Go to [Azure Portal](https://portal.azure.com)
2. Click **Create a resource**
3. Search for **Template deployment** (or "Deploy a custom template")
4. Click **Create**

#### 2. Upload the ARM Template

1. Select **Build your own template in the editor**
2. Copy the entire content of `azuredeploy.json`
3. Paste into the editor
4. Click **Save**

#### 3. Fill in Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Subscription** | Select your subscription | |
| **Resource Group** | Create new or select existing | |
| **Location** | Choose region (e.g., eastus) | Should match your Log Analytics workspace region |
| **Logic App Name** | ArconToSentinel-Connector | Custom name for your Logic App |
| **Arcon Api Url** | https://api.arcon.example.com/v1 | Your Arcon API endpoint |
| **Arcon Api Key** | [paste your API key] | Keep secure, don't share |
| **Workspace Id** | [paste workspace ID] | From Log Analytics workspace |
| **Workspace Key** | [paste primary key] | Keep secure, don't share |
| **Log Table Name** | ArconLogs_CL | Custom log table name |
| **Pull Interval Minutes** | 5 | Adjust based on log volume |

#### 4. Review and Create

1. Click **Review + create**
2. Verify all parameters are correct
3. Accept terms and conditions
4. Click **Create**

#### 5. Monitor Deployment

1. Wait for deployment to complete (usually 1-2 minutes)
2. Click **Go to resource** to view the Logic App
3. Check the run history to verify first execution

---

## Option 2: Deploy via PowerShell

### Complete PowerShell Deployment Script

```powershell
# ============================================================
# Arcon to Sentinel Connector - PowerShell Deployment Script
# ============================================================

# 1. Set Parameters
$subscriptionId = "your-subscription-id"
$resourceGroupName = "your-resource-group"
$location = "eastus"
$logicAppName = "ArconToSentinel-Connector"
$templatePath = "./azuredeploy.json"

# Arcon Configuration
$arconApiUrl = "https://api.arcon.example.com/v1"
$arconApiKey = "your-arcon-api-key"

# Log Analytics Configuration
$workspaceId = "your-workspace-id"
$workspaceKey = "your-workspace-key"

# Additional Settings
$logTableName = "ArconLogs_CL"
$pullIntervalMinutes = 5

# ============================================================
# 2. Connect to Azure
# ============================================================
Write-Host "Connecting to Azure..." -ForegroundColor Cyan
Connect-AzAccount
Set-AzContext -SubscriptionId $subscriptionId

# ============================================================
# 3. Create Resource Group (if it doesn't exist)
# ============================================================
Write-Host "Checking resource group..." -ForegroundColor Cyan
$rg = Get-AzResourceGroup -Name $resourceGroupName -ErrorAction SilentlyContinue

if (-not $rg) {
    Write-Host "Creating resource group: $resourceGroupName" -ForegroundColor Green
    New-AzResourceGroup -Name $resourceGroupName -Location $location
} else {
    Write-Host "Resource group already exists" -ForegroundColor Green
}

# ============================================================
# 4. Test Arcon API Connection
# ============================================================
Write-Host "Testing Arcon API connection..." -ForegroundColor Cyan
try {
    $headers = @{
        "X-API-Key" = $arconApiKey
        "Content-Type" = "application/json"
    }
    
    $response = Invoke-WebRequest -Uri "$arconApiUrl/logs?limit=1" `
        -Headers $headers -Method GET -TimeoutSec 10
    
    Write-Host "✅ Arcon API connection successful" -ForegroundColor Green
} catch {
    Write-Host "❌ Failed to connect to Arcon API" -ForegroundColor Red
    Write-Host "Error: $_" -ForegroundColor Red
    exit 1
}

# ============================================================
# 5. Deploy ARM Template
# ============================================================
Write-Host "Deploying ARM template..." -ForegroundColor Cyan

$deploymentName = "ArconSentinelDeploy-$(Get-Date -Format 'yyyyMMdd-HHmmss')"

$deploymentParams = @{
    ResourceGroupName = $resourceGroupName
    TemplateFile = $templatePath
    TemplateParameterObject = @{
        location = $location
        logicAppName = $logicAppName
        arconApiUrl = $arconApiUrl
        arconApiKey = $arconApiKey
        workspaceId = $workspaceId
        workspaceKey = $workspaceKey
        logTableName = $logTableName
        pullIntervalMinutes = $pullIntervalMinutes
    }
    Name = $deploymentName
}

try {
    $deployment = New-AzResourceGroupDeployment @deploymentParams
    
    if ($deployment.ProvisioningState -eq "Succeeded") {
        Write-Host "✅ Deployment successful!" -ForegroundColor Green
        Write-Host "Deployment ID: $($deployment.DeploymentId)" -ForegroundColor Green
    } else {
        Write-Host "❌ Deployment failed with state: $($deployment.ProvisioningState)" -ForegroundColor Red
        exit 1
    }
} catch {
    Write-Host "❌ Error during deployment: $_" -ForegroundColor Red
    exit 1
}

# ============================================================
# 6. Get Logic App Details
# ============================================================
Write-Host "Retrieving Logic App details..." -ForegroundColor Cyan

$logicApp = Get-AzLogicApp -ResourceGroupName $resourceGroupName -Name $logicAppName

Write-Host "`n=== Deployment Summary ===" -ForegroundColor Green
Write-Host "Logic App Name: $($logicApp.Name)"
Write-Host "Resource Group: $($logicApp.ResourceGroupName)"
Write-Host "Location: $($logicApp.Location)"
Write-Host "State: $($logicApp.State)"
Write-Host "Resource ID: $($logicApp.ResourceId)"

Write-Host "`n=== Next Steps ===" -ForegroundColor Cyan
Write-Host "1. Go to the Logic App in Azure Portal"
Write-Host "2. Monitor the run history to verify logs are being collected"
Write-Host "3. Query Log Analytics to verify logs are arriving"
Write-Host "`nQuery: $logTableName | limit 10"

Write-Host "`n✅ Deployment complete!" -ForegroundColor Green
```

### Run PowerShell Script

```powershell
# Save script as deploy.ps1
# Run the script
.\deploy.ps1

# If you get execution policy error:
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
.\deploy.ps1
```

---

## Option 3: Deploy via Azure CLI

```bash
#!/bin/bash

# ============================================================
# Arcon to Sentinel Connector - Azure CLI Deployment Script
# ============================================================

# Set variables
SUBSCRIPTION_ID="your-subscription-id"
RESOURCE_GROUP="your-resource-group"
LOCATION="eastus"
LOGIC_APP_NAME="ArconToSentinel-Connector"
TEMPLATE_FILE="azuredeploy.json"

# Arcon Configuration
ARCON_API_URL="https://api.arcon.example.com/v1"
ARCON_API_KEY="your-arcon-api-key"

# Log Analytics Configuration
WORKSPACE_ID="your-workspace-id"
WORKSPACE_KEY="your-workspace-key"

# Additional Settings
LOG_TABLE_NAME="ArconLogs_CL"
PULL_INTERVAL_MINUTES=5

# ============================================================
# Connect to Azure
# ============================================================
echo "Connecting to Azure..."
az login
az account set --subscription "$SUBSCRIPTION_ID"

# ============================================================
# Create Resource Group
# ============================================================
echo "Creating resource group..."
az group create \
    --name "$RESOURCE_GROUP" \
    --location "$LOCATION"

# ============================================================
# Deploy Template
# ============================================================
echo "Deploying Logic App..."
az deployment group create \
    --resource-group "$RESOURCE_GROUP" \
    --template-file "$TEMPLATE_FILE" \
    --parameters \
        location="$LOCATION" \
        logicAppName="$LOGIC_APP_NAME" \
        arconApiUrl="$ARCON_API_URL" \
        arconApiKey="$ARCON_API_KEY" \
        workspaceId="$WORKSPACE_ID" \
        workspaceKey="$WORKSPACE_KEY" \
        logTableName="$LOG_TABLE_NAME" \
        pullIntervalMinutes="$PULL_INTERVAL_MINUTES"

# ============================================================
# Get Logic App Details
# ============================================================
echo ""
echo "=== Deployment Complete ==="
az logicapp show \
    --resource-group "$RESOURCE_GROUP" \
    --name "$LOGIC_APP_NAME"

echo ""
echo "✅ Deployment successful!"
```

### Run Azure CLI Script

```bash
# Save script as deploy.sh
# Make it executable
chmod +x deploy.sh

# Run the script
./deploy.sh
```

---

## Post-Deployment Verification

### Step 1: Verify Logic App Deployment

```powershell
# Check Logic App status
Get-AzLogicApp -ResourceGroupName "your-resource-group" `
    -Name "ArconToSentinel-Connector" | Select-Object Name, State, Location
```

### Step 2: Monitor First Run

```powershell
# Get Logic App run history
$runs = Get-AzLogicAppRunHistory -ResourceGroupName "your-resource-group" `
    -Name "ArconToSentinel-Connector" -MaximumItemCount 5

$runs | Select-Object Name, Status, StartTime, EndTime | Format-Table
```

### Step 3: Check Logs in Log Analytics

Run this query in **Log Analytics Workspace**:

```kusto
ArconLogs_CL
| where TimeGenerated > ago(1h)
| project TimeGenerated, data_d
| order by TimeGenerated desc
| limit 10
```

### Step 4: Verify Data in Sentinel

1. Go to **Microsoft Sentinel** → **Logs**
2. Run the query above
3. Verify logs are appearing

---

## Monitoring Setup

### Create Action Group for Alerts

```powershell
# Create action group
$actionGroup = New-AzActionGroup -ResourceGroupName "your-resource-group" `
    -Name "ArconSentinelAlerts" `
    -ShortName "ArconSent" `
    -EmailReceiver -Name "AdminEmail" `
    -EmailAddress "admin@example.com"
```

### Monitor Logic App Health

```kusto
// Alert if Logic App hasn't run in 30 minutes
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.LOGIC"
| where ResourceId contains "ArconToSentinel-Connector"
| where operationName == "Microsoft.Logic/workflows/workflowRunCompleted"
| where TimeGenerated > ago(30m)
| summarize LastRun = max(TimeGenerated)
| where LastRun < ago(30m)
```

---

## Rollback Procedure

If you need to remove the deployment:

### PowerShell Rollback

```powershell
# Remove Logic App
Remove-AzLogicApp -ResourceGroupName "your-resource-group" `
    -Name "ArconToSentinel-Connector" -Force

# Remove resource group (if needed)
Remove-AzResourceGroup -Name "your-resource-group" -Force
```

### Azure CLI Rollback

```bash
# Remove resource group
az group delete --name "your-resource-group" --yes

# Or delete just the Logic App
az logicapp delete --resource-group "your-resource-group" \
    --name "ArconToSentinel-Connector"
```

---

## Troubleshooting Deployment

### Issue: Authentication Failed

**Cause:** Incorrect Azure credentials  
**Solution:**
```powershell
# Clear cached credentials
Remove-AzAccount -ErrorAction SilentlyContinue

# Reconnect
Connect-AzAccount
```

### Issue: Template Validation Error

**Cause:** Invalid parameter values  
**Solution:**
1. Verify all parameters in the template
2. Check that strings are quoted correctly
3. Validate JSON syntax

### Issue: Insufficient Permissions

**Cause:** User doesn't have Contributor role  
**Solution:**
1. Ask subscription owner to grant Contributor role
2. Use Azure Portal with proper account

### Issue: Resource Group Not Found

**Cause:** Resource group doesn't exist  
**Solution:**
```powershell
# Create it first
New-AzResourceGroup -Name "your-resource-group" -Location "eastus"
```

---

## Success Indicators

✅ Logic App created and in "Enabled" state  
✅ First run completed successfully  
✅ Logs appearing in Log Analytics workspace  
✅ Custom table `ArconLogs_CL` exists  
✅ Data visible in Microsoft Sentinel  
✅ No failed runs in run history  

---

## Next Steps

1. Review [README.md](README.md) for overview
2. Check [CONFIG_TEMPLATE.md](CONFIG_TEMPLATE.md) for configuration options
3. Refer to [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common issues
4. Set up monitoring and alerts
5. Configure Sentinel analytics rules

---

**For additional help, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md)**
