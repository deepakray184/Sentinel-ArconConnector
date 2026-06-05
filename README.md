# Arcon to Microsoft Sentinel Custom Connector

A complete Custom Connector Framework (CCF) solution for integrating Arcon logs into Microsoft Sentinel using Azure Logic Apps and REST API.

## 📋 Overview

This project provides a production-ready connector that:
- ✅ Retrieves logs from Arcon API in JSON format
- ✅ Authenticates using API Key
- ✅ Sends data to Microsoft Sentinel via Log Analytics Data Collector API
- ✅ Handles pagination and batch processing
- ✅ Includes automatic retry and error handling
- ✅ Deployable via ARM template or PowerShell

## 📁 Project Structure

```
├── azuredeploy.json                    # Azure ARM template for Logic App
├── README.md                           # This file
├── DEPLOYMENT_GUIDE.md                 # Step-by-step deployment guide
├── CONFIG_TEMPLATE.md                  # Configuration checklist
└── TROUBLESHOOTING.md                  # Common issues and solutions
```

## 🚀 Quick Start

### Prerequisites
- Azure subscription with Contributor role
- Log Analytics Workspace
- Arcon API access with valid API key
- Azure CLI or PowerShell installed

### 1. Deploy via Azure Portal (Quickest)

1. Go to your resource group
2. Click "Create" → "Template Deployment"
3. Select "Build your own template"
4. Copy contents of `azuredeploy.json`
5. Fill in parameters:
   - **logicAppName**: Name for your Logic App (e.g., ArconToSentinel-Connector)
   - **arconApiUrl**: Your Arcon API base URL (e.g., https://api.arcon.example.com/v1)
   - **arconApiKey**: Your Arcon API Key
   - **workspaceId**: Your Log Analytics Workspace ID
   - **workspaceKey**: Your Log Analytics Workspace Primary Key
   - **logTableName**: Custom log table name (default: ArconLogs_CL)
   - **pullIntervalMinutes**: How often to pull logs (default: 5)
6. Click "Review + Create" → "Create"

### 2. Deploy with PowerShell

```powershell
# Connect to Azure
Connect-AzAccount

# Set variables
$resourceGroup = "your-resource-group"
$templateFile = "azuredeploy.json"
$location = "eastus"

# Deploy template
New-AzResourceGroupDeployment `
    -ResourceGroupName $resourceGroup `
    -TemplateFile $templateFile `
    -location $location `
    -logicAppName "ArconToSentinel-Connector" `
    -arconApiUrl "https://api.arcon.example.com/v1" `
    -arconApiKey (ConvertTo-SecureString "your-api-key" -AsPlainText -Force) `
    -workspaceId "your-workspace-id" `
    -workspaceKey (ConvertTo-SecureString "your-workspace-key" -AsPlainText -Force)
```

### 3. Verify Deployment

```kusto
// Query in Log Analytics to verify logs are arriving
ArconLogs_CL
| where TimeGenerated > ago(1h)
| project TimeGenerated, data_d
| order by TimeGenerated desc
| limit 10
```

## 🔧 Configuration

### Environment Variables Required

| Variable | Description | Example |
|----------|-------------|---------|
| `arconApiUrl` | Base URL of Arcon API | `https://api.arcon.example.com/v1` |
| `arconApiKey` | API key for Arcon authentication | `sk-xxxxxxxxxxxxx` |
| `workspaceId` | Log Analytics Workspace ID | `12345678-1234-1234-1234-123456789012` |
| `workspaceKey` | Log Analytics Workspace Primary Key | `xxxxxxxxxxxxxxxx==` |

### Logic App Settings

| Setting | Default | Description |
|---------|---------|-------------|
| **Pull Interval** | 5 minutes | Adjust based on log volume |
| **Batch Size** | 100 logs | Logs per API request |
| **Retry Policy** | 3 attempts | Exponential backoff |
| **Timeout** | 5 minutes | Per API call |

## 📊 Data Flow

```
┌─────────────┐
│  Arcon API  │
│  (JSON)     │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│  Logic App Trigger  │ (Every 5 minutes)
│  (Recurrence)       │
└──────┬──────────────┘
       │
       ▼
┌────────────────────────────────┐
│  Call Arcon API                │
│  (GET /logs with pagination)   │
└──────┬─────────────────────────┘
       │
       ▼
┌────────────────────────────��───┐
│  Batch Processing              │
│  (100 logs per batch)          │
└──────┬─────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│  Send to Log Analytics                      │
│  (Data Collector API)                       │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────┐
│  ArconLogs_CL Custom Table       │
│  (Log Analytics Workspace)       │
└──────┬───────────────────────────┘
       │
       ▼
┌──────────────────────────────────┐
│  Microsoft Sentinel              │
│  (Analytics Rules, Dashboards)   │
└──────────────────────────────────┘
```

## 🔐 Security Best Practices

### API Key Management
- ✅ Store API keys in **Azure Key Vault** (not in code)
- ✅ Never commit keys to source control
- ✅ Rotate keys regularly (every 90 days recommended)
- ✅ Use different keys for dev/test/prod environments

### Log Analytics Access
- ✅ Use **Managed Identity** when possible
- ✅ Limit permissions to Log Analytics workspace only
- ✅ Enable audit logging in Log Analytics
- ✅ Implement least-privilege access

### Data Protection
- ✅ All data transmitted over HTTPS
- ✅ Data encrypted at rest (Log Analytics default)
- ✅ Implement data retention policies
- ✅ Archive old logs to Azure Storage if needed

## 📈 Performance & Optimization

### Adjust Pull Interval Based on Log Volume

| Log Volume | Recommended Interval | Notes |
|-----------|----------------------|-------|
| 0-1,000/hour | 5 minutes | Default setting |
| 1,000-10,000/hour | 10 minutes | Moderate volume |
| 10,000+/hour | 15+ minutes | High volume, consider parallel processing |

### Optimization Tips

1. **Filter at source** (if Arcon API supports it):
   ```
   /logs?filter=severity:ERROR,CRITICAL
   ```

2. **Implement parallel processing**:
   - Enable concurrency in For_Each loop settings

3. **Monitor Logic App costs**:
   - Check runs history in Azure Portal
   - Monitor throttling and failures
   - Adjust batch size if needed

4. **Archive old logs**:
   - Use Log Analytics data export rules
   - Store in Azure Storage for compliance

## 🐛 Troubleshooting

### Common Issues & Solutions

#### Issue: Logic App runs fail immediately
**Solution**: 
1. Check API key and URL in Logic App parameters
2. Verify Arcon API is accessible from Azure
3. Check Logic App run history for detailed error messages

#### Issue: No logs appearing in Sentinel
**Solution**:
1. Verify Log Analytics Workspace ID and Key are correct
2. Check custom log table exists (ArconLogs_CL by default)
3. Review Logic App run history for Send_to_Log_Analytics action
4. Query Log Analytics: `ArconLogs_CL | limit 1`

#### Issue: Logic App timeout errors
**Solution**:
1. Increase timeout in HTTP actions (currently 5 minutes)
2. Reduce batch size from 100 to 50
3. Increase pull interval to reduce load
4. Check Arcon API performance

#### Issue: Authentication failures
**Solution**:
1. Verify arconApiKey hasn't expired
2. Confirm API key format is correct
3. Test API key directly with Arcon API
4. Check API key permissions for /logs endpoint

### Quick Diagnostics

```powershell
# Check Logic App deployment
Get-AzLogicApp -ResourceGroupName "your-rg" -Name "ArconToSentinel-Connector"

# View run history
Get-AzLogicAppRunHistory -ResourceGroupName "your-rg" -Name "ArconToSentinel-Connector" | 
    Select-Object Name, Status, StartTime, EndTime | Format-Table

# Check failed runs
Get-AzLogicAppRunHistory -ResourceGroupName "your-rg" -Name "ArconToSentinel-Connector" | 
    Where-Object {$_.Status -eq "Failed"} | 
    Select-Object -First 5 | 
    ForEach-Object { Get-AzLogicAppRun -ResourceGroupName "your-rg" -Name "ArconToSentinel-Connector" -RunName $_.Name }
```

## 📚 Documentation

- **[DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)** - Detailed step-by-step setup
- **[CONFIG_TEMPLATE.md](CONFIG_TEMPLATE.md)** - Pre-deployment configuration checklist
- **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** - Comprehensive troubleshooting guide

## 🔄 Integration Examples

### Sentinel Alert Rule (KQL)
```kusto
// Alert on high severity events
ArconLogs_CL
| where TimeGenerated > ago(5m)
| project severity_s, event_type_s, message_s, TimeGenerated
```

### Dashboard Query
```kusto
// Logs by severity over time
ArconLogs_CL
| summarize count() by bin(TimeGenerated, 1h), severity_s
| render timechart
```

### Automation Runbook Integration
- Trigger on critical events
- Auto-remediate issues
- Create incidents in ITSM

## 🔌 API Compatibility

| Requirement | Details |
|------------|---------|
| **Arcon API Version** | v1+ |
| **Protocol** | HTTPS/REST |
| **Authentication** | API Key (X-API-Key header) |
| **Data Format** | JSON |
| **Pagination** | Offset-based with `has_next` flag |
| **Response Schema** | `{success: bool, data: array, pagination: {has_next: bool}}` |

## 🆘 Support & Help

### Before Opening an Issue
1. Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
2. Review [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)
3. Verify Arcon API documentation
4. Check Azure Logic Apps documentation

### Information to Include When Reporting Issues
- Logic App run ID (from run history)
- Full error message and stack trace
- Relevant Log Analytics queries
- Arcon API version and endpoint used
- Azure resource group and Logic App name

### Resources
- [Azure Logic Apps Documentation](https://docs.microsoft.com/en-us/azure/logic-apps/)
- [Log Analytics Data Collector API](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/data-collector-api)
- [Microsoft Sentinel Documentation](https://docs.microsoft.com/en-us/azure/sentinel/)
- [Arcon API Documentation](https://api.arcon.example.com/docs)

## ✅ Production Deployment Checklist

- [ ] Arcon API credentials obtained and tested
- [ ] Log Analytics Workspace ID and key obtained
- [ ] API key stored in Azure Key Vault
- [ ] Resource group and location selected
- [ ] azuredeploy.json template reviewed
- [ ] Logic App deployed successfully
- [ ] Test run executed and verified
- [ ] Logs appearing in Sentinel
- [ ] Custom alerts configured
- [ ] Monitoring dashboards created
- [ ] Team trained on connector
- [ ] Backup and disaster recovery plan documented
- [ ] Cost monitoring enabled
- [ ] Data retention policies set

## 📊 Monitoring & Alerts

### Key Metrics to Monitor
- Logic App run success rate
- Average run duration
- Logs ingested per hour
- API response times
- Failed authentication attempts

### Set Up Alert Rule
```kusto
// Alert if ingestion fails for 30 minutes
ArconLogs_CL
| where TimeGenerated > ago(30m)
| summarize LastLog = max(TimeGenerated)
| where LastLog < ago(30m)
```

## 📝 Changelog

### Version 1.0.0 (June 2026)
- ✅ Initial release
- ✅ Support for Arcon API v1
- ✅ Logic App deployment via ARM template
- ✅ Pagination and batch processing
- ✅ Error handling and retry logic
- ✅ Comprehensive documentation
- ✅ Troubleshooting guide
- ✅ Security best practices
- ✅ Performance optimization tips

## 📄 License

This project is provided as-is for use within your organization. Modify and distribute as needed.

## 🎯 Key Features

| Feature | Status | Details |
|---------|--------|---------|
| REST API Integration | ✅ | Full JSON support |
| API Key Authentication | ✅ | Secure X-API-Key header |
| Pagination Support | ✅ | Handles large datasets |
| Batch Processing | ✅ | 100 logs per batch |
| Error Handling | ✅ | Retry logic included |
| ARM Template | ✅ | One-click deployment |
| Custom Table | ✅ | ArconLogs_CL |
| Monitoring | ✅ | Built-in logs |
| Security | ✅ | API key management |
| Documentation | ✅ | Complete guides |

## 🏗️ Architecture Notes

### Why Logic Apps?
- ✅ Native Azure service (no infrastructure)
- ✅ Pay-per-use pricing
- ✅ Built-in error handling and retries
- ✅ Easy to monitor and debug
- ✅ Supports complex workflows

### Why Batch Processing?
- ✅ Better performance (fewer API calls)
- ✅ Lower costs (reduced executions)
- ✅ Natural handling of rate limiting
- ✅ More efficient data transfer

### Why Custom REST Connector?
- ✅ Direct control over API calls
- ✅ Flexible authentication
- ✅ Easy to maintain and update
- ✅ Works with any REST API

## 🔮 Future Enhancements

- [ ] Terraform deployment templates
- [ ] Bicep templates for Infrastructure as Code
- [ ] Azure Functions alternative
- [ ] Event Grid real-time ingestion
- [ ] Advanced data transformation
- [ ] Multi-workspace support
- [ ] Self-updating connector
- [ ] Custom schema mapping

## 📞 Support & Contribution

### Getting Help
1. Review this README
2. Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
3. Contact your Arcon support team
4. Refer to Azure documentation

### Contributing Improvements
Contributions welcome! Consider:
- Enhanced error handling
- Performance optimizations
- Additional documentation
- Extended features
- Alternative deployment methods

---

**Version**: 1.0.0  
**Last Updated**: June 2026  
**Author**: Deepak Kumar Ray  
**Repository**: [deepakray184/Sentinel-ArconConnector](https://github.com/deepakray184/Sentinel-ArconConnector)  

**Compatibility**:
- Azure CLI 2.0+
- PowerShell 7+
- ARM API 2019-04-01+
- Arcon API v1+
- Microsoft Sentinel Standard or higher

---

**🚀 Ready to get started? Follow the [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) for step-by-step instructions!**
